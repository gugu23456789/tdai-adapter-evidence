# TDAI Adapter - Test Evidence for PR #339 (Issue #235)

Test evidence for cross-platform TDAI adapter SDK + MCP stdio transport layer.

PR: https://github.com/TencentCloud/TencentDB-Agent-Memory/pull/339
Evidence repo: https://github.com/gugu23456789/tdai-adapter-evidence

## PR at a Glance

**One engine, three entry points** - unified access layer for TDAI Memory Gateway v2:

| Entry Point | Implementation | Use Case |
|:---|---|:---|
| **Python SDK** | `TdaiAdapter` ABC + `BridgeAdapter` | Python-native agent frameworks |
| **TypeScript SDK** | `MemoryAdapter` interface + `BaseMemoryAdapter` + `TdaiHttpClient` | TypeScript agent frameworks |
| **MCP stdio** | JSON-RPC 2.0 server (5 tools, 5 defense gates) | Any MCP-compliant platform |

## Test Summary

### PR-Submitted Tests (in fork `bridge-adapter` branch)

| Suite | Location | Tests | Scope |
|:---|---:|---:|:---|
| MCP protocol compliance | `bridge/mcp/tests/test_protocol.py` | 14 | JSON-RPC 2.0 spec, tool routing, error codes |
| MCP red-team | `bridge/mcp/tests/test_redteam.py` | 13 | injection, auth bypass, rate-limit, circuit breaker |
| MCP config (env-var fallback) | `bridge/mcp/tests/test_config.py` | 7 | API Key resolution, TDAI_* inheritance, loopback mode |
| MCP dual-path (local + multi-tenant) | `bridge/mcp/tests/test_dual_path.py` | 4 | local mode, multi-tenant mode, API key override, startup demo |
| TS MemoryAdapter | `src/core/memory-adapter.test.ts` | 19 | interface contract, bounds, lifecycle, concurrency |
| TS TdaiHttpClient | `src/core/tdai-http-client.test.ts` | 10 | 8 endpoints, 7 error types, env-var defaults |
| **PR-submitted total** | | **67** | |

### Bridge Internal Tests (local, not in fork)
Local Bridge tests that exercise the SDK through the Bridge engine - these exist in the `.codex` repository, not in the TDAI fork.

| Suite | Tests | Passed | Scope |
|:---|---:|:---:|:---|
| Bridge integration | 353 | 353 OK | engine.py hooks, memory layer, TDAI wiring, session recall |
| BridgeAdapter provider | 20 | 20 OK | recall/capture/search, graceful degradation, health check |
| SDK red-team | 37 | 37 OK | registry, bounds, concurrency, ABC enforcement, buffering |
| **Local total** | **410** | **410** | **100%** |

## CI Status (Fork Actions)

6/6 jobs passing: https://github.com/gugu23456789/TencentDB-Agent-Memory/actions

| Job | Result |
|:---|---:|
| Test TS (29 tests) | OK |
| Test Python (import + SHA256) | OK |
| Install | OK |
| Pack (npm build) | OK |
| Manifest | OK |
| Size Guard | OK |

Node: 24 (upgraded from 22, CI warning-free)

## MCP Defense Gates (G0-G4)

The MCP stdio server (`bridge/mcp/server.py`) applies 5 layers of defense on every request:

| Gate | Mechanism | What it stops |
|:---:|:---|---:|
| G0 | JSON-RPC format validation | Malformed requests, protocol injection |
| G1 | HMAC API Key comparison | Unauthorized access (empty key = local loopback mode) |
| G2 | Sliding window rate-limit (60r/60s) | Request flooding, DoS |
| G3 | Circuit breaker (10 failures -> 60s cooldown) | Cascading failures, backend saturation |
| G4 | Full audit log (method, params, duration, result) | Zero audit trail for forensics |

**Graceful fallback chain**: `python -m bridge.mcp.server` (5 tools) <-> `python -m bridge.mcp_health` (1 tool: `tdai_health`, 4 gates, independent process).

### API Key Resolution Order

The MCP server resolves the API key in this order:
1. `MCP_BRIDGE_API_KEY` env var (MCP-specific override)
2. `TDAI_API_KEY` env var (shared SDK key)
3. Empty string -> loopback mode (no auth required tested)

This ensures users only need to set one `TDAI_API_KEY` for all three entry points.
See `test_config.py` (7 tests) for the fallback chain verification.

## Architecture

```
+------------------------------------------------------------------+
|                       Python SDK                                  |
|  TdaiAdapter ABC (retry + middleware + cache + degradation)       |
|    +-- BridgeAdapter (httpx -> Gateway)                           |
|    +-- HermesV2Adapter (chat-completion proxy)                    |
|    +-- BufferedAdapter (local JSONL dedup + compaction)          |
|  TdaiAdapterRegistry - create("bridge"|"hermes_v2"), list()       |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|                      TypeScript SDK                                |
|  MemoryAdapter interface                                          |
|  BaseMemoryAdapter (retry + middleware + cache + degradation)     |
|  TdaiHttpClient (fetch-based, 8 Gateway endpoints, 7 typed errors)|
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|                    MCP stdio Transport                            |
|  JSON-RPC 2.0 over stdin/stdout, zero framework dependency        |
|  +-- 5 tools: tdai_health/recall/capture/memory_search/conv_search|
|  +-- 5 gates: G0-G4 (always active)                               |
|                                                                   |
|  Clients: Codex CLI / Claude Code / CodeBuddy / Trae IDE / Cursor|
+------------------------------------------------------------------+

                         +--------------+
                         | TDAI Gateway  |
                         | (port 8420)   |
                         +------+-------+
                                v
                          TdaiCore -> SQLite
```

## Key Design Decisions

1. **ABC over concrete** - Subclasses implement only 4 `_impl` methods; base class provides retry, middleware, caching, degradation.
2. **MCP over per-platform adapters** - One stdio server replaces N platform-specific adapters. Any MCP client connects with one config line.
3. **Defense gates always active** - G0-G4 run on every request regardless of transport.
4. **Zero MCP framework dependency** - Pure JSON-RPC 2.0, no supply-chain risk.
5. **Graceful degradation at every layer** - MCP fallback -> SDK safe defaults -> return.
6. **Dual-language parity** - Python ABC and TS interface+client share equivalent abstractions.

## Related Issues

- **#120 prompt cache degradation**: Session-level recall cache added to `TdaiAdapter` (SHA256 query -> cached result). Same query within session returns cached context without re-fetching from Gateway. Full fix (move injection to message end) belongs in platform plugin layer, not SDK.
- **#235 cross-platform adapters**: This PR implements the unified access layer.

## PR Body & Commit History

- `pr_body_updated.txt` - current PR #339 description
- `pr.json` - PR #339 metadata (API snapshot)
- `pr339_body.json` - PR body as JSON
- `commit_msg.txt` - git commit history (bridge-adapter branch)

## Test Log Files

| File | Content |
|:---|:---|
| `verify_integration.log` | 353 integration tests (Bridge engine + memory + TDAI) |
| `test_provider.log` | 20 provider tests (BridgeAdapter full cycle) |
| `test_redteam.log` | 37 SDK red team tests (security + edge cases + concurrency) |
| `test_config-run.log` | 7 MCP config tests (env-var fallback, loopback, inheritance) |
| `test_dual_path-run.log` | 4 MCP dual-path tests (local mode, multi-tenant, API key override) |

## Gateway Health

TDAI Gateway was running at `http://127.0.0.1:8420` during provider and red team tests.
Circuit breaker warnings in logs (`Illegal header value`, `WinError 10061`) are expected -
they come from graceful degradation tests that intentionally clear env vars
to verify the adapter doesn't crash.

## Verification (Local Bridge Repo)

```bash
cd path/to/bridge-repo
python verify_integration.py     # 353 tests
python test_bridge_provider.py   # 20 tests
python test_tdai_sdk_redteam.py  # 37 tests
python -m pytest bridge/mcp/tests/test_config.py -v  # 7 tests
```

## Environment

- TDAI Gateway: `http://127.0.0.1:8420` (local standalone)
- Python: 3.11+
- Node: 20+
- httpx: 0.28.1
