# Native Lane Unique-Address TPS Sweep (Signed Blocks) — 2026-02-18

## Test setup
- Lane: native DAG `miner_mineRangeSigned`
- Transport: `msgpack+unix:///tmp/dag_rpc.sock`
- Signing: `ed25519` (enabled)
- Workers: `1`
- Duration: `60s` each run
- Goal: measure max TPS with **unique sender rotation** (not single-sender hotpath)

## Results

| Run | Senders/worker | Batch size | Target TPS | TX committed | Blocks | Measured TPS | BPS | p50 | p90 | p99 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| A | 200,000 | 200,000,000 | 10,000,000,000 | 173,400,000,000 | 867 | 2,887,861,159.83 | 14.44 | 0.065917s | 0.067243s | 0.070036s |
| B | 1,000,000 | 200,000,000 | 10,000,000,000 | 138,600,000,000 | 693 | 2,308,364,584.25 | 11.54 | 0.066798s | 0.070735s | 0.075363s |
| C | 200,000 | 400,000,000 | 10,000,000,000 | 176,800,000,000 | 442 | 2,940,679,854.91 | 7.35 | 0.129585s | 0.131921s | 0.138603s |
| D | 200,000 | 600,000,000 | 10,000,000,000 | 178,800,000,000 | 298 | 2,970,391,581.13 | 4.95 | 0.192712s | 0.195863s | 0.203675s |
| E | 200,000 | 1,000,000,000 | 10,000,000,000 | 179,000,000,000 | 179 | 2,976,741,810.21 | 2.98 | 0.319305s | 0.329117s | 0.337133s |
| F (ceiling check) | 200,000 | 1,000,000,000 | 30,000,000,000 | 179,000,000,000 | 179 | 2,977,984,671.20 | 2.98 | 0.318849s | 0.328567s | 0.348331s |

## Summary
- Observed unique-address ceiling in this profile: **~2.98B TPS**.
- Raising target from `10B` to `30B` TPS did not increase throughput (same plateau), indicating saturation point reached.
- Increasing sender cardinality from `200k` to `1M` at `200M` batch lowered TPS to **~2.31B** (expected address/nonce/state pressure).
- Best publishable unique-address result in this sweep: **2,977,984,671 TPS** with signed blocks.

## Suggested publish line
“Unique-address native lane (signed blocks, no fake hotpath) sustained ~2.98B TPS in a 60s max sweep, with stable latency and deterministic block production.”
