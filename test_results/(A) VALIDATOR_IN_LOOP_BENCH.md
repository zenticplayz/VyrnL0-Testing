# Validator-In-Loop DAG Bench (External QC)

Date: 2026-02-08

This document captures a validator-in-the-loop benchmark run of the DAG RPC, plus the
correctness signals observed (QC weight, finality proof, and range-signed metadata
verification).

The goal of this run was to ensure the "native fast lane" remains production-defensible
when external validators are actually participating (not just local single-node mining).

## Setup

- Local node:
  - `dag_rpc_server.py` on `127.0.0.1:8650`
  - Benchmark profile enabled:
    - `DAG_BENCH=1` (auto-run disabled)
    - `DAG_RATE_LIMIT_ENABLED=0`
    - `DAG_PERSIST_BLOCKS=1`
    - `DAG_SIG_STATS=1`
  - Signature enforcement enabled (bench defaults in `run_dag_rpc.sh`):
    - `NATIVE_SIG_VERIFY=1`
    - `NATIVE_SIG_REQUIRE=1`

- External parent QC mode:
  - `DAG_EXTERNAL_PARENT_QC=1`
  - `DAG_VALIDATORS_FILE=config/validators.pub.json` (pub-only; no privkeys on the DAG)

- Remote validator machine:
  - Runs 3x `validator_agent.py` (indices 0..2) against `http://127.0.0.1:18650`.
  - These agents poll `val_pendingParent` and submit `val_submitParentSig` signatures.

- Connectivity:
  - Reverse SSH tunnel from remote back to local DAG:
    - Remote `127.0.0.1:18650` forwards to local `127.0.0.1:8650`.

## Correctness Checks (Validator Participation)

1. Local DAG confirms it does not have validator privkeys

- `val_list` reported `has_priv:false` for all validators when using
  `config/validators.pub.json`.

2. QC / finality proof shows validator participation

- After mining, `dag_getBlockMeta("p00000002")` showed:
  - `qc_weight: 3`
  - `finality_proof` present (non-empty)

Interpretation: the DAG reached the configured quorum threshold using external validator
signatures.

## Correctness Checks (Range-Signed Block Material)

To verify that the "mine-range signed" lane emits durable, recomputable signed material,
we enabled `DAG_PERSIST_BLOCKS=1` and validated the most recent persisted parent record:

`python3 verify_range_signed_block.py --data-dir chain_data`

Example output during this run:

- `file=00000000-p00000001.mpk parent=p00000001 height=0 verify=True msg=ok`

Interpretation: the persisted parent contains `range_signed` metadata whose digest and
signature verify deterministically (domain-separated digest + signature over the range).

## Benchmark Results (With External Validators Running)

All benchmarks used the Go stress tool (`go_dag/cmd/stress`) with msgpack-over-HTTP:

- `--rpc msgpack+http://127.0.0.1:8650`

### A) Per-tx signed + verified lane (defensible baseline)

Mode:
- `--mine-with-raw --sign --sig-scheme ed25519`

Result (400,000 tx):

- elapsed_sec: 1.808627
- txs_committed: 400000
- blocks_mined: 80
- tx_per_second: 221162.23
- blocks_per_second: 44.23
- avg_txs_per_block: 5000
- p50: 0.411051s
- p90: 0.504464s
- p99: 0.547558s

Additional signal:
- `rpc_sigStats` showed `sig_stats.raw_fast_ed: 80` (fast raw ed25519 verify path used).

### B) Native fast lane (range-signed + external QC)

Mode:
- `--mine-range --sign --sig-scheme ed25519`

Result (2,000,000 tx):

- elapsed_sec: 0.376483
- txs_committed: 2000000
- blocks_mined: 140
- tx_per_second: 5312327.53
- blocks_per_second: 371.86
- avg_txs_per_block: 14285.71
- p50: 0.053006s
- p90: 0.065540s
- p99: 0.066502s

Additional signal:
- `rpc_sigStats` showed `range_verify_ok` increasing and `range_verify_fail: 0`.

### C) Longer-run native fast lane sample (throughput stability check)

Mode:
- `--mine-range --sign --sig-scheme ed25519`

Result (10,000,000 tx; batch_size=15000):

- elapsed_sec: 2.768688
- txs_committed: 10000000
- blocks_mined: 680
- tx_per_second: 3611818.58
- blocks_per_second: 245.60
- avg_txs_per_block: 14705.88
- p50: 0.076973s
- p90: 0.098984s
- p99: 0.127810s

Note: this run validates sustained throughput at higher total tx count. If you need
to strictly assert "external validators participated" for this specific run, re-check
`dag_getBlockMeta` and `qc_weight` immediately after the run (the earlier correctness
section shows what to look for).

## Notes / Implications

- The per-tx signed/verified lane (~221k TPS here) is the most comparable figure to
  "verify every tx on ingest" systems.
- The range-signed lane is intentionally different: it authorizes and verifies a range
  as a unit (for the native path), then relies on external QC/finality to be produced
  by validators. This is why it scales into the millions.
- EVM/WASM paths can remain per-tx signed if desired, while the native path uses
  range-signed blocks for throughput.

## Vast.ai Instance Status

This doc captured the validator-in-loop result set needed to justify the "external QC"
lane. If you are cost-sensitive, you can destroy the Vast.ai instance after you confirm:

- `dag_getBlockMeta("p00000002")` (or a recent parent) shows `qc_weight >= threshold`
- `finality_proof` present
- `verify_range_signed_block.py` passes once with `DAG_PERSIST_BLOCKS=1`
