# MLB Stats MCP Server

[![Tests](https://github.com/etweisberg/mcp-baseball-stats/actions/workflows/test.yml/badge.svg)](https://github.com/etweisberg/baseball/mcp-baseball-stats/workflows/test.yml)
[![Pre-commit](https://github.com/etweisberg/mcp-baseball-stats/actions/workflows/pre-commit.yml/badge.svg)](https://github.com/etweisberg/mcp-baseball-stats/actions/workflows/pre-commit.yml)
[![smithery badge](https://smithery.ai/badge/@etweisberg/mlb-mcp)](https://smithery.ai/server/@etweisberg/mlb-mcp)

A Python project that creates a Model Context Protocol (MCP) server for accessing MLB statistics data through the MLB Stats API and `pybaseball` library for statcast, fangraphs, and baseball reference statistics. This server provides structured API access to baseball statistics that can be used with MCP-compatible clients.

## Quick start (Docker)

The server's primary runtime is **Docker over Streamable HTTP**. Build, start, and verify:

```bash
docker compose up -d --build            # build image + start → http://localhost:12000/mcp
docker compose logs -f baseball-mcp     # wait for "Uvicorn running on http://0.0.0.0:8081"
```

Confirm it is serving tools (prints the HTTP status code and the parsed JSON payload):

```bash
curl -sS -X POST http://localhost:12000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -w '\nHTTP %{http_code}\n' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' \
  | sed -n -e 's/^data: //p' -e '/^HTTP /p'
```

See [Running with Docker](#running-with-docker-http-transport) for the full command reference, verification, and troubleshooting.

## Project Structure

- `mlb_stats_mcp/` - Main package directory
  - `server.py` - Core MCP server implementation
  - `tools/` - MCP tool implementations
    - `mlb_statsapi_tools.py` - MLB StatsAPI tool definitions
    - `statcast_tools.py` - Statcast data tool definitions
    - `pybaseball_plotting_tools.py` - Additional `pybaseball` tools provided for generating matplotlib plots and returning base64 encoded images
    - `pybaseball_supp_tools.py` - Supplemental `pybaseball` functions for interfacing with fangraphs, baseball reference, and other data sources
  - `utils/` - Utility modules
    - `logging_config.py` - Logging configuration
    - `images.py` - functions related to handling plot images
  - `tests/` - Test suite for verifying server functionality
- `pyproject.toml` - Project configuration and dependencies
- `.pre-commit-config.yaml` - Pre-commit hooks configuration
- `.github/` - GitHub Actions workflows

## Tools

The server exposes 50+ MCP tools spanning the MLB Stats API, Statcast,
`pybaseball` plotting, and supplemental `pybaseball` data sources. Call the MCP
`tools/list` method (see [Running with Docker](#running-with-docker-http-transport)
below) for the authoritative list, or browse the implementations under
`mlb_stats_mcp/tools/`.

## Setup

1. Install uv if you haven't already:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

1. Create and activate a virtual environment:

```bash
uv venv
source .venv/bin/activate  # On Unix/macOS
# or
.venv\Scripts\activate  # On Windows
```

1. Install dependencies:

```bash
uv pip install -e .
```

### Installing via Smithery

To install MLB Stats Server for Claude Desktop automatically via [Smithery](https://smithery.ai/server/@etweisberg/mlb-mcp):

```bash
npx -y @smithery/cli install @etweisberg/mlb-mcp --client claude
```

### Running Tests

The project includes comprehensive pytest tests for the MCP server functionality:

```bash
uv run pytest -v
```

Tests verify all MLB StatsAPI tools work correctly with the MCP protocol, establishing connections, making API calls, and processing responses.

## Running with Docker (HTTP transport)

In addition to stdio, the server can run over **Streamable HTTP**, which is how
non-stdio clients (for example the `mlb-projections` API) consume it, via the
provided `docker-compose.yml`.

### Common commands

```bash
docker compose up -d --build          # build image + start in the background
docker compose build baseball-mcp     # rebuild the image (dependency/Dockerfile changes)
docker compose restart baseball-mcp   # reload Python edits (source is bind-mounted)
docker compose logs -f baseball-mcp   # follow logs ("Uvicorn running on http://0.0.0.0:8081")
docker compose down                   # stop and remove the container
```

Notes on the Compose service:

- It publishes the in-container port `8081` on host port `12000` (endpoint `/mcp`).
- `restart: unless-stopped` makes the container return automatically after
  Docker Desktop or the host restarts.
- `./mlb_stats_mcp` is bind-mounted, so `docker compose restart baseball-mcp`
  picks up Python edits; `docker compose build` is only needed for dependency
  changes or to bake a change into the image itself (e.g. for Smithery).

> If a `docker` command hangs, Docker Desktop is probably still starting or
> wedged — wait or restart Docker Desktop, then retry. (Non-interactive
> scripts/agents that cannot Ctrl-C should wrap calls in a timeout; note that
> macOS has no `timeout` built in.)

### Verifying the server

Responses are JSON-RPC over Server-Sent Events, so print the status code and pipe
through `sed` to read the JSON payload (the `-e '/^HTTP /p'` keeps the status
line, which the `data:` filter would otherwise drop):

```bash
curl -sS -X POST http://localhost:12000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -w '\nHTTP %{http_code}\n' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' \
  | sed -n -e 's/^data: //p' -e '/^HTTP /p'
```

Call a real tool to confirm live data end-to-end:

```bash
curl -sS -X POST http://localhost:12000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -w '\nHTTP %{http_code}\n' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_standings","arguments":{"season":2024,"league_id":104}}}' \
  | sed -n -e 's/^data: //p' -e '/^HTTP /p'
```

Expect `HTTP 200` and a JSON-RPC `result`. A plain `GET /mcp` returns
`406 Not Acceptable` — this is expected, because the Streamable HTTP transport
requires the `Accept: application/json, text/event-stream` header.

### Reaching the server from another container

The HTTP transport enables the MCP SDK's DNS-rebinding protection, which by
default only allows `localhost` Host headers. When another container (such as
the `mlb-projections` API in its own Compose stack) connects via
`host.docker.internal:12000`, that host is allowed out of the box. To permit
additional hosts/origins without code changes, set `MCP_ALLOWED_HOSTS` and/or
`MCP_ALLOWED_ORIGINS` (see [Environment Variables](#environment-variables)).

## Environment Variables

The project uses environment variables stored in `.env` to configure settings.

Use `ANTHROPIC_API_KEY` to enable MCP Server.

### Logging Configuration

The MLB Stats MCP Server supports configurable logging via environment variables:

- `MLB_STATS_LOG_LEVEL` - Sets the logging level (DEBUG, INFO, WARNING, ERROR, CRITICAL)
- `MLB_STATS_LOG_FILE` - Path to log file (if not set, logs to stdout)

### HTTP Transport

These apply when running over Streamable HTTP (`python -m mlb_stats_mcp.server --http`,
which is what the Docker image runs):

- `PORT` - Port the HTTP server binds to inside the process (default `8081`).
- `MCP_ALLOWED_HOSTS` - Comma-separated extra `Host` header values permitted by
  the transport's DNS-rebinding protection (e.g. `host.docker.internal:*`).
  The localhost variants and `host.docker.internal` are allowed by default;
  `host:port` patterns support `*` (e.g. `mcp.internal:*`).
- `MCP_ALLOWED_ORIGINS` - Comma-separated extra `Origin` header values permitted.

## Claude Desktop Integration

To connect this MCP server to Claude Desktop, add a configuration to your `claude_desktop_config.json` file. Here's a template configuration:

```json
"mcp-baseball-stats": {
  "command": "{PATH_TO_UV}",
  "args": [
    "--directory",
    "{PROJECT_DIRECTORY}",
    "run",
    "python",
    "-m",
    "mlb_stats_mcp.server"
  ],
  "env": {
    "MLB_STATS_LOG_FILE": "{LOG_FILE_PATH}",
    "MLB_STATS_LOG_LEVEL": "DEBUG"
  }
}
```

Replace the following placeholders:

- `{PATH_TO_UV}`: Path to your uv installation (e.g., `~/.local/bin/uv`)
- `{PROJECT_DIRECTORY}`: Path to your project directory
- `{LOG_FILE_PATH}`: Path where you want to store the log file

## Technologies Used

- `mcp[cli]` - Machine-Learning Chat Protocol for tool definition
- `mlb-statsapi` - Python wrapper for the MLB Stats API
- `httpx` - HTTP client for making API requests
- `pytest` and `pytest-asyncio` - Test frameworks
- `uv` - Fast Python package manager and installer

## Linting

This project uses [Ruff](https://github.com/astral-sh/ruff) for linting and code formatting, with pre-commit hooks to ensure code quality.

### Setup Pre-commit Hooks

1. Install pre-commit
   - `pip install pre-commit`
2. Initialize pre-commit hooks:
   - `pre-commit install`

Now, the linting checks will run automatically whenever you commit code. You can also run them manually:

```bash
pre-commit run --all-files
```

### Linting Configuration

Linting rules are configured in the `pyproject.toml` file under the `[tool.ruff]` section. The project follows PEP 8 style guidelines with some customizations.

### CI Integration

GitHub Actions workflows automatically run tests, linting, and pre-commit checks on all pull requests and pushes to the main branch.
