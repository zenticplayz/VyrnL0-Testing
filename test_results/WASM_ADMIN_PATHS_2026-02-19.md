# WASM CosmWasm Admin Paths (2026-02-19)

## Goal
Improve CosmWasm compatibility for admin-style entrypoints without touching native DAG hot path.

## Implemented
- Added direct CW payload/dispatch parity for exported wasm entrypoints:
  - `migrate`
  - `sudo`
  - `reply` (with `reply` alias payload)
- Added submessage normalization support for:
  - `msg.wasm.migrate`
- Preserved existing behavior for:
  - fallback mutable → `execute`
  - fallback view → `query`

## Files touched
- `contract_runtime_stub.py`
- `test_wasm_cw_admin_paths_v1.py` (new)
- `(A) README_RPC.md`

## Validation
- `python3 test_wasm_cw_admin_paths_v1.py` ✅
- `python3 test_wasm_compat_v1.py` ✅
- `python3 test_wasm_dispatch_cw_v1.py` ✅
- `python3 test_wasm_host_env_v1.py` ✅
- `python3 test_wasm_submsg_reply_guardrails_v1.py` ✅
- `python3 test_wasm_cw_submsg_bank_v1.py` ✅
- `python3 test_wasm_cw_bank_payload_v1.py` ✅
- `python3 compat_smoke.py` ✅

## Notes
- No native DAG lane logic changed.
- This pass only raised wasm contract-island compatibility surface.
