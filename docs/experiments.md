# Experiments — Evaluation Protocol

This document defines the evaluation methodology for HYSTERoC. Follow this protocol to produce comparable experimental results across different runs.

---

## 1. Configuration points

Evaluate the complete ablation matrix for all reported experimental data. The default configuration in `tb/env/test.sv` represents HYSTERoC (`DUAL=1, USE_EWMA=1, THIGH=160, TLOW=96`).

| # | Name | `DUAL` | `USE_EWMA` | Thresholds | `SWITCH_POLICY` | Evaluation Purpose |
|---|---|---|---|---|---|---|
| 0 | XY reference | — | — | `XY_ONLY=1` | 0 | Baseline deterministic routing without adaptation. |
| 1 | **HYSTERoC** | 1 | 1 | THIGH=160, TLOW=96 | 0 | The proposed hysteresis architecture. |
| 2 | Ablation B | 1 | 0 | THIGH=160, TLOW=96 | 0 | Dual thresholds on raw signals (isolates filtering). |
| 3 | Ablation A | 0 | 1 | THRESH_SINGLE=128 | 0 | Single threshold on filtered signals (isolates hysteresis). |
| 4 | Conventional hybrid | 0 | 0 | THRESH_SINGLE=128 | 0 | Conventional hybrid baseline. |
| 1′ | HYSTERoC no-drain | 1 | 1 | THIGH=160, TLOW=96 | 1 | FPGA-friendly variant without drain delays (`DET_VARIANT` = 0, 1, 2). |

Comparison objectives:
* Compare **Point 1 against Point 4** to demonstrate the primary benefit of HYSTERoC.
* Compare **Point 1 against Point 2** to evaluate EWMA filter performance.
* Compare **Point 1 against Point 3** to evaluate the dual-threshold hysteresis band.
* Report **Point 1′** separately to show the throughput benefit of the no-drain variant.

---

## 2. Workloads

| Workload | Options and Flags | Purpose |
|---|---|---|
| Uniform | `--pat 0` with load sweep | Measures baseline sustained traffic. |
| Hotspot | `--pat 1` | Exercises maximum congestion detection across nodes. |
| Transpose | `--pat 2` | Measures structured permutation traffic. |
| Complement | `--pat 3` | Stresses longest-path routing in the mesh. |
| Neighbour | `--pat 4` | Evaluates single-hop nearest-neighbour traffic. |
| Oscillating (square wave) | `hy_oscillate_test --load 400 --cycles 300` | Tests ping-pong reduction under oscillating congestion. |
| Phased (low → high → low) | `hy_phased_test --load 350` | Tests transitions across hysteresis thresholds in both directions. |
| Load sweep | `hy_compare_test --lo 100 --hi 350 --step 25 --dwell 2000` | Measures latency and throughput across offered load levels. |
| All patterns | `hy_patterns_all_test --load 250` | Verifies stability across all traffic patterns. |
| Zero load | `hy_zero_load_test` | Confirms minimum latency equals `distance + 4`. |

Specify loads in units of per-mille flits per node per cycle (`--load 400` equals 0.4 flits/node/cycle).

---

## 3. Reported metrics (`performance_monitor`)

Record these metrics from the `performance_monitor` module:

| Metric | Signal Name | Description |
|---|---|---|
| Completed switches | `w_switches` (`w_to_adapt`, `w_to_xy`) | Total count of completed mode transitions. |
| Ping-pong events | `w_pp0` to `w_pp3` | Switches occurring within 50, 100, 200, or 400 cycles of previous switch. |
| Dwell time | `w_dwell_sum`, `min`, `max` | Duration the network remains in a mode before switching. |
| Blocked cycles | `w_blocked` | Cycles during which `inject_block_req` is asserted. |
| Blocked fraction | `w_blocked / measure` | Proportion of measurement window lost to drain blocks. |
| Drain duration | `drain_cnt`, `sum`, `min`, `max` | Cycles needed to drain all packets before switching. |
| Packet latency | `w_lat_*` | Mean, standard deviation, and maximum total and network latency. |
| Delivered throughput | `w_ej_flits` | Flits accepted at ejection sinks within measurement window. |
| Stall watchdog | `stall_max` | Maximum consecutive cycles with in-flight packets and zero deliveries. |

Always calculate rate-relative values:
* Switches per 1,000 delivered packets.
* Injection-blocked time fraction.
* Ping-pong events per 1,000 mode switches.

---

## 4. Statistical methodology

* Execute each configuration with **at least three distinct random seeds**. Report mean values and error ranges.
* Exclude the warmup period from all reported statistics. Measure only during the configured measurement window.
* For oscillating workloads, log the congestion trajectories (`global_congestion` and `global_raw`) to show band crossings.

---

## 5. Reproducibility checklist

Include these details in all reported experiment summaries:
- [ ] Git commit SHA and status of working tree.
- [ ] Simulator version and synthesis/compilation flags.
- [ ] Top-level parameters (`policy`, `det`, `dual`, `ewma`, `thigh`, `tlow`).
- [ ] Test name and all plusarg parameters (`+HY_LOAD=`, `+HY_CYCLES=`, `+HY_PAT=`).
- [ ] Random seed values, warmup length, and measurement length.
- [ ] Raw performance counter values from `performance_monitor`.
- [ ] Status of watchdog alerts or error indicators.

---

## 6. Evaluation success criteria

A successful experimental evaluation must show:
1. **Fewer switches and fewer ping-pong events** for HYSTERoC compared to the conventional hybrid baseline at equal or better latency and throughput.
2. **Predictable ablation trends**: Point 2 and Point 3 must perform between Point 1 and Point 4.
3. **No-drain efficiency**: Point 1′ must eliminate drain delays while maintaining safe packet routing.
4. **Safe timeout recovery**: The watchdog test must transition cleanly to `ERROR` without deadlock when sources stall.
