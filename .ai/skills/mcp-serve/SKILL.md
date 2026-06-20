---
name: mcp-serve
description: Build, run, restart, and verify the mlb-mcp "baseball" MCP server (Docker, Streamable HTTP on port 12000). Use this whenever the MCP server won't start, isn't reachable, returns HTTP 421/406, needs a restart or rebuild after editing files under mlb_stats_mcp/, or when another service (e.g. the mlb-projections API) can't reach host.docker.internal:12000/mcp.
---

# Serve & verify the mlb-mcp server

This repo ships a FastMCP server (`mlb_stats_mcp/server.py`) exposing 50+ MLB
tools. In production-like setups it runs over **Streamable HTTP** in Docker; the
`mlb-projections` API consumes it at `http://localhost:12000/mcp` (local) or
`http://host.docker.internal:12000/mcp` (when that API runs in its own
container).

## Topology (know this before debugging)

- Compose service: `baseball-mcp` → container `mlb-mcp-baseball-mcp-1`.
- Port: host `12000` → container `8081`; MCP path is `/mcp`.
- Transport: `FastMCP("baseball", stateless_http=True)` → no session/initialize
  handshake needed; you can POST `tools/list` / `tools/call` directly.
- `docker-compose.yml` bind-mounts `./mlb_stats_mcp` into the container and sets
  `restart: unless-stopped`.

## Start / restart / rebuild

```bash
cd ~/Projects/github/etweisberg/mlb-mcp

# First run or after dependency/Dockerfile changes (bakes code into the image):
docker compose up -d --build

# After editing ONLY Python under mlb_stats_mcp/ (source is bind-mounted, so a
# restart re-imports it — no rebuild needed):
docker compose restart baseball-mcp

# Rebuild the image so a change is baked in (Smithery / fresh clones):
docker compose build baseball-mcp && docker compose up -d
```

Watch startup until you see `Uvicorn running on http://0.0.0.0:8081`:

```bash
docker logs -f mlb-mcp-baseball-mcp-1
```

## Verify it works

```bash
MCP=http://localhost:12000/mcp
CT='Content-Type: application/json'; ACC='Accept: application/json, text/event-stream'

# 1) protocol handshake — expect HTTP 200 and a long tool list
curl -sS -X POST "$MCP" -H "$CT" -H "$ACC" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'

# 2) a real tool call returning data
curl -sS -X POST "$MCP" -H "$CT" -H "$ACC" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"lookup_player","arguments":{"name":"Aaron Judge"}}}'
```

Responses are SSE; extract the JSON with `sed -n 's/^data: //p'`.

## Reachability & status-code cheat sheet

- `200` — healthy.
- `406 Not Acceptable` on `GET /mcp` — expected. The transport requires the
  `Accept: application/json, text/event-stream` header; it's a sign the server
  is up, not an error.
- `421 Misdirected Request` — the MCP SDK's DNS-rebinding protection rejected the
  `Host` header. Only `localhost`/`127.0.0.1`/`[::1]` and `host.docker.internal`
  are allowed by default. To allow more without code changes, set
  `MCP_ALLOWED_HOSTS` / `MCP_ALLOWED_ORIGINS` (comma-separated, `host:port`
  patterns support `*`) in the service `environment:`. The allow-list is built
  in `server.py` via `TransportSecuritySettings`.

Prove the cross-container path (what the dockerized API uses):

```bash
docker run --rm curlimages/curl:latest -sS -X POST \
  http://host.docker.internal:12000/mcp \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

## When the container "won't start"

Most "won't start" reports are environmental, not code:

1. **Container exited and didn't come back.** `docker ps -a` — if it shows
   `Exited (255)` right when Docker Desktop last restarted, the cause is a
   missing restart policy, which is already fixed (`restart: unless-stopped`).
   Just `docker compose up -d`.
2. **Docker Desktop is wedged / CLI hangs.** The Desktop processes exist but
   `docker ps` hangs because the daemon is mid-startup or stuck. macOS has no
   `timeout`; bound any docker call so it can't hang your shell:

   ```bash
   run_with_timeout() { local s="$1"; shift; "$@" & local p=$!; ( sleep "$s"; kill -9 "$p" 2>/dev/null ) & local k=$!; wait "$p" 2>/dev/null; local rc=$?; kill "$k" 2>/dev/null; return $rc; }
   run_with_timeout 12 docker ps -a
   ```

   If it stays stuck, fully quit and reopen Docker Desktop (or reinstall), then
   `docker compose up -d`.

## Gotchas

- Don't rely on `docker restart` for **dependency** changes — only source is
  bind-mounted; deps live in the image, so rebuild.
- There is no real `.env` file; configuration comes from the Compose
  `environment:` block. Don't re-add a `./.env:/app/.env` bind mount — Docker
  will recreate it as an empty directory.
