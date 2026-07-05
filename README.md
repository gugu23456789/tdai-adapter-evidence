# TDAI Adapter - Test Evidence for r3-v1 / bridge-adapter

## Summary

- **PR**: [#339](https://github.com/TencentCloud/TencentDB-Agent-Memory/pull/339) — TDAI Unified Adapter SDK
- **Issue**: [#235](https://github.com/TencentCloud/TencentDB-Agent-Memory/issues/235) — Unified MCP adapter
- **Status**: CI 8/8 all green (commit `9ec2ca4`)
- **Total tests**: **~90** (32 SDK TypeScript + 29 core TS + 4 Python redteam + Python smoke)

## Test Breakdown

| Category | Files | Count | Notes |
|---|---|---|---|
| SDK TypeScript | `sdk/typescript/tests/client.test.ts`, `cos.test.ts` | 32 | mock transport (client + cos) |
| Core TS | `src/core/*.test.ts` + `src/core/gates/*.test.ts` | 29 | interface contract + HTTP client + gates (rate-limit, circuit-breaker, audit, redteam) |
| Utils TS | `src/utils/*.test.ts` | ~5 | no-think-fetch, sanitize, time |
| Python | `bridge/mcp/tests/` | ~24 | MCP config, dual-path, protocol, redteam |
| **Total** | | **~90** | |

## CI Results

**8/8 jobs passing** (latest: `9ec2ca4`):

| Job | Status | Scope |
|---|---|---|
| Install | OK | npm ci + npm audit (supply chain) |
| Test TS | OK | 6 test files + SDK 2 files + Utils 3 files |
| Test Python | OK | bridge_adapter import + SHA256 integrity |
| Encoding Defense | OK | L1 (BOM) + L5 (non-ASCII scan on bridge/bridge_adapter/docs) |
| File Integrity | OK | UTF-8 validity + Python syntax + tsc --noEmit + truncation check |
| Manifest | OK | openclaw.plugin.json + package.json metadata |
| Pack | OK | npm pack verification |
| Size Guard | OK | 5MB limit (current: 1.0MB) |

## Infrastructure

- **Node version**: 24 (upgraded from 22)
- **CI**: GitHub Actions (8 jobs)
- **Supply chain**: npm audit --audit-level=high
- **Encoding defense**: 5-layer PowerShell mojibake prevention
- **Evidence repo**: This repository

## Encoding Defense (5-layer)

| Layer | Defense | Scope |
|---|---|---|
| L5 | CI non-ASCII byte scan | bridge/, bridge_adapter/, docs/ |
| L4 | PowerShell pipeline encoding | Out-File -Encoding utf8 |
| L3 | Shell bypass | git commit -F msg.txt |
| L2 | Git config | core.quotepath false, i18n.commitEncoding utf-8 |
| L1 | UTF-8 BOM verification | scripts/*.ps1 |

Evidence: [#1](https://github.com/gugu23456789/tdai-adapter-evidence/issues/1)

## Files in this repo

| File | Description |
|---|---|
| `README.md` | This file |
| `commit_msg.txt` | Full commit log |
| `pr.json` | PR metadata (API snapshot) |
| `pr339_body.json` | PR description (initial version) |
| `pr_body_updated.txt` | PR description (latest version) |
| Various `.log` files | Test output logs |
