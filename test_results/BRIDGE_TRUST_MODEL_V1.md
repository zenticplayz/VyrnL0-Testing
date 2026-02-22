# Bridge Trust Model (V1)

This document locks the launch-time L2 bridge trust model decision.

## Decision

- Launch model: **Threshold attestations** (operator/relayer attest path).
- Fail-closed finalization: **enabled**.

## Required controls

For launch and testnet security gates, the bridge must run with:

- `require_observed_before_final=true`
- `require_idempotency_key=true`
- `allow_replay_overwrite=false`

## Why this model now

- Matches current architecture and tests.
- Keeps operational complexity lower than proof-client integration at this stage.
- Preserves strict anti-replay and status-transition guarantees.

## Enforcement points

- Launch scripts:
  - `run_stack.sh`
  - `run_l2_rpc.sh`
- Runtime policy checks:
  - `l2_bridge_v1.py`
- Automated gates:
  - `security_selftest.py`
  - `test_l2_bridge_security_ab_v1.py`

## Out of scope for V1

- Proof-based bridge light-client verification.
- Hybrid trust path (proof + threshold).

Those can be introduced in a future bridge model revision with a new decision doc.
