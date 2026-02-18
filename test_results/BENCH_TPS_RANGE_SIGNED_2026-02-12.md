# Range-Signed TPS Bench (2026-02-12)

Goal: push the **native DAG lane** (`miner_mineRangeSigned`, ed25519) into a stable **3-5M TPS** region without disabling signature verification/quorum enforcement.

This bench is the "native throughput lane" (range-signed / range-root). It is not an EVM-style per-tx execution benchmark.

## Code change

Optimized the hot loop in `NodeRuntime.mine_incr_range()` to avoid per-tx function calls and per-iteration branching when:
- `mode=direct`
- `root_mode=range` (no per-tx hashing)
- `km==0` (unbounded keyspace)

File:
- `Documents/Vyrn-Chain/multi_dag_runtime.py`

## Bench configs + results

All runs used:
- RPC: `msgpack+http://127.0.0.1:8650`
- Method: `miner_mineRangeSigned`
- Signature scheme: `ed25519`
- Workers: `20`
- Total tx: `10,000,000`
- Progress output disabled: `--progress-every 0`

### A) Compute-only upper bound (memory store)

DAG start:
```bash
cd ~/Documents/Vyrn-Chain
pkill -f dag_rpc_server.py || true

DAG_BENCH=1 DAG_FAST=1 DAG_STORE=memory \
  DAG_PERSIST_BLOCKS=0 DAG_CHECKPOINT_EVERY=0 \
  DAG_TRACK_RECENT=0 DAG_SIG_STATS=0 DAG_RATE_LIMIT_ENABLED=0 \
  DAG_RANGE_KEY_MOD=0 DAG_MAX_RANGE_COUNT=100000 \
  NATIVE_SIG_VERIFY=1 NATIVE_SIG_REQUIRE=1 \
  DAG_ENFORCE_VALIDATOR_QUORUM=1 DAG_ENFORCE_PARENT_QUORUM=1 DAG_ENFORCE_CHILD_QUORUM=1 \
  ./run_dag_rpc.sh
```

Stress:
```bash
cd ~/Documents/Vyrn-Chain/go_dag
./stress --rpc msgpack+http://127.0.0.1:8650 \
  --workers 20 --txs-total 10000000 --batch-size 100000 \
  --mine-range --sign --sig-scheme ed25519 \
  --progress-every 0 --sender-base addr:stress_mem
```

Observed:
- **~4.3M TPS** with plain HTTP+JSON on this workload (example: **4,328,499 TPS** with `--batch-size 100000`).
- **~3.9M TPS** with `msgpack+http` (example run: **3,905,438 TPS**).

Note: on this specific benchmark path (compute-bound state mutation), JSON vs msgpack is not the primary limiter; msgpack can be slightly slower depending on client/server framing.

### B) Testnet-fast style durability (log store + persist blocks)

DAG start:
```bash
cd ~/Documents/Vyrn-Chain
pkill -f dag_rpc_server.py || true

DAG_BENCH=1 DAG_FAST=1 DAG_STORE=log \
  DAG_PERSIST_BLOCKS=1 DAG_BLOCK_FSYNC=0 \
  DAG_CHECKPOINT_EVERY=0 DAG_LOG_FSYNC_EVERY=0 \
  DAG_TRACK_RECENT=0 DAG_SIG_STATS=0 DAG_RATE_LIMIT_ENABLED=0 \
  DAG_RANGE_KEY_MOD=0 DAG_MAX_RANGE_COUNT=100000 \
  NATIVE_SIG_VERIFY=1 NATIVE_SIG_REQUIRE=1 \
  DAG_ENFORCE_VALIDATOR_QUORUM=1 DAG_ENFORCE_PARENT_QUORUM=1 DAG_ENFORCE_CHILD_QUORUM=1 \
  ./run_dag_rpc.sh
```

Stress:
```bash
cd ~/Documents/Vyrn-Chain/go_dag
./stress --rpc msgpack+http://127.0.0.1:8650 \
  --workers 20 --txs-total 10000000 --batch-size 100000 \
  --mine-range --sign --sig-scheme ed25519 \
  --progress-every 0 --sender-base addr:stress_log
```

Observed:
- **~3.3–3.5M TPS** (example runs: **3,317,916 TPS**, **3,475,449 TPS**)

### C) Long-soak-friendly bounded keyspace (km>0)

This keeps the synthetic bench lane from growing the in-memory state without bound during long soaks.
It is expected to reduce peak TPS vs `DAG_RANGE_KEY_MOD=0`.

DAG start:
```bash
cd ~/Documents/Vyrn-Chain
pkill -f dag_rpc_server.py || true

DAG_BENCH=1 DAG_FAST=1 DAG_STORE=memory \
  DAG_PERSIST_BLOCKS=0 DAG_CHECKPOINT_EVERY=0 \
  DAG_TRACK_RECENT=0 DAG_SIG_STATS=0 DAG_RATE_LIMIT_ENABLED=0 \
  DAG_RANGE_KEY_MOD=200000 DAG_MAX_RANGE_COUNT=15000 \
  NATIVE_SIG_VERIFY=1 NATIVE_SIG_REQUIRE=1 \
  DAG_ENFORCE_VALIDATOR_QUORUM=1 DAG_ENFORCE_PARENT_QUORUM=1 DAG_ENFORCE_CHILD_QUORUM=1 \
  ./run_dag_rpc.sh
```

Stress:
```bash
cd ~/Documents/Vyrn-Chain/go_dag
./stress --rpc http://127.0.0.1:8650 \
  --workers 20 --txs-total 2000000 --batch-size 15000 \
  --mine-range --sign --sig-scheme ed25519 \
  --progress-every 0 --sender-base addr:stress_km
```

Observed:
- **~2.3-2.4M TPS** (example: **2,384,922 TPS**) after optimizing the km>0 hot loop.

## Notes / caveats

- `--fast` is the native lane benchmark path (range-root + no per-tx receipts). EVM/WASM/BPF will remain much slower by design.
- `DAG_RANGE_KEY_MOD=0` grows state without bound. Use a bounded keyspace for long soaks, but expect lower peak TPS.
- Checkpointing/snapshots (`--checkpoint-every`) will introduce periodic stalls; tune separately from peak TPS tests.

## What "mineRangeSigned" commits (block semantics)

This lane produces a real committed parent block in the DAG:

- Parent height advances (visible in `dag_getTips` / `dag_getBlockMeta`).
- A `state_root` is committed into the parent block.
- Range authorization is persisted as `range_signed` metadata:
  - `{sender, start_nonce, count, delta, sig_scheme, pubkey, sig, digest}`
  - `digest` is domain-separated and deterministic (validators can recompute it).
- QC / finality material is produced in the same way as other parents (unless you run external-QC mode).

What it does NOT do (by design of this native throughput lane):

- It does not store per-tx bodies, per-tx receipts, or per-tx logs in the parent.
  - `dag_getBlockByHeight` will typically show `txs_len: 0` for these parents.
  - Use `dag_getBlockMeta(parent_id)` to inspect the `range_signed` material.

## State update semantics (and GO_RANGE_STATE)

This lane still applies state updates; it just applies them in a specialized way:

- The committed `state_root` is computed in "fast root" mode:
  - It is a rolling hash over per-branch diff roots (not a full merkle over all keys).
  - This keeps the lane deterministic and cheap at very high TPS.

Two execution backends exist for the actual key/value updates:

1) Python state map (default, `GO_RANGE_STATE=0`)
   - Updates are applied to the Python in-memory state (`StateExecutor._state`).
   - `rpc_sigStats.result.state_keys` will grow over time.

2) Go range-state store (`GO_RANGE_STATE=1`)
   - Updates are applied to a Go-backed store (ctypes -> `go_range_state/librangestate.so`).
   - The Python state map is bypassed on this lane (so `state_keys` stays 0).
   - `rpc_sigStats.result.go_range_state` reports:
     - `km` (key_mod / bounded keyspace size)
     - `keys` (number of applied entries)
     - `senders`

Important: GO range-state currently accelerates the "native fast lane" only. It does not
change EVM/WASM/BPF semantics.

## Verification / introspection

Quick live checks:

```bash
# Shows whether the Go range-state is active + how many keys it has applied
curl -s -X POST http://127.0.0.1:8650 -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"rpc_sigStats","params":[]}' | jq .result

# See range-signed metadata on the latest tip
curl -s -X POST http://127.0.0.1:8650 -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":2,"method":"dag_getTips","params":[]}' | jq .result

curl -s -X POST http://127.0.0.1:8650 -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":3,"method":"dag_getBlockMeta","params":["p00000XYZ"]}' | jq .result.range_signed
```

Durable signature verification (requires `DAG_PERSIST_BLOCKS=1`):

```bash
python3 verify_range_signed_block.py --data-dir chain_data
```
