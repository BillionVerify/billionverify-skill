---
name: billionverify
description: Verify email addresses using the BillionVerify API. Use when user wants to verify single emails, batch verify email lists, upload files for bulk verification, check credit balance, view verification history or statistics, or manage webhooks.
version: 1.1.0
allowed-tools: Bash
---

# BillionVerify API Skill

Call the [BillionVerify](https://billionverify.com/) API to verify email addresses — single, batch, or bulk file processing.

## Setup

API key must be set in environment variable `BILLIONVERIFY_API_KEY`.
Get your API key at: https://billionverify.com/auth/sign-in?next=/home/api-keys

Before the first call, check the key is available:

```bash
[ -n "$BILLIONVERIFY_API_KEY" ] && echo set || echo missing
```

If it is missing (for example in claude.ai, where skills have no environment variables):

1. Tell the user the key is not set and ask them to paste it. Warn them that a key pasted into a chat stays in the conversation history, and that the hosted MCP connector (`https://mcp.billionverify.com/mcp`, OAuth sign-in, no key needed) avoids this.
2. Once they provide it, export it in the same shell command as each request (`BILLIONVERIFY_API_KEY=... curl ...`). Never write it to a file and never echo it back.

If a request fails with a DNS, connection or proxy error (not an HTTP error), the sandbox is blocking outbound traffic. In claude.ai the user must allow `api.billionverify.com` under Settings → Capabilities (network egress / allowed domains); on Team / Enterprise an Owner has to add the domain.

## Base URL

```
https://api.billionverify.com
```

## Authentication

All requests require an API key header:
```bash
-H "BV-API-KEY: $BILLIONVERIFY_API_KEY"
```

## Endpoints

### Verify Single Email
```bash
curl -X POST "https://api.billionverify.com/v1/verify/single" \
  -H "BV-API-KEY: $BILLIONVERIFY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "check_smtp": true
  }'
```

Response `data` includes: `status` (valid/invalid/unknown/risky/disposable/catchall/role), `score` (0-1), `is_deliverable`, `is_disposable`, `is_catchall`, `is_role`, `is_free`, `domain`, `mx_records`, `domain_reputation`, `check_smtp`, `smtp_result`, `reason`, `verification_mode`, `retryable`, `response_time` (ms), `credits_used`. Some fields (e.g. `suggestion`, `domain_age`, `catch_all_evidence`) only appear when relevant.

### Verify Batch Emails (max 50)
```bash
curl -X POST "https://api.billionverify.com/v1/verify/bulk" \
  -H "BV-API-KEY: $BILLIONVERIFY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "emails": ["user1@example.com", "user2@example.com"],
    "check_smtp": true
  }'
```

### Upload File for Bulk Verification
Upload CSV, Excel (.xlsx/.xls), or TXT files (max 20MB, 100,000 emails):
```bash
curl -X POST "https://api.billionverify.com/v1/verify/file" \
  -H "BV-API-KEY: $BILLIONVERIFY_API_KEY" \
  -F "file=@/path/to/emails.csv" \
  -F "check_smtp=true" \
  -F "email_column=email" \
  -F "preserve_original=true"
```

Returns `task_id` for tracking the async job.

### Get File Job Status
Supports long-polling with `timeout` parameter (0-300 seconds):
```bash
curl -X GET "https://api.billionverify.com/v1/verify/file/{task_id}?timeout=30" \
  -H "BV-API-KEY: $BILLIONVERIFY_API_KEY"
```

Status values: `pending`, `processing`, `completed`, `failed`.

### Download Verification Results
Without filters returns redirect to full result file. With filters returns CSV of matching emails (filters combined with OR logic):
```bash
curl -X GET "https://api.billionverify.com/v1/verify/file/{task_id}/results?valid=true&invalid=true" \
  -H "BV-API-KEY: $BILLIONVERIFY_API_KEY" \
  -L -o results.csv
```

Filter parameters: `valid`, `invalid`, `catchall`, `role`, `unknown`, `disposable`, `risky`.

### Get Credit Balance
```bash
curl -X GET "https://api.billionverify.com/v1/credits" \
  -H "BV-API-KEY: $BILLIONVERIFY_API_KEY"
```

### Get Verification History
Paginated, newest first, covers every key and the web dashboard of the account. Optional filters: `email` (substring), `status`, `method` (`web` / `api`):
```bash
curl -X GET "https://api.billionverify.com/v1/usage?page=1&limit=20" \
  -H "BV-API-KEY: $BILLIONVERIFY_API_KEY"
```

`limit` is 1-100. Pagination is returned in the top-level `pagination` object (`page`, `limit`, `total`, `total_pages`).

### Get Verification Statistics
`period` is one of `7d` (default), `30d`, `90d`, `1y`:
```bash
curl -X GET "https://api.billionverify.com/v1/stats?period=30d" \
  -H "BV-API-KEY: $BILLIONVERIFY_API_KEY"
```

Returns `total_verified`, `valid_emails`, `invalid_emails`, `credits_used`, `average_score`.

### Create Webhook
```bash
curl -X POST "https://api.billionverify.com/v1/webhooks" \
  -H "BV-API-KEY: $BILLIONVERIFY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-app.com/webhooks/billionverify",
    "events": ["file.completed", "file.failed"]
  }'
```

The `secret` is only returned on creation — store it securely.

### List Webhooks
```bash
curl -X GET "https://api.billionverify.com/v1/webhooks" \
  -H "BV-API-KEY: $BILLIONVERIFY_API_KEY"
```

### Delete Webhook
```bash
curl -X DELETE "https://api.billionverify.com/v1/webhooks/{webhook_id}" \
  -H "BV-API-KEY: $BILLIONVERIFY_API_KEY"
```

### Health Check (no auth required)
```bash
curl -X GET "https://api.billionverify.com/health"
```

## Credits & Billing

- **Invalid** / **Unknown**: 0 credits (free)
- All other statuses (valid, risky, disposable, catchall, role): 1 credit each

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| Single Verification | 6,000/min |
| Batch Verification | 1,500/min |
| File Upload | 300/min |
| Other endpoints | 200/min |

## User Request

$ARGUMENTS
