# Verification — HYSTERoC UVM Testbench

This document describes the UVM 1.2 verification environment for the HYSTERoC network-on-chip. The testbench verifies `noc_top` across the complete 4×4 mesh with active injection and ejection agents.

---

## 1. Interface timing contract (`hy_if`)

All testbench components must follow this timing rule:
* Drive signals using non-blocking assignments (`<=`) immediately after `@(posedge clk)`.
* Sample signals in the active region of `@(posedge clk)`.

This rule matches what registers capture on each clock edge. It prevents race conditions across different simulators.

The interface model defines node status signals:

```systemverilog
src_empty[n]        = !(src_busy[n] | src_force_busy[n]);
inject_block_ack[n] = inject_block_req & src_empty[n];
```

* `src_busy[n]`: Asserted by inject driver while a packet is transferring.
* `src_force_busy[n]`: Testbench control bit. Prevents acknowledgment to trigger the watchdog timer (`hy_watchdog_test`).
* `test_rst_n`: Testbench control bit for mid-test resets (`hy_reset_test`).

Ejection ready (`ej_ready`) must not depend combinationally on `ej_valid`. The sink driver uses registered random logic.

---

## 2. Running tests

Execute test scripts with VCS, Xcelium, or Questa:

```bash
./scripts/run_uvm.sh --sim vcs   --test hy_smoke_test
./scripts/run_uvm.sh --sim xrun  --test hy_oscillate_test --load 400 --cycles 300
./scripts/run_uvm.sh --sim questa --test hy_nodrain_test --policy 1 --det 1
```

Common command-line arguments:

| Argument | Description |
|---|---|
| `--test <name>` | Selects test name (`+UVM_TESTNAME=`). See §5. |
| `--policy 0\|1` | Selects `SWITCH_POLICY` (0 = drain-and-switch, 1 = no-drain). |
| `--det 0\|1\|2` | Selects `DET_VARIANT` (0 = `oe_fixed`, 1 = `oe_par`, 2 = `oe_x21`). |
| `--thigh <val>` | Sets high hysteresis threshold (default: 160). |
| `--tlow <val>` | Sets low hysteresis threshold (default: 96). |
| `--dual 0\|1` | Enables or disables dual-threshold mode. |
| `--ewma 0\|1` | Enables or disables EWMA filter. |
| `--load <val>` | Sets offered load in flits/node/cycle × 1000 (`+HY_LOAD=`). |
| `--cycles <val>`| Sets duration in clock cycles (`+HY_CYCLES=`). |
| `--timeout <val>`| Sets maximum cycles before test abort (default: 3,000,000). |
| `--campaign` | Executes all tests in sequence and reports status. |

Pass criteria: Test completes with `UVM_ERROR = 0` and `UVM_FATAL = 0`.

---

## 3. Testbench components

| File | Class / Scope | Function |
|---|---|---|
| `tb/iface/hy_if.sv` | `hy_if` | Physical interface to DUT. Exposes links and error signals. |
| `tb/pkg/hy_types.svh` | Functions | Constants, flit definitions, and header conversion functions. |
| `tb/pkg/hy_item.svh` | `hy_item` | Sequence item representing network packets. |
| `tb/pkg/hy_cfg.svh` | `hy_cfg` | Configuration object for sink modes, knobs, and checks. |
| `tb/pkg/hy_route.svh` | Functions | Golden routing calculation functions for testbench checks. |
| `tb/env/hy_sb_core.svh` | `hy_sb_core` | Core reassembly, ordering checks, and packet verification. |
| `tb/pkg/hy_traffic.svh` | Functions | Helper routines to generate traffic patterns. |
| `tb/env/hy_model.svh` | `hy_model` | Cycle-exact reference model for all control registers and logic. |
| `tb/pkg/hy_sequencer.svh` | `hy_vseqr` | Virtual sequencer executing virtual sequences. |
| `tb/agents/hy_inject_driver.svh` | `hy_inject_driver` | Injects flits into the 16 router input ports. |
| `tb/agents/hy_monitors.svh` | Monitors | Monitors packet transfers at injection and ejection ports. |
| `tb/agents/hy_sink_driver.svh` | `hy_sink_driver` | Controls `ej_ready` with random, burst, or continuous readiness. |
| `tb/agents/hy_hop_monitor.svh` | `hy_hop_monitor` | Monitors router links. Validates hop legality against turn rules. |
| `tb/env/hy_scoreboard.svh` | `hy_scoreboard` | Validates delivery count, destination, order, and checksum. |
| `tb/env/hy_protocol_checker.svh` | `hy_protocol_checker` | Validates transition rules, drain invariants, and timeouts. |
| `tb/env/hy_coverage.svh` | `hy_coverage` | Functional coverage for modes, thresholds, and patterns. |
| `tb/agents/hy_agents.svh` | Agents | UVM agent wrappers for drivers and monitors. |
| `tb/env/hy_sva.sv` | `hy_sva` | Concurrent SystemVerilog assertions for interface rules. |
| `tb/env/test.sv` | Module | Testbench top module: clock, reset, DUT, and configuration. |

---

## 4. Concurrent assertions (`hy_sva`)

Assertions verify protocol rules during simulation. Failures increment `vif.sva_errors`.

| Assertion | Policy | Verified Invariant |
|---|---|---|
| `a_blk_implies_busy` | Both | `inject_block_req` implies `transition_busy`. |
| `a_state_legal` | Both | FSM state does not equal invalid state 7. |
| `a_timeout_is_error` | Both | `drain_timeout` is asserted if and only if state is `ERROR`. |
| `a_mode_change_only_after_switch` | 0 | `active_mode` changes only in `SWITCH` state when drained. |
| `a_mode_change_sources_idle` | 0 | Mode changes only when all sources are idle. |
| `a_start_only_from_run` | 0 | Transitions begin only from `RUN` when request differs from active. |
| `a_block_rises_from_run` | 0 | Injection block asserts only from `RUN`. |
| `a_release_in_resume` | 0 | Injection block deasserts only in `RESUME`. |
| `a_error_sticky` | 0 | `ERROR` state persists until system reset. |
| `a_no_inject_in_drain_switch` | 0 | Injection `valid` remains zero during `DRAIN` and `SWITCH`. |
| `a_nd_run_only` | 1 | No-drain mode remains in `RUN`. Block and timeout stay low. |
| `a_nd_mode_follows_req` | 1 | No-drain mode updates `active_mode` one cycle after request. |
| `a_drained_means_empty` | Both | `network_drained` implies all FIFOs and sources are empty. |
| `a_no_internal_errors` | Both | Internal protocol and allocation errors stay zero. |
| `a_no_x_control` | Both | Control and handshake signals contain no X or Z values. |
| `a_ej_hold` | Both | Ejection outputs hold data until accepted by the receiver. |

---

## 5. Test suite catalogue

| Test Name | Description |
|---|---|
| `hy_smoke_test` | Sends one packet from node 0 to node 15. |
| `hy_zero_load_test` | Verifies latency equals `distance + 4` in an empty mesh. |
| `hy_all_pairs_test` | Sends packets across all 240 source-destination pairs. |
| `hy_flow_test` | Transmits 200 packets along a single flow to verify ordering. |
| `hy_uniform_test` | Evaluates uniform random traffic under specified load. |
| `hy_hotspot_test` | Evaluates traffic directed toward a single node. |
| `hy_transpose_test` | Evaluates matrix transpose traffic pattern. |
| `hy_phased_test` | Sweeps load low → high → low to cross hysteresis thresholds. |
| `hy_oscillate_test` | Alternates load in a square wave to test ping-pong suppression. |
| `hy_nodrain_test` | Verifies in-flight mode changes under `SWITCH_POLICY = 1`. |
| `hy_compare_test` | Sweeps load across multiple steps to measure latency and switches. |
| `hy_patterns_all_test` | Executes all traffic patterns under heavy load. |
| `hy_lengths_test` | Transmits packets with lengths between 2 and 16 flits. |
| `hy_loopback_test` | Sends packets where source equals destination. |
| `hy_bubble_test` | Injects packets with intra-packet delays and stalled sinks. |
| `hy_backpressure_test` | Dynamically changes sink ready behaviour during transmission. |
| `hy_watchdog_test` | Simulates an unacknowledged source to verify `ERROR` timeout. |
| `hy_reset_test` | Applies reset during active packet transmission. |
| `hy_regress_test` | Runs the complete sequence of tests in a single simulation. |

---

## 6. Verification procedure

1. Write sequences in `tb/seq/`.
2. Add new test classes in `tb/tests/hy_tests_pkg.sv` by extending `hy_base_test`.
3. Read configuration knobs through `plus("HY_LOAD", 100)` instead of fixed constants.
4. Set `cfg.expect_drain_timeout = 1` for tests that intentionally trigger watchdogs.
5. Set `cfg.require_inflight_switch = 1` for tests that require in-flight mode switches.
6. Execute the test using `./scripts/run_uvm.sh`.

---

## 7. Sign-off checklist

Before marking a configuration complete, confirm:
- [ ] `hy_smoke_test`, `hy_zero_load_test`, and `hy_all_pairs_test` pass without errors.
- [ ] `hy_oscillate_test` passes under both policies (`SWITCH_POLICY = 0` and `1`).
- [ ] `hy_nodrain_test` passes and observes an in-flight switch under `SWITCH_POLICY = 1`.
- [ ] `hy_watchdog_test` reaches the `ERROR` state safely without hanging under `SWITCH_POLICY = 0`.
- [ ] `hy_reset_test` and `hy_regress_test` complete successfully.
- [ ] Functional coverage bins for modes, threshold bands, and patterns are covered.
- [ ] Performance metrics match expected values in [docs/experiments.md](experiments.md).
