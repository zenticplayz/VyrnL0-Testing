# Go DAG Endurance + Ceiling (range-signed native lane)

Date: 2026-02-13

This documents the updated Go stress/soak harness and the latest short ceiling probes
against the native DAG `miner_mineRangeSigned` lane (valid, block-signed range blocks).

## What changed (Go stress tool)

File: `Documents/Vyrn-Chain/go_dag/cmd/stress/main.go`

- Soak mode is now time-based (runs until `--duration`), instead of precomputing
  `txs_total = tps * seconds`.
- Latency percentile stats use fixed-size reservoir sampling (`--lat-samples`)
  so multi-day runs do not grow memory.
- Soak banner prints `soak_tps` + `duration` (instead of a misleading `txs_total`).

Rebuilt binary:
- `Documents/Vyrn-Chain/go_dag/stress`

## Ceiling probes (local, msgpack+http)

Server profile used:
- `GO_RANGE_STATE=1`
- `DAG_FAST=1`
- `DAG_BENCH=1`
- `DAG_AUTO_RUN=0`
- `DAG_STORE=log`
- `DAG_PERSIST_BLOCKS=1`
- `DAG_TRACK_RECENT=0`
- `DAG_SIG_STATS=0`
- `DAG_RATE_LIMIT_ENABLED=0`
- `DAG_RANGE_KEY_MOD=200000` (bounded state growth for long soaks)
- `DAG_LOG_FSYNC_EVERY=1` (durability-friendly; still fast)

### Low-BPS mode (recommended for multi-day soaks)

Key idea: keep TPS high while reducing blocks/sec by increasing range block size.

Server override:
- `DAG_MAX_RANGE_COUNT=60000`

Client:
- `--batch-size 60000`

Results (10s probes):
- target 7,000,000 TPS: achieved ~6,997,578 TPS, ~116.6 BPS, p99 ~62ms
- target 8,000,000 TPS: achieved ~7,997,907 TPS, ~133.3 BPS, p99 ~64ms

## Recommended endurance command (multi-day)

Start DAG (highspeed lane, low overhead):

```bash
cd ~/Documents/Vyrn-Chain
pkill -f dag_rpc_server.py || true

GO_RANGE_STATE=1 \
DAG_FAST=1 DAG_BENCH=1 DAG_AUTO_RUN=0 \
DAG_STORE=log DAG_PERSIST_BLOCKS=1 \
DAG_RATE_LIMIT_ENABLED=0 \
DAG_TRACK_RECENT=0 DAG_SIG_STATS=0 \
DAG_RANGE_KEY_MOD=200000 \
DAG_MAX_RANGE_COUNT=60000 \
DAG_LOG_FSYNC_EVERY=1 \
./run_dag_rpc.sh
```

Run soak (example: 72 hours @ 6M TPS, ~100 BPS):

```bash
cd ~/Documents/Vyrn-Chain/go_dag
./stress \
  --rpc msgpack+http://127.0.0.1:8650 \
  --workers 20 \
  --sender-base addr:soak_6m_$(date +%Y%m%d_%H%M%S) \
  --soak --tps 6000000 --duration 72h \
  --batch-size 60000 \
  --mine-range \
  --sign --sig-scheme ed25519 \
  --progress-every 50 \
  --lat-samples 50000
```

Notes:
- If you re-run with the *same* sender, either change `--sender-base` or add
  `--nonce-sync` so it resumes at the chain nonce.
- For maximum throughput at the cost of durability, set `DAG_LOG_FSYNC_EVERY=0`.

---

## Addendum: extreme low-BPS probes (2026-02-15)

These runs push the native `miner_mineRangeSigned` lane into *very* large range-block
sizes (millions to billions of range-ops per block). This is still a **valid**
block-signed lane (Ed25519 per block), but it is **not** per-tx signature verification
and it is **not** EVM.

Key identity to keep in mind:

`range_ops_per_sec ~= blocks_per_sec * ops_per_block`

### Probe: ~2.5B range-ops/sec (1 worker)

Server profile:
- `GO_RANGE_STATE=1`
- `DAG_FAST=1`
- `DAG_BENCH=1`
- `DAG_AUTO_RUN=0`
- `DAG_STORE=log`
- `DAG_PERSIST_BLOCKS=1`
- `DAG_RATE_LIMIT_ENABLED=0`
- `DAG_TRACK_RECENT=0`
- `DAG_SIG_STATS=0`
- `DAG_RANGE_KEY_MOD=200000`
- `DAG_MAX_RANGE_COUNT=1000000000` (must be >= client `--batch-size`)
- `DAG_LOG_FSYNC_EVERY=5` (reduced IO spikes)
- `DAG_CHECKPOINT_EVERY=25000` (avoid checkpointing too frequently at low BPS)

Client:
- `--workers 1`
- `--soak --duration 60s`
- `--batch-size 1000000000` (1B ops/block)
- `--tps 10000000000` (target; achieved is limited by block apply time)
- `--mine-range --sign --sig-scheme ed25519`

Result (60s):
- `tx_per_second`: **2,502,563,319.59**
- `blocks_per_second`: **2.50**
- `avg_txs_per_block`: **1,000,000,000**
- `p99`: **~0.407s**

Interpretation:
- At 1B ops/block the system becomes *block-apply-time bound*: BPS collapses to ~2.5,
  but peak range-ops/sec remains very high.

### 10-hour durability run: ~2.48B range-ops/sec (1 worker)

Profile used:
- `./run_dag_profile_lowbps.sh`
  - `GO_RANGE_STATE=1`
  - `DAG_MAX_RANGE_COUNT=1000000000`
  - `DAG_LOG_FSYNC_EVERY=5`
  - `DAG_CHECKPOINT_EVERY=1500`

Client:
- `--workers 1`
- `--soak --duration 10h`
- `--tps 10000000000`
- `--batch-size 1000000000`
- `--mine-range --sign --sig-scheme ed25519`

Result (10h):
- `elapsed_sec`: **36000.186634**
- `txs_committed`: **89,439,000,000,000**
- `blocks_mined`: **89,439**
- `tx_per_second`: **2,484,403,786.85**
- `blocks_per_second`: **2.48**
- `avg_txs_per_block`: **1,000,000,000.00**
- `p50`: **0.391003s**
- `p90`: **0.402394s**
- `p99`: **0.409173s**

Interpretation:
- Stable long-run behavior in the huge-block / low-BPS operating mode.
- No late-run collapse visible in tail latency (p99 remained ~0.41s).

### Checkpoint cadence findings (stability)

Observed behavior in huge-block / low-BPS mode (1B ops/block):

- `DAG_CHECKPOINT_EVERY=25000` can cause heavy IO bursts and long pauses.
  - At ~2.5 BPS, 25k means one checkpoint roughly every ~2.8 hours.
  - Each checkpoint then has a very large snapshot/rotation workload, which can
    create multi-second to tens-of-seconds stalls on desktop/dev hosts.
- `DAG_CHECKPOINT_EVERY=1000..2000` spreads IO and smooths latency.
  - At ~2.5 BPS, this is ~6.7 to ~13.3 minutes per checkpoint.
  - In practice, this reduced severe stalls and produced steadier `last` times
    (for example ~0.3-0.4s instead of sporadic long spikes).

Practical recommendation for 1B/block lane:

- Use `DAG_CHECKPOINT_EVERY=1500` as a balanced default.
- Increase only if checkpoint overhead is proven low on the target hardware.

### Optional auto-checkpoint heuristic

You can now enable an adaptive checkpoint cadence:

- `DAG_CHECKPOINT_TARGET_SEC` (off when `0`)
- `DAG_CHECKPOINT_MIN` / `DAG_CHECKPOINT_MAX`
- `DAG_CHECKPOINT_TUNE_SEC`

Heuristic:

`checkpoint_every ~= observed_blocks_per_sec * DAG_CHECKPOINT_TARGET_SEC`

This keeps restart replay bounded while avoiding fixed-cadence IO hammering when
BPS changes across test profiles.
