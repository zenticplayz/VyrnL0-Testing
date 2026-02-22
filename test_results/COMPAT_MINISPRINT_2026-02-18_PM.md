# Compat Mini-Sprint (1/2/3) — 2026-02-18 PM

## Scope
1. Native + EVM + WASM coexistence regression
2. EVM edge-case compatibility hardening
3. WASM parity smoke pass (upload/instantiate/execute/query + guardrails)

## Results

### 1) Coexistence regression
- Native lane (`mineRangeSigned`, unix msgpack, signed):  
  - `tx_per_second`: **3,008,428,910.49**
  - `blocks_per_second`: `15.04`
  - `avg_txs_per_block`: `200,000,000`
  - `p99`: `0.072353s`
- Concurrent main-RPC contract mix bench (`evm + wasm`, 8 workers, 30s):
  - `calls_per_sec`: **1,066.61**
  - `calls_err`: `0`
  - `p99`: `18.7596ms`

### 2) EVM edge hardening
- `evm_edge_smoke.py`: PASS
  - blake2f vectors
  - revert returndata behavior
  - ABI dynamic decode + strict invalid/canonical cases
- `rpc_server_v1.py` updates:
  - Headless ETH-read fallback for fresh data dirs (no `head.json` yet):
    - `_eth_head()` returns empty-chain head instead of raising.
    - `_rehydrate_world_from_head()` returns empty world instead of hard-failing.
  - `eth_getTransactionCount(..., "pending")` now includes contiguous sender nonces from mempool.
    - Prevents accidental nonce reuse/replacement by clients that use pending mode.

### 3) WASM parity pass
- PASS:
  - `test_wasm_compat_v1.py`
  - `test_wasm_dispatch_cw_v1.py`
  - `test_wasm_host_env_v1.py`
  - `test_wasm_submsg_reply_guardrails_v1.py`
  - `test_wasm_cw_submsg_bank_v1.py`
  - `test_wasm_cw_bank_payload_v1.py`
- Coverage includes deploy/upload + instantiate/execute/query dispatch, storage/env host bindings, submsg/reply semantics, rollback, and reentrancy/depth guardrails.

## Notes
- `evm_log_smoke.py` was improved to:
  - normalize bare tx hashes to `0x...`
  - use `pending` nonce tag
  - ignore `"status":"pending"` pseudo-receipts
- In a fresh standalone compat data dir, receipt lookup can still lag depending on local tx/receipt indexing mode. This does not affect the edge-smoke or coexistence results above.
