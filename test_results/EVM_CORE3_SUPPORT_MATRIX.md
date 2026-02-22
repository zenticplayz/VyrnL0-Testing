# EVM Core-3 Support Matrix (Launch Contract)

This file defines "EVM 100%" for Vyrn's current launch target:

- Tooling target: **MetaMask + ethers.js + Hardhat (basic flows)**
- Policy: **unsupported methods fail closed with explicit JSON-RPC errors**
- Constraint: **native DAG lane performance must not regress**

## 1) Required method surface (Core-3)

These methods are in scope and must remain stable:

- `web3_clientVersion`
- `net_version`
- `eth_chainId`
- `eth_blockNumber`
- `eth_getBalance`
- `eth_getTransactionCount`
- `eth_call`
- `eth_estimateGas`
- `eth_sendRawTransaction`
- `eth_getTransactionByHash`
- `eth_getTransactionReceipt`
- `eth_getLogs`
- `eth_newFilter`
- `eth_getFilterChanges`
- `eth_getFilterLogs`
- `eth_uninstallFilter`
- `eth_newBlockFilter`
- `eth_newPendingTransactionFilter`

## 2) Explicitly unsupported (fail-closed)

These are intentionally not part of the launch EVM contract:

- `eth_subscribe` -> `-32601` with explicit websocket/pubsub disabled message.
- `eth_unsubscribe` -> `-32601` with explicit websocket/pubsub disabled message.
- `eth_sendTransaction` -> `-32000` with explicit "sign locally and use eth_sendRawTransaction".

## 3) Receipt/log/filter behavior guarantees

- Deterministic receipt/log resolution order:
  1. recent in-memory cache
  2. tx/log index fast path
  3. receipt store
  4. bounded WAL fallback
  5. bounded block scan fallback (if enabled)
- Filter lifecycle is bounded:
  - active filter cap
  - TTL expiry
  - bounded max changes per poll
- Runtime fail-closed when EVM lane is disabled:
  - `dag_primary=true` and `evm_force_main=false`

## 4) Conformance gate command

Run the full Core-3 gate:

```bash
cd ~/Documents/Vyrn-Chain
python3 test_evm_core3_conformance_v1.py
```

Optional live smoke on running RPC:

```bash
python3 test_evm_core3_conformance_v1.py \
  --rpc http://127.0.0.1:8545 \
  --admin-token secret123
```

## 5) Native-lane guardrail

Any EVM parity change must keep native lane isolated:

- no EVM coupling into DAG hot path state-apply loop
- no measurable native throughput regression beyond expected noise
