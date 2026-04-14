# NullBore API Reference

Base URL: `${NULLBORE_SERVER}/v1` (default: `https://tunnel.nullbore.com/v1`)

Auth: `Authorization: Bearer $NULLBORE_API_KEY`

## POST /v1/tunnels — Create tunnel

Request:
```json
{
  "local_port": 3000,        // required, 1-65535
  "name": "myapp",           // optional, 2-63 chars, lowercase alnum + hyphens
  "ttl": "1h",               // optional, Go duration, default "1h"
  "idle_ttl": true            // optional, resets TTL on activity
}
```

Response (201):
```json
{
  "id": "uuid",
  "slug": "myapp",           // or random hex like "a7f3bc"
  "client_id": "client-1",
  "local_port": 3000,
  "name": "myapp",
  "ttl": "1h0m0s",
  "mode": "relay",
  "created_at": "2026-03-30T14:30:00Z",
  "expires_at": "2026-03-30T15:30:00Z",
  "idle_ttl": true,
  "bytes_in": 0,
  "bytes_out": 0,
  "requests": 0
}
```

Tunnel URL: `https://{slug}.tunnel.nullbore.com` or `https://tunnel.nullbore.com/t/{slug}`

Errors: 400 (bad request), 409 (name taken), 429 (rate limit: 10 creates/min)

## GET /v1/tunnels — List tunnels

Response: `{"tunnels": [...]}`

## GET /v1/tunnels/{id} — Get tunnel

Response: tunnel object (same as create response)

## DELETE /v1/tunnels/{id} — Close tunnel

Response: `{"status": "closed"}`

## PUT /v1/tunnels/{id}/extend — Extend TTL

Request: `{"ttl": "4h"}`

## GET /health — Health check (no auth)

Response: `{"status": "ok"}`

## TTL Limits by Plan

| Plan | Max TTL | Idle TTL |
|------|---------|----------|
| Free | 2h | 2h |
| Dev   | 7 days | 7 days |
| Pro | Unlimited | Unlimited |

## Name Rules

- 2-63 characters
- Lowercase alphanumeric + hyphens
- Must start/end with alphanumeric
- Reserved names blocked: www, api, admin, dashboard, app, etc.
