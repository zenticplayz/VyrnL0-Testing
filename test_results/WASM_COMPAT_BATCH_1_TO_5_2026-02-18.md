# WASM Compat Batch 1-5 (2026-02-18)

Scope completed in one pass:
1. Real bank bridge adapter (`token` default + `native` mode)
2. Submessage + reply-lite flow
3. Contract-to-contract guardrails (depth/reentrancy/submsg/reply limits)
4. Event/attribute parity + indexed log filters
5. WASM policy hardening endpoint payload/guardrail visibility

## Code touched
- `contract_runtime_stub.py`
- `wasm_runtime.py`
- `rpc_server_v1.py`
- `test_wasm_submsg_reply_guardrails_v1.py` (new)
- `(A) README_RPC.md`
- `docs/(o) contract_layer_overview.md`

## Error log (captured during implementation)

### Error 1
- Symptom: new test failed immediately with `[FAIL] deploy stub contracts`.
- Root cause: both test contracts used identical bytecode, yielding the same deterministic `ctr:` address; second deploy collided.
- Fix: changed second stub deploy bytecode from `"00"` to `"01"`.
- Result: deploy section passed.

### Error 2
- Symptom: runtime crash during submessage failure path:
  - `NameError: name '_rollback_err' is not defined`
- Root cause: atomic rollback helper was accidentally inserted in `call_view` instead of `call_mutable` scope.
- Fix: moved rollback helper into the wasm branch inside `call_mutable`.
- Result: rollback and guardrail tests passed.

## Validation run
- `python3 test_wasm_compat_v1.py` ✅
- `python3 test_wasm_dispatch_cw_v1.py` ✅
- `python3 test_wasm_host_env_v1.py` ✅
- `python3 test_wasm_cw_bank_payload_v1.py` ✅
- `python3 test_wasm_submsg_reply_guardrails_v1.py` ✅
- `python3 compat_smoke.py` ✅

## X/Twitter-ready note
"WASM compat batch 1-5 shipped: bank bridge modes, submessage/reply-lite, C2C guardrails, indexed event filters, and hardened policy introspection. Full test suite green; two implementation bugs found and fixed in-flight with reproducible notes."
