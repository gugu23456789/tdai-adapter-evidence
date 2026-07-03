# TDAI Adapter - Test Evidence for PR #339 (Issue #235)

## Summary

- **PR**: [#339](https://github.com/TencentCloud/TencentDB-Agent-Memory/pull/339) — TDAI Unified Adapter SDK
- **Issue**: [#235](https://github.com/TencentCloud/TencentDB-Agent-Memory/issues/235) — Unified MCP adapter
- **Status**: CI 8/8 all green (commit `860f225e`)
- **Total tests**: **67** (38 MCP Python + 29 TypeScript)

## Test Breakdown

| Category | File | Count | Notes |
|:---|---|:---:|:---|
| MCP config | `bridge/mcp/tests/test_config.py` | 7 | env-var fallback chain, API key override, loopback mode |
| MCP dual-path | `bridge/mcp/tests/test_dual_path.py` | 4 | local mode, multi-tenant mode, API key override, startup demo |
| MCP protocol | `bridge/mcp/tests/test_protocol.py` | 27 | tool list, tool call, error handling, gate behavior |
| TS SDK | `src/core/*.test.ts` + `sdk/typescript/tests/*.ts` | 29 | unit + e2e tests |
| **Total** | | **67** | |

## CI Results

**8/8 jobs passing** (latest: `860f225e`):

| Job | Status |
|:---|---:|
| Install | OK |
| Test TS (29) | OK |
| Test Python (import + SHA256) | OK |
| Encoding Defense (L1+L5) | OK |
| Manifest | OK |
| Pack | OK |
| Size Guard | OK |

## Encoding Defense (new in PR #339)

5-layer defense against PowerShell -> Git encoding corruption (mojibake):

- **L5**: CI non-ASCII byte scan on .py/.ts files
- **L4**: PowerShell pipeline encoding (Out-File -Encoding utf8)
- **L3**: Shell bypass (git commit -F msg.txt)
- **L2**: Git config (core.quotepath false, i18n.commitEncoding utf-8)
- **L1**: UTF-8 BOM verification on .ps1 files

Evidence: https://github.com/gugu23456789/tdai-adapter-evidence/issues/1

## Infrastructure

- **Node version**: 24 (upgraded from 22)
- **CI**: GitHub Actions (7 jobs + 1 encoding defense job)
- **Evidence repo**: This repository

## Files in this repo

| File | Description |
|:---|---|
| `README.md` | This file |
| `commit_msg.txt` | Full commit log for the PR branch |
| `pr.json` | PR metadata (API snapshot) |
| `pr339_body.json` | PR description (initial version) |
| `pr_body_updated.txt` | PR description (latest version) |
| `test_config-run.log` | Test output: test_config.py (7/7 passed) |
| `test_dual_path-run.log` | Test output: test_dual_path.py (4/4 passed) |
| `test_provider.log` | Test output: provider tests |
| `test_redteam.log` | Test output: redteam tests |
| `verify_integration.log` | Test output: integration verification |
