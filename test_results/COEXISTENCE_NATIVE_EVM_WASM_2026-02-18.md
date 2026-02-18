# Coexistence Test: Native + EVM + WASM (2026-02-18)

## Goal
Run all three lanes for documentation:
- Native DAG lane (`miner_mineRangeSigned`)
- EVM contract lane
- WASM contract lane
- Coexistence run (Native + EVM/WASM mixed load at the same time)

## Environment
- DAG RPC: `http://127.0.0.1:8650`
- Main RPC: `http://127.0.0.1:8545`
- Stress tool: `go_dag/stress`
- Contract RPC bench: `bench_contract_rpc.py`

---

## 1) Native-only (signed range blocks)

Command:
```bash
cd ~/Documents/Vyrn-Chain/go_dag
./stress \
  --rpc msgpack+http://127.0.0.1:8650 \
  --workers 1 \
  --sender-base addr:doc_native_ok_$(date +%Y%m%d_%H%M%S) \
  --txs-total 640000 \
  --batch-size 8000 \
  --mine-range \
  --sign --sig-scheme ed25519 \
  --progress-every 20 \
  --lat-samples 20000
```

Result:
- `txs_committed`: `640000`
- `blocks_mined`: `80`
- `tx_per_second`: `16114.43`
- `blocks_per_second`: `2.01`
- `avg_txs_per_block`: `8000`
- `p50`: `0.074695s`
- `p90`: `0.083579s`
- `p99`: `7.424768s`

Note: separate longer native runs hit intermittent nonce rejection after ~800k tx (`bad nonce (want 800000, got 808000)`), so this specific run was capped below that edge.

---

## 2) EVM-only contract calls

Command:
```bash
cd ~/Documents/Vyrn-Chain
python3 bench_contract_rpc.py \
  --rpc http://127.0.0.1:8545 \
  --engine evm \
  --duration 60 \
  --workers 4 \
  --admin-token secret123
```

Result:
- `calls_ok`: `91069`
- `calls_err`: `0`
- `calls_per_sec`: `1517.78`
- `p50`: `2.607ms`
- `p90`: `3.291ms`
- `p99`: `4.001ms`

Status: stable.

---

## 3) WASM-only contract calls

Command:
```bash
cd ~/Documents/Vyrn-Chain
python3 bench_contract_rpc.py \
  --rpc http://127.0.0.1:8545 \
  --engine wasm \
  --duration 60 \
  --workers 4 \
  --admin-token secret123
```

Result:
- `calls_ok`: `0`
- `calls_err`: `66616`
- `calls_per_sec`: `0`
- top error: `WASM_EXEC_ERROR`

Observed runtime error body:
- `list indices must be integers or slices, not tuple`

Status: currently failing under this path; requires WASM runtime fix before throughput claim on WASM lane.

---

## 4) Coexistence run (Native + mixed EVM/WASM)

Native command (background):
```bash
cd ~/Documents/Vyrn-Chain/go_dag
./stress \
  --rpc msgpack+http://127.0.0.1:8650 \
  --workers 1 \
  --sender-base addr:doc_coexist_ok_$(date +%Y%m%d_%H%M%S) \
  --txs-total 640000 \
  --batch-size 8000 \
  --mine-range \
  --sign --sig-scheme ed25519 \
  --progress-every 20 \
  --lat-samples 20000
```

Mixed contract command (parallel):
```bash
cd ~/Documents/Vyrn-Chain
python3 bench_contract_rpc.py \
  --rpc http://127.0.0.1:8545 \
  --engine mix \
  --duration 30 \
  --workers 8 \
  --admin-token secret123
```

Coexistence results:

Native side:
- `txs_committed`: `640000`
- `blocks_mined`: `80`
- `tx_per_second`: `43917.78`
- `blocks_per_second`: `5.49`
- `p50`: `0.075754s`
- `p99`: `0.089261s`

Mix side:
- `calls_ok`: `18700`
- `calls_err`: `18871`
- `calls_per_sec`: `623.27`
- `p50`: `5.229ms`
- `p90`: `7.051ms`
- `p99`: `8.920ms`
- top error: `WASM_EXEC_ERROR` (all observed errors)

Interpretation:
- Coexistence itself is working (native and EVM continue processing under simultaneous load).
- Current limiting issue is WASM execution reliability, not EVM.

---

## Summary
- Native lane: operational with signed range blocks.
- EVM lane: stable and benchmarkable.
- WASM lane: currently erroring (`WASM_EXEC_ERROR`) and needs fix before meaningful TPS/call-rate claims.
- Coexistence: valid for Native + EVM; mixed mode currently inherits WASM failures.
