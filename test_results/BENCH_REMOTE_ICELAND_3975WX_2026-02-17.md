# Remote Bench: Iceland Node (Threadripper 3975WX) - 2026-02-17

## Scope
- Objective: validate signed native DAG `mineRangeSigned` throughput on a remote data-center host.
- Constraint: keep the remote SSH terminal active (do not close background session).
- RPC path: `msgpack+unix:///tmp/dag_rpc.sock`

## Remote Setup
- SSH tunnel/session: `ssh -p 26548 root@213.181.123.107 -L 8080:localhost:8080`
- Repo copied to remote: `~/Vyrn-Chain`
- DAG start profile:
  - `DAG_UNIX=/tmp/dag_rpc.sock ./run_dag_profile_lowbps.sh`
  - Confirmed listening: `dag_rpc_server.py ... --unix /tmp/dag_rpc.sock`
- Signing lane: enabled (`--mine-range --sign --sig-scheme ed25519`)

## Test Command (sweep)
```bash
cd ~/Vyrn-Chain/go_dag
./stress \
  --rpc msgpack+unix:///tmp/dag_rpc.sock \
  --workers <1|2|4|8|12> \
  --sender-base addr:iceland_<workers>_$(date +%Y%m%d_%H%M%S) \
  --soak --tps 10000000000 --duration 60s \
  --batch-size 200000000 \
  --mine-range \
  --sign --sig-scheme ed25519 \
  --lat-samples 50000
```

## Results (60s each)

| Workers | TPS | BPS | Avg tx/block | p99 |
|---:|---:|---:|---:|---:|
| 1 | 1,719,885,661.11 | 8.60 | 200,000,000 | 0.132015s |
| 2 | 1,708,025,143.44 | 8.54 | 200,000,000 | 0.253901s |
| 4 | 1,685,580,253.34 | 8.43 | 200,000,000 | 0.496527s |
| 8 | 1,652,627,284.28 | 8.26 | 200,000,000 | 0.996066s |
| 12 | 1,664,773,663.47 | 8.32 | 200,000,000 | 1.490284s |

## Key Findings
- Best throughput in this profile is at `workers=1`.
- Throughput stays in the same band as workers increase, but p99 grows sharply.
- This indicates the bottleneck is on serialized mining/apply path pressure (not client worker count).
- Blocks remain valid/signed in the tested lane (`mineRangeSigned` with ed25519).

## Transport Verification (5m)
- Follow-up run to verify transport impact using identical workload:
  - workers: `1`
  - batch size: `200,000,000`
  - duration: `5m`
  - mode: `mineRangeSigned` + ed25519

| Transport | elapsed_sec | tx_per_second | blocks_per_second | avg_txs_per_block | p99 |
|---|---:|---:|---:|---:|---:|
| msgpack+unix (`/tmp/dag_rpc.sock`) | 300.039175 | 1,762,436,519.83 | 8.81 | 200,000,000 | 0.129160s |
| msgpack+http (`127.0.0.1:8650`) | 300.077862 | 1,712,222,273.84 | 8.56 | 200,000,000 | 0.133878s |

- Measured delta (HTTP vs unix): roughly `-2.85%` TPS.
- Interpretation: unix is faster, but transport is not the main limiter in this lane.

## Validation Suite (1h + checkpoint A/B + transport A/B)
- Automated suite output: `/tmp/iceland_suite_20260217_131743/summary.md` (mirrored locally in artifacts folder below).
- Suite ordering:
  1) `1h` signed soak
  2) checkpoint durability A/B (`1000` vs `5000`)
  3) transport A/B (`unix` then `http`)

| Test | elapsed_sec | tx_per_second | blocks_per_second | avg_txs_per_block | p99 |
|---|---:|---:|---:|---:|---:|
| 1h_unix_w1_ckpt1500 | 3600.068684 | 1,752,133,238.01 | 8.76 | 200,000,000.00 | 0.130928s |
| ckpt_1000_run | 600.031382 | 1,761,574,529.06 | 8.81 | 200,000,000.00 | 0.129185s |
| ckpt_5000_run | 600.057286 | 1,753,499,249.23 | 8.77 | 200,000,000.00 | 0.129941s |
| transport_unix_5m | 300.039175 | 1,762,436,519.83 | 8.81 | 200,000,000.00 | 0.129160s |

- Checkpoint restart timings from suite:
  - `checkpoint_every=1000`: restart-to-healthy `~1957ms`
  - `checkpoint_every=5000`: restart-to-healthy `~2180ms`
- Suite's initial `transport_http_5m` entry was invalid because DAG was running unix-only at that point; corrected HTTP run is reported in **Transport Verification (5m)** above.

## Post-suite Confidence Rerun (30m)
- Command profile: signed `mineRangeSigned`, `workers=1`, `batch=200,000,000`, `msgpack+unix`.
- Result:
  - `elapsed_sec`: `1800.097834`
  - `txs_committed`: `3,157,600,000,000`
  - `blocks_mined`: `15,788`
  - `tx_per_second`: `1,754,126,881.43`
  - `blocks_per_second`: `8.77`
  - `p50/p90/p99`: `0.111716s / 0.124065s / 0.129555s`
- Interpretation: consistent with 5m/10m/1h bands; no drift/regression signal.

## Artifacts
- Remote sweep log: `/tmp/iceland_sweep_20260217_124247.log`
- Remote 5m HTTP verification log: `/tmp/iceland_http_5m.log`
- Remote DAG log: `/tmp/dag_remote.log`
- Remote DAG socket: `/tmp/dag_rpc.sock`
- Local archived artifacts (after instance teardown):
  - `bench_artifacts/iceland_2026-02-17/summary.md`
  - `bench_artifacts/iceland_2026-02-17/suite.log`
  - `bench_artifacts/iceland_2026-02-17/hour_unix_w1.log`
  - `bench_artifacts/iceland_2026-02-17/ckpt_1000_10m.log`
  - `bench_artifacts/iceland_2026-02-17/ckpt_5000_10m.log`
  - `bench_artifacts/iceland_2026-02-17/transport_unix_5m.log`
  - `bench_artifacts/iceland_2026-02-17/transport_http_5m.log`
  - `bench_artifacts/iceland_2026-02-17/iceland_http_5m.log`
  - `bench_artifacts/iceland_2026-02-17/iceland_confidence_30m.log`

## Network Latency (local PC -> remote SSH host)
- Target: `213.181.123.107`
- Method: `ping -c 20 213.181.123.107`
- Result:
  - Packet loss: `0%` (20/20)
  - RTT min/avg/max/mdev: `183.674 / 184.283 / 184.984 / 0.440 ms`

## TPS vs Latency Context (Investor-facing)
- This benchmark runs the hot data path over a local unix socket on the remote host, so measured TPS reflects node compute + storage path, not internet RTT.
- The `~184ms` RTT matters for user-facing round-trip confirmation time and remote operator UX, but it does not cap in-node batch execution throughput in this setup.
- Practical read: high throughput remains valid at distance; what degrades first over WAN is interactivity/confirmation feel, not raw local block processing capacity.

## Operational Note
- During this bench window, the remote SSH session was intentionally left active per request and DAG stayed up after sweep completion.
