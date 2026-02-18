# Stability & Durability Notes (A)

This is a practical checklist + scripts for keeping the stack stable and recoverable.

---

## 1) What stability means in Vyrn
- **No hangs** on health probes
- **Predictable restart** (no surprise rehydrate from 0)
- **No data loss** after crash (WAL + checkpoint)
- **Clear recovery steps**

---

## 2) Primary components to monitor
- **Main RPC** (8545) – handles writes, WAL, receipts
- **DAG RPC** (8650) – fast solver, tips
- **L2 RPC** (8660) – batching/bridge
- **Go Gateway** (9545) – optional single ingress

---

## 3) Restart & recovery (safe)
Use the new script (below) or do this manually:

1. Kill stale ports
2. Restart stack
3. Verify health

### Manual:
```bash
sudo fuser -k 9545/tcp 8545/tcp 8650/tcp 8660/tcp || true
./stop_stack.sh || true
./run_stack.sh
```

### Verify:
```bash
curl -s -X POST http://127.0.0.1:8545 \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"rpc_health","params":[]}'
```

---

## 4) WAL / checkpoint durability (main RPC)
Main RPC uses WAL + optional checkpointing.

Useful env flags:
- `WAL_ENABLED=1`
- `WAL_CHECKPOINT_EVERY=200` (or 100)
- `WAL_FSYNC_EVERY=0` (fast) or `1` (durable)
- `WAL_BUFFER_BYTES=1048576`
- `WAL_FLUSH_MS=5`

Checkpoint info:
```bash
python3 - <<'PY'
import json,urllib.request
rpc='http://127.0.0.1:8545'
body=json.dumps({"jsonrpc":"2.0","id":1,"method":"checkpoint_info","params":[]}).encode()
req=urllib.request.Request(rpc,data=body,headers={'Content-Type':'application/json'})
print(urllib.request.urlopen(req).read().decode())
PY
```

---

## 5) DAG snapshot / rehydrate
DAG uses `dag_parents.log` + `.snap` for fast rehydrate. The snapshot file should be ahead of log for fast boot.

Snapshot info:
```bash
curl -s -X POST http://127.0.0.1:8650 \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"snapshot_info","params":[{"auth":"secret123"}]}'
```

---

## 5.1) DAG txpool durability (accepted txs survive restart)
By default the DAG txpool lives in RAM (mempool + deferred nonces). This means a restart
can lose **accepted-but-not-yet-mined** txs unless we persist them.

Durable txpool is implemented via a small snapshot + journal colocated with the DAG checkpoint.

Enable (default):
```
DAG_TXPOOL_PERSIST=1
```

Tune durability vs speed:
```
DAG_TXPOOL_FLUSH_MS=5          # group commit window (ms)
DAG_TXPOOL_BUFFER_BYTES=262144
DAG_TXPOOL_FSYNC_EVERY=0       # 0=fast, 1=stronger durability
```

Opt out (microbench only):
```
DAG_TXPOOL_PERSIST=0
```

Files (relative to the DAG checkpoint path):
- `*.txpool.snap.mpk` — snapshot (NextNonce + mempool + deferred)
- `*.txpool.log` — journal (tx ingress + commits since last snapshot)

---

## 6) Health reliability
**Default behavior:** `rpc_health` is cached (won’t block on DAG or disk).

Optional flags:
- `HEALTH_USE_CHAIN_HEAD=1` (includes actual chain head; may block)
- `HEALTH_USE_DAG=1` (includes cached DAG tip)

---

## 7) Hangs / slow requests (watchdog)
The RPC now logs slow/hung handlers:

- `RPC_WATCHDOG=1`
- `RPC_SLOW_MS=2000`
- `RPC_HANG_MS=8000`
- `RPC_HANG_DUMP=1`

When a hang happens you’ll see:
```
[watchdog] slow RPC: method=state_getBalance ...
[watchdog] RPC hang: method=tx_getTransactionReceipt ...
```

---

## 8) Scripts added
- `scripts/restart_stack_safe.sh` – kills stale ports + restarts
- `scripts/snapshot_status.sh` – shows main + DAG checkpoint info

---

## 9) Recommended baseline (fast + safe)
```
WAL_ENABLED=1
WAL_CHECKPOINT_EVERY=200
WAL_FSYNC_EVERY=0
WAL_BUFFER_BYTES=1048576
WAL_FLUSH_MS=5
STATE_CACHE_TTL=1.0
MEMPOOL_CACHE_TTL=1.0
RPC_WATCHDOG=1
RPC_SLOW_MS=2000
RPC_HANG_MS=8000
```

---

## 10) Next stability step ideas
- **Process supervisor** (systemd or watchdog) for auto‑restart
- **Gateway fail‑fast**: return error if main RPC is hung
- **Health fan‑out**: return cached state if backend is slow
