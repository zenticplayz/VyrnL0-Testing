# L2 Hardening Test Summary (2026-02-21)

Status: pass-focused summary for public evidence packaging.

## What was covered

- Sequencer trust model behavior (`single`, `failover`, `committee`) with restart persistence.
- Bridge security gates:
  - observed-before-final
  - stale escalation and bounded retry behavior
  - sender caps and anti-spam controls
- L2 health and overload observability surface.
- Settlement hardening policy visibility (confirmation/fail-closed controls).

## Test evidence (scripts)

- `test_l2_sequencer_trust_model_v1.py`
- `test_l2_bridge_security_ab_v1.py`
- `test_l2_bridge_fault_matrix_v1.py`
- `test_l2_settlement_invariants_v1.py`
- `test_l2_bridge_settlement_hardening_v1.py`
- `security_selftest.py` (L2 behavioral gates section)

## Recorded result highlights

- Sequencer trust model run: `PASS` across single/failover/committee transitions.
- Security self-test run: L2 bridge/security checks reported `PASS`.
- Settlement hardening tests and compile checks completed successfully in the recent integration cycle.

## Operational notes

- L2 startup races during heavy stack boot were observed in soak orchestration.
- Mitigation added: L2 stress harness now waits/retries on `rpc_health` before failing.
- This reduces false negatives in long soak starts where L2 is still binding to port `8660`.

