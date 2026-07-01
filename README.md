# Bridge Adapter — Test Evidence for TDAI #235

Test results for `bridge_adapter` module submission (PR for issue #235).

## Test Summary

| Suite | Tests | Passed | Scope |
|:---|:---:|:---:|:---|
| Bridge integration tests | 353 | 353 ✅ | engine.py hooks, memory layer, TDAI wiring, session recall |
| BridgeAdapter provider tests | 20 | 20 ✅ | recall/capture/search, graceful degradation, health check |
| TDAI SDK red team tests | 37 | 37 ✅ | registry, parameter bounds, concurrency, ABC enforcement, buffering |
| TypeScript MemoryAdapter | 19 | 19 ✅ | interface contract, bounds, lifecycle, concurrency |
| **Total** | **429** | **429** | **100%** |

## CI Status (Fork Actions)

6/6 jobs passing: https://github.com/gugu23456789/TencentDB-Agent-Memory/actions/runs/28502102262

| Job | Result |
|:---|---:|
| Test TS (19 MemoryAdapter) | ✅ |
| Test Python (import + SHA256) | ✅ |
| Install | ✅ |
| Pack (npm build) | ✅ |
| Manifest | ✅ |
| Size Guard | ✅ |

## Related Issues

- **#120 prompt cache degradation**: Session-level recall cache added to `TdaiAdapter` (SHA256 query → cached result, `base.py:285-298`). Mitigation: same query within session returns cached `prepend_context` without re-fetching from Gateway. Root cause: `prependContext` injected at user message start every turn breaks prefix-matching caches (DeepSeek, MiMo). Full fix (move injection to message end) belongs in OpenClaw plugin layer, not SDK.

  **Reference — Reasonix `PrefixShape` pattern** (v1.13.1, `internal/agent/cache_shape.go`):
  Reasonix tracks `SystemHash + ToolsHash + PrefixHash` per turn and compares across turns to explain cache misses. If TDAI adopted similar diagnostics, it could report: *"cache miss: system=✓ tools=✓ prefix=✗ (L1 memories changed)"* — giving maintainers exact evidence instead of speculation.

## PR Body

- `pr_body_updated.txt` — PR #339 description with `Closes #235`

## Test Log Files

| File | Content |
|:---|:---|
| `verify_integration.log` | 353 integration tests (Bridge engine + memory + TDAI) |
| `test_provider.log` | 20 provider tests (BridgeAdapter full cycle) |
| `test_redteam.log` | 37 SDK red team tests (security + edge cases + concurrency) |

## Gateway Health

TDAI Gateway was running at `http://127.0.0.1:8420` during provider and red team tests.
Circuit breaker warnings in logs (`Illegal header value`, `WinError 10061`) are expected —
they come from graceful degradation tests (B7/B8) that intentionally clear env vars
to verify the adapter doesn't crash.

## Adapter Files (submitted in PR)

| File | Lines | Purpose |
|:---|:---:|:---|
| `bridge_adapter/base.py` | 382 | TdaiAdapter ABC, BufferedAdapter, errors, retry, middleware, registry |
| `bridge_adapter/__init__.py` | 300 | BridgeAdapter(TdaiAdapter) implementation |
| `bridge_adapter/client.py` | 120 | TdaiHttpClient (httpx, 7 Gateway endpoints) |
| `bridge_adapter/hermes_v2_adapter.py` | 130 | HermesV2Adapter cross-platform reference |
| `bridge_adapter/plugin.yaml` | 12 | TDAI plugin metadata |
| `bridge_adapter/pyproject.toml` | 13 | pip install configuration |
| `bridge_adapter/README.md` | 145 | Quick start, architecture, deployment modes |
| `docs/platform-adapter-comparison.md` | 145 | 4-platform matrix + unified SDK guide |

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Bridge / Any Platform                    │
│  engine.py → BridgeAdapter.recall()                          │
│                     │                                        │
│                     ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ TdaiAdapter SDK (bridge_adapter.base)                   ││
│  │  recall(query, limit)                                   ││
│  │   ├─ _sanitize_query()      100K truncation + type     ││
│  │   ├─ _sanitize_limit()       [1, 1000] clamping        ││
│  │   ├─ middleware.before()     metrics / auth / logging  ││
│  │   ├─ _with_retry()           3 attempts, exp backoff   ││
│  │   │   └─ _recall_impl()      platform implementation  ││
│  │   ├─ middleware.after()      record latency + counts  ││
│  │   └─ graceful degradation    exception → safe empty   ││
│  │                                                        ││
│  │  BufferedAdapter: optional mixin, local JSONL buffer  ││
│  │  TdaiAdapterRegistry: name-based lookup + health_all() ││
│  └────────────────────┬───────────────────────────────────┘│
└───────────────────────┼─────────────────────────────────────┘
                        │ TdaiHttpClient (httpx)
                        ▼
                 TDAI Gateway (port 8420) → TdaiCore → SQLite
```

## Verification Command

```bash
# Run all Bridge-side tests
cd path/to/bridge-repo
python verify_integration.py
python test_bridge_provider.py
python test_tdai_sdk_redteam.py
```

## Environment

- TDAI Gateway: `http://127.0.0.1:8420` (local standalone)
- Python: 3.11+
- httpx: 0.28.1
