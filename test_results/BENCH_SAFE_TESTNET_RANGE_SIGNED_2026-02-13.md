# Safe Testnet Bench — Range-Signed Native Lane (2026-02-13)

Goal: measure "high-speed but testnet-defensible" throughput for the native DAG lane
using `miner_mineRangeSigned` (one signature per block/range) with msgpack transport.

This bench is for the *native fast lane only* (range-root + no per-tx receipts/bodies).
EVM/WASM/BPF lanes will remain much slower by design.

## Profile (safe testnet-ish)

DAG process (isolated port + isolated data dir under `/tmp`):

- `--store log` (WAL-style log store)
- `--persist-blocks` (writes parent blocks to disk)
- `--checkpoint-every 100` (snapshots)
- `--fast` (native lane)
- rate limit disabled for throughput measurement (`DAG_RATE_LIMIT_ENABLED=0`)
- wallet/UX extras off (`DAG_TRACK_RECENT=0`, `DAG_SIG_STATS=0`)
- strict signature gating on (`NATIVE_SIG_VERIFY=1`, `NATIVE_SIG_REQUIRE=1`)
- external batch submit capped low (`DAG_MAX_TX_PER_BATCH=5000`)
- range mining cap independent (`DAG_MAX_RANGE_COUNT=15000`)
- Go range-state enabled (`GO_RANGE_STATE=1`)

Note: in `DAG_BENCH=1`, `run_dag_rpc.sh` defaults `DAG_RANGE_KEY_MOD=200000` to keep
state bounded during long runs.

## Stress tool

Go stress (`go_dag/stress`) with:

- `--rpc msgpack+http://127.0.0.1:18650`
- `--mine-range` (calls `miner_mineRangeSigned`)
- `--sign --sig-scheme ed25519`
- mineRange opts used by go-stress: `mode=direct root_mode=range assume_new=true`

## Results (20 workers, 400 blocks)

### range=5,000 tx per block (2,000,000 tx total)

1) `DAG_LOG_FSYNC_EVERY=0`
- TPS: 2,821,970.89
- blocks/s: 564.39
- elapsed: 0.708725s

2) `DAG_LOG_FSYNC_EVERY=1` (fsync per parent)
- TPS: 2,093,813.80
- blocks/s: 418.76
- elapsed: 0.955195s

### range=15,000 tx per block (6,000,000 tx total)

1) `DAG_LOG_FSYNC_EVERY=0`
- TPS: 8,553,061.10
- blocks/s: 570.20
- elapsed: 0.701503s

2) `DAG_LOG_FSYNC_EVERY=1` (fsync per parent)
- TPS: 6,350,828.55
- blocks/s: 423.39
- elapsed: 0.944759s

### Larger blocks (fsync per parent)

These runs used `DAG_MAX_RANGE_COUNT=50000` so the miner can commit larger ranges.

1) range=30,000 tx per block (6,000,000 tx total; 200 blocks)
- TPS: 12,835,329.48
- blocks/s: 427.84
- elapsed: 0.467460s

2) range=50,000 tx per block (10,000,000 tx total; 200 blocks)
- TPS: 21,680,986.46
- blocks/s: 433.62
- elapsed: 0.461234s

## Commands used

Start (no fsync):

```bash
cd ~/Documents/Vyrn-Chain
DATA_DIR=/tmp/vyrn_bench_safe_range15k \
DAG_HOST=127.0.0.1 DAG_PORT=18650 DAG_STORE=log DAG_FAST=1 DAG_BENCH=1 \
GO_RANGE_STATE=1 DAG_RATE_LIMIT_ENABLED=0 DAG_TRACK_RECENT=0 DAG_SIG_STATS=0 \
DAG_PERSIST_BLOCKS=1 DAG_CHECKPOINT_EVERY=100 DAG_LOG_FSYNC_EVERY=0 \
DAG_MAX_TX_PER_BATCH=5000 DAG_MAX_RANGE_COUNT=15000 \
./run_dag_rpc.sh
```

Bench (15k/block):

```bash
cd ~/Documents/Vyrn-Chain/go_dag
./stress --rpc msgpack+http://127.0.0.1:18650 \
  --workers 20 \
  --txs-total 6000000 \
  --batch-size 15000 \
  --mine-range \
  --sign --sig-scheme ed25519
```

Start (fsync per parent):

```bash
cd ~/Documents/Vyrn-Chain
DATA_DIR=/tmp/vyrn_bench_safe_range15k_fsync \
DAG_HOST=127.0.0.1 DAG_PORT=18650 DAG_STORE=log DAG_FAST=1 DAG_BENCH=1 \
GO_RANGE_STATE=1 DAG_RATE_LIMIT_ENABLED=0 DAG_TRACK_RECENT=0 DAG_SIG_STATS=0 \
DAG_PERSIST_BLOCKS=1 DAG_CHECKPOINT_EVERY=100 DAG_LOG_FSYNC_EVERY=1 \
DAG_MAX_TX_PER_BATCH=5000 DAG_MAX_RANGE_COUNT=15000 \
./run_dag_rpc.sh
```

## Note on caps (bugfix)

`dag_rpc_server._cap_method_params()` now caps `miner_mineRange*` by
`DAG_MAX_RANGE_COUNT` (not `DAG_MAX_TX_PER_BATCH`). This keeps external batch-submit
strict while allowing larger native range-signed blocks safely.
