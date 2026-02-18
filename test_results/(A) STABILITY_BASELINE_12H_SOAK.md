# Vyrn Stability Baseline (12h Soak)

## Scope
This baseline captures the native DAG signed soak profile and records both:
- The prior failure mode that caused silent DAG drops.
- The post-fix 12h+ stable run behavior.

Date window: 2026-02-10 (local test run)

## Known-Good Profile
DAG launch profile (frozen):
- `./run_dag_bench_profile.sh`
- Key settings:
  - `DAG_BENCH=1`
  - `DAG_AUTO_RUN=0`
  - `DAG_TRACK_RECENT=0`
  - `NATIVE_SIG_VERIFY=1`
  - `NATIVE_SIG_REQUIRE=1`
  - `DAG_RANGE_KEY_MOD=200000`
  - `DAG_MAX_TX_PER_BATCH=50000000`

Soak traffic profile:
- `cd ~/Documents/Vyrn-Chain/go_dag`
- `./stress --rpc msgpack+http://127.0.0.1:8650 --workers 8 --soak --tps 90000 --duration 22h --mine-range --sign --sig-scheme ed25519 --progress-every 50`

## Prior Failure Mode (Before Keyspace Bounding)
Observed signatures:
- `read: connection reset by peer`
- `Post "http://127.0.0.1:8650": EOF`
- `empty response (status 200)`
- Main RPC proxy errors when DAG dropped:
  - `DAG RPC proxy failed (dag_getTips): timed out`
  - `DAG RPC proxy failed ...: [Errno 111] Connection refused`

Typical stress termination sample:
- workers reached high block counts, then a burst of EOF/reset errors across workers.

Root cause category:
- Unbounded growth in mine-range keyspace under long signed soak, causing instability over time.

## Fix Applied
Implemented bounded keyspace for mine-range benchmark lane:
- Runtime supports `range_key_mod` mapping.
- DAG profile sets `DAG_RANGE_KEY_MOD=200000` in bench mode.

Operational hardening added:
- `dag_guard_watchdog.sh` (health poll + restart with explicit reason logging).
- `dag_metrics_snapshot.py` (periodic JSONL metrics snapshots).
- Stack integration via `run_stack.sh`, `stop_stack.sh`, `status_stack.sh`.

## Post-Fix Evidence (Stable Run)
Reported stable progress sample:
- `[worker 0..7] blocks=40250 txs=452812500 elapsed=40250s ...`
- No DAG silent death after ~12h.

Telemetry snapshot (post-fix):
- `range_key_mod: 200000`
- `state_keys: 1600000`
- `mine_lock_locked: false`

Validator cache note:
- `hit: 0`, `miss: 1010688`, `evict: 1000688` indicates low cache reuse in this workload.
- This is a performance tuning issue, not a stability blocker for this profile.

## Interpretation
What this means for the chain now:
- Native signed DAG lane demonstrates sustained stability for at least ~12h at high constant load.
- The prior hard-fail pattern is materially mitigated under bounded keyspace profile.
- The chain is in a stronger position for repeatable endurance tests and investor-facing stability evidence.

What this does **not** prove yet:
- Multi-node/validator WAN stability under packet loss/jitter.
- EVM/WASM/BPF parity behavior under the same soak profile.
- 24h/48h persistence guarantees with all production durability toggles enabled.

## Next Validation Targets
1. Repeat 24h soak with the same profile and archive snapshots.
2. Run one WAN validator-in-loop soak and compare error envelope.
3. Add a receipt/index integrity pass every N minutes during soak.
