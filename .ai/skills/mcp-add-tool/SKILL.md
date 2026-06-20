---
name: mcp-add-tool
description: Add, modify, or fix a tool in the mlb-mcp "baseball" MCP server. Use whenever you edit files under mlb_stats_mcp/tools/, register a new @mcp_tool_wrapper tool in server.py, or debug a FastMCP structured-output / "wrapperOutput" validation error (e.g. integer dict keys, non-JSON-serializable return values, a tool returning isError unexpectedly).
---

# Add or fix an mlb-mcp tool

## How tools are wired

- Implementation lives in `mlb_stats_mcp/tools/*.py` as a plain `async def`
  returning `Dict[str, Any]` (JSON-serializable).
- `server.py` exposes it with a thin wrapper:

  ```python
  @mcp_tool_wrapper
  async def get_thing(arg: int) -> Dict[str, Any]:
      """One-line summary shown to the model, then details/args."""
      return await some_tools_module.get_thing(arg)
  ```

- `mcp_tool_wrapper` (in `server.py`) registers the function via
  `mcp.tool(name=func.__name__, description=func.__doc__)`. So **the function
  name becomes the tool name and the docstring becomes the tool description** —
  write both deliberately.

## Adding a tool

1. Implement the logic in the appropriate `mlb_stats_mcp/tools/` module,
   returning a JSON-serializable `Dict[str, Any]`. Wrap the body in try/except
   and re-raise with context (match the existing modules).
2. Add a `@mcp_tool_wrapper` wrapper in `server.py` that delegates to it, with a
   clear docstring (this is what the model reads to decide when/how to call it).
3. Restart or rebuild (see the `mcp-serve` skill) and verify with a
   `tools/call` curl. Add/extend a test under `mlb_stats_mcp/tests/`.

## The #1 gotcha: structured-output validation ("wrapperOutput")

FastMCP (mcp SDK ≥ 1.x) generates a Pydantic **output schema** from the return
annotation. For `Dict[str, Any]` it validates that every top-level object key is
a **string**. If your data has non-string keys the call fails at the tool layer
with, e.g.:

```
Error executing tool get_standings: 3 validation errors for wrapperOutput
result.201.[key]
  Input should be a valid string [type=string_type, input_value=201, input_type=int]
```

This is exactly what bit `get_standings`: `statsapi.standings_data()` returns a
dict keyed by **integer** division IDs (`200`, `201`, …). Fix = coerce keys to
strings before returning (JSON object keys are strings anyway):

```python
result = statsapi.standings_data(**kwargs)
if isinstance(result, dict):
    result = {str(div_id): division for div_id, division in result.items()}
return result
```

General rules to avoid this class of bug:

- Return only JSON-serializable values; convert pandas objects with
  `df.to_dict(...)` and stringify any non-string dict keys.
- The wrapper validates **top-level** keys for `Dict[str, Any]`; nested values
  are `Any`. If you tighten a return type, expect deeper validation.
- A tool that raises returns `isError: true` with the message in
  `content[0].text`. A successful response also contains `"isError": false`, so
  don't treat the mere presence of the string `isError` as a failure — check the
  value / look for the real payload.

## Verify a fix

```bash
docker compose restart baseball-mcp && sleep 4
curl -sS -X POST http://localhost:12000/mcp \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_standings","arguments":{"season":2023,"league_id":103}}}' \
  | sed -n 's/^data: //p'
```

Expect real data with string keys (e.g. `"201": {"div_name": ...}`), not a
`wrapperOutput` error. Then run the matching test, e.g.
`uv run pytest mlb_stats_mcp/tests/test_mlbstats.py -k standings -v`.

## Note on duplicate names

There are two standings tools: `get_standings` (MLB StatsAPI) and
`get_pybaseball_standings` (Baseball Reference via pybaseball). Keep them
distinct; downstream clients may prefer the generic `get_stats`
(`endpoint=standings`) wrapper for league/division standings.
