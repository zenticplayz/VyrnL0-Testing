# CosmWasm 75% Status (Core Execution Parity)

This tracks the "75% done" milestone focused on runtime semantics parity while
keeping native DAG hotpath isolated.

## Implemented in this milestone

- Entrypoint support and fallback routing:
  - direct entrypoint calls (`instantiate/execute/query/reply/sudo/migrate`)
  - fallback routing: unknown mutable -> `execute`, unknown view -> `query`
- Mutable rollback atomicity:
  - failed mutable execution rolls back storage/log changes
  - submessage + reply chain guarded with rollback-on-failure behavior
- Guardrails:
  - call depth, submessage, reply caps
  - payload/memory/log caps exposed in `rpc_wasm_policy`
- Real CosmWasm ABI host import compatibility:
  - DB host imports (`db_read/write/remove/scan/next`) backed by persisted wasm storage
  - API/crypto/query imports linked (`addr_*`, `secp256k1_*`, `ed25519_*`, `debug`, `query_chain`, `abort`)
  - CosmWasm region-pointer call ABI for `instantiate/execute/query/migrate` with env/info/msg regions
  - ContractResult/Binary unwrapping on return path (query returns decoded binary payload)
- RPC surface:
  - `contract_querySmart`
  - `contract_queryRaw` (alias)
  - `contract_info`
  - existing `contract_simulate`, `contract_storage`, `contract_rawQuery`
- Policy/observability:
  - `rpc_wasm_policy` now includes `execution` and query mode surfaces
  - explicit compatibility list and 75% target marker
- IBC verifier-lite + governance surface:
  - channel handshake RPCs (`ibc_channel_openInit/openTry/openAck/close`)
  - packet RPCs (`ibc_transfer_send`, `ibc_packet_recv`, `ibc_packet_ack`, `ibc_packet_timeout`)
  - trusted checkpoint anchors (`ibc_setTrustedCheckpoint`, `ibc_checkpoint_status`)
  - proof/header checks on recv+ack (membership + trusted-header anchoring)
  - governance ops with multisig+timelock (`ibc_gov_*`)
  - CosmJS helper client (`cosmjs_compat_client_v1.py`) for workflow parity

## Deferred (toward 100%)

- Full IBC protocol semantics parity with production chains
  - ordered-channel semantics, timeout-on-close, and proof-of-absence paths
- Full light-client security model
  - real consensus proof verification (not verifier-lite commitment/header checks)
- Full CosmJS tooling parity and schema/codegen UX
- Advanced contract migration governance and admin-policy lifecycle
- Deep gas/economic calibration against production Cosmos chains

## Evidence (tests)

- Existing runtime semantics suites:
  - `test_wasm_dispatch_cw_v1.py`
  - `test_wasm_submsg_reply_guardrails_v1.py`
  - `test_wasm_cw_submsg_bank_v1.py`
  - `test_wasm_cw_submsg_instantiate_v1.py`
  - `test_wasm_host_env_v1.py`
- New RPC surface suites:
  - `test_wasm_query_modes_v1.py`
  - `test_wasm_policy_surface_v1.py`
- New IBC/verifier-lite suites:
  - `test_ibc_semantics_v1.py`
  - `test_ibc_rpc_surface_v1.py`
- Real fixture suite:
  - `test_wasm_cw20_base_integration_v1.py`
