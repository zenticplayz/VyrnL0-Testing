# Startup Method Speed Bench (Go stress, no signing)

This benchmark compares DAG startup profiles using the same Go stress workload.

## Workload (constant across all profiles)

```bash
cd ~/Documents/Vyrn-Chain/go_dag
go run ./cmd/stress \
  --rpc msgpack+http://127.0.0.1:8650 \
  --workers 20 \
  --txs-total 2000000 \
  --batch-size 5000 \
  --mine-range \
  --progress-every 0
```

- Signing: **off**
- Method: `miner_mineRange`
- Tx/block target: `5000`

## Profiles tested (7)

1. `secure_default_log_autorun`
2. `secure_log_no_autorun`
3. `secure_log_lean`
4. `bench_memory`
5. `bench_memory_persist_blocks`
6. `bench_fast_log`
7. `launch_fast_log_autorun`

## Results

| Profile | tx/s | blocks/s | elapsed (s) | p50 (s) | p90 (s) | p99 (s) |
|---|---:|---:|---:|---:|---:|---:|
| secure_default_log_autorun | 50,347.21 | 10.07 | 39.724147 | 2.023416 | 3.954364 | 5.163965 |
| secure_log_no_autorun | 50,677.24 | 10.14 | 39.465452 | 2.023728 | 3.749335 | 4.952872 |
| secure_log_lean | 50,584.87 | 10.12 | 39.537516 | 2.011277 | 3.812219 | 5.093984 |
| bench_memory | **2,050,935.49** | **410.19** | **0.975165** | 0.045410 | 0.068825 | 0.108418 |
| bench_memory_persist_blocks | 1,383,370.89 | 276.67 | 1.445744 | 0.069343 | 0.083815 | 0.117131 |
| bench_fast_log | 50,576.16 | 10.12 | 39.544322 | 2.049581 | 3.856604 | 5.056556 |
| launch_fast_log_autorun | 50,859.40 | 10.17 | 39.324101 | 2.036446 | 3.774359 | 4.918626 |

## Raw artifacts

- TSV: `bench_artifacts/startup_bench_results.tsv`
- Per-profile stress output: `bench_artifacts/*.stress.txt`
- Per-profile DAG logs: `chain_data/logs/dag_bench_*.log`

## Quick read

- Memory mode (`bench_memory`) is the clear max-throughput profile.
- Enabling block persistence in memory mode reduces throughput (~2.05M -> ~1.38M tx/s).
- Log-store profiles in this test shape cluster around ~50k tx/s.

## Go test status

```text
go test ./...   (from go_dag)
?    vyrn-go-dag             [no test files]
?    vyrn-go-dag/cmd/stress  [no test files]
ok   vyrn-go-dag/rawtx       (cached)
```
