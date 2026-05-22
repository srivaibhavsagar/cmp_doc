# Terraform, ECR Setup & Release Pipeline Guide

> Complete guide for setting up vendor infrastructure (ECR, S3, KMS), deploying customer environments, and understanding the release pipeline.

---

## Table of Contents

1. [ECR Setup — Step by Step](#ecr-setup--step-by-step)
2. [Terraform — Vendor Infrastructure](#terraform--vendor-infrastructure)
3. [Terraform — Customer Deployment](#terraform--customer-deployment)
4. [Single EC2 vs Production-Grade (HA)](#single-ec2-vs-production-grade-ha)
5. [What release.yml Does — Stage by Stage](#what-releaseyml-does--stage-by-stage)
6. [GitHub Secrets Required](#github-secrets-required)

---

## ECR Setup — Step by Step

### Option A: Using Terraform (Recommended)

```bash
cd cloud_management_platform/terraform/vendor

# 1. Initialize Terraform
terraform init

# 2. Review what will be created
terraform plan

# 3. Apply (creates ECR repos, S3, KMS, IAM)
terraform apply

# 4. Note the outputs
terraform output ecr_registry_url
# Example: 123456789012.dkr.ecr.us-east-1.amazonaws.com

terraform output -raw github_actions_access_key_id
terraform output -raw github_actions_secret_access_key
# → Add these to GitHub Secrets
```

### Option B: Manual AWS CLI

If you prefer to create ECR manually:

```bash
# Set your region
export AWS_REGION=us-east-1

# 1. Create ECR repositories
aws ecr create-repository \
  --repository-name autonimbus/cmp/cmp-backend \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --region $AWS_REGION

aws ecr create-repository \
  --repository-name autonimbus/cmp/cmp-frontend \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --region $AWS_REGION

aws ecr create-repository \
  --repository-name autonimbus/cmp/cmp-worker \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --region $AWS_REGION

aws ecr create-repository \
  --repository-name autonimbus/cmp/cmp-ai \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --region $AWS_REGION

# 2. Verify repositories
aws ecr describe-repositories --region $AWS_REGION | jq '.repositories[].repositoryUri'

# 3. Get your registry URL
aws sts get-caller-identity --query Account --output text
# Your registry: <account-id>.dkr.ecr.<region>.amazonaws.com

# 4. Test login
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin \
  <account-id>.dkr.ecr.$AWS_REGION.amazonaws.com

# 5. Set lifecycle policy (auto-clean old images)
aws ecr put-lifecycle-policy \
  --repository-name autonimbus/cmp/cmp-backend \
  --lifecycle-policy-text '{
    "rules": [{
      "rulePriority": 1,
      "description": "Keep last 20 images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 20
      },
      "action": {"type": "expire"}
    }]
  }'
```

### ECR Cross-Account Access (For Customers)

To let a customer pull images from your ECR:

```bash
# Set repository policy allowing customer's AWS account to pull
aws ecr set-repository-policy \
  --repository-name autonimbus/cmp/cmp-backend \
  --policy-text '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "AllowCustomerPull",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::CUSTOMER_ACCOUNT_ID:root"},
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability"
      ]
    }]
  }'
```

### How Customers Authenticate to ECR

**Option 1: Cross-account IAM (recommended for AWS customers)**
```bash
# Customer runs this on their EC2 (needs ECR read policy on their IAM role)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  <YOUR_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

**Option 2: You generate a token and send it (non-AWS customers)**
```bash
# You generate a 12-hour token
TOKEN=$(aws ecr get-login-password --region us-east-1)
echo "Registry: <account-id>.dkr.ecr.us-east-1.amazonaws.com"
echo "Username: AWS"
echo "Password: $TOKEN"
echo "Expires: 12 hours"
# Send to customer — they use: docker login --username AWS --password-stdin
```

---

## Terraform — Vendor Infrastructure

### What It Creates

| Resource | Purpose |
|----------|---------|
| 4 ECR repositories | cmp-backend, cmp-frontend, cmp-worker, cmp-ai |
| S3 bucket | Release packages, air-gapped archives |
| KMS key | Encrypt license signing private key |
| 2 Secrets Manager secrets | License private key, code-signing key |
| IAM user + access key | GitHub Actions CI/CD access |
| ECR lifecycle policies | Auto-delete old images (keep last 20) |

### How to Execute

```bash
cd cloud_management_platform/terraform/vendor

# Prerequisites: AWS CLI configured with admin access
aws sts get-caller-identity  # Verify you're in the right account

# 1. Initialize
terraform init

# 2. Plan (review changes)
terraform plan -out=plan.tfplan

# 3. Apply
terraform apply plan.tfplan

# 4. Get outputs (save these!)
terraform output

# Key outputs:
#   ecr_registry_url              → Use in docker-compose.production.yml
#   s3_releases_bucket            → Upload release packages here
#   github_actions_access_key_id  → Add to GitHub Secrets
#   github_actions_secret_access_key → Add to GitHub Secrets
```

### After Terraform Apply

1. Add GitHub Secrets:
   - `AWS_ACCESS_KEY_ID` → from terraform output
   - `AWS_SECRET_ACCESS_KEY` → from terraform output
   - `AWS_REGION` → your region (e.g., `us-east-1`)
   - `AWS_ACCOUNT_ID` → your AWS account ID

2. Store your RSA private key in Secrets Manager:
   ```bash
   aws secretsmanager put-secret-value \
     --secret-id autonimbus-cmp/license-private-key \
     --secret-string "$(cat vendor_private_key.pem)"
   ```

3. Store your code-signing key:
   ```bash
   aws secretsmanager put-secret-value \
     --secret-id autonimbus-cmp/code-signing-key \
     --secret-string "$(cat code_signing_key.pem)"
   ```

---

## Terraform — Customer Deployment

### What It Creates

| Resource | Single Mode | HA Mode |
|----------|-------------|---------|
| VPC + Subnets | 1 VPC, 1 subnet | 1 VPC, 2 subnets (multi-AZ) |
| Internet Gateway | Yes (except air-gapped) | Yes (except air-gapped) |
| Security Groups | 1 (instance) | 2 (instance + ALB) |
| EC2 Instance | 1x t3.xlarge | 2x m5.xlarge |
| EBS Volumes | 100GB root + 200GB data | 100GB root + 200GB data (each) |
| ALB | Optional | Yes |
| IAM Role | ECR pull (online only) | ECR pull (online only) |

### How to Execute

```bash
cd cloud_management_platform/terraform/customer

# 1. Initialize
terraform init

# 2. Create a tfvars file for the customer
cat > customer.tfvars << 'EOF'
customer_name    = "acme-corp"
aws_region       = "us-east-1"
deployment_mode  = "single"        # "single" or "ha"
deployment_model = "on-premises"   # "on-premises", "air-gapped", "hybrid", "saas"
instance_type    = "t3.xlarge"
data_volume_size = 200
enable_alb       = true
key_pair_name    = "my-key-pair"
ecr_registry_url = "123456789012.dkr.ecr.us-east-1.amazonaws.com/autonimbus/cmp"
ssh_allowed_cidr = "203.0.113.0/32"  # Your office IP
EOF

# 3. Plan
terraform plan -var-file=customer.tfvars -out=plan.tfplan

# 4. Apply
terraform apply plan.tfplan

# 5. Get connection info
terraform output ssh_command
terraform output instance_public_ip
terraform output alb_dns_name
```

### Example: Deploy for Air-Gapped Customer

```bash
cat > airgapped-customer.tfvars << 'EOF'
customer_name    = "gov-agency"
aws_region       = "us-gov-west-1"
deployment_mode  = "single"
deployment_model = "air-gapped"
instance_type    = "m5.xlarge"
data_volume_size = 300
enable_alb       = false           # No ALB for air-gapped
key_pair_name    = "gov-key"
ecr_registry_url = ""              # No ECR for air-gapped
ssh_allowed_cidr = "10.0.0.0/8"   # Internal network only
EOF

terraform apply -var-file=airgapped-customer.tfvars
```

### After Terraform Apply (Customer Side)

```bash
# SSH into the instance
ssh -i ~/.ssh/my-key-pair.pem ubuntu@<instance-ip>

# The bootstrap script already installed Docker and created /opt/cmp/
# Now complete the setup:

# 1. Edit configuration
vi /opt/cmp/.env
# Set SECRET_KEY, ENCRYPTION_KEY, etc.

# 2. Place license
cp /path/to/license.json /opt/cmp/license/license.json

# 3. Place docker-compose.yml
cp /path/to/docker-compose.production.yml /opt/cmp/docker-compose.yml

# 4. Login to ECR (online deployments)
/opt/cmp/ecr-login.sh

# 5. Pull and start (with --profile local to include bundled Redis + DynamoDB)
cd /opt/cmp
docker compose --profile local pull
docker compose --profile local up -d

# 6. Verify (wait 60 seconds for startup)
sleep 60
curl -s http://localhost:8000/health | jq .
```

---

## Docker Compose Profiles — How It Works

The `docker-compose.production.yml` uses Docker Compose **profiles** to control whether Redis and DynamoDB run as local containers or are expected as external managed services.

### The `--profile local` Flag

| Command | What Starts | Use When |
|---------|-------------|----------|
| `docker compose --profile local up -d` | All 6 services (app + Redis + DynamoDB) | Standard deployment — customer's infra team manages the EC2 |
| `docker compose up -d` | Only 4 app services (frontend, backend, worker, ai) | Customer provides their own Redis + DynamoDB externally |

### Standard Deployment (Recommended for All Customers)

```bash
# Start everything including bundled Redis and DynamoDB Local
docker compose --profile local -f docker-compose.production.yml up -d
```

`.env` settings:
```bash
DYNAMODB_ENDPOINT=http://database:8000
REDIS_URL=redis://redis:6379/0
```

This is the default for all deployment models (on-premises, air-gapped, hybrid). The customer's backup/DR team handles EBS snapshots of the instance — that covers the database and Redis data.

### Optional: External Services (Customer's Choice)

If a customer's infrastructure team prefers to use their own managed Redis or DynamoDB (because they already have them, or their DR policy requires it):

```bash
# Start only app services — no local Redis/DynamoDB containers
docker compose -f docker-compose.production.yml up -d
```

`.env` settings:
```bash
DYNAMODB_ENDPOINT=https://dynamodb.us-east-1.amazonaws.com
REDIS_URL=redis://their-redis-cluster.internal:6379/0
```

**This is the customer's decision, not yours.** You ship with `--profile local` as the standard. If their team wants to point to external services, they just change the `.env` and drop the `--profile local` flag.

### Why This Design

- **One compose file** for all scenarios (no separate files to maintain)
- **Default includes everything** — works out of the box
- **Customer's infra team decides** whether to use bundled or external services
- **No vendor responsibility** for their Redis/DynamoDB infrastructure
- **Air-gapped works perfectly** — everything is local, no external dependencies

---

## What release.yml Does — Stage by Stage

The release pipeline (`.github/workflows/release.yml`) runs when you push a version tag (`v*.*.*`) or manually trigger it.

### Pipeline Flow

```
Tag Push (v1.0.0)
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Stage 1: PREPARE                                             │
│ • Extracts version from tag (e.g., "1.0.0")                │
│ • Gets commit short hash (e.g., "a1b2c3d4")                │
│ • Computes image tag: "1.0.0-a1b2c3d4"                     │
│ • Outputs: version, commit_short, image_tag                 │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Stage 2: VERIFY WORKFLOWS                                    │
│ • Checks git diff of the release commit                     │
│ • Ensures deploy-dev.yml, deploy-azure-dev.yml, deploy.yml, │
│   and dev-ops.yml are NOT modified                          │
│ • FAILS if any protected workflow was changed               │
│ • Purpose: Guarantees dev workflow isolation                 │
└─────────────────────────────────────────────────────────────┘
    │
    ▼ (parallel)
┌──────────────────────────┐  ┌──────────────────────────────┐
│ Stage 3: COMPILE BACKEND │  │ Stage 4: BUILD FRONTEND      │
│ • Installs Python 3.12   │  │ • Installs Node 20           │
│ • Installs Nuitka        │  │ • Runs npm ci                │
│ • Compiles backend/app/  │  │ • Vite production build      │
│   into standalone binary │  │   (minified, no source maps) │
│ • Binary: cmp-backend    │  │ • JavaScript obfuscation     │
│ • Verifies --host/--port │  │ • Gzip + Brotli compression  │
│ • Uploads as artifact    │  │ • Validates chunk sizes      │
│                          │  │ • Uploads dist/ as artifact  │
└──────────────────────────┘  └──────────────────────────────┘
    │                              │
    ▼                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Stage 5: BUILD IMAGES (matrix: 4 services)                   │
│ • Downloads backend binary artifact                         │
│ • Downloads frontend dist artifact                          │
│ • Builds multi-stage Docker images:                         │
│   - cmp-backend (binary + minimal runtime)                  │
│   - cmp-frontend (dist + nginx)                             │
│   - cmp-worker (binary + minimal runtime)                   │
│   - cmp-ai (binary + minimal runtime)                       │
│ • Tags: <version>-<commit>, <version>, latest               │
│ • Verifies: no source files, no .map files, < 2GB           │
│ • Saves images as tar.gz artifacts                          │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Stage 6: SCAN IMAGES (Trivy — matrix: 4 services)            │
│ • Loads each image from artifact                            │
│ • Runs Trivy vulnerability scan                             │
│ • Severity filter: CRITICAL + HIGH                          │
│ • FAILS pipeline if any critical/high CVE found             │
│ • Reports: image name, CVE IDs, affected packages           │
│ • Can be skipped with skip_scan=true (emergency only)       │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Stage 7: PUBLISH IMAGES                                      │
│ • Configures AWS credentials                                │
│ • Logs into ECR                                             │
│ • Pushes all 4 images with 3 tags each:                     │
│   - <version>-<commit> (immutable, unique)                  │
│   - <version> (latest for this version)                     │
│   - latest (rolling latest)                                 │
│ • Images now available for customer pull                    │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Stage 8: ASSEMBLE PACKAGE                                    │
│ • Creates release package directory structure                │
│ • Includes: docker-compose, config templates, migrations,   │
│   scripts, docs, license template                           │
│ • Pulls Redis + DynamoDB images for air-gapped package      │
│ • Generates release notes from git log (features/fixes/     │
│   breaking changes grouped by commit prefix)                │
│ • Generates SHA-256 manifest of all files                   │
│ • Signs manifest with vendor code-signing key               │
│ • FAILS if signing key unavailable                          │
│ • FAILS if any required artifact is missing                 │
│ • Creates tar.gz archive                                    │
│ • Uploads as GitHub Actions artifact (30-day retention)     │
└─────────────────────────────────────────────────────────────┘
```

### What Triggers It

| Trigger | How | Example |
|---------|-----|---------|
| Version tag push | `git tag -a v1.0.0 -m "Release" && git push origin v1.0.0` | Automatic |
| Manual dispatch | GitHub Actions UI → Run workflow → enter version | Emergency/hotfix |

### What It Produces

| Output | Location | Purpose |
|--------|----------|---------|
| Container images | ECR (`<account>.dkr.ecr.<region>.amazonaws.com`) | Customer pulls these |
| Release package | GitHub Actions artifact (30 days) | Download for air-gapped delivery |
| Release notes | Inside the package (`docs/RELEASE-NOTES.md`) | Customer reads before upgrading |
| Signed manifest | Inside the package (`MANIFEST.sha256` + `.sig`) | Integrity verification |

### Pipeline Duration

| Stage | Typical Duration |
|-------|-----------------|
| Prepare | 10 seconds |
| Verify workflows | 15 seconds |
| Compile backend (Nuitka) | 10-20 minutes |
| Build frontend | 2-5 minutes |
| Build images | 5-10 minutes |
| Scan images | 3-5 minutes |
| Publish images | 2-3 minutes |
| Assemble package | 3-5 minutes |
| **Total** | **~25-45 minutes** |

---

## GitHub Secrets Required

After running the vendor Terraform, add these secrets to your GitHub repository:

| Secret Name | Value | Source |
|-------------|-------|--------|
| `AWS_ACCESS_KEY_ID` | IAM access key for GitHub Actions | `terraform output -raw github_actions_access_key_id` |
| `AWS_SECRET_ACCESS_KEY` | IAM secret key | `terraform output -raw github_actions_secret_access_key` |
| `AWS_REGION` | Your AWS region | e.g., `us-east-1` |
| `AWS_ACCOUNT_ID` | Your AWS account ID | `aws sts get-caller-identity --query Account --output text` |
| `CODE_SIGNING_PRIVATE_KEY` | PEM private key for manifest signing | Your code-signing key |

### How to Add GitHub Secrets

```bash
# Using GitHub CLI
gh secret set AWS_ACCESS_KEY_ID --body "AKIA..."
gh secret set AWS_SECRET_ACCESS_KEY --body "wJalr..."
gh secret set AWS_REGION --body "us-east-1"
gh secret set AWS_ACCOUNT_ID --body "123456789012"
gh secret set CODE_SIGNING_PRIVATE_KEY < code_signing_key.pem
```

Or via GitHub UI: Repository → Settings → Secrets and variables → Actions → New repository secret

---

## Quick Reference

| Task | Command |
|------|---------|
| Create vendor infra | `cd terraform/vendor && terraform apply` |
| Create customer infra | `cd terraform/customer && terraform apply -var-file=customer.tfvars` |
| Login to ECR | `aws ecr get-login-password \| docker login --username AWS --password-stdin <registry>` |
| Push image to ECR | `docker push <registry>/autonimbus/cmp/cmp-backend:1.0.0` |
| Trigger release | `git tag -a v1.0.0 -m "Release" && git push origin v1.0.0` |
| Manual release | GitHub Actions → Run workflow → enter version |
| List ECR images | `aws ecr list-images --repository-name autonimbus/cmp/cmp-backend` |
| Destroy vendor infra | `cd terraform/vendor && terraform destroy` |
| Destroy customer infra | `cd terraform/customer && terraform destroy -var-file=customer.tfvars` |
