---
name: nullbore-openclaw-skill
description: Manage NullBore tunnels via the REST API. Use when the user wants to expose a local port to the internet, create/list/close tunnels, check tunnel status, or expose an MCP server or local service temporarily. Also use when the user mentions NullBore, tunnel URLs, or needs a public URL for a local service.
---

# NullBore Tunnels

Manage on-demand HTTPS tunnels via the NullBore API.

## Prerequisites

Requires two environment variables:
- `NULLBORE_SERVER` — server URL (default: `https://tunnel.nullbore.com`)
- `NULLBORE_API_KEY` — API key (prefix `nbk_`)

## Core Operations

### Open a tunnel

```bash
curl -s -X POST "${NULLBORE_SERVER:-https://tunnel.nullbore.com}/v1/tunnels" \
  -H "Authorization: Bearer $NULLBORE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"local_port": PORT, "ttl": "1h"}'
```

Response includes `slug` — the tunnel URL is `https://{slug}.tunnel.nullbore.com`.

Optional fields:
- `name` (string) — custom subdomain (Hobby+ plans)
- `ttl` (string) — Go duration: `30m`, `4h`, `168h`. Default `1h`.
- `idle_ttl` (bool) — if true, TTL resets on traffic (idle timeout mode)

### List tunnels

```bash
curl -s "${NULLBORE_SERVER:-https://tunnel.nullbore.com}/v1/tunnels" \
  -H "Authorization: Bearer $NULLBORE_API_KEY"
```

### Close a tunnel

```bash
curl -s -X DELETE "${NULLBORE_SERVER:-https://tunnel.nullbore.com}/v1/tunnels/{id}" \
  -H "Authorization: Bearer $NULLBORE_API_KEY"
```

### Extend a tunnel

```bash
curl -s -X PUT "${NULLBORE_SERVER:-https://tunnel.nullbore.com}/v1/tunnels/{id}/extend" \
  -H "Authorization: Bearer $NULLBORE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"ttl": "4h"}'
```

### Health check (no auth)

```bash
curl -s "${NULLBORE_SERVER:-https://tunnel.nullbore.com}/health"
```

## Patterns

### Expose an MCP server

Open with idle TTL so it stays alive during agent work:

```bash
curl -s -X POST "$NULLBORE_SERVER/v1/tunnels" \
  -H "Authorization: Bearer $NULLBORE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"local_port": 4000, "ttl": "1h", "idle_ttl": true}'
```

### Get the public URL from response

```bash
RESP=$(curl -s -X POST "$NULLBORE_SERVER/v1/tunnels" ...)
SLUG=$(echo "$RESP" | jq -r .slug)
URL="https://${SLUG}.tunnel.nullbore.com"
```

### Clean up all tunnels

```bash
TUNNELS=$(curl -s "$NULLBORE_SERVER/v1/tunnels" -H "Authorization: Bearer $NULLBORE_API_KEY")
echo "$TUNNELS" | jq -r '.tunnels[].id' | while read id; do
  curl -s -X DELETE "$NULLBORE_SERVER/v1/tunnels/$id" -H "Authorization: Bearer $NULLBORE_API_KEY"
done
```

## Reference

For full API docs: [references/api.md](references/api.md)
