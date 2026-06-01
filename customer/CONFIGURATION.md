# CMP Configuration Reference

This document provides a complete reference for all CMP configuration settings, including how to manage secrets, encrypt sensitive values, and configure each subsystem.

---

## Table of Contents

1. [Configuration Precedence](#configuration-precedence)
2. [Required Settings](#required-settings)
3. [Complete Configuration Reference](#complete-configuration-reference)
4. [Encrypting Sensitive Values](#encrypting-sensitive-values)
5. [Docker Secrets Setup](#docker-secrets-setup)
6. [Secret Rotation](#secret-rotation)
7. [SMTP Configuration](#smtp-configuration)
8. [SSO / SAML Configuration](#sso--saml-configuration)
9. [Cloud Provider Credentials](#cloud-provider-credentials)
10. [Redis Configuration](#redis-configuration)
11. [Database Configuration](#database-configuration)
12. [Feature Flags and Toggles](#feature-flags-and-toggles)
13. [Deployment Model Selection](#deployment-model-selection)

---

## Configuration Precedence

CMP loads configuration from multiple sources. When the same key is defined in multiple places, the highest-precedence source wins:

```
1. Environment variables          (highest priority)
2. .env file
3. Encrypted configuration file
4. Built-in defaults              (lowest priority)
```

**Example:** If `REDIS_URL` is set as an environment variable AND in the `.env` file, the environment variable value is used.

When a value is overridden by a higher-precedence source, CMP logs the key name and source (e.g., `"REDIS_URL loaded from: env"`) but never logs the actual value.

---

## Required Settings

The application will **refuse to start** if any of these are missing:

| Key | Description | How to Generate |
|-----|-------------|-----------------|
| `SECRET_KEY` | JWT signing key (min 32 characters) | `openssl rand -hex 32` |
| `ENCRYPTION_KEY` | Fernet encryption key (base64, 32 bytes) | `python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"` |
| `DYNAMODB_ENDPOINT` | Database endpoint URL | e.g., `http://database:8000` |
| `AWS_REGION` | AWS region for DynamoDB | e.g., `us-east-1` |

If any required key is missing, the startup error will list **all** missing keys:

```
FATAL: Required configuration missing: SECRET_KEY, ENCRYPTION_KEY
Application cannot start. Please set the above values in your .env file or environment.
```

---

## Complete Configuration Reference

### Core Application Settings

| Key | Type | Required | Default | Description | Example |
|-----|------|----------|---------|-------------|---------|
| `SECRET_KEY` | string | Yes | — | JWT signing secret (min 32 chars) | `a1b2c3d4...` (64 hex chars) |
| `ENCRYPTION_KEY` | string | Yes | — | Fernet encryption key (base64) | `gAAAAABh...` |
| `DEPLOYMENT_MODEL` | enum | Yes | — | Deployment type | `on-premises` |
| `LICENSE_FILE_PATH` | string | No | `/app/license/license.json` | Path to license file inside container | `/app/license/license.json` |
| `DEBUG` | boolean | No | `false` | Enable debug mode (never in production) | `false` |
| `LOG_LEVEL` | string | No | `INFO` | Logging level | `INFO`, `WARNING`, `ERROR` |

### Database Settings

| Key | Type | Required | Default | Description | Example |
|-----|------|----------|---------|-------------|---------|
| `DYNAMODB_ENDPOINT` | URL | Yes | — | DynamoDB endpoint | `http://database:8000` |
| `AWS_REGION` | string | Yes | — | AWS region | `us-east-1` |
| `AWS_ACCESS_KEY_ID` | string | No | — | AWS access key (if using AWS DynamoDB) | `AKIA...` |
| `AWS_SECRET_ACCESS_KEY` | string | No | — | AWS secret key (sensitive) | `wJalr...` |
| `DYNAMODB_TABLE_PREFIX` | string | No | `cmp_` | Table name prefix | `cmp_prod_` |

### Redis Settings

| Key | Type | Required | Default | Description | Example |
|-----|------|----------|---------|-------------|---------|
| `REDIS_URL` | URL | No | `redis://redis:6379/0` | Redis connection URL | `redis://:password@redis:6379/0` |
| `REDIS_PASSWORD` | string | No | — | Redis password (sensitive) | `your-redis-password` |
| `REDIS_MAX_CONNECTIONS` | integer | No | `10` | Max connection pool size | `20` |

### Frontend Settings

| Key | Type | Required | Default | Description | Example |
|-----|------|----------|---------|-------------|---------|
| `FRONTEND_URL` | URL | No | `http://localhost:3000` | Frontend base URL (for CORS) | `https://cmp.company.com` |
| `ALLOWED_ORIGINS` | string | No | `*` | Comma-separated CORS origins | `https://cmp.company.com` |

### SMTP Settings

| Key | Type | Required | Default | Description | Example |
|-----|------|----------|---------|-------------|---------|
| `SMTP_HOST` | string | No | — | SMTP server hostname | `smtp.company.com` |
| `SMTP_PORT` | integer | No | `587` | SMTP port (1–65535) | `587` |
| `SMTP_USE_TLS` | boolean | No | `true` | Enable TLS for SMTP | `true` |
| `SMTP_USERNAME` | string | No | — | SMTP authentication username | `notifications@company.com` |
| `SMTP_PASSWORD` | string | No | — | SMTP password (sensitive) | `smtp-password` |
| `SMTP_FROM_EMAIL` | string | No | — | Default sender email | `cmp@company.com` |

### SSO / SAML Settings

| Key | Type | Required | Default | Description | Example |
|-----|------|----------|---------|-------------|---------|
| `SSO_ENABLED` | boolean | No | `false` | Enable SSO/SAML authentication | `true` |
| `SSO_ENTITY_ID` | string | No | — | SAML Service Provider entity ID | `https://cmp.company.com` |
| `SSO_METADATA_URL` | URL | No | — | IdP metadata URL | `https://idp.company.com/metadata` |
| `SSO_CERTIFICATE` | string | No | — | SAML signing certificate (base64) | `MIICpD...` |

### Cloud Provider Settings

| Key | Type | Required | Default | Description | Example |
|-----|------|----------|---------|-------------|---------|
| `AWS_ACCESS_KEY_ID` | string | No | — | AWS access key | `AKIA...` |
| `AWS_SECRET_ACCESS_KEY` | string | No | — | AWS secret key (sensitive) | `wJalr...` |
| `AZURE_CLIENT_ID` | string | No | — | Azure service principal client ID | `12345-abcde-...` |
| `AZURE_CLIENT_SECRET` | string | No | — | Azure client secret (sensitive) | `azure-secret` |
| `AZURE_TENANT_ID` | string | No | — | Azure AD tenant ID | `67890-fghij-...` |
| `GOOGLE_APPLICATION_CREDENTIALS` | string | No | — | Path to GCP service account JSON | `/app/secrets/gcp-sa.json` |

### AI Service Settings

| Key | Type | Required | Default | Description | Example |
|-----|------|----------|---------|-------------|---------|
| `GEMINI_API_KEY` | string | No | — | Google Gemini API key (sensitive) | `AIza...` |
| `AI_ENABLED` | boolean | No | `true` | Enable AI assistant features | `true` |

### Notification Settings

| Key | Type | Required | Default | Description | Example |
|-----|------|----------|---------|-------------|---------|
| `SLACK_WEBHOOK_URL` | URL | No | — | Slack webhook (sensitive) | `https://hooks.slack.com/...` |
| `TEAMS_WEBHOOK_URL` | URL | No | — | Microsoft Teams webhook (sensitive) | `https://outlook.office.com/...` |
| `PAGERDUTY_ROUTING_KEY` | string | No | — | PagerDuty routing key (sensitive) | `abc123...` |

### Resource Limits (Container Overrides)

| Key | Type | Required | Default | Description | Example |
|-----|------|----------|---------|-------------|---------|
| `BACKEND_CPU_LIMIT` | string | No | `2.0` | Backend CPU limit | `4.0` |
| `BACKEND_MEMORY_LIMIT` | string | No | `2g` | Backend memory limit | `4g` |
| `FRONTEND_CPU_LIMIT` | string | No | `1.0` | Frontend CPU limit | `2.0` |
| `FRONTEND_MEMORY_LIMIT` | string | No | `512m` | Frontend memory limit | `1g` |
| `WORKER_CPU_LIMIT` | string | No | `2.0` | Worker CPU limit | `4.0` |
| `WORKER_MEMORY_LIMIT` | string | No | `2g` | Worker memory limit | `4g` |
| `AI_CPU_LIMIT` | string | No | `1.0` | AI service CPU limit | `2.0` |
| `AI_MEMORY_LIMIT` | string | No | `1g` | AI service memory limit | `2g` |

---

## Encrypting Sensitive Values

CMP supports Fernet symmetric encryption for sensitive configuration values stored in files. This protects secrets at rest.

### How It Works

1. You provide a Fernet encryption key (`ENCRYPTION_KEY` in your `.env`)
2. Sensitive values in the encrypted config file are stored as Fernet-encrypted strings
3. CMP decrypts them at runtime using your key

### Encrypting a Value

```bash
# Generate your encryption key (do this once, store securely)
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
# Output: gAAAAABh... (this is your ENCRYPTION_KEY)

# Encrypt a sensitive value
python3 -c "
from cryptography.fernet import Fernet
key = b'YOUR_ENCRYPTION_KEY_HERE'
f = Fernet(key)
encrypted = f.encrypt(b'my-secret-password')
print(encrypted.decode())
"
```

### Using Encrypted Values in Configuration

In your encrypted config file, prefix encrypted values with `ENC:`:

```bash
# config/encrypted.env
SMTP_PASSWORD=ENC:gAAAAABhXYZ123...encrypted-value...
AWS_SECRET_ACCESS_KEY=ENC:gAAAAABhABC456...encrypted-value...
SLACK_WEBHOOK_URL=ENC:gAAAAABhDEF789...encrypted-value...
```

CMP automatically detects the `ENC:` prefix and decrypts the value using your `ENCRYPTION_KEY`.

### Key Management Best Practices

- Store the `ENCRYPTION_KEY` separately from encrypted values (e.g., as an environment variable or Docker secret)
- Never commit the encryption key to version control
- Rotate the encryption key periodically (re-encrypt all values with the new key)
- Keep a secure backup of the encryption key — if lost, encrypted values cannot be recovered

---

## Docker Secrets Setup

For enhanced security, CMP supports Docker secrets for sensitive values. Docker secrets are stored encrypted by Docker and mounted as files inside containers.

### Creating Docker Secrets

```bash
# Create secrets from values
echo "your-secret-key-value" | docker secret create cmp_secret_key -
echo "your-encryption-key" | docker secret create cmp_encryption_key -
echo "your-smtp-password" | docker secret create cmp_smtp_password -
```

Or from files:

```bash
docker secret create cmp_secret_key ./secrets/secret_key.txt
docker secret create cmp_encryption_key ./secrets/encryption_key.txt
```

### Docker Compose Configuration

In your `docker-compose.yml`, reference secrets:

```yaml
services:
  cmp-backend:
    image: <account_id>.dkr.ecr.<region>.amazonaws.com/cmp-backend:8.10.0
    secrets:
      - cmp_secret_key
      - cmp_encryption_key
      - cmp_smtp_password

secrets:
  cmp_secret_key:
    external: true
  cmp_encryption_key:
    external: true
  cmp_smtp_password:
    external: true
```

### How CMP Reads Docker Secrets

CMP automatically checks `/run/secrets/<secret_name>` for each sensitive configuration key. The mapping is:

| Config Key | Docker Secret Name |
|------------|-------------------|
| `SECRET_KEY` | `cmp_secret_key` |
| `ENCRYPTION_KEY` | `cmp_encryption_key` |
| `SMTP_PASSWORD` | `cmp_smtp_password` |
| `AWS_SECRET_ACCESS_KEY` | `cmp_aws_secret_key` |
| `AZURE_CLIENT_SECRET` | `cmp_azure_secret` |
| `GEMINI_API_KEY` | `cmp_gemini_key` |
| `REDIS_PASSWORD` | `cmp_redis_password` |

---

## Secret Rotation

CMP supports zero-downtime secret rotation. When you update a secret source, CMP detects the change and applies the new value within 60 seconds.

### Rotation Procedure

1. **Update the secret source** (Docker secret, encrypted file, or environment variable)
2. **Wait up to 60 seconds** for CMP to detect and apply the change
3. **Verify** the new secret is in use (e.g., test SMTP delivery, verify cloud connectivity)

### Rotating Docker Secrets

Docker secrets are immutable — you must create a new version:

```bash
# Create new secret version
echo "new-password" | docker secret create cmp_smtp_password_v2 -

# Update docker-compose.yml to reference new secret
# Then redeploy the service
docker compose up -d cmp-backend
```

### Rotating Encrypted Config Values

1. Encrypt the new value with your encryption key
2. Update the encrypted config file with the new value
3. CMP detects the file change within 60 seconds and applies it

### Rotating the Encryption Key Itself

This requires re-encrypting all values:

1. Generate a new encryption key
2. Decrypt all values with the old key
3. Re-encrypt all values with the new key
4. Update `ENCRYPTION_KEY` to the new key
5. Restart the application (encryption key rotation requires restart)

---

## SMTP Configuration

Configure email notifications for alerts, approvals, and user communications.

### Basic SMTP Setup

```bash
SMTP_HOST=smtp.company.com
SMTP_PORT=587
SMTP_USE_TLS=true
SMTP_USERNAME=cmp-notifications@company.com
SMTP_PASSWORD=your-smtp-password
SMTP_FROM_EMAIL=cmp@company.com
```

### Common SMTP Providers

| Provider | Host | Port | TLS |
|----------|------|------|-----|
| Office 365 | `smtp.office365.com` | 587 | Yes |
| Gmail (Workspace) | `smtp.gmail.com` | 587 | Yes |
| Amazon SES | `email-smtp.us-east-1.amazonaws.com` | 587 | Yes |
| SendGrid | `smtp.sendgrid.net` | 587 | Yes |

### Validation Rules

- `SMTP_PORT` must be between 1 and 65535
- If `SMTP_USE_TLS=true`, then `SMTP_HOST` must be configured (startup will fail otherwise)
- `SMTP_HOST` must be a valid hostname or IP address

### Testing SMTP

After configuration, send a test email from the CMP UI: **Settings → Notifications → Test Email**.

---

## SSO / SAML Configuration

Configure single sign-on for enterprise identity providers.

### Basic SSO Setup

```bash
SSO_ENABLED=true
SSO_ENTITY_ID=https://cmp.company.com
SSO_METADATA_URL=https://idp.company.com/app/metadata
SSO_CERTIFICATE=MIICpDCCAYwCCQD...base64-certificate...
```

### Identity Provider Setup

1. Register CMP as a Service Provider in your IdP
2. Set the ACS (Assertion Consumer Service) URL to: `https://cmp.company.com/api/v1/auth/saml/callback`
3. Set the Entity ID to match your `SSO_ENTITY_ID` value
4. Download the IdP metadata URL or certificate

### Supported Identity Providers

- Azure Active Directory (Entra ID)
- Okta
- OneLogin
- Google Workspace
- Any SAML 2.0 compliant IdP

---

## Cloud Provider Credentials

Configure credentials for managing cloud resources across AWS, Azure, and GCP.

### AWS Configuration

```bash
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_REGION=us-east-1
```

**Best Practice:** Use IAM roles with minimum required permissions. Create a dedicated IAM user for CMP with only the permissions needed for resource management.

### Azure Configuration

```bash
AZURE_CLIENT_ID=12345678-1234-1234-1234-123456789012
AZURE_CLIENT_SECRET=your-azure-client-secret
AZURE_TENANT_ID=87654321-4321-4321-4321-210987654321
AZURE_SUBSCRIPTION_ID=abcdefgh-abcd-abcd-abcd-abcdefghijkl
```

**Best Practice:** Create a dedicated Azure Service Principal with Contributor role scoped to specific resource groups.

### GCP Configuration

```bash
GOOGLE_APPLICATION_CREDENTIALS=/app/secrets/gcp-service-account.json
```

Mount the service account JSON file into the container via a volume or Docker secret.

---

## Redis Configuration

Redis is used for caching, session management, and background task queuing.

### Basic Redis Setup

```bash
REDIS_URL=redis://redis:6379/0
```

### Redis with Authentication

```bash
REDIS_URL=redis://:your-redis-password@redis:6379/0
# Or separately:
REDIS_PASSWORD=your-redis-password
```

### Redis with TLS

```bash
REDIS_URL=rediss://:password@redis:6380/0
```

Note the `rediss://` scheme (double 's') for TLS connections.

### Redis Behavior

- Redis is used for token blacklisting, rate limiting, and Celery task queuing
- If Redis is unavailable, CMP degrades gracefully (auth continues to work, but token blacklisting is disabled)
- Redis is considered a critical service for health monitoring

---

## Database Configuration

CMP uses DynamoDB as its primary database.

### Local DynamoDB (Included in Package)

```bash
DYNAMODB_ENDPOINT=http://database:8000
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=local
AWS_SECRET_ACCESS_KEY=local
```

### AWS DynamoDB

```bash
DYNAMODB_ENDPOINT=https://dynamodb.us-east-1.amazonaws.com
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=wJalr...
```

### Validation

- `DYNAMODB_ENDPOINT` must be a valid URL
- CMP validates connectivity to the endpoint at startup (10-second timeout)
- If unreachable, startup fails with a descriptive error

---

## Feature Flags and Toggles

CMP uses a two-layer feature gating system:

1. **License features:** Controlled by your license file (cannot be enabled beyond license)
2. **Admin toggles:** Controlled by administrators in the CMP UI

A feature is available to users only if it is **both** licensed AND enabled by admin toggle.

### Viewing Feature Status

```bash
# API endpoint (requires authentication)
curl -s -H "Authorization: Bearer <token>" \
  http://localhost:8001/api/v1/settings/feature-toggles/me | jq .
```

### Available Feature Keys

| Feature Key | Description |
|-------------|-------------|
| `catalog` | Self-service catalog |
| `workflows` | Workflow automation |
| `cost_management` | Cost tracking and optimization |
| `policy_governance` | Policy engine and compliance |
| `ai_assistant` | AI-powered assistant |
| `sso` | Single sign-on |
| `multi_tenancy` | Multi-tenant support |
| `budgets` | Budget management |
| `scheduled_jobs` | Scheduled automation |

### Enabling/Disabling Features

Administrators can toggle features in the CMP UI under **Settings → Feature Toggles**. Note that disabling a feature via admin toggle takes effect immediately, but enabling a feature only works if it's also included in your license.

---

## Deployment Model Selection

The `DEPLOYMENT_MODEL` environment variable determines how CMP operates in your environment.

### Valid Values

| Value | Description | Network | License Verification | Updates |
|-------|-------------|---------|---------------------|---------|
| `saas` | Vendor-managed SaaS | Full internet | Online (periodic) | Automatic |
| `hybrid` | Shared responsibility | Partial internet | Online (periodic) | Vendor-assisted |
| `on-premises` | Customer-managed | Optional internet | Offline | Manual |
| `air-gapped` | Fully isolated | No internet | Offline only | Manual (package) |

### Setting the Deployment Model

```bash
# In your .env file
DEPLOYMENT_MODEL=on-premises
```

### Important Rules

1. `DEPLOYMENT_MODEL` **must** match the `deployment_model` field in your license file
2. If missing or set to an invalid value, the application will not start
3. The error message will list valid values:
   ```
   FATAL: Invalid DEPLOYMENT_MODEL value: "production"
   Valid values: saas, hybrid, on-premises, air-gapped
   ```

### Behavior by Deployment Model

**`air-gapped`:**
- All outbound network calls disabled
- No license verification calls to vendor
- No update checks or telemetry
- All operations are fully offline

**`on-premises`:**
- License validation is offline
- Optional outbound connectivity for update notifications
- Manual upgrade process

**`hybrid`:**
- Vendor may have monitoring access
- Periodic license verification enabled
- Vendor-assisted upgrades available

**`saas`:**
- Full vendor management
- Periodic license verification with vendor server
- Automatic update notifications

---

## Configuration Validation

At startup, CMP validates all configuration values and reports specific errors:

| Validation | Error Message |
|------------|---------------|
| Malformed URL | `"DYNAMODB_ENDPOINT: invalid URL format"` |
| Invalid port | `"SMTP_PORT: value 99999 outside valid range 1-65535"` |
| Unreachable endpoint | `"REDIS_URL: connection failed within 10 seconds"` |
| Incompatible combination | `"SMTP_USE_TLS enabled but SMTP_HOST not configured"` |
| Missing required key | `"Required configuration missing: SECRET_KEY"` |
| Invalid deployment model | `"Invalid DEPLOYMENT_MODEL value"` |

All validation completes within 30 seconds of startup. If any validation fails, the application refuses to start and reports all errors (not just the first one).

---

## Getting Help

For configuration questions:

1. Check the deployment-model-specific template (`.env.template.<model>`) for recommended settings
2. Review the [Troubleshooting Guide](TROUBLESHOOTING.md) for common configuration errors
3. Contact Autonimbus support with your customer ID
