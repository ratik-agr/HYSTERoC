# Experiments — evaluation protocol

This document defines how a HYSTERoC result is produced so that two runs are comparable. The *claims* live in the [README](../README.md) §2.3; this is the protocol.

---

## 1. Configuration points

Always run the full ablation matrix for any claimed result. `DUAL=1, USE_EWMA=1, THIGH=160, TLOW=96` (defaults in `tb/env/test.sv`) is HYSTERoC.

| # | Name | `DUAL` | `USE_EWMA` | Thresholds | `SWITCH_POLICY` |
|---|---|---|---|---|---|
| 0 | XY-only reference | — | — | `XY_ONLY=1` | 0 |
| 1 | **HYSTERoC** | 1 | 1 | THIGH=160, TLOW=96 | 0 |
| 2 | Ablation B | 1 | 0 | THIGH=160, TLOW=96 | 0 |
| 3 | Ablation A | 0 | 1 | THRESH_SINGLE=128 | 0 |
| 4 | Conventional hybrid | 0 | 0 | THRESH_SINGLE=128 | 0 |
| 1′ / 2′ | HYSTERoC / Ablation B, no-drain | 1 | 1 (0) | THIGH=160, TLOW=96 | **1** (`DET_VARIANT` = 0, 1, 2 reported separately) |

Reading: **1 vs 4** is the headline claim; **1 vs 2** isolates filtering; **1 vs 3** isolates the hysteresis band; **1′** is the FPGA-friendly variant (report its switch count and latency separately — the non-adaptive mode is deterministic Odd–Even, not XY).

## 2. Workloads

| Workload | Plusargs | Purpose |
|---|---|---|
| uniform | `--pat 0` uniform test, load sweep | baseline sustained congestion |
| hotspot | `--pat 1` | concentrated congestion (max-of-16 signal exercised) |
| transpose | `--pat 2` | structured permutation |
| complement | `--pat 3` | worst-case XY paths |
| neighbour | `--pat 4` | minimal-hop traffic |
| oscillating (square wave) | `hy_oscillate_test --load 400 --cycles 300` | **the** ping-pong criterion: sustained threshold crossings |
| phased (low→high→low) | `hy_phased_test --load 350` | hysteresis band traversal, both directions |
| load sweep | `hy_compare_test --lo 100 --hi 350 --step 25 --dwell 2000` | one configuration, many loads; plot latency & throughput vs load |
| all patterns | `hy_patterns_all_test --load 250` | no pattern-specific pathology |
| zero-load | `hy_zero_load_test` | latency floor = distance + 4 (check before sweeping) |

Loads are in per-mille flits/node/cycle (`--load 400` = 0.4). State every load in both units.

## 3. Metrics to report (from `performance_monitor`)

| Metric | Signal | Why |
|---|---|---|
| Completed switches | `w_switches` (split `w_to_adapt` / `w_to_xy`) | the headline HYSTERoC-vs-baseline number |
| **Ping-pong events** | `w_pp0…w_pp3` (dwell ≤ 50/100/200/400) | the failure mode being fixed; report pp100 and pp200 at minimum |
| Dwell time | `w_dwell_sum/cnt/min/max` | is the controller actually dwelling in each mode? |
| Blocked cycles + fraction | `w_blocked`, `w_blocked / measure` | transition overhead as a throughput loss |
| Drain length | `drain_cnt/sum/min/max` | worst-case injection stall per switch |
| Packet latency | mean/std/max of total and network latency | `w_lat_*` + exact percentiles from the testbench packet log |
| Throughput | `w_inj_flits` / `w_ej_flits` in the window | must be ≥ baseline at equal latency, not just fewer switches |
| Stall watchdog | `stall_max` | empirical liveness evidence (not a proof — say so) |

Report **rate-relative** numbers too: switches per 1000 packets, blocked fraction, ping-pong per 1000 switches.

## 4. Statistics

* Run each (configuration, workload) point at **≥ 3 seeds** (the `traffic_generator` PRNG differs per seed in the bench flow; the UVM randomized sequences differ run-to-run) and report mean ± range.
* Keep the measurement window *after* warmup (`warmup`/`measure` in `performance_monitor` parameters); never measure across a transition burst.
* For the oscillating workload, additionally record the congestion signal trajectory (`global_congestion`/`global_raw`, `tb/iface/hy_if` taps) so the band-crossing argument is visible, not just the switch count.

## 5. Reproducibility checklist (attach to every result table)

- [ ] Commit SHA (+ `git status` clean) and `sim/` file lists used
- [ ] Simulator + version; DUT parameterization (`policy/det/dual/ewma/thigh/tlow/xy-only`)
- [ ] Test name and every plusarg, incl. `--load/--cycles/--pat/--timeout`
- [ ] Seed(s), warmup/measure cycles, number of repetitions
- [ ] `performance_monitor` raw counters (not just derived means)
- [ ] Unexpected ERROR / timeout / watchdog state: reported, never omitted

## 6. What would make the evaluation convincing

1. The headline comparison (1 vs 4) on the oscillating workload: **fewer switches and fewer ping-pong events at equal-or-better latency and throughput.** Fewer switches with worse latency is a failed claim, not a result.
2. The ablation ordering (2 and 3 between 1 and 4, in the direction predicted by §1) — a result where disabling either mechanism beats the full design is a bug or a mis-calibrated threshold, not a finding.
3. Policy 1′ at its best `DET_VARIANT` with the in-flight switch requirement observed (`hy_nodrain_test` green) — no drain stalls, comparable switch behaviour.
4. The watchdog test proving failure is *safe* (ERROR, no forced switch) under a misbehaving source.
