---
name: outline
description: Interact with the Outline wiki API — create, read, update, search documents and manage collections. Use when the user asks to write to Outline, create docs, search the wiki, manage collections, or when you need to store/retrieve collaborative documentation.
---

# Outline Wiki

Self-hosted Outline wiki at `https://dg726iuay4ymx.cloudfront.net`.

## Authentication

Retrieve the API key from AWS Secrets Manager:
```bash
aws secretsmanager get-secret-value --secret-id outline/api-key --region us-east-1 --query SecretString --output text
```

Use as Bearer token: `Authorization: Bearer <key>`

## API Basics

Base URL: `https://dg726iuay4ymx.cloudfront.net/api`

All endpoints are POST with JSON body. Example:
```bash
curl -s -X POST "$BASE/api/documents.list" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"collectionId": "<id>"}'
```

## Key Endpoints

### Documents
| Endpoint | Purpose | Key params |
|----------|---------|------------|
| `documents.create` | Create doc | `title`, `text` (markdown), `collectionId`, `publish: true` |
| `documents.info` | Get doc | `id` or `shareId` |
| `documents.update` | Update doc | `id`, `title`, `text`, `append: true` (to append) |
| `documents.delete` | Delete doc | `id` |
| `documents.search` | Search docs | `query`, `collectionId` (optional) |
| `documents.list` | List docs | `collectionId`, `sort`, `direction` |
| `documents.import` | Import file | multipart/form-data with `file`, `collectionId` |

### Collections
| Endpoint | Purpose | Key params |
|----------|---------|------------|
| `collections.list` | List all | — |
| `collections.create` | Create | `name`, `description`, `permission` (`read_write`) |
| `collections.info` | Get one | `id` |
| `collections.delete` | Delete | `id` |

### Other
| Endpoint | Purpose |
|----------|---------|
| `auth.info` | Verify token, get current user |
| `apiKeys.create` | Create new API key (`name`) |

## Python Pattern (recommended for multi-doc operations)

```python
import json, urllib.request, os

API_KEY = os.popen("aws secretsmanager get-secret-value --secret-id outline/api-key --region us-east-1 --query SecretString --output text").read().strip()
BASE = "https://dg726iuay4ymx.cloudfront.net"

def outline_api(endpoint, data={}):
    body = json.dumps(data).encode('utf-8')
    req = urllib.request.Request(
        f"{BASE}/api/{endpoint}",
        data=body,
        headers={"Authorization": f"Bearer {API_KEY}", "Content-Type": "application/json"}
    )
    return json.loads(urllib.request.urlopen(req, timeout=15).read())

# Create a doc
result = outline_api("documents.create", {
    "title": "My Doc",
    "text": "# Hello\nContent here",
    "collectionId": "<collection-id>",
    "publish": True
})
print(result["data"]["url"])
```

## Infrastructure

- **ECS Fargate ARM64** on `outline-cluster` / `outline-service`
- **Aurora PostgreSQL** (shared cluster `hedgedoc-aurora`, database `outline`)
- **S3** uploads: `outline-uploads-20260227091639722700000001`
- **CloudFront**: `ER6E3XD6I5VO4`
- **Cognito**: pool `us-east-1_XgaUUc0TG`, client `1vi1ltvj8obug8n41bknl5otq7`
- **Terraform**: `infra/outline/terraform/`

## Users

- **Roy** (`roy@osherove.com`) — admin
- **Loki** (`loki@openclaw.ai`) — member

## Tips

- Always use `publish: True` when creating docs (unpublished docs are drafts, not visible to others).
- Use `append: True` in `documents.update` to add content without replacing.
- The `text` field is standard Markdown.
- Doc URLs follow pattern: `https://dg726iuay4ymx.cloudfront.net/doc/<slug>-<urlId>`
- Health check: `GET /_health` (returns 200 when healthy).
