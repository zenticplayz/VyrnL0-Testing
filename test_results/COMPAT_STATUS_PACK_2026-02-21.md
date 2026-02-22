# Compatibility Status Pack (2026-02-21)

This file is a single pointer doc for current compatibility artifacts.

## Included compatibility tracks

### EVM

- Core-3 method target and fail-closed policy:
  - `EVM_CORE3_SUPPORT_MATRIX.md`
- Existing coexistence benchmark and notes:
  - `COEXISTENCE_NATIVE_EVM_WASM_2026-02-18.md`

### CosmWasm / WASM

- 75% progress snapshot and scope boundaries:
  - `COSMWASM_75_STATUS.md`
- Existing WASM compatibility artifacts in this folder:
  - `WASM_COMPAT_BATCH_1_TO_5_2026-02-18.md`
  - `WASM_COMPAT_BATCH_6_2026-02-19.md`
  - `WASM_COSMWASM_85_PROGRESS_2026-02-18.md`
  - `WASM_ADMIN_PATHS_2026-02-19.md`

### Bridge trust model

- Launch trust decision and enforcement references:
  - `BRIDGE_TRUST_MODEL_V1.md`

## Interpretation

- EVM lane: documented with explicit scope and unsupported-method fail-closed behavior.
- WASM lane: active compatibility buildout with staged progress docs.
- Bridge model: launch trust posture locked to threshold attestations with fail-closed finalization controls.

