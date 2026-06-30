# AI Knowledge Sources — Admin Guide

## Overview

AI Knowledge Sources allow administrators to configure external documentation, wikis, and custom content that the AI assistant uses as additional context when answering questions. This makes the copilot organization-specific — it knows your internal standards, runbooks, architecture decisions, and policies.

## How It Works

1. **Admin configures sources** — Add URLs (Confluence, GitHub, websites) or paste plain text
2. **Admin triggers sync** — Click the refresh button to fetch and cache content
3. **AI uses context at query time** — When a user asks a question, relevant cached content is injected into the AI prompt
4. **Admin refreshes when needed** — Content stays cached until the admin manually syncs again

## Accessing the Configuration

Navigate to: **Administration → Settings → AI Knowledge Sources**

## Source Types

| Type | Description | Authentication |
|------|-------------|---------------|
| **Website** | Any public or private web page (HTML content extracted) | None, Bearer, API Key |
| **Confluence** | Atlassian Confluence pages via REST API | Bearer Token (PAT) |
| **GitHub** | GitHub files or pages (raw content) | None (public) or Bearer (PAT) |
| **Custom API** | Any URL returning text or JSON | Bearer, API Key |
| **Plain Text** | Direct text input (no URL needed) | None |

## Adding a Knowledge Source

### Website / URL

1. Click **Add Knowledge Source**
2. Enter a name (e.g., "Cloud Architecture Standards")
3. Select type: **Website / URL**
4. Enter the URL
5. If the page requires authentication, select the auth type and provide the token
6. Add tags for relevance matching (e.g., "architecture, standards, naming")
7. Click **Add Source**

### Confluence

1. Select type: **Confluence**
2. Enter the Confluence REST API URL:
   ```
   https://your-domain.atlassian.net/wiki/rest/api/content/{page-id}?expand=body.storage
   ```
3. Authentication: **Bearer Token** with your Confluence Personal Access Token
4. Add relevant tags

### GitHub

1. Select type: **GitHub**
2. Enter the raw file URL:
   ```
   https://raw.githubusercontent.com/org/repo/main/docs/standards.md
   ```
3. For private repos: **Bearer Token** with a GitHub PAT

### Plain Text

1. Select type: **Plain Text**
2. Paste your content directly (naming conventions, internal policies, etc.)
3. No URL needed

## Configuration Options

| Field | Description | Default |
|-------|-------------|---------|
| **Name** | Display name for the source | Required |
| **URL** | Source URL to fetch content from | Required (except plain text) |
| **Type** | Source type (website, confluence, github, custom_api, plain_text) | website |
| **Authentication** | Auth method for protected sources | none |
| **Tags** | Comma-separated keywords for relevance matching | Empty |
| **Description** | Brief description of what the source contains | Optional |
| **Max Characters** | Maximum content to cache per source | 8,000 |

## How Relevance Matching Works

When a user asks a question, the system:

1. Checks all enabled sources
2. Scores each source by matching:
   - Source tags against the user's message keywords
   - Source tags against the current page category
   - Source name/description against message words
3. Selects the top 3 most relevant sources
4. Injects their cached content into the AI prompt (max 20KB total)

**Tip:** Use specific, descriptive tags to improve relevance matching. For example:
- A naming conventions doc → tags: `naming, conventions, standards, resources`
- An architecture guide → tags: `architecture, design, patterns, infrastructure`
- A cost policy → tags: `cost, budget, spending, limits`

## Managing Sources

### Sync (Refresh)

- Click the **refresh icon** on any source to manually fetch the latest content
- Use **Sync All** to refresh all enabled sources at once
- Content stays cached indefinitely until you refresh — no automatic background fetching
- After updating your internal docs, come here and click refresh to update the AI's knowledge

### Enable/Disable

- Toggle sources on/off without deleting them
- Disabled sources are not included in AI responses

### Delete

- Removes the source configuration and its cached content

## Security

- **Auth tokens are encrypted** at rest using Fernet encryption (same as cloud credentials)
- **Tokens are never exposed** in API responses — only `has_auth_token: true/false` is returned
- **Content is fetched server-side** — never exposed to the frontend
- **Admin-only access** — only users with the `admin` role can configure sources

## Limits

| Limit | Value |
|-------|-------|
| Max sources | No hard limit (recommended: 10-20) |
| Max content per source | 50,000 characters |
| Max total injected per query | 20,000 characters |
| Max sources injected per query | 3 |
| Fetch timeout | 15 seconds |

## Best Practices

1. **Keep sources focused** — One topic per source works better than one large document
2. **Use descriptive tags** — More tags = better relevance matching
3. **Refresh after doc updates** — When your internal docs change, click refresh to update the AI
4. **Test with the AI** — After adding a source, ask the AI a related question to verify it uses the content
5. **Monitor sync errors** — Check the source list for red error messages
6. **Limit total sources** — 10-20 well-tagged sources perform better than 50 untagged ones

## Example Use Cases

| Use Case | Source Type | Tags |
|----------|------------|------|
| Company naming conventions | Plain Text | naming, conventions, standards |
| Cloud architecture decisions | Confluence | architecture, design, cloud |
| Terraform module documentation | GitHub | terraform, modules, infrastructure |
| Cost policies and limits | Website | cost, budget, policy, limits |
| Onboarding runbook | Confluence | onboarding, setup, getting-started |
| Approved instance types | Plain Text | instances, sizing, approved, types |
