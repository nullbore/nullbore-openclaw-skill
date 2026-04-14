---
name: nullbore
description: Manage NullBore tunnels — expose local ports to the internet. Use when the user wants to create, list, close, or extend tunnels, check tunnel status, expose an MCP server or local service temporarily, or mentions NullBore or tunnel URLs.
---

# NullBore

Expose local services to the internet via on-demand HTTPS tunnels.

## Commands

Usage: `/nullbore <command> [args]`

| Command | Description |
|---------|-------------|
| `open <port>` | Expose a local port (default TTL: 1h) |
| `open <port> --ttl 4h` | Expose with custom TTL |
| `open <port> --name myapp` | Expose with a named subdomain (Dev+) |
| `open <port> --idle` | Idle timeout mode — TTL resets on traffic |
| `list` | List active tunnels |
| `close <id>` | Close a tunnel by ID |
| `extend <id> --ttl 2h` | Extend a tunnel's TTL |
| `status` | Health check the NullBore server |
| `cleanup` | Close all active tunnels |

## Examples

```
/nullbore open 3000
/nullbore open 8080 --ttl 30m --name api
/nullbore list
/nullbore close abc123
/nullbore cleanup
```

## Prerequisites

Requires environment variables:
- `NULLBORE_SERVER` — server URL (default: `https://tunnel.nullbore.com`)
- `NULLBORE_API_KEY` — API key (prefix `nbk_`)

## Implementation

### open — IMPORTANT: uses the client binary, not just the API

The `open` command MUST use the `nullbore` client binary, not a raw API call.
The REST API (`POST /v1/tunnels`) only registers the tunnel on the server —
it does NOT establish the WebSocket relay connection needed to actually forward
traffic. Without the client binary running, the tunnel URL will return 502.

**Step 1: Ensure the client binary is installed**

```bash
# Check if nullbore is already installed
which nullbore || {
  # Install it (auto-detects OS/arch, installs to ~/.local/bin)
  curl -fsSL https://nullbore.com/install.sh | sh
  export PATH="$HOME/.local/bin:$PATH"
}
```

**Step 2: Open the tunnel using the client binary**

```bash
# Basic — expose a port with default 1h TTL
nullbore open --port PORT

# With options
nullbore open --port PORT --ttl TTL --name NAME

# With idle timeout (TTL resets on each request)
nullbore open --port PORT --ttl 1h --idle-ttl
```

The client binary:
1. Calls the REST API to register the tunnel
2. Connects a WebSocket control channel to the server
3. Relays incoming HTTP traffic from the public URL to localhost:PORT
4. Runs in the foreground until Ctrl+C or the TTL expires

**Run it in the background** so it stays alive during the session:

```bash
nohup nullbore open --port PORT --ttl TTL > /tmp/nullbore-tunnel.log 2>&1 &
echo "Tunnel PID: $!"
# Wait a moment for it to connect, then check the log for the public URL
sleep 2 && cat /tmp/nullbore-tunnel.log
```

**Important notes:**
- The binary installs to `~/.local/bin/nullbore` and will NOT persist across
  container restarts. Re-run the install step if the container was restarted.
- Do NOT use the REST API (`curl POST /v1/tunnels`) alone for opening tunnels.
  It only registers the tunnel on the server without establishing the WebSocket
  relay — the URL will return 502 "tunnel client unavailable".

**Exposing a service on a Docker network** (e.g., grampsweb:5000):

The nullbore client connects to localhost:PORT. If the target service is on
a Docker network (not localhost), you need a local TCP proxy.

**Use a Node.js proxy** (socat is blocked in OpenClaw containers):

```bash
node -e "
const net = require('net');
const srv = net.createServer(c => {
  const up = net.connect({host: 'CONTAINER_HOST', port: SERVICE_PORT});
  c.pipe(up); up.pipe(c);
  c.on('error', () => up.destroy());
  up.on('error', () => c.destroy());
});
srv.listen(PROXY_PORT, '127.0.0.1', () => console.log('proxy :PROXY_PORT → CONTAINER_HOST:SERVICE_PORT'));
" &
# Then expose the proxy port
nullbore open --port PROXY_PORT
```

Replace `CONTAINER_HOST`, `SERVICE_PORT`, and `PROXY_PORT` with actual values.
For example, to expose grampsweb:5000 via a proxy on port 5050:

```bash
node -e "
const net = require('net');
const srv = net.createServer(c => {
  const up = net.connect({host: 'grampsweb', port: 5000});
  c.pipe(up); up.pipe(c);
  c.on('error', () => up.destroy());
  up.on('error', () => c.destroy());
});
srv.listen(5050, '127.0.0.1', () => console.log('proxy :5050 → grampsweb:5000'));
" &
nullbore open --port 5050
```

### list

```bash
curl -s "${NULLBORE_SERVER:-https://tunnel.nullbore.com}/v1/tunnels" \
  -H "Authorization: Bearer $NULLBORE_API_KEY"
```

### close

```bash
curl -s -X DELETE "${NULLBORE_SERVER:-https://tunnel.nullbore.com}/v1/tunnels/{id}" \
  -H "Authorization: Bearer $NULLBORE_API_KEY"
```

### extend

```bash
curl -s -X POST "${NULLBORE_SERVER:-https://tunnel.nullbore.com}/v1/tunnels/{id}/extend" \
  -H "Authorization: Bearer $NULLBORE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"ttl": "TTL"}'
```

### status

```bash
curl -s "${NULLBORE_SERVER:-https://tunnel.nullbore.com}/health"
```

### cleanup

```bash
# Close all tunnels via API
curl -s "${NULLBORE_SERVER:-https://tunnel.nullbore.com}/v1/tunnels" \
  -H "Authorization: Bearer $NULLBORE_API_KEY" | \
  jq -r '.[].id' | while read id; do
    curl -s -X DELETE "${NULLBORE_SERVER:-https://tunnel.nullbore.com}/v1/tunnels/$id" \
      -H "Authorization: Bearer $NULLBORE_API_KEY"
  done

# Also kill any local nullbore processes
pkill -x nullbore 2>/dev/null || true
```

## Docker Sidecar (persistent tunnels for docker-compose projects)

For services running in Docker, the recommended approach is a **sidecar container**
rather than installing the binary inside another container. The sidecar runs
alongside your services on the same Docker network and handles all tunnel
management. It persists across restarts, resolves container names via Docker
DNS, and doesn't require modifying your existing containers.

**Image:** `ghcr.io/nullbore/tunnel:latest` (linux/amd64 + linux/arm64)

### Add to any docker-compose.yml

```yaml
services:
  # Your existing service (no ports needed)
  webapp:
    image: your-app
    # ...

  # NullBore sidecar — exposes services through tunnels
  nullbore-tunnel:
    image: ghcr.io/nullbore/tunnel:latest
    environment:
      - NULLBORE_API_KEY=${NULLBORE_API_KEY}
      - NULLBORE_SERVER=${NULLBORE_SERVER:-https://tunnel.nullbore.com}
      - NULLBORE_TUNNELS=webapp:3000
    depends_on:
      - webapp
    restart: unless-stopped
```

### NULLBORE_TUNNELS format

Comma-separated list: `host:port` or `host:port:name`

```bash
# Single service (random slug, works on free tier)
NULLBORE_TUNNELS=webapp:3000

# Named tunnel (requires Dev+ with claimed subdomain)
NULLBORE_TUNNELS=webapp:3000:my-app

# Multiple services
NULLBORE_TUNNELS=api:8080:api,frontend:3000:app,admin:8081:admin
```

- `host` = the Docker service name (resolved via Docker's internal DNS)
- `port` = the port inside that container
- `name` = optional tunnel name (becomes the subdomain slug)

### Config file mode (for TTL, auth, idle timeout)

Mount a config file for more control:

```yaml
  nullbore-tunnel:
    image: ghcr.io/nullbore/tunnel:latest
    environment:
      - NULLBORE_API_KEY=${NULLBORE_API_KEY}
    volumes:
      - ./nullbore.toml:/home/nullbore/.config/nullbore/config.toml:ro
```

```toml
# nullbore.toml
server = "https://tunnel.nullbore.com"

[[tunnels]]
name = "api"
host = "webapp"
port = 3000
ttl  = "8h"
auth = "demo:s3cret"

[[tunnels]]
name = "docs"
host = "docs-server"
port = 4000
```

### Dashboard-driven mode

Omit `NULLBORE_TUNNELS` and the container runs in daemon mode, managing
tunnels configured in the NullBore dashboard UI:

```yaml
  nullbore-tunnel:
    image: ghcr.io/nullbore/tunnel:latest
    environment:
      - NULLBORE_API_KEY=${NULLBORE_API_KEY}
```

### When to use sidecar vs /nullbore open

| Approach | Use case |
|----------|----------|
| **Sidecar container** | Persistent tunnels for services in docker-compose. Survives restarts. Production-grade. |
| **`/nullbore open`** | Ad-hoc, temporary tunnels. Quick demos. Doesn't persist. |

## Documentation

- Docker integration guide: https://nullbore.com/docs/integrations/docker.html
- CI/CD previews: https://nullbore.com/docs/use-cases/cicd-previews.html
- Bot/agent exposure: https://nullbore.com/docs/use-cases/bot-exposure.html
- Remote home services: https://nullbore.com/docs/use-cases/remote-home.html
- Webhook testing: https://nullbore.com/docs/integrations/webhooks.html
- MCP server exposure: https://nullbore.com/docs/integrations/mcp.html
- OpenClaw skill: https://nullbore.com/docs/integrations/openclaw.html
- Self-hosting: https://nullbore.com/docs/self-hosting.html
- Full API reference: [references/api.md](references/api.md)
