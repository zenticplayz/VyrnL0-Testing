# WASM CosmWasm 85% Push (2026-02-18)

## Goal
Raise CosmWasm compatibility without touching native DAG hot path.

## Implemented in this pass
- Submessage normalization for:
  - `submessages` / `submsgs` / `messages`
  - `msg.wasm.execute`
  - `msg.bank.send`
- `reply_on` enum compatibility:
  - string values: `never/success/error/always`
  - int values: `0/1/2/3`
- Bank-send submessage execution through configured bank adapter.
- Reply fanout behavior:
  - one reply payload call per reply-triggered submessage.
- Added policy surface fields showing CosmWasm compatibility scope.

## Files
- `contract_runtime_stub.py`
- `rpc_server_v1.py`
- `test_wasm_cw_submsg_bank_v1.py` (new)
- `(A) README_RPC.md`
- `docs/(o) contract_layer_overview.md`

## Error encountered and fixed
- Initial test failure: bank submessage send failed due to test adapter mapping `from_raw` incorrectly for contract addresses (`ctr:...`).
- Fix: corrected test adapter to honor raw contract sender string.

## Validation
- `test_wasm_cw_submsg_bank_v1.py` PASS
- `test_wasm_submsg_reply_guardrails_v1.py` PASS
- `test_wasm_dispatch_cw_v1.py` PASS
- `test_wasm_cw_bank_payload_v1.py` PASS
- `test_wasm_host_env_v1.py` PASS
- `test_wasm_compat_v1.py` PASS
