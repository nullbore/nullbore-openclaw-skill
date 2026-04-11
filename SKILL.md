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
| `open <port> --name myapp` | Expose with a named subdomain (Hobby+) |
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

**Exposing a service on a Docker network** (e.g., grampsweb:5000):

The nullbore client connects to localhost:PORT. If the target service is on
a Docker network (not localhost), you need a local proxy:

```bash
# Start a TCP proxy from localhost:PROXY_PORT to container:SERVICE_PORT
socat TCP-LISTEN:PROXY_PORT,fork,reuseaddr TCP:CONTAINER_HOST:SERVICE_PORT &
# Then expose the proxy port
nullbore open --port PROXY_PORT
```

Or if `socat` is not available, use a simple netcat relay or install socat:
```bash
apt-get install -y socat 2>/dev/null || apk add socat 2>/dev/null
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

## Reference

For full API docs: [references/api.md](references/api.md)
