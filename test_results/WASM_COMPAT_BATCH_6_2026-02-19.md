# WASM Compat Batch 6 (2026-02-19)

## Goal
Push CosmWasm compatibility further (admin + instantiate paths) and verify wasm lane reliability under RPC load.

## Implemented
- Added direct wasm entrypoint parity for:
  - `instantiate`
  - `execute`
  - `query`
  - `reply`
  - `sudo`
  - `migrate`
- Added submessage normalization/execution support for:
  - `msg.wasm.instantiate`
  - `msg.wasm.migrate`
  - `msg.wasm.execute`
  - `msg.bank.send`
- Added child contract deployment + instantiate execution path for `msg.wasm.instantiate`.

## Files changed
- `contract_runtime_stub.py`
- `(A) README_RPC.md`
- `test_wasm_cw_submsg_instantiate_v1.py` (new)

## Validation
- `python3 test_wasm_cw_submsg_instantiate_v1.py` ✅
- `python3 test_wasm_cw_admin_paths_v1.py` ✅
- `python3 test_wasm_submsg_reply_guardrails_v1.py` ✅
- `python3 test_wasm_dispatch_cw_v1.py` ✅
- `python3 test_wasm_cw_submsg_bank_v1.py` ✅
- `python3 test_wasm_cw_bank_payload_v1.py` ✅

## Throughput sanity (Main RPC contract lane)

### WASM-only (`bench_contract_rpc.py`)
- duration: 15s
- workers: 4
- calls_ok: 8577
- calls_err: 0
- calls_per_sec: 571.60
- p50/p90/p99: 6.89ms / 7.97ms / 9.65ms

### MIX (EVM+WASM) (`bench_contract_rpc.py`)
- duration: 15s
- workers: 8
- calls_ok: 11773
- calls_err: 0
- calls_per_sec: 784.41
- p50/p90/p99: 3.32ms / 21.35ms / 27.46ms

## Notes
- No native DAG hot-path changes.
- This batch targets wasm contract-island compatibility and coexistence reliability.
