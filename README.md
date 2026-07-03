# TDAI Adapter 鈥?Test Evidence for PR #339 (Issue #235)

Test evidence for cross-platform TDAI adapter SDK + MCP stdio transport layer.

PR: https://github.com/TencentCloud/TencentDB-Agent-Memory/pull/339
Evidence repo: https://github.com/gugu23456789/tdai-adapter-evidence

## PR at a Glance

**One engine, three entry points** 鈥?unified access layer for TDAI Memory Gateway v2:

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
| TS MemoryAdapter | `src/core/memory-adapter.test.ts` | 19 | interface contract, bounds, lifecycle, concurrency |
| TS TdaiHttpClient | `src/core/tdai-http-client.test.ts` | 10 | 8 endpoints, 7 error types, env-var defaults |
| **PR-submitted total** | | **56** | |

### Bridge Internal Tests (local, not in fork)
Local Bridge tests that exercise the SDK through the Bridge engine 鈥?these exist in the `.codex` repository, not in the TDAI fork.

| Suite | Tests | Passed | Scope |
|:---|---:|:---:|:---|
| Bridge integration | 353 | 353 鉁?| engine.py hooks, memory layer, TDAI wiring, session recall |
| BridgeAdapter provider | 20 | 20 鉁?| recall/capture/search, graceful degradation, health check |
| SDK red-team | 37 | 37 鉁?| registry, bounds, concurrency, ABC enforcement, buffering |
| **Local total** | **410** | **410** | **100%** |

## CI Status (Fork Actions)

6/6 jobs passing: https://github.com/gugu23456789/TencentDB-Agent-Memory/actions

| Job | Result |
|:---|---:|
| Test TS (30 tests) | 鉁?|
| Test Python (import + SHA256) | 鉁?|
| Install | 鉁?|
| Pack (npm build) | 鉁?|
| Manifest | 鉁?|
| Size Guard | 鉁?|

## MCP Defense Gates (G0鈥揋4)

The MCP stdio server (`bridge/mcp/server.py`) applies 5 layers of defense on every request:

| Gate | Mechanism | What it stops |
|:---:|:---|---:|
| G0 | JSON-RPC format validation | Malformed requests, protocol injection |
| G1 | HMAC API Key comparison | Unauthorized access (empty key = local loopback mode) |
| G2 | Sliding window rate-limit (60r/60s) | Request flooding, DoS |
| G3 | Circuit breaker (10 failures 鈫?60s cooldown) | Cascading failures, backend saturation |
| G4 | Full audit log (method, params, duration, result) | Zero audit trail for forensics |

**Graceful fallback chain**: `python -m bridge.mcp.server` (5 tools) 鈫?鈫?`python -m bridge.mcp_health` (1 tool: `tdai_health`, 4 gates, independent process).

The fallback server shares the same environment variables and backend, making it a drop-in replacement for health-only diagnosis without the full MCP tool surface.

## Architecture

```
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?                       Python SDK                                 鈹?鈹? TdaiAdapter ABC (retry + middleware + cache + degradation)       鈹?鈹?   鈹斺攢 BridgeAdapter (httpx 鈫?Gateway)                             鈹?鈹?   鈹斺攢 HermesV2Adapter (chat-completion proxy)                     鈹?鈹?   鈹斺攢 BufferedAdapter (local JSONL dedup + compaction)           鈹?鈹? TdaiAdapterRegistry 鈥?create("bridge"|"hermes_v2"), list()       鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?                     TypeScript SDK                               鈹?鈹? MemoryAdapter interface                                          鈹?鈹? BaseMemoryAdapter (retry + middleware + cache + degradation)     鈹?鈹? TdaiHttpClient (fetch-based, 8 Gateway endpoints, 7 typed errors)鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?                   MCP stdio Transport                            鈹?鈹? JSON-RPC 2.0 over stdin/stdout, zero framework dependency        鈹?鈹? 鈹屸攢 5 tools: tdai_health/recall/capture/memory_search/conv_search鈹?鈹? 鈹斺攢 5 gates: G0-G4 (always active)                               鈹?鈹?                                                                  鈹?鈹? Clients: Codex CLI / Claude Code / CodeBuddy / Trae IDE / Cursor鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?
                         鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?                         鈹?TDAI Gateway  鈹?                         鈹?(port 8420)   鈹?                         鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹?                                鈻?                          TdaiCore 鈫?SQLite
```

## Key Design Decisions

1. **ABC over concrete** 鈥?Subclasses implement only 4 `_impl` methods; base class provides retry, middleware, caching, degradation.
2. **MCP over per-platform adapters** 鈥?One stdio server replaces N platform-specific adapters. Any MCP client connects with one config line.
3. **Defense gates always active** 鈥?G0鈥揋4 run on every request regardless of transport.
4. **Zero MCP framework dependency** 鈥?Pure JSON-RPC 2.0, no supply-chain risk.
5. **Graceful degradation at every layer** 鈥?MCP fallback 鈫?SDK safe defaults 鈫?return.
6. **Dual-language parity** 鈥?Python ABC and TS interface+client share equivalent abstractions.

## Related Issues

- **#120 prompt cache degradation**: Session-level recall cache added to `TdaiAdapter` (SHA256 query 鈫?cached result). Same query within session returns cached context without re-fetching from Gateway. Full fix (move injection to message end) belongs in platform plugin layer, not SDK.
- **#235 cross-platform adapters**: This PR implements the unified access layer.

## PR Body & Commit History

- `pr_body_updated.txt` 鈥?current PR #339 description
- `pr.json` 鈥?PR #339 metadata (API snapshot)
- `pr339_body.json` 鈥?PR body as JSON
- `commit_msg.txt` 鈥?git commit history (bridge-adapter branch)

## Test Log Files

| File | Content |
|:---|:---|
| `verify_integration.log` | 353 integration tests (Bridge engine + memory + TDAI) |
| `test_provider.log` | 20 provider tests (BridgeAdapter full cycle) |
| `test_redteam.log` | 37 SDK red team tests (security + edge cases + concurrency) |

## Gateway Health

TDAI Gateway was running at `http://127.0.0.1:8420` during provider and red team tests.
Circuit breaker warnings in logs (`Illegal header value`, `WinError 10061`) are expected 鈥?they come from graceful degradation tests that intentionally clear env vars
to verify the adapter doesn't crash.

## Verification (Local Bridge Repo)

```bash
cd path/to/bridge-repo
python verify_integration.py     # 353 tests
python test_bridge_provider.py   # 20 tests
python test_tdai_sdk_redteam.py  # 37 tests
```

## Environment

- TDAI Gateway: `http://127.0.0.1:8420` (local standalone)
- Python: 3.11+
- Node: 20+
- httpx: 0.28.1
