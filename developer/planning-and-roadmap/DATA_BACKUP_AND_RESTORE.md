# CMP Data Backup & Restore Guide

## Overview

This guide covers how to back up and restore all CMP application data. It applies to any scenario:

- Moving from one cloud to another (Azure → GCP, AWS → Azure, etc.)
- Rebuilding infrastructure within the same cloud
- Disaster recovery
- Spinning up a new environment with existing data
- Creating a staging/test environment from production data

The focus is entirely on **data portability** — independent of where the infrastructure lives.

---

## What Data Exists

| Data Store | Technology | Location on VM | Criticality |
|-----------|-----------|----------------|-------------|
| Application Database | DynamoDB Local (Docker) | `/opt/cmp/data/dynamodb/` | **Critical** — all tenant data, users, resources, workflows, catalogs |
| Cache & Token Blacklist | Redis (Docker) | Docker volume `redis_data` | **Low** — ephemeral; users just re-login if lost |
| Application Secrets | `.env` file | `/opt/cmp/.env` | **Critical** — encryption keys, JWT signing key |
| Encrypted Credentials | Inside DynamoDB | Part of DB backup | **Critical** — cloud provider credentials stored Fernet-encrypted |

### What Makes the Data Portable

CMP uses DynamoDB Local (a self-contained Docker container) rather than any cloud-managed database. This means:
- Data is just files on disk — no proprietary cloud lock-in
- The same backup works regardless of target cloud
- No database migration or schema conversion needed between clouds

---

## Critical Secrets to Preserve

These values must be **identical** on the new environment for existing data to remain functional:

| Secret | Why It Matters | Consequence If Lost |
|--------|---------------|-------------------|
| `SECRET_KEY` | Signs all JWT tokens | All users forced to re-login |
| `ENCRYPTION_KEY` | Fernet key encrypting stored cloud credentials | **All saved cloud credentials become unreadable** — must re-enter every credential |

**Always back up your `.env` file alongside the database.**

---

## Backup Procedures

### Method A: Application-Level JSON Export (Recommended)

Exports all data as portable JSON. Works regardless of DynamoDB Local version, Docker version, or OS. Includes table schemas and GSI definitions for full recreation.

**Step 1: SSH into the running CMP server**

```bash
ssh <admin_username>@<server-ip>
cd /opt/cmp
```

**Step 2: Run the export**

```bash
sudo docker compose exec backend python -c "
import json
import boto3
import base64
from decimal import Decimal
from boto3.dynamodb.types import Binary

class DynamoDBEncoder(json.JSONEncoder):
    def default(self, o):
        if isinstance(o, Decimal):
            return float(o)
        if isinstance(o, Binary):
            return {'__binary__': True, 'b64': base64.b64encode(o.value).decode('utf-8')}
        if isinstance(o, bytes):
            return {'__binary__': True, 'b64': base64.b64encode(o).decode('utf-8')}
        if isinstance(o, set):
            return list(o)
        return super().default(o)

dynamodb = boto3.resource('dynamodb', endpoint_url='http://dynamodb-local:8000',
    aws_access_key_id='local', aws_secret_access_key='local', region_name='us-east-1')

client = boto3.client('dynamodb', endpoint_url='http://dynamodb-local:8000',
    aws_access_key_id='local', aws_secret_access_key='local', region_name='us-east-1')

tables = client.list_tables()['TableNames']
backup = {}

for table_name in tables:
    table = dynamodb.Table(table_name)
    items = []
    response = table.scan()
    items.extend(response['Items'])
    while 'LastEvaluatedKey' in response:
        response = table.scan(ExclusiveStartKey=response['LastEvaluatedKey'])
        items.extend(response['Items'])

    desc = client.describe_table(TableName=table_name)['Table']
    backup[table_name] = {
        'items': items,
        'key_schema': desc['KeySchema'],
        'attribute_definitions': desc['AttributeDefinitions'],
        'global_secondary_indexes': [
            {k: v for k, v in gsi.items() if k in ['IndexName', 'KeySchema', 'Projection']}
            for gsi in desc.get('GlobalSecondaryIndexes', [])
        ]
    }
    print(f'  {table_name}: {len(items)} items')

with open('/tmp/cmp-full-backup.json', 'w') as f:
    json.dump(backup, f, cls=DynamoDBEncoder, indent=2)

print(f'\\nBackup complete: {len(tables)} tables exported')
"
```

**Step 3: Copy backup out of the container**

```bash
sudo docker compose cp backend:/tmp/cmp-full-backup.json /tmp/cmp-full-backup.json
ls -lh /tmp/cmp-full-backup.json
```

**Step 4: Back up secrets**

```bash
cp /opt/cmp/.env /tmp/cmp-env-backup-$(date +%Y%m%d)
```

**Step 5: Transfer to safe storage**

```bash
# Option A: Download to your local machine
scp <admin_username>@<server-ip>:/tmp/cmp-full-backup.json ./
scp <admin_username>@<server-ip>:/tmp/cmp-env-backup-* ./

# Option B: Upload to a cloud storage bucket
# GCS
gsutil cp /tmp/cmp-full-backup.json gs://your-bucket/cmp-backups/
# AWS S3
aws s3 cp /tmp/cmp-full-backup.json s3://your-bucket/cmp-backups/
# Azure Blob
az storage blob upload --file /tmp/cmp-full-backup.json --container cmp-backups --name cmp-full-backup.json
```

---

### Method B: Direct File Copy (Faster for Large Datasets)

Copies the raw DynamoDB Local database files. Faster but requires the target to run the same DynamoDB Local image version.

**Step 1: Stop writes to ensure consistency**

```bash
ssh <admin_username>@<server-ip>
cd /opt/cmp
sudo docker compose stop backend
sudo docker compose stop dynamodb-local
```

**Step 2: Archive the data directory**

```bash
sudo tar -czf /tmp/cmp-dynamodb-backup-$(date +%Y%m%d).tar.gz -C /opt/cmp/data dynamodb
```

**Step 3: Back up secrets**

```bash
cp /opt/cmp/.env /tmp/cmp-env-backup-$(date +%Y%m%d)
```

**Step 4: Restart services**

```bash
sudo docker compose up -d
```

**Step 5: Transfer the archive**

```bash
scp <admin_username>@<server-ip>:/tmp/cmp-dynamodb-backup-*.tar.gz ./
scp <admin_username>@<server-ip>:/tmp/cmp-env-backup-* ./
```

---

### Method C: AWS CLI Per-Table Export

Useful for exporting a single table or for quick inspection.

```bash
# List all tables
aws dynamodb list-tables \
  --endpoint-url http://localhost:8000 \
  --region us-east-1

# Export a specific table
aws dynamodb scan \
  --table-name cmp_table \
  --endpoint-url http://localhost:8000 \
  --region us-east-1 \
  --output json > cmp_table_backup.json
```

---

## Restore Procedures

### Restoring from JSON Export (Method A)

Use this on any fresh CMP deployment — same cloud, different cloud, or local.

**Step 1: Ensure services are running on the target**

```bash
ssh <admin_username>@<new-server-ip>
cd /opt/cmp
sudo docker compose up -d
# Wait for healthy state
sudo docker compose ps
```

**Step 2: Copy backup file to the target VM and into the backend container**

From your local machine, transfer the backup to the target VM first, then into the container:

```bash
# Transfer from local to the target VM
scp -i <ssh-key-path> /path/to/cmp-full-backup.json <admin_username>@<new-server-ip>:/tmp/cmp-full-backup.json

#scp -i /Users/vaibhavsrivastava/Downloads/cmptest.pem ../../../backups/cmp-full-backup.json cmpdev@35.232.129.58:/tmp/cmp-full-backup.json

# SSH into the target VM
ssh -i <ssh-key-path> <admin_username>@<new-server-ip>

# Copy from VM into the backend container
cd /opt/cmp
sudo docker compose cp /tmp/cmp-full-backup.json backend:/tmp/cmp-full-backup.json
```

**Step 3: Run the restore**

```bash
sudo docker compose exec backend python -c "
import json
import boto3
import base64
from decimal import Decimal
from boto3.dynamodb.types import Binary
import time

def convert_values(obj):
    \"\"\"Convert JSON backup values back to DynamoDB-compatible types.\"\"\"
    if isinstance(obj, float):
        return Decimal(str(obj))
    if isinstance(obj, dict):
        if obj.get('__binary__'):
            return Binary(base64.b64decode(obj['b64']))
        return {k: convert_values(v) for k, v in obj.items()}
    if isinstance(obj, list):
        return [convert_values(i) for i in obj]
    return obj

dynamodb = boto3.resource('dynamodb', endpoint_url='http://dynamodb-local:8000',
    aws_access_key_id='local', aws_secret_access_key='local', region_name='us-east-1')

client = boto3.client('dynamodb', endpoint_url='http://dynamodb-local:8000',
    aws_access_key_id='local', aws_secret_access_key='local', region_name='us-east-1')

with open('/tmp/cmp-full-backup.json') as f:
    backup = json.load(f)

existing_tables = client.list_tables()['TableNames']

for table_name, data in backup.items():
    # Create table if it doesn't exist
    if table_name not in existing_tables:
        create_params = {
            'TableName': table_name,
            'KeySchema': data['key_schema'],
            'AttributeDefinitions': data['attribute_definitions'],
            'BillingMode': 'PAY_PER_REQUEST'
        }
        if data.get('global_secondary_indexes'):
            gsis = []
            for gsi in data['global_secondary_indexes']:
                gsi_def = {
                    'IndexName': gsi['IndexName'],
                    'KeySchema': gsi['KeySchema'],
                    'Projection': gsi['Projection']
                }
                gsis.append(gsi_def)
            create_params['GlobalSecondaryIndexes'] = gsis
        try:
            client.create_table(**create_params)
            time.sleep(1)  # Brief pause for table creation
            print(f'  Created table: {table_name}')
        except client.exceptions.ResourceInUseException:
            print(f'  Table already exists: {table_name}')
    else:
        print(f'  Table exists: {table_name}')

    # Write all items
    table = dynamodb.Table(table_name)
    count = 0
    with table.batch_writer() as batch:
        for item in data['items']:
            batch.put_item(Item=convert_values(item))
            count += 1
    print(f'  Restored {table_name}: {count} items')

print('\\nRestore complete!')
"
```

**Step 4: Restore secrets**

Copy the backed-up `.env` to the new server's `/opt/cmp/.env`, ensuring `SECRET_KEY` and `ENCRYPTION_KEY` match the original.

**Step 5: Restart to pick up the restored data**

```bash
sudo docker compose restart backend
```

---

### Restoring from File Copy (Method B)

**Step 1: Stop services on the target**

```bash
cd /opt/cmp
sudo docker compose stop dynamodb-local backend
```

**Step 2: Extract the archive**

```bash
sudo rm -rf /opt/cmp/data/dynamodb/*
sudo tar -xzf cmp-dynamodb-backup-*.tar.gz -C /opt/cmp/data/
sudo chown -R 1000:1000 /opt/cmp/data/dynamodb
```

**Step 3: Start services**

```bash
sudo docker compose up -d
```

---

## Pre-Backup Checklist

- [ ] Note the current DynamoDB Local image version: `docker compose images dynamodb-local`
- [ ] Note the backend image version: `docker compose images backend`
- [ ] Copy `/opt/cmp/.env` (contains `SECRET_KEY`, `ENCRYPTION_KEY`, and other secrets)
- [ ] Verify enough disk space for backup: `df -h /tmp`
- [ ] If using Method B, plan for brief downtime (backend stopped during archive)

## Post-Restore Verification

Run these checks after restoring on the new environment:

| Check | Command / Action | Expected Result |
|-------|-----------------|-----------------|
| Health endpoint | `curl https://<domain>/health` | `200 OK` with version info |
| Admin login | Log in with existing admin credentials | Successful — JWT signs correctly |
| Tenant data | Navigate to dashboard | All tenants, resources, workflows visible |
| Cloud credentials | Go to Settings → Cloud Credentials | Credentials decrypt and display (requires same `ENCRYPTION_KEY`) |
| Catalog items | Browse service catalog | All catalog items present |
| Event history | Check activity/events page | Historical events preserved |

---

## Scenarios

### Same Cloud, New Infrastructure

Example: Rebuilding the Azure VM with fresh Terraform.

1. Back up data (Method A or B) from old VM
2. Terraform apply to create new VM
3. Wait for cloud-init to finish (services running with empty DB)
4. Restore data onto new VM
5. Copy `.env` with original secrets
6. Restart backend — done

### Cross-Cloud Migration

Example: Azure → GCP, or AWS → Azure.

1. Deploy new infrastructure on target cloud (Terraform / manual)
2. Verify CMP stack starts with empty database on target
3. Back up data from source (Method A recommended — format-independent)
4. Transfer backup file to target VM
5. Restore data on target
6. Copy `.env` with original `SECRET_KEY` and `ENCRYPTION_KEY`
7. Restart backend
8. Verify (login, data integrity, credential decryption)
9. Update DNS to point to new server
10. Decommission source infrastructure

### Disaster Recovery

1. Retrieve latest backup from storage (S3, GCS, Azure Blob, or local)
2. Provision new VM (any cloud)
3. Deploy CMP stack
4. Restore from backup
5. Update DNS

### Clone to Staging/Test

1. Back up production data (Method A)
2. Restore onto staging environment
3. **Change `SECRET_KEY`** on staging (invalidates prod tokens — intentional for isolation)
4. Optionally scrub sensitive data from staging DB

---

## Automating Backups

For scheduled backups, add a cron job on the CMP server:

```bash
# /etc/cron.d/cmp-backup
0 2 * * * root /usr/local/bin/cmp-backup.sh >> /var/log/cmp-backup.log 2>&1
```

Example `/usr/local/bin/cmp-backup.sh`:

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/opt/cmp/backups"
RETENTION_DAYS=7
DATE=$(date +%Y%m%d-%H%M%S)

mkdir -p "$BACKUP_DIR"

# JSON export via backend container
docker compose -f /opt/cmp/docker-compose.yml exec -T backend python -c "
import json, boto3, base64
from decimal import Decimal
from boto3.dynamodb.types import Binary

class DynamoDBEncoder(json.JSONEncoder):
    def default(self, o):
        if isinstance(o, Decimal):
            return float(o)
        if isinstance(o, Binary):
            return {'__binary__': True, 'b64': base64.b64encode(o.value).decode('utf-8')}
        if isinstance(o, bytes):
            return {'__binary__': True, 'b64': base64.b64encode(o).decode('utf-8')}
        if isinstance(o, set):
            return list(o)
        return super().default(o)

client = boto3.client('dynamodb', endpoint_url='http://dynamodb-local:8000',
    aws_access_key_id='local', aws_secret_access_key='local', region_name='us-east-1')
dynamodb = boto3.resource('dynamodb', endpoint_url='http://dynamodb-local:8000',
    aws_access_key_id='local', aws_secret_access_key='local', region_name='us-east-1')

tables = client.list_tables()['TableNames']
backup = {}
for t in tables:
    table = dynamodb.Table(t)
    items, resp = [], table.scan()
    items.extend(resp['Items'])
    while 'LastEvaluatedKey' in resp:
        resp = table.scan(ExclusiveStartKey=resp['LastEvaluatedKey'])
        items.extend(resp['Items'])
    desc = client.describe_table(TableName=t)['Table']
    backup[t] = {
        'items': items,
        'key_schema': desc['KeySchema'],
        'attribute_definitions': desc['AttributeDefinitions'],
        'global_secondary_indexes': [
            {k: v for k, v in g.items() if k in ['IndexName','KeySchema','Projection']}
            for g in desc.get('GlobalSecondaryIndexes', [])
        ]
    }
print(json.dumps(backup, cls=DynamoDBEncoder))
" > "$BACKUP_DIR/cmp-backup-$DATE.json"

# Compress
gzip "$BACKUP_DIR/cmp-backup-$DATE.json"

# Upload to cloud storage (uncomment your provider)
# gsutil cp "$BACKUP_DIR/cmp-backup-$DATE.json.gz" gs://your-bucket/cmp-backups/
# aws s3 cp "$BACKUP_DIR/cmp-backup-$DATE.json.gz" s3://your-bucket/cmp-backups/
# az storage blob upload --file "$BACKUP_DIR/cmp-backup-$DATE.json.gz" --container cmp-backups

# Clean old local backups
find "$BACKUP_DIR" -name "cmp-backup-*.json.gz" -mtime +$RETENTION_DAYS -delete

echo "[$DATE] Backup complete: $BACKUP_DIR/cmp-backup-$DATE.json.gz"
```

```bash
sudo chmod +x /usr/local/bin/cmp-backup.sh
```

---

## Summary

| Method | Best For | Downtime | Portability |
|--------|---------|----------|-------------|
| **A: JSON Export** | Cross-cloud moves, version mismatches, long-term archival | None (live export) | Any environment |
| **B: File Copy** | Same DynamoDB Local version, fastest restore, large datasets | ~1–2 min (stop backend) | Same image version required |
| **C: AWS CLI** | Quick single-table inspection or export | None | Standard DynamoDB format |

**Recommendation:** Use Method A for any migration or backup you plan to restore elsewhere. Use Method B for quick same-environment snapshots. Set up the automated cron script for ongoing protection.
