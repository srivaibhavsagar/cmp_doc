# CMP Infrastructure — Registry, AWS Resources & Deployment Architecture

> **For:** Vaibhav (Vendor/Developer perspective)
> **Decision:** Use AWS ECR as private registry, EC2 + Docker Compose for customer deployments

---

## Table of Contents

1. [Private Registry — Options & Decision](#private-registry--options--decision)
2. [Vendor Infrastructure (Your AWS Account)](#vendor-infrastructure-your-aws-account)
3. [Customer Deployment on AWS](#customer-deployment-on-aws)
4. [AWS Resources Created During Deployment](#aws-resources-created-during-deployment)
5. [Cost Estimates](#cost-estimates)
6. [Recommended Starting Setup](#recommended-starting-setup)

---

## Private Registry — Options & Decision

### Options Compared

| Option | Pros | Cons | Best For |
|--------|------|------|----------|
| **AWS ECR** | Managed, HA, IAM auth, per-repo policies, vulnerability scanning built-in, scales to zero cost when idle | AWS-specific, customers need AWS CLI or Docker credential helper | ✅ Your use case |
| **GitHub Container Registry (GHCR)** | Free for public, integrates with Actions, token-based auth | Limited access control granularity, tied to GitHub | Open source projects |
| **Self-hosted Docker Registry** | Full control, any domain | You manage HA, TLS, auth, backups, scaling — operational burden | Air-gapped vendor environments |
| **Harbor** | Open source, RBAC, vulnerability scanning, replication | Complex to operate, needs dedicated infra | Large orgs with dedicated platform teams |
| **Docker Hub (private)** | Simple, well-known | Expensive at scale, rate limits, limited access control per-customer | Small teams |

### Decision: AWS ECR

```
registry.autonimbus.com → CNAME → <your-account>.dkr.ecr.<region>.amazonaws.com
```

**Why ECR works for Autonimbus:**
- Per-repository IAM policies = per-customer access control
- Lifecycle policies auto-clean old images (save cost)
- Built-in vulnerability scanning (Trivy alternative)
- Cross-region replication for global customers
- No servers to manage
- Integrates natively with GitHub Actions pipeline
- Cost: ~$0.10/GB/month storage + $0.09/GB data transfer

**Custom domain:** Put CloudFront in front of ECR to serve from `registry.autonimbus.com`.

### Is Self-Hosted Docker Registry Good?

**No, not for your use case.** Self-hosted Docker Registry requires:
- Dedicated EC2 instance (always running = cost)
- TLS certificate management
- Authentication layer (htpasswd or token service)
- Storage backend (S3 or EBS)
- High availability setup
- Backup strategy
- Monitoring and alerting
- Security patching

ECR gives you all of this managed for ~$5-20/month. Self-hosted would cost $50-100/month minimum and require ongoing maintenance.

**Exception:** If you ever need a registry inside a customer's air-gapped network, Harbor or a simple Docker Registry on their infra makes sense. But that's their problem, not yours — you deliver tar files for air-gapped.

---

## Vendor Infrastructure (Your AWS Account)

### AWS Resources for Autonimbus (Vendor Side)

| Resource | Purpose | Estimated Cost |
|----------|---------|---------------|
| **ECR** (Elastic Container Registry) | Store production images | ~$5-20/month |
| **S3 Bucket** | Store release packages, air-gapped archives, license backups | ~$5/month |
| **KMS Key** | Encrypt the RSA private key (license signing) | $1/month + $0.03/10k requests |
| **Secrets Manager** | Store registry credentials, signing keys | $0.40/secret/month |
| **CloudFront** (optional) | Custom domain for registry (`registry.autonimbus.com`) | ~$10/month |
| **Route 53** | DNS for `autonimbus.com`, `registry.autonimbus.com` | $0.50/zone/month |
| **GitHub Actions** | Build pipeline (free tier or paid) | Free for public, $0.008/min for private |

**Total vendor cost: ~$25-50/month** (scales with number of customers and image versions)

### Architecture Diagram (Vendor Side)

```
GitHub Actions (release.yml)
    │
    ├── Compile backend (Nuitka)
    ├── Build frontend (Vite)
    ├── Build Docker images
    ├── Scan with Trivy
    │
    ▼
AWS ECR (registry.autonimbus.com)
    ├── autonimbus/cmp-backend:v1.0.0-a1b2c3d4
    ├── autonimbus/cmp-frontend:v1.0.0-a1b2c3d4
    ├── autonimbus/cmp-worker:v1.0.0-a1b2c3d4
    └── autonimbus/cmp-ai:v1.0.0-a1b2c3d4
    │
    ▼
S3 (release packages)
    ├── releases/cmp-release-v1.0.0.tar.gz
    ├── releases/cmp-airgapped-v1.0.0.tar.gz
    └── licenses/cust_acme_789/license.json

KMS + Secrets Manager
    ├── vendor-rsa-private-key (license signing)
    └── code-signing-key (manifest signing)
```

### ECR Repository Structure

```
<account-id>.dkr.ecr.<region>.amazonaws.com/
├── autonimbus/cmp-backend
│     ├── :1.0.0-a1b2c3d4
│     ├── :1.0.0
│     ├── :1.1.0-b2c3d4e5
│     ├── :1.1.0
│     └── :latest
├── autonimbus/cmp-frontend
│     └── (same tag pattern)
├── autonimbus/cmp-worker
│     └── (same tag pattern)
└── autonimbus/cmp-ai
      └── (same tag pattern)
```

### Per-Customer Access Control (ECR Policy)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCustomerPull",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::CUSTOMER_ACCOUNT:root"
      },
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability"
      ],
      "Condition": {
        "StringLike": {
          "ecr:ResourceTag/customer": "cust_acme_789"
        }
      }
    }
  ]
}
```

---

## Customer Deployment on AWS

### Option A: Single EC2 Instance (Small/Medium Customers)

**Simplest deployment — everything on one machine.**

| Resource | Spec | Purpose | Monthly Cost |
|----------|------|---------|--------------|
| **EC2 Instance** | t3.xlarge (4 vCPU, 16 GB) or m5.xlarge | Run all Docker containers | ~$120-170/month |
| **EBS Volume (root)** | 100 GB gp3 | OS + Docker images | ~$8/month |
| **EBS Volume (data)** | 200 GB gp3 | DynamoDB Local data + backups | ~$16/month |
| **Elastic IP** | 1 | Static IP for the instance | Free (when attached to running instance) |
| **Security Group** | 1 | Firewall rules | Free |
| **ALB** (optional) | 1 | TLS termination, load balancing | ~$22/month |
| **ACM Certificate** | 1 | Free TLS certificate | Free |

**Total: ~$170-220/month**

```
Internet → ALB (443/TLS) → EC2 Instance
                              ├── cmp-frontend (port 3000)
                              ├── cmp-backend (port 8000)
                              ├── cmp-worker
                              ├── cmp-ai (port 8001)
                              ├── redis (port 6379)
                              └── dynamodb-local (port 8000)
```

### Option B: Production-Grade HA (Large Customers)

**For customers who want high availability and managed services.**

| Resource | Spec | Purpose | Monthly Cost |
|----------|------|---------|--------------|
| **EC2 Instances** | 2x m5.xlarge (multi-AZ) | Run CMP containers | ~$340/month |
| **ALB** | 1 | Load balancing + TLS + health checks | ~$22/month |
| **DynamoDB** (AWS managed) | On-demand billing | Database (replaces local) | ~$25-100/month |
| **ElastiCache Redis** | cache.t3.medium | Managed Redis (multi-AZ) | ~$50/month |
| **EBS Volumes** | 2x 100 GB gp3 | Instance storage | ~$16/month |
| **S3 Bucket** | 1 | Backups, audit logs | ~$5/month |
| **ACM Certificate** | 1 | TLS | Free |
| **Security Groups** | 3 | ALB, app tier, data tier | Free |
| **VPC** | 1 (or existing) | Network isolation | Free |
| **NAT Gateway** | 1 (if private subnets) | Outbound internet for updates | ~$45/month |

**Total: ~$500-600/month**

```
Internet → ALB (443/TLS)
              ├── EC2 Instance 1 (AZ-a)
              │     ├── cmp-frontend
              │     ├── cmp-backend
              │     ├── cmp-worker
              │     └── cmp-ai
              └── EC2 Instance 2 (AZ-b) [standby/HA]
                    └── (same containers)

AWS Managed Services:
├── DynamoDB (multi-AZ, auto-scaling)
└── ElastiCache Redis (multi-AZ, failover)
```

### Option C: Air-Gapped (Government/Banking)

**No internet, no AWS managed services, complete isolation.**

| Resource | Spec | Purpose | Monthly Cost |
|----------|------|---------|--------------|
| **EC2 Instance** | m5.xlarge | Run all containers | ~$170/month |
| **EBS Volumes** | 300 GB gp3 | Everything local | ~$24/month |
| **Security Group** | 1 | No outbound rules | Free |
| **VPC** | Private only (no IGW, no NAT) | Complete isolation | Free |

**Total: ~$200/month**

No NAT Gateway, no internet gateway, no managed services. Everything runs locally inside the instance.

```
VPC (private, no internet gateway)
└── Private Subnet
    └── EC2 Instance (no public IP)
          ├── cmp-frontend
          ├── cmp-backend
          ├── cmp-worker
          ├── cmp-ai
          ├── redis (local)
          └── dynamodb-local (local)
```

---

## AWS Resources Created During Deployment

### Minimum Resources (Single EC2 — All Models)

```
AWS Account
├── VPC (or use existing default VPC)
│     ├── Subnet (public for online, private for air-gapped)
│     ├── Internet Gateway (online models only)
│     ├── Route Table
│     └── Security Group
│           ├── Inbound: 443 (HTTPS from anywhere)
│           ├── Inbound: 22 (SSH from admin IP only)
│           └── Outbound: 443 (registry pull — online only)
│
├── EC2 Instance
│     ├── Instance Type: t3.xlarge or m5.xlarge
│     ├── AMI: Ubuntu 22.04 LTS or Amazon Linux 2023
│     ├── EBS Root Volume: 100 GB gp3
│     ├── EBS Data Volume: 200 GB gp3 (mounted at /opt/cmp/data)
│     ├── IAM Instance Profile (for ECR pull — online only)
│     └── User Data script (Docker install + CMP setup)
│
├── Elastic IP (optional, for static addressing)
│
├── ALB (optional, for TLS termination)
│     ├── Listener: 443 (HTTPS)
│     ├── Target Group → EC2 Instance:3000
│     └── ACM Certificate (auto-renewed)
│
└── S3 Bucket (optional, for backups)
```

### What's Running Inside the EC2 Instance

```
/opt/cmp/
├── docker-compose.yml          (production compose file)
├── .env                        (configuration)
├── license/
│   └── license.json            (cryptographic license)
├── config/
│   └── encrypted.env           (encrypted secrets, optional)
├── logs/
│   ├── upgrade-audit.log
│   └── alerts.log
├── backups/
│   └── backup-20260522-020000/
└── data/
    ├── dynamodb/               (DynamoDB Local data files)
    └── redis/                  (Redis persistence)

Docker Containers (6 total):
├── cmp-frontend    (nginx + obfuscated JS, port 3000)
├── cmp-backend     (compiled binary, port 8000)
├── cmp-worker      (compiled Celery binary)
├── cmp-ai          (compiled AI binary, port 8001)
├── redis           (cache + message broker, port 6379)
└── database        (DynamoDB Local, port 8000)
```

### IAM Resources (Online Deployments Only)

```
IAM Role: cmp-ec2-role
├── Policy: AmazonEC2ContainerRegistryReadOnly (ECR pull)
├── Policy: Custom (S3 backup write, if using S3 for backups)
└── Instance Profile: cmp-instance-profile
```

### Network Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Customer VPC                           │
│                                                          │
│  ┌──────────────┐     ┌──────────────────────────────┐  │
│  │     ALB      │────▶│        EC2 Instance           │  │
│  │  (port 443)  │     │                              │  │
│  └──────────────┘     │  ┌────────┐  ┌────────────┐ │  │
│         ▲              │  │frontend│  │  backend   │ │  │
│         │              │  │ :3000  │  │   :8000    │ │  │
│    Internet            │  └────────┘  └────────────┘ │  │
│    (users)             │  ┌────────┐  ┌────────────┐ │  │
│                        │  │ worker │  │  ai :8001  │ │  │
│                        │  └────────┘  └────────────┘ │  │
│                        │  ┌────────┐  ┌────────────┐ │  │
│                        │  │ redis  │  │  database  │ │  │
│                        │  │ :6379  │  │   :8000    │ │  │
│                        │  └────────┘  └────────────┘ │  │
│                        └──────────────────────────────┘  │
│                                                          │
│  Outbound (online only):                                 │
│  → registry.autonimbus.com:443 (image pull)              │
│  → license.autonimbus.com:443 (verification, hybrid/saas)│
└─────────────────────────────────────────────────────────┘
```

---

## Cost Estimates

### Vendor Side (Autonimbus — Your AWS Account)

| Item | Monthly Cost |
|------|-------------|
| ECR (4 repos, ~5 versions each, ~2GB per image) | $4-8 |
| S3 (release packages, ~50GB) | $1-2 |
| KMS (1 key) | $1 |
| Secrets Manager (3-5 secrets) | $2 |
| CloudFront (registry domain) | $5-10 |
| Route 53 (1 zone) | $0.50 |
| **Total** | **~$15-25/month** |

### Per Customer (Their AWS Account)

| Deployment Model | Monthly Cost | Notes |
|-----------------|-------------|-------|
| Single EC2 (basic) | $170-220 | t3.xlarge, local DB + Redis |
| Single EC2 + ALB | $200-250 | Add TLS termination |
| HA (2 EC2 + managed services) | $500-600 | DynamoDB + ElastiCache |
| Air-gapped (single EC2) | $170-200 | No internet, no managed services |

### Comparison: EC2 vs Other Options

| Platform | Monthly Cost | Complexity | Best For |
|----------|-------------|------------|----------|
| **Single EC2 + Docker Compose** | $170-220 | Low | Most customers ✅ |
| **ECS Fargate** | $300-500 | Medium | Customers who want serverless containers |
| **EKS (Kubernetes)** | $500-800 | High | Future — large enterprise customers |
| **Bare metal (customer DC)** | $0 (their hardware) | Low | Air-gapped, on-prem |

---

## Recommended Starting Setup

### Phase 1: Now (0-10 customers)

**Vendor side:**
- AWS ECR (4 repositories)
- S3 bucket for releases
- KMS key for license signing
- GitHub Actions for CI/CD

**Customer deployments:**
- Single EC2 instance per customer
- Docker Compose
- DynamoDB Local (bundled in compose)
- Redis (bundled in compose)

**Total vendor cost:** ~$25/month
**Per customer cost:** ~$170-220/month

### Phase 2: Growth (10-50 customers)

**Add:**
- CloudFront for registry custom domain
- Cross-region ECR replication (if global customers)
- Offer managed DynamoDB + ElastiCache as premium option
- Consider ECS Fargate for SaaS customers

### Phase 3: Scale (50+ customers)

**Add:**
- EKS (Kubernetes) support for enterprise customers
- Multi-region deployment option
- Dedicated vendor monitoring infrastructure
- Customer portal for license management

---

## Key Decisions Summary

| Decision | Choice | Reason |
|----------|--------|--------|
| Private registry | AWS ECR | Managed, cheap, IAM-based access control |
| Customer compute | EC2 + Docker Compose | Simple, portable, works everywhere |
| Customer database | DynamoDB Local (bundled) | No AWS dependency, works air-gapped |
| Customer cache | Redis (bundled) | No AWS dependency, works air-gapped |
| TLS termination | ALB + ACM (optional) | Free certs, auto-renewal |
| Backups | Local EBS + optional S3 | Works in all deployment models |
| Air-gapped delivery | tar.gz packages | No network dependency |
| License signing | KMS-protected RSA key | Secure, auditable |
| Build pipeline | GitHub Actions | Already using it, free tier available |
