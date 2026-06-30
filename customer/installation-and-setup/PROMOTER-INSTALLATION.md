# CMP Environment Promoter — Installation & Setup Guide

This guide walks you through installing and configuring the CMP Environment Promoter, a standalone tool for promoting configuration artifacts between CMP environments (e.g., dev → test → prod).

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Quick Start with Docker Compose](#quick-start-with-docker-compose)
3. [Environment Variables](#environment-variables)
4. [DynamoDB Table Setup](#dynamodb-table-setup)
5. [Initial Configuration](#initial-configuration)
6. [Troubleshooting](#troubleshooting)
7. [Upgrading](#upgrading)
8. [Production Deployment Considerations](#production-deployment-considerations)

---

## Prerequisites

### Software Requirements

| Software | Version | Notes |
|----------|---------|-------|
| Docker Engine | 24.0+ | Required for containerized deployment |
| Docker Compose | v2.20+ | Plugin or standalone |
| Python | 3.12+ | Required only for non-Docker development |
| Node.js | 18+ | Required only for non-Docker frontend development |
| AWS Account | — | For DynamoDB (or use DynamoDB Local for development) |

### Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 2 cores | 4+ cores |
| RAM | 4 GB | 8+ GB |
| Disk | 20 GB SSD | 50+ GB SSD |
| Network | 100 Mbps | 1 Gbps |

### Network Requirements

| Port | Service | Direction | Notes |
|------|---------|-----------|-------|
| 8002 | Backend API | Inbound | Promoter backend (FastAPI) |
| 3001 | Frontend | Inbound | Promoter web interface |
| 8100 | DynamoDB Local | Internal | Local development only |

The Promoter must have outbound HTTPS access to your CMP environment(s) for artifact retrieval and promotion operations.

### CMP Platform Compatibility

The Promoter communicates with CMP exclusively through its public REST API. No direct database access or shared code is required.

| Promoter Version | Compatible CMP Versions | Notes |
|------------------|------------------------|-------|
| 0.1.x | 2.0.0+ | Initial release |

---

## Quick Start with Docker Compose

The fastest way to get the Promoter running locally.

### Step 1: Clone or Extract the Promoter Package

```bash
cd /path/to/cmp_promoter/backend
```

### Step 2: Generate Security Keys

```bash
# Generate ENCRYPTION_KEY (Fernet-compatible base64 key)
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

# Generate JWT_SECRET_KEY (random 64-character string)
openssl rand -hex 32
```

### Step 3: Create a `.env` File (Optional)

You can set environment variables directly in `docker-compose.yml` or create a `.env` file in the same directory:

```bash
ENCRYPTION_KEY=your-generated-fernet-key
JWT_SECRET_KEY=your-generated-jwt-secret
DYNAMODB_TABLE=cmp_promoter
AWS_REGION=us-east-1
CORS_ORIGINS=http://localhost:3001,http://127.0.0.1:3001
```

### Step 4: Start All Services

```bash
docker compose up -d
```

This starts three services:
- **dynamodb-local** — Local DynamoDB instance on port 8100
- **backend** — FastAPI backend on port 8002
- **frontend** — React frontend on port 3001

### Step 5: Verify Installation

```bash
# Check all containers are running
docker compose ps

# Check backend health
curl -s http://localhost:8002/health | jq .
```

Expected response:
```json
{
  "status": "healthy",
  "version": "0.1.0"
}
```

### Step 6: Access the Web Interface

Open your browser and navigate to `http://localhost:3001`. You should see the Promoter login page.

---

## Environment Variables

All configuration is managed through environment variables. These can be set in a `.env` file or directly in your `docker-compose.yml`.

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `ENCRYPTION_KEY` | Fernet encryption key for storing API tokens securely. **Must be a valid Fernet key.** | `gAAAAABk...` (base64) |
| `DYNAMODB_TABLE` | Name of the DynamoDB table used by the Promoter | `cmp_promoter` |
| `JWT_SECRET_KEY` | Secret key for signing JWT tokens (mapped to `SECRET_KEY` internally) | `a1b2c3d4...` (hex string) |

### AWS / DynamoDB Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DYNAMODB_ENDPOINT` | DynamoDB endpoint URL | `http://dynamodb-local:8000` |
| `AWS_REGION` | AWS region for DynamoDB | `us-east-1` |
| `AWS_ACCESS_KEY_ID` | AWS access key (use `local` for DynamoDB Local) | — |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key (use `local` for DynamoDB Local) | — |

### Application Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `CORS_ORIGINS` | Comma-separated list of allowed CORS origins | `http://localhost:3000,http://127.0.0.1:3000` |
| `APP_DOMAIN` | Application domain (automatically added to CORS origins as HTTPS) | — |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | JWT token expiry in minutes | `30` |

### Frontend Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `VITE_API_URL` | Backend API URL for the frontend to connect to | `http://localhost:8002` |

### Generating the Encryption Key

The `ENCRYPTION_KEY` must be a valid Fernet key. Generate one with:

```bash
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

> **Warning:** If you lose or change the encryption key, all stored environment connection credentials become unreadable. Back up this key securely.

---

## DynamoDB Table Setup

The Promoter uses a single DynamoDB table with a composite primary key (partition key + sort key).

### Local Development (Automatic)

When using Docker Compose with DynamoDB Local, the table is created automatically at application startup. No manual setup is required.

### AWS DynamoDB (Production)

For production deployments using AWS DynamoDB, create the table manually or via infrastructure-as-code:

#### AWS CLI

```bash
aws dynamodb create-table \
  --table-name cmp_promoter \
  --attribute-definitions \
    AttributeName=PK,AttributeType=S \
    AttributeName=SK,AttributeType=S \
  --key-schema \
    AttributeName=PK,KeyType=HASH \
    AttributeName=SK,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

#### Terraform

```hcl
resource "aws_dynamodb_table" "cmp_promoter" {
  name         = "cmp_promoter"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "PK"
  range_key    = "SK"

  attribute {
    name = "PK"
    type = "S"
  }

  attribute {
    name = "SK"
    type = "S"
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Application = "cmp-promoter"
    Environment = "production"
  }
}
```

### Table Schema

| Key | Type | Pattern | Example |
|-----|------|---------|---------|
| PK (Partition Key) | String | `TENANT#{tenant_id}` | `TENANT#default` |
| SK (Sort Key) | String | `{ENTITY}#{id}` | `CONN#abc123`, `SESSION#xyz789` |

The Promoter stores the following entity types in this single table:

| Entity | SK Pattern | Description |
|--------|-----------|-------------|
| Connection | `CONN#{connection_id}` | Environment connections |
| Session | `SESSION#{session_id}` | Promotion session records |
| Session Artifact | `SESSION#{session_id}#ART#{artifact_id}` | Per-artifact outcomes |
| ID Mapping | `IDMAP#{session_id}#SRC#{source_id}` | Source → target ID mappings |
| Snapshot | `SNAP#{session_id}#ART#{artifact_id}` | Pre-promotion artifact state (for rollback) |

---

## Initial Configuration

After installation, configure the Promoter to connect to your CMP environments.

### Creating Your First Environment Connection

1. Open the Promoter web interface at `http://localhost:3001`
2. Navigate to **Connections** in the sidebar
3. Click **Add Connection**
4. Fill in the connection details:

| Field | Description | Example |
|-------|-------------|---------|
| Name | A friendly name for this environment (unique, 1–100 chars) | `Production CMP` |
| Base URL | The CMP instance URL (HTTP or HTTPS) | `https://cmp.yourcompany.com` |
| API Token | A valid API token from the target CMP environment (1–500 chars) | `eyJhbGciOi...` |
| Tenant ID | The tenant identifier in the target CMP environment (1–100 chars) | `default` |

5. Click **Save** — the Promoter will validate connectivity by calling the CMP health endpoint with a 10-second timeout

### Validating Connectivity

After creating a connection, you can test it at any time:

1. Go to **Connections**
2. Click the **Test** button next to the connection
3. The Promoter calls the CMP health endpoint and reports the result

A successful test confirms:
- The CMP instance is reachable from the Promoter
- The API token is valid
- The tenant ID is accessible

### Setting Up Multiple Environments

For a typical dev → test → prod promotion workflow, create one connection per environment:

| Connection Name | Base URL | Purpose |
|----------------|----------|---------|
| Development | `https://cmp-dev.yourcompany.com` | Source for promotions |
| Testing | `https://cmp-test.yourcompany.com` | Intermediate target |
| Production | `https://cmp.yourcompany.com` | Final target |

### Authentication Requirements

The Promoter requires **admin** role access in both source and target CMP environments to perform promotions. Ensure the API token belongs to a user with admin privileges and access to the specified tenant.

---

## Troubleshooting

### Connection Timeout Errors

**Symptom:** "Connection validation failed: timeout" when creating or testing a connection.

**Causes and Solutions:**

| Cause | Solution |
|-------|----------|
| CMP instance is down | Verify the CMP instance is running: `curl -s https://your-cmp-url/health` |
| Network firewall blocking | Ensure the Promoter host can reach the CMP URL on port 443 (HTTPS) or 8001 (HTTP) |
| DNS resolution failure | Verify DNS resolves: `nslookup your-cmp-url` from the Promoter host |
| Slow network | The Promoter uses a 10-second timeout — check network latency between hosts |

**Diagnostic steps:**

```bash
# From the Promoter container
docker compose exec backend curl -v --max-time 10 https://your-cmp-url/health

# Check DNS resolution
docker compose exec backend nslookup your-cmp-url
```

### Authentication Failures

**Symptom:** "Connection validation failed: authentication error" or 401/403 responses.

**Causes and Solutions:**

| Cause | Solution |
|-------|----------|
| Expired API token | Generate a new token from the CMP instance and update the connection |
| Insufficient permissions | Ensure the token belongs to a user with **admin** role |
| Wrong tenant ID | Verify the tenant ID matches one in the user's authorized tenant list |
| Token format invalid | Ensure the full JWT token is pasted without truncation |

**Diagnostic steps:**

```bash
# Test the token directly against the CMP API
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  https://your-cmp-url/api/v1/auth/me | jq .
```

### Unreachable Host Errors

**Symptom:** "Connection validation failed: host unreachable" or connection refused.

**Causes and Solutions:**

| Cause | Solution |
|-------|----------|
| Wrong URL | Double-check the base URL (include `https://` or `http://`) |
| CMP not listening on expected port | Verify the CMP backend port (typically 443 for HTTPS or 8001 for HTTP) |
| Docker network isolation | If both run in Docker, ensure they share a network or use host networking |
| VPN required | Connect to the required VPN before testing |

### Encryption Key Issues

**Symptom:** "Invalid encryption key" or garbled credential data.

**Causes and Solutions:**

| Cause | Solution |
|-------|----------|
| Missing `ENCRYPTION_KEY` | Set the variable in `.env` or `docker-compose.yml` |
| Invalid Fernet key format | Regenerate: `python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"` |
| Key changed after connections were created | Restore the original key — connections encrypted with the old key cannot be decrypted with a new one |

### DynamoDB Connection Issues

**Symptom:** Backend fails to start or returns 500 errors.

**Causes and Solutions:**

```bash
# Check if DynamoDB Local is running
docker compose ps dynamodb-local

# Check backend logs for DynamoDB errors
docker compose logs backend --tail 50

# Verify DynamoDB is accessible
curl -s http://localhost:8100
```

If using AWS DynamoDB, verify:
- `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are set correctly
- The IAM role/user has `dynamodb:*` permissions on the table
- `DYNAMODB_ENDPOINT` is removed or set to the AWS endpoint for your region

---

## Upgrading

### Version Compatibility

The CMP Environment Promoter follows independent semantic versioning. It is not tied to CMP platform release versions.

| Version Change | Impact | Action Required |
|---------------|--------|-----------------|
| Patch (0.1.x → 0.1.y) | Bug fixes only | Safe to upgrade without migration |
| Minor (0.1.x → 0.2.x) | New features, backward-compatible | Review release notes, no data migration |
| Major (0.x → 1.x) | Breaking changes possible | Follow migration guide in release notes |

### Upgrade Procedure

#### Step 1: Back Up Your Data

```bash
# Back up the DynamoDB data directory (local development)
cp -r data/dynamodb data/dynamodb.backup.$(date +%Y%m%d)

# For AWS DynamoDB, ensure point-in-time recovery is enabled
aws dynamodb describe-continuous-backups --table-name cmp_promoter
```

#### Step 2: Pull the New Version

```bash
# Update your docker-compose.yml or image tags
docker compose pull
```

#### Step 3: Stop the Current Services

```bash
docker compose down
```

#### Step 4: Start the New Version

```bash
docker compose up -d
```

#### Step 5: Verify the Upgrade

```bash
# Check the version
curl -s http://localhost:8002/health | jq .

# Verify existing connections still work
# Navigate to Connections page and test each connection
```

### Rolling Back an Upgrade

If the new version has issues:

```bash
# Stop the new version
docker compose down

# Restore the previous docker-compose.yml or image tags
# Restore the data backup if needed
cp -r data/dynamodb.backup.YYYYMMDD data/dynamodb

# Start the previous version
docker compose up -d
```

### Data Retention During Upgrades

- **Promotion sessions** are retained for 90 days from creation
- **Rollback snapshots** are retained for 30 days from the associated promotion
- **Environment connections** are retained indefinitely
- Upgrades do not delete existing data

---

## Production Deployment Considerations

### Use AWS DynamoDB Instead of DynamoDB Local

For production, replace DynamoDB Local with AWS DynamoDB:

1. Remove the `dynamodb-local` service from `docker-compose.yml`
2. Set `DYNAMODB_ENDPOINT` to your AWS region endpoint (or remove it to use the default)
3. Configure proper AWS credentials via IAM roles (preferred) or access keys
4. Enable point-in-time recovery and encryption at rest on the table

### Security Hardening

| Area | Recommendation |
|------|---------------|
| Encryption Key | Store `ENCRYPTION_KEY` in a secrets manager (AWS Secrets Manager, HashiCorp Vault) |
| JWT Secret | Use a strong, unique `JWT_SECRET_KEY` (minimum 32 characters) |
| CORS | Restrict `CORS_ORIGINS` to your exact frontend domain |
| HTTPS | Terminate TLS at a load balancer or reverse proxy in front of the backend |
| Network | Place the backend in a private subnet; expose only through a load balancer |
| IAM | Use least-privilege IAM policies for DynamoDB access |

### Reverse Proxy Configuration (Nginx Example)

```nginx
server {
    listen 443 ssl;
    server_name promoter.yourcompany.com;

    ssl_certificate /etc/ssl/certs/promoter.crt;
    ssl_certificate_key /etc/ssl/private/promoter.key;

    location /api/ {
        proxy_pass http://localhost:8002;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        proxy_pass http://localhost:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Monitoring and Health Checks

- **Health endpoint:** `GET /health` returns `{"status": "healthy", "version": "0.1.0"}`
- **API documentation:** Available at `/docs` (Swagger UI) and `/redoc` (ReDoc)
- Configure your load balancer or monitoring tool to poll `/health` every 30 seconds
- Set up alerts for consecutive health check failures

### Capacity Planning

| Workload | DynamoDB Capacity | Notes |
|----------|-------------------|-------|
| < 10 promotions/day | On-demand (PAY_PER_REQUEST) | Recommended for most deployments |
| 10–100 promotions/day | On-demand or provisioned (5 RCU / 5 WCU) | Monitor and adjust |
| > 100 promotions/day | Provisioned with auto-scaling | Contact support for guidance |

### Backup Strategy

- Enable DynamoDB point-in-time recovery for continuous backups
- Schedule periodic exports to S3 for long-term retention
- Back up the `ENCRYPTION_KEY` securely — without it, stored credentials are unrecoverable

---

## Next Steps

After successful installation:

1. Create connections to your CMP environments (dev, test, prod)
2. Test connectivity to each environment
3. Try your first promotion: select artifacts from dev and promote to test
4. Review the [CMP Environment Promoter Functionality Guide](../functionality/CMP-ENVIRONMENT-PROMOTER.md) for detailed usage instructions

---

## Getting Help

If you encounter issues not covered here:

1. Check the backend logs: `docker compose logs backend --tail 100`
2. Verify your environment variables are set correctly
3. Ensure your CMP environments are accessible from the Promoter host
4. Contact your CMP administrator for API token and access issues
