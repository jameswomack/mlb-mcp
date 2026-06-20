# Changelog

## Unreleased

### Features
- **Default season updated to 2026** — `get_team_roster` now defaults to the 2026 season in both the MCP server wrapper and the underlying statsapi tool.
- **New MCP tools** — Added `get_batting_stats_bref`, `get_bwar_bat`, and `get_bwar_pitch` endpoints for Baseball Reference batting stats and bWAR data.
- **Optimized bWAR batting payload** — `bwar_bat()` now fetches full data (`return_all=True`) but selects only essential columns (name, IDs, WAR components, salary) to keep response size manageable.

### Bug Fixes
- **Robust Baseball Savant CSV parsing** — Added `_fetch_savant_csv()` helper that strips BOM and falls back through Python CSV engine → `on_bad_lines='skip'` when pybaseball's C parser fails.
- **Exit-velo/barrel fallback** — Added `_safe_exitvelo_barrels()` wrapper that retries via direct HTTP fetch when pybaseball raises errors for batter and pitcher exit-velo/barrel leaderboards.
- **Pitch arsenal fallback** — `get_statcast_pitcher_pitch_arsenal` now falls back to direct Baseball Savant CSV fetch on pybaseball errors.
- **Typo corrections** — Fixed "pthe" → "the", "overlayed" → "overlaid", "leaguewide" → "league-wide" in docstrings; added `type: ignore` for `__signature__` assignment.
- **Cross-container HTTP reachability** — The Streamable HTTP transport now allows the `host.docker.internal` Host header (plus any extra `MCP_ALLOWED_HOSTS` / `MCP_ALLOWED_ORIGINS`) so the `mlb-projections` API running in Docker no longer receives `HTTP 421`; localhost-only DNS-rebinding protection is otherwise retained.
- **`get_standings` output validation** — `statsapi.standings_data()` returns integer division-id keys; they are now coerced to strings so FastMCP structured-output validation no longer fails with a `wrapperOutput` error.

### Infrastructure
- **Docker port mapping** — Changed `docker-compose.yml` from `8000:8000` to `12000:8081`.
- **CORS origins** — Added `http://localhost:3001` to FastAPI allowed origins for additional Next.js dev server support.
- **Auto-restart** — `docker-compose.yml` now sets `restart: unless-stopped` so the `baseball-mcp` container returns automatically after Docker Desktop or the host restarts.
- **Compose cleanup** — Removed the broken `./.env:/app/.env` bind mount (Docker auto-created it as an empty directory) and dropped the obsolete Compose `version` key.

### Development Tooling
- Added Poetry configuration (`poetry.toml`, `poetry.lock`) alongside existing uv setup.
- Pinned Python 3.12.7 (`.python-version`) and Node 22.16.0 (`mlb_stats_mcp_client/.nvmrc`).
- Added Pyright type-checker config (`pyrightconfig.json`) and PEP 561 `py.typed` marker.
- Added Claude Code local settings (`.claude/settings.local.json`).
- Added empty `prompts` module stub for future use.

### Documentation
- **README** — Documented running the server over Streamable HTTP with Docker, verification `curl` commands, and the new `PORT` / `MCP_ALLOWED_HOSTS` / `MCP_ALLOWED_ORIGINS` environment variables.
- **Repo skills** — Added `.ai/skills/mcp-serve` (operate and verify the dockerized server) and `.ai/skills/mcp-add-tool` (add/fix tools, structured-output gotchas) to speed up future sessions.
