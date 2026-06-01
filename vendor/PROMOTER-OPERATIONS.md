# CMP Environment Promoter — Operations Guide

> Internal operations guide for deploying, monitoring, and maintaining the CMP Environment Promoter.
> Covers deployment procedures, monitoring, DynamoDB capacity planning, data retention, runbooks, CI/CD, and disaster recovery.

---

## Table of Contents

1. [Deployment Procedures](#deployment-procedures)
2. [Monitoring](#monitoring)
3. [DynamoDB Capacity Planning and Cost](#dynamodb-capacity-planning-and-cost)
4. [Data Retention Policies](#data-retention-policies)
5. [Runbook: Common Operational Issues](#runbook-common-operational-issues)
6. [CI/CD Pipeline Configuration](#cicd-pipeline-configuration)
7. [Backup and Disaster Recovery](#backup-and-disaster-recovery)

---

## Deployment Procedures

### Architecture Overview

The CMP Environment Promoter is a standalone application consisting of:

- **Backend**: Python 3.12 / FastAPI / Uvicorn (port 8000 internal)
- **Frontend**: React 18 / TypeScript / Vite (static assets served via Nginx or CDN)
- **Database**: AWS DynamoDB (single table: `cmp_promoter`)

The Promoter communicates with CMP environments exclusively via their public REST APIs.

### Environment Variables

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `SECRET_KEY` | Yes | JWT signing key (HS256) | `openssl rand -hex 32` |
| `ENCRYPTION_KEY` | Yes | Fernet key for credential encryption | `python -c "import secrets; print(secrets.token_urlsafe(32))"` |
| `DYNAMODB_TABLE` | Yes | DynamoDB table name | `cmp_promoter` |
| `AWS_REGION` | Yes | AWS region for DynamoDB | `us-east-1` |
| `AWS_ACCESS_KEY_ID` | Yes* | AWS credentials (not needed with IAM roles) | — |
| `AWS_SECRET_ACCESS_KEY` | Yes* | AWS credentials (not needed with IAM roles) | — |
| `CORS_ORIGINS` | No | Comma-separated allowed origins | `https://promoter.example.com` |
| `APP_DOMAIN` | No | Application domain for CORS | `promoter.example.com` |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | No | JWT token TTL (default: 30) | `30` |

> *When running on AWS ECS with a task role, `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are not required — the SDK uses the task role automatically.

### AWS ECS Deployment

#### Prerequisites

- ECR repository for the Promoter backend image
- DynamoDB table `cmp_promoter` provisioned in the target region
- ECS cluster with Fargate or EC2 capacity
- Application Load Balancer (ALB) with HTTPS listener
- IAM task execution role and task role with DynamoDB access

#### IAM Task Role Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:CreateTable",
        "dynamodb:DescribeTable"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/cmp_promoter"
    }
  ]
}
```

#### ECS Task Definition (Key Fields)

```json
{
  "family": "cmp-promoter-backend",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "promoter-backend",
      "image": "<account>.dkr.ecr.<region>.amazonaws.com/cmp-promoter:latest",
      "portMappings": [{ "containerPort": 8000, "protocol": "tcp" }],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
        "interval": 10,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 15
      },
      "environment": [
        { "name": "DYNAMODB_TABLE", "value": "cmp_promoter" },
        { "name": "AWS_REGION", "value": "us-east-1" },
        { "name": "CORS_ORIGINS", "value": "https://promoter.example.com" }
      ],
      "secrets": [
        { "name": "SECRET_KEY", "valueFrom": "arn:aws:secretsmanager:...:secret:promoter/secret-key" },
        { "name": "ENCRYPTION_KEY", "valueFrom": "arn:aws:secretsmanager:...:secret:promoter/encryption-key" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/cmp-promoter",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "backend"
        }
      }
    }
  ]
}
```

#### Deployment Steps (ECS)

1. Build and push the Docker image:
   ```bash
   docker build -t cmp-promoter:latest ./cmp_promoter/backend
   docker tag cmp-promoter:latest <account>.dkr.ecr.<region>.amazonaws.com/cmp-promoter:latest
   aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com
   docker push <account>.dkr.ecr.<region>.amazonaws.com/cmp-promoter:latest
   ```

2. Register or update the ECS task definition:
   ```bash
   aws ecs register-task-definition --cli-input-json file://task-definition.json
   ```

3. Update the ECS service to use the new task definition:
   ```bash
   aws ecs update-service \
     --cluster cmp-promoter-cluster \
     --service cmp-promoter-service \
     --task-definition cmp-promoter-backend:<revision> \
     --force-new-deployment
   ```

4. Verify deployment health:
   ```bash
   aws ecs describe-services --cluster cmp-promoter-cluster --services cmp-promoter-service \
     --query 'services[0].deployments'
   ```

5. Confirm the health endpoint responds:
   ```bash
   curl https://promoter-api.example.com/health
   # Expected: {"status": "healthy", "version": "0.1.0"}
   ```

### Azure Container Apps Deployment

#### Prerequisites

- Azure Container Registry (ACR) with the Promoter image
- DynamoDB-compatible table (use AWS DynamoDB cross-region or Azure Cosmos DB with DynamoDB API)
- Azure Container Apps environment
- Azure Key Vault for secrets

#### Container App Configuration

```bash
# Create the Container App
az containerapp create \
  --name cmp-promoter-backend \
  --resource-group cmp-promoter-rg \
  --environment cmp-promoter-env \
  --image <acr-name>.azurecr.io/cmp-promoter:latest \
  --target-port 8000 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 5 \
  --cpu 0.5 \
  --memory 1.0Gi \
  --env-vars \
    DYNAMODB_TABLE=cmp_promoter \
    AWS_REGION=us-east-1 \
    CORS_ORIGINS=https://promoter.example.com \
  --secrets \
    secret-key=keyvaultref:<vault-uri>/secrets/promoter-secret-key,identityref:<identity-id> \
    encryption-key=keyvaultref:<vault-uri>/secrets/promoter-encryption-key,identityref:<identity-id> \
  --secret-env-vars \
    SECRET_KEY=secret-key \
    ENCRYPTION_KEY=encryption-key \
    AWS_ACCESS_KEY_ID=keyvaultref:<vault-uri>/secrets/aws-access-key,identityref:<identity-id> \
    AWS_SECRET_ACCESS_KEY=keyvaultref:<vault-uri>/secrets/aws-secret-key,identityref:<identity-id>
```

#### Health Probes

```bash
az containerapp update \
  --name cmp-promoter-backend \
  --resource-group cmp-promoter-rg \
  --set-env-vars ... \
  --probe-path /health \
  --probe-type liveness \
  --probe-interval 10 \
  --probe-timeout 5
```

#### Deployment Steps (Azure Container Apps)

1. Build and push the image to ACR:
   ```bash
   az acr build --registry <acr-name> --image cmp-promoter:latest ./cmp_promoter/backend
   ```

2. Update the Container App revision:
   ```bash
   az containerapp update \
     --name cmp-promoter-backend \
     --resource-group cmp-promoter-rg \
     --image <acr-name>.azurecr.io/cmp-promoter:<new-tag>
   ```

3. Verify the new revision is active:
   ```bash
   az containerapp revision list \
     --name cmp-promoter-backend \
     --resource-group cmp-promoter-rg \
     --query "[?properties.active==\`true\`].name"
   ```

4. Confirm health:
   ```bash
   curl https://cmp-promoter-backend.<env-domain>.azurecontainerapps.io/health
   ```

### Frontend Deployment

The frontend is a static React build. Deploy to any static hosting:

- **AWS**: S3 + CloudFront
- **Azure**: Azure Static Web Apps or Blob Storage + CDN

```bash
cd cmp_promoter/frontend
npm ci
VITE_API_URL=https://promoter-api.example.com npm run build
# Upload dist/ to S3 or Azure Blob Storage
```

---

## Monitoring

### Health Endpoint

The Promoter exposes a health check at `GET /health`:

```json
{"status": "healthy", "version": "0.1.0"}
```

Use this endpoint for:
- Load balancer health checks (interval: 10s, timeout: 5s, unhealthy threshold: 3)
- Uptime monitoring (Pingdom, UptimeRobot, or equivalent)
- Kubernetes/ECS liveness probes

### Key Metrics to Monitor

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| Health endpoint response time | ALB/Ingress | > 2s for 3 consecutive checks |
| HTTP 5xx error rate | ALB access logs | > 1% of requests over 5 minutes |
| DynamoDB consumed RCU/WCU | CloudWatch | > 80% of provisioned capacity |
| DynamoDB throttled requests | CloudWatch | Any throttling events |
| Container CPU utilization | ECS/Container Apps | > 80% sustained for 5 minutes |
| Container memory utilization | ECS/Container Apps | > 85% sustained for 5 minutes |
| Promotion session duration | Application logs | > 60 seconds for single promotion |
| Token refresh failures | Application logs | Any occurrence |

### CloudWatch Configuration (AWS)

#### Log Group Setup

```bash
aws logs create-log-group --log-group-name /ecs/cmp-promoter
aws logs put-retention-policy --log-group-name /ecs/cmp-promoter --retention-in-days 90
```

#### Metric Filters

Create metric filters for key application events:

```bash
# Track promotion failures
aws logs put-metric-filter \
  --log-group-name /ecs/cmp-promoter \
  --filter-name PromotionFailures \
  --filter-pattern '"ERROR" "promotion" "failed"' \
  --metric-transformations metricName=PromotionFailures,metricNamespace=CMPPromoter,metricValue=1
```

```bash
# Track DynamoDB write retries
aws logs put-metric-filter \
  --log-group-name /ecs/cmp-promoter \
  --filter-name DynamoDBRetries \
  --filter-pattern '"WARNING" "DynamoDB" "retry"' \
  --metric-transformations metricName=DynamoDBRetries,metricNamespace=CMPPromoter,metricValue=1

# Track token refresh failures
aws logs put-metric-filter \
  --log-group-name /ecs/cmp-promoter \
  --filter-name TokenRefreshFailures \
  --filter-pattern '"ERROR" "token" "refresh" "failed"' \
  --metric-transformations metricName=TokenRefreshFailures,metricNamespace=CMPPromoter,metricValue=1
```

#### CloudWatch Alarms

```bash
# Alarm: DynamoDB throttling
aws cloudwatch put-metric-alarm \
  --alarm-name cmp-promoter-dynamodb-throttle \
  --metric-name ThrottledRequests \
  --namespace AWS/DynamoDB \
  --dimensions Name=TableName,Value=cmp_promoter \
  --statistic Sum \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:<region>:<account>:cmp-promoter-alerts

# Alarm: High error rate
aws cloudwatch put-metric-alarm \
  --alarm-name cmp-promoter-high-errors \
  --metric-name PromotionFailures \
  --namespace CMPPromoter \
  --statistic Sum \
  --period 300 \
  --threshold 5 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:<region>:<account>:cmp-promoter-alerts
```

### Azure Monitor Configuration

#### Diagnostic Settings

```bash
az monitor diagnostic-settings create \
  --name cmp-promoter-diagnostics \
  --resource /subscriptions/<sub>/resourceGroups/cmp-promoter-rg/providers/Microsoft.App/containerApps/cmp-promoter-backend \
  --logs '[{"category":"ContainerAppConsoleLogs","enabled":true},{"category":"ContainerAppSystemLogs","enabled":true}]' \
  --workspace /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.OperationalInsights/workspaces/<workspace>
```

#### Log Analytics Queries

```kusto
// Promotion failures in the last 24 hours
ContainerAppConsoleLogs_CL
| where ContainerName_s == "cmp-promoter-backend"
| where Log_s contains "ERROR" and Log_s contains "promotion"
| summarize count() by bin(TimeGenerated, 1h)
| order by TimeGenerated desc

// Average response time by endpoint
ContainerAppConsoleLogs_CL
| where ContainerName_s == "cmp-promoter-backend"
| where Log_s contains "completed in"
| parse Log_s with * "completed in " duration:double "ms"
| summarize avg(duration), percentile(duration, 95) by bin(TimeGenerated, 5m)
```

#### Azure Alerts

```bash
az monitor metrics alert create \
  --name cmp-promoter-cpu-alert \
  --resource-group cmp-promoter-rg \
  --scopes /subscriptions/<sub>/resourceGroups/cmp-promoter-rg/providers/Microsoft.App/containerApps/cmp-promoter-backend \
  --condition "avg UsageNanoCores > 400000000" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Insights/actionGroups/cmp-promoter-alerts
```

### Log Analysis Patterns

Key log patterns to search for during troubleshooting:

| Pattern | Meaning |
|---------|---------|
| `DynamoDB connection ready` | Startup successful |
| `Failed to connect to DynamoDB after 5 attempts` | DynamoDB unreachable at startup |
| `ERROR - promotion - failed` | Promotion execution failure |
| `WARNING - DynamoDB - retry` | DynamoDB write required retry |
| `ERROR - token - refresh - failed` | CMP environment auth expired |
| `ERROR - source unreachable` | Source CMP environment down |
| `ERROR - target unreachable` | Target CMP environment down |

---

## DynamoDB Capacity Planning and Cost

### Table Design

The Promoter uses a single DynamoDB table (`cmp_promoter`) with the following access patterns:

| Entity | PK | SK | Access Pattern |
|--------|----|----|----------------|
| Connection | `TENANT#{tenant_id}` | `CONN#{connection_id}` | CRUD per user (low volume) |
| Session | `TENANT#{tenant_id}` | `SESSION#{session_id}` | Write once, read many |
| SessionArtifact | `TENANT#{tenant_id}` | `SESSION#{session_id}#ART#{artifact_id}` | Batch write during promotion |
| IDMapping | `TENANT#{tenant_id}` | `IDMAP#{session_id}#SRC#{source_id}` | Write during promotion, read during rollback |
| Snapshot | `TENANT#{tenant_id}` | `SNAP#{session_id}#ART#{artifact_id}` | Write during promotion, read during rollback, TTL expiry |

### Capacity Estimation

#### Low Usage (1-5 promotions/day, ~50 artifacts each)

| Metric | Estimate |
|--------|----------|
| Write Capacity Units (WCU) | 5 WCU sustained, 25 WCU burst |
| Read Capacity Units (RCU) | 10 RCU sustained, 50 RCU burst |
| Storage | < 1 GB/month |
| Monthly cost (on-demand) | ~$5-15 |

#### Medium Usage (10-20 promotions/day, ~200 artifacts each)

| Metric | Estimate |
|--------|----------|
| Write Capacity Units (WCU) | 25 WCU sustained, 100 WCU burst |
| Read Capacity Units (RCU) | 50 RCU sustained, 200 RCU burst |
| Storage | 1-5 GB/month |
| Monthly cost (on-demand) | ~$25-75 |

#### High Usage (50+ promotions/day, ~500 artifacts each)

| Metric | Estimate |
|--------|----------|
| Write Capacity Units (WCU) | 100 WCU sustained, 500 WCU burst |
| Read Capacity Units (RCU) | 200 RCU sustained, 1000 RCU burst |
| Storage | 5-20 GB/month |
| Monthly cost (on-demand) | ~$100-300 |

### Capacity Mode Recommendations

| Scenario | Recommended Mode | Rationale |
|----------|-----------------|-----------|
| Development/staging | On-demand | Unpredictable, low-volume traffic |
| Production (steady load) | Provisioned with auto-scaling | Cost-effective for predictable patterns |
| Production (bursty) | On-demand | Bulk promotions create large write spikes |

#### Provisioned Mode with Auto-Scaling

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/cmp_promoter" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
  --min-capacity 5 \
  --max-capacity 500

aws application-autoscaling put-scaling-policy \
  --service-namespace dynamodb \
  --resource-id "table/cmp_promoter" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
  --policy-name "cmp-promoter-write-scaling" \
  --policy-type "TargetTrackingScaling" \
  --target-tracking-scaling-policy-configuration \
    '{"TargetValue": 70.0, "PredefinedMetricSpecification": {"PredefinedMetricType": "DynamoDBWriteCapacityUtilization"}}'
```

### Cost Optimization Tips

1. **Use TTL for snapshots**: Snapshots expire after 30 days. Enable DynamoDB TTL on the `expires_at` attribute to auto-delete expired items at no cost.
2. **On-demand for dev/staging**: Avoid paying for idle provisioned capacity in non-production environments.
3. **Batch writes during promotion**: The executor batches artifact outcomes to reduce individual write operations.
4. **Monitor consumed vs. provisioned**: If consumed capacity is consistently < 30% of provisioned, switch to on-demand or reduce provisioned capacity.

#### Enable TTL

```bash
aws dynamodb update-time-to-live \
  --table-name cmp_promoter \
  --time-to-live-specification "Enabled=true,AttributeName=expires_at"
```

---

## Data Retention Policies

### Session Retention (90 Days)

Promotion session records (`SESSION#` and `SESSION#...#ART#` items) are retained for a minimum of 90 days per requirement 7.6.

**Implementation**: Sessions do not use DynamoDB TTL by default. Implement a scheduled cleanup job to delete sessions older than 90 days:

```bash
# Example: Lambda or cron job to purge old sessions
# Query sessions with started_at older than 90 days and delete them
```

**Recommended approach**: Add an `expires_at` attribute to session records set to 90 days from creation, and enable TTL. This provides automatic, zero-cost cleanup.

### Snapshot Retention (30 Days)

Rollback snapshots (`SNAP#` items) are retained for 30 days to support the rollback window (requirement 8.1).

**Implementation**: Snapshots include an `expires_at` field set to 30 days from creation. With DynamoDB TTL enabled on `expires_at`, expired snapshots are automatically deleted within 48 hours of expiry.

### ID Mapping Retention

ID mappings (`IDMAP#` items) are retained as long as the parent session exists (90 days). They share the session's lifecycle and are cleaned up together.

### Retention Summary

| Data Type | Retention Period | Cleanup Mechanism |
|-----------|-----------------|-------------------|
| Promotion sessions | 90 days minimum | TTL or scheduled job |
| Artifact outcomes | 90 days (with session) | TTL or scheduled job |
| Rollback snapshots | 30 days | DynamoDB TTL on `expires_at` |
| ID mappings | 90 days (with session) | TTL or scheduled job |
| Environment connections | Indefinite (user-managed) | Manual deletion |

---

## Runbook: Common Operational Issues

### Issue 1: High Latency on Promotion Execution

**Symptoms**: Promotion operations taking > 60 seconds, users reporting slow plan generation or execution.

**Diagnosis**:

1. Check DynamoDB consumed capacity:
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/DynamoDB \
     --metric-name ConsumedWriteCapacityUnits \
     --dimensions Name=TableName,Value=cmp_promoter \
     --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
     --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
     --period 60 --statistics Sum
   ```

2. Check if target CMP environment is responding slowly:
   ```bash
   curl -w "@curl-format.txt" -o /dev/null -s https://target-cmp.example.com/health
   ```

3. Review application logs for slow operations:
   ```bash
   aws logs filter-log-events \
     --log-group-name /ecs/cmp-promoter \
     --filter-pattern '"completed in" "ms"' \
     --start-time $(date -u -v-1H +%s)000
   ```

**Resolution**:

- If DynamoDB is the bottleneck: increase provisioned capacity or switch to on-demand mode
- If target CMP is slow: check target environment health, consider increasing httpx timeout
- If bundle is large (close to 500 artifacts): this is expected behavior; inform the user that large promotions take longer
- If CPU is saturated: scale up ECS task count or increase CPU allocation

### Issue 2: DynamoDB Throttling

**Symptoms**: `ThrottledRequests` CloudWatch alarm firing, application logs showing retry warnings, intermittent 500 errors.

**Diagnosis**:

1. Identify which operations are being throttled:
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/DynamoDB \
     --metric-name ThrottledRequests \
     --dimensions Name=TableName,Value=cmp_promoter Name=Operation,Value=PutItem \
     --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
     --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
     --period 60 --statistics Sum
   ```

2. Check if a bulk promotion is in progress (high write volume is expected during bulk operations).

3. Review the partition key distribution — hot partitions can cause throttling even with sufficient overall capacity.

**Resolution**:

- **Immediate**: Switch to on-demand capacity mode (takes effect within minutes):
  ```bash
  aws dynamodb update-table \
    --table-name cmp_promoter \
    --billing-mode PAY_PER_REQUEST
  ```

- **If provisioned mode**: Increase WCU/RCU or enable auto-scaling with a higher max:
  ```bash
  aws application-autoscaling update-scalable-target \
    --service-namespace dynamodb \
    --resource-id "table/cmp_promoter" \
    --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
    --max-capacity 1000
  ```

- **Long-term**: If throttling is frequent during bulk promotions, consider implementing write batching with exponential backoff in the application layer (already implemented with 3 retries).

### Issue 3: Token Refresh Failures

**Symptoms**: Promotions failing mid-execution with "authentication failed" errors, `TokenRefreshFailures` metric increasing.

**Diagnosis**:

1. Check application logs for token refresh attempts:
   ```bash
   aws logs filter-log-events \
     --log-group-name /ecs/cmp-promoter \
     --filter-pattern '"token" "refresh"' \
     --start-time $(date -u -v-1H +%s)000
   ```

2. Verify the target CMP environment's auth endpoint is responding:
   ```bash
   curl -X POST https://target-cmp.example.com/api/v1/auth/login \
     -H "Content-Type: application/json" \
     -d '{"username":"test","password":"test"}'
   ```

3. Check if the user's credentials have been rotated or expired in the CMP environment.

**Resolution**:

- If the CMP environment auth endpoint is down: wait for it to recover; the Promoter retries 3 times with 5-second intervals
- If user credentials have changed: the user must update their Environment Connection with new API token
- If JWT secret has been rotated on the CMP environment: all active tokens are invalidated; users must re-authenticate
- If network connectivity is intermittent: check security groups, NACLs, or NSGs between the Promoter and CMP environments

### Issue 4: Health Endpoint Returning Unhealthy

**Symptoms**: Load balancer marking targets as unhealthy, container restarts.

**Diagnosis**:

1. Check container logs for startup errors:
   ```bash
   aws logs filter-log-events \
     --log-group-name /ecs/cmp-promoter \
     --filter-pattern '"ERROR" "startup"' \
     --start-time $(date -u -v-10M +%s)000
   ```

2. Common startup failures:
   - `Failed to connect to DynamoDB after 5 attempts` — DynamoDB endpoint unreachable
   - Missing `ENCRYPTION_KEY` — container will start but encryption operations will fail

**Resolution**:

- Verify DynamoDB endpoint is accessible from the container's VPC/subnet
- Verify security groups allow outbound traffic to DynamoDB (port 443 for AWS service endpoint, or port 8000 for local DynamoDB)
- Verify all required environment variables are set (especially `ENCRYPTION_KEY` and `SECRET_KEY`)
- Check that the IAM task role has DynamoDB permissions

### Issue 5: Partial Promotion Failures

**Symptoms**: Promotion sessions completing with status `partially_completed`, some artifacts showing `failed` or `skipped` outcomes.

**Diagnosis**:

1. Retrieve the session details:
   ```bash
   curl https://promoter-api.example.com/api/v1/sessions/<session_id> \
     -H "Authorization: Bearer <token>"
   ```

2. Review failed artifact outcomes — the error message indicates the specific failure reason.

3. Common failure reasons:
   - `target unreachable` — target CMP went down mid-promotion
   - `artifact not found` — source artifact was deleted between plan generation and execution
   - `conflict` — target was modified after plan generation (stale plan)
   - `dependency failed` — a parent artifact failed, so dependents were skipped

**Resolution**:

- If target was temporarily unreachable: re-run the promotion for failed artifacts only
- If stale plan: regenerate the promotion plan and re-execute
- If dependency chain failure: fix the root cause (first failed artifact) and re-promote
- Consider using rollback if the partial state is problematic, then re-promote cleanly

### Issue 6: Rollback Failures

**Symptoms**: Rollback completing with status `partially_rolled_back`.

**Diagnosis**:

1. Check if the rollback window (30 days) has not expired
2. Review per-artifact rollback outcomes for specific failure reasons
3. Verify target CMP environment is accessible

**Resolution**:

- If specific artifacts fail to restore: manually restore them via the CMP UI using the snapshot data stored in the session
- If target is unreachable: retry the rollback once connectivity is restored
- If snapshots have expired (> 30 days): manual restoration from backups is required

---

## CI/CD Pipeline Configuration

### Pipeline Overview

The CMP Promoter maintains its own CI/CD pipeline, independent of the CMP platform pipeline.

```
┌──────────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│   Lint/Type  │───▶│   Test   │───▶│  Build   │───▶│   Scan   │───▶│  Deploy  │
└──────────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

### GitHub Actions Workflow

```yaml
# .github/workflows/promoter-deploy.yml
name: CMP Promoter — Build & Deploy

on:
  push:
    branches: [main]
    paths:
      - 'cmp_promoter/**'
  pull_request:
    branches: [main]
    paths:
      - 'cmp_promoter/**'

env:
  PYTHON_VERSION: '3.12'
  NODE_VERSION: '20'

jobs:
  lint-and-type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - name: Install dependencies
        run: pip install -r cmp_promoter/backend/requirements.txt
      - name: Lint (ruff)
        run: ruff check cmp_promoter/backend/
      - name: Type check (mypy)
        run: mypy cmp_promoter/backend/app/ --ignore-missing-imports

  test:
    runs-on: ubuntu-latest
    needs: lint-and-type-check
    services:
      dynamodb:
        image: amazon/dynamodb-local:latest
        ports:
          - 8000:8000
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - name: Install dependencies
        run: pip install -r cmp_promoter/backend/requirements.txt
      - name: Run tests
        env:
          DYNAMODB_ENDPOINT: http://localhost:8000
          DYNAMODB_TABLE: cmp_promoter_test
          AWS_ACCESS_KEY_ID: local
          AWS_SECRET_ACCESS_KEY: local
          AWS_REGION: us-east-1
          ENCRYPTION_KEY: test-encryption-key-32chars!!
          SECRET_KEY: test-secret-key
        run: pytest cmp_promoter/backend/ -v --tb=short

  build-backend:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}
      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2
      - name: Build and push
        run: |
          docker build -t cmp-promoter:${{ github.sha }} ./cmp_promoter/backend
          docker tag cmp-promoter:${{ github.sha }} ${{ secrets.ECR_REGISTRY }}/cmp-promoter:${{ github.sha }}
          docker tag cmp-promoter:${{ github.sha }} ${{ secrets.ECR_REGISTRY }}/cmp-promoter:latest
          docker push ${{ secrets.ECR_REGISTRY }}/cmp-promoter:${{ github.sha }}
          docker push ${{ secrets.ECR_REGISTRY }}/cmp-promoter:latest

  build-frontend:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      - name: Install and build
        working-directory: cmp_promoter/frontend
        run: |
          npm ci
          VITE_API_URL=${{ secrets.PROMOTER_API_URL }} npm run build
      - name: Upload to S3
        run: |
          aws s3 sync cmp_promoter/frontend/dist/ s3://${{ secrets.FRONTEND_BUCKET }}/ --delete

  deploy:
    runs-on: ubuntu-latest
    needs: [build-backend, build-frontend]
    steps:
      - name: Update ECS service
        run: |
          aws ecs update-service \
            --cluster ${{ secrets.ECS_CLUSTER }} \
            --service ${{ secrets.ECS_SERVICE }} \
            --force-new-deployment
      - name: Wait for stable
        run: |
          aws ecs wait services-stable \
            --cluster ${{ secrets.ECS_CLUSTER }} \
            --services ${{ secrets.ECS_SERVICE }}
      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DIST_ID }} \
            --paths "/*"
```

### Secrets Management

#### Required CI/CD Secrets

| Secret | Purpose | Rotation Frequency |
|--------|---------|-------------------|
| `AWS_ACCESS_KEY_ID` | CI/CD AWS access for ECR push and ECS deploy | 90 days |
| `AWS_SECRET_ACCESS_KEY` | CI/CD AWS secret key | 90 days |
| `AWS_REGION` | Target AWS region | Static |
| `ECR_REGISTRY` | ECR registry URL | Static |
| `ECS_CLUSTER` | ECS cluster name | Static |
| `ECS_SERVICE` | ECS service name | Static |
| `PROMOTER_API_URL` | Backend API URL for frontend build | Static |
| `FRONTEND_BUCKET` | S3 bucket for frontend static assets | Static |
| `CLOUDFRONT_DIST_ID` | CloudFront distribution for cache invalidation | Static |

#### Application Secrets (Runtime)

| Secret | Storage | Rotation Procedure |
|--------|---------|-------------------|
| `SECRET_KEY` | AWS Secrets Manager / Azure Key Vault | Rotate and restart all instances; existing JWTs will be invalidated |
| `ENCRYPTION_KEY` | AWS Secrets Manager / Azure Key Vault | **Cannot rotate without re-encrypting all stored credentials**; see below |

#### Encryption Key Rotation

The `ENCRYPTION_KEY` is used to encrypt stored API tokens for Environment Connections. Rotating it requires:

1. Decrypt all existing connection tokens with the old key
2. Re-encrypt with the new key
3. Update the secret in Secrets Manager/Key Vault
4. Restart all application instances

```python
# Script to rotate encryption key (run as one-time migration)
from cryptography.fernet import Fernet

old_key = "old-encryption-key"
new_key = "new-encryption-key"

old_fernet = Fernet(old_key.encode() if len(old_key) == 44 else Fernet.generate_key())
new_fernet = Fernet(new_key.encode() if len(new_key) == 44 else Fernet.generate_key())

# For each connection in DynamoDB:
# 1. Decrypt api_token_encrypted with old_fernet
# 2. Re-encrypt with new_fernet
# 3. Update the item in DynamoDB
```

---

## Backup and Disaster Recovery

### DynamoDB Backup Strategy

#### Point-in-Time Recovery (PITR)

Enable PITR for continuous backups with 35-day retention:

```bash
aws dynamodb update-continuous-backups \
  --table-name cmp_promoter \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true
```

PITR allows restoring the table to any second within the last 35 days. This is the primary recovery mechanism.

#### On-Demand Backups

Create on-demand backups before major operations or deployments:

```bash
# Create a backup before deployment
aws dynamodb create-backup \
  --table-name cmp_promoter \
  --backup-name "cmp-promoter-pre-deploy-$(date +%Y%m%d-%H%M%S)"

# List existing backups
aws dynamodb list-backups --table-name cmp_promoter

# Restore from backup (creates a new table)
aws dynamodb restore-table-from-backup \
  --target-table-name cmp_promoter_restored \
  --backup-arn arn:aws:dynamodb:<region>:<account>:table/cmp_promoter/backup/<backup-id>
```

#### Scheduled Backups (AWS Backup)

```bash
# Create a backup plan for daily backups with 30-day retention
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "cmp-promoter-daily",
  "Rules": [{
    "RuleName": "DailyBackup",
    "TargetBackupVaultName": "Default",
    "ScheduleExpression": "cron(0 3 * * ? *)",
    "Lifecycle": { "DeleteAfterDays": 30 }
  }]
}'
```

### Disaster Recovery Scenarios

#### Scenario 1: DynamoDB Table Corruption or Accidental Deletion

**Recovery**:
1. Restore from PITR to the last known good state:
   ```bash
   aws dynamodb restore-table-to-point-in-time \
     --source-table-name cmp_promoter \
     --target-table-name cmp_promoter_restored \
     --restore-date-time "2025-01-15T10:00:00Z"
   ```
2. Verify restored data integrity
3. Rename or swap tables (delete corrupted, rename restored)
4. Restart application instances

**RTO**: 30-60 minutes (depending on table size)
**RPO**: Seconds (PITR granularity)

#### Scenario 2: Application Secret Compromise

**Recovery**:
1. Immediately rotate `SECRET_KEY` in Secrets Manager — this invalidates all active JWTs
2. Rotate `ENCRYPTION_KEY` using the migration script (re-encrypt all stored tokens)
3. Force restart all application instances
4. Notify affected users to re-authenticate
5. Audit session logs for unauthorized access during the compromise window

#### Scenario 3: Complete Region Failure

**Recovery** (if multi-region is configured):
1. DynamoDB Global Tables replicate to the secondary region automatically
2. Update DNS to point to the secondary region's ALB/Container App
3. Verify health endpoint in the secondary region
4. Monitor for data consistency after failover

**If single-region**: Restore from the most recent on-demand backup or PITR in a new region.

#### Scenario 4: Encryption Key Loss

**Impact**: All stored Environment Connection API tokens become unrecoverable.

**Recovery**:
1. Deploy with a new `ENCRYPTION_KEY`
2. Users must re-enter API tokens for all Environment Connections
3. Existing connections will fail decryption — mark them as requiring reconfiguration

**Prevention**: Store the encryption key in at least two independent secret stores (e.g., AWS Secrets Manager + encrypted backup in a separate account).

### Recovery Time Objectives

| Scenario | RTO | RPO |
|----------|-----|-----|
| DynamoDB table issue | 30-60 min | Seconds (PITR) |
| Application failure | 5-10 min | Zero (stateless) |
| Secret compromise | 15-30 min | Zero |
| Region failure (multi-region) | 5-15 min | Seconds (Global Tables) |
| Region failure (single-region) | 1-4 hours | Last backup |

### Pre-Deployment Checklist

- [ ] PITR enabled on `cmp_promoter` table
- [ ] On-demand backup created before deployment
- [ ] `ENCRYPTION_KEY` stored in Secrets Manager with backup
- [ ] `SECRET_KEY` stored in Secrets Manager
- [ ] Health endpoint accessible after deployment
- [ ] CloudWatch alarms configured and tested
- [ ] Log retention set to 90 days
- [ ] DynamoDB TTL enabled on `expires_at` attribute
