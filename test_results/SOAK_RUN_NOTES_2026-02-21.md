# Full-Stack Soak Run Notes (2026-02-21)

## Goal

Run native lane, L2 stress lane, and compat lane together for long-duration validation.

## Runner

- Script: `run_soak_24h_all.sh`
- Profile: `SOAK_STACK_PROFILE=web` (prevents localhost admin-auth deadlocks)
- Data dir: `chain_data_soak24h`

## Issues found and fixed

1. `dag_guard` restart loop in unix-socket mode
   - Cause: guard listener check is TCP-based and reports false unhealthy on unix-only DAG.
   - Mitigation in soak runner: default native stress path uses `msgpack+http`, unix mode optional.

2. L2 stress early startup failure (`connection refused`)
   - Cause: stress started before L2 was fully ready.
   - Fix: `stress_l2_absolute_v1.py` now supports startup wait/retry:
     - `--startup-wait-sec`
     - `--startup-poll-sec`
   - Wired through `run_l2_absolute_stress.sh`.

3. Soak wait handling
   - Fix: runner now waits on stress worker PIDs only, not the infinite monitor loop.

## Current observed behavior snapshot

- DAG RPC up on `127.0.0.1:8650` and processing range-signed native load.
- L2 RPC up on `127.0.0.1:8660` with bridge/contract engines enabled.
- Main RPC up on `127.0.0.1:8545`.

Native log sample from this run:

- `blocks=390`
- `txs=390000000000`
- elapsed around `131.56s`
- per-block latency around `0.33s` in the sampled segment

## Note for publication

These notes are for transparency on operator behavior and soak harness reliability.
Use benchmark docs in this folder for headline throughput claims.

