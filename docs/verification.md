# Verification — HYSTERoC UVM testbench

The testbench is UVM 1.2, SystemVerilog, simulator-portable (no vendor extensions except the tool-provided `uvm_pkg`/`uvm_macros.svh`). It verifies `noc_top` in the *full* 4×4 mesh with real injection/ejection agents — there are no stub unit tests in this repo (per-block testbenches existed upstream of the snapshot and were superseded by the environment-level model below).

---

## 1. The `hy_if` contract (timing rule)

**One rule, observed by every component:** *drive with non-blocking assignments right after `@(posedge clk)`; sample in the active region of `@(posedge clk)`.* Samples then equal what the DUT flip-flops see at that edge — no clocking blocks, no simulator-specific sampling semantics.

The transition-protocol sources are modelled **inside the interface** so that 16 independent drivers can own one bit each (`tb/iface/hy_if.sv`):

```
src_empty[n]        = !(src_busy[n] | src_force_busy[n])
inject_block_ack[n] = inject_block_req & src_empty[n]
```

* `src_busy[n]` is set by node n's inject driver while a started packet is not yet fully accepted.
* `src_force_busy[n]` is a test knob: a source that never acknowledges (used to provoke the watchdog; `hy_watchdog_test`).
* Mid-run reset: the testbench top builds `assign rst_n = por_n & vif.test_rst_n`; a test pulls `test_rst_n` low for a few cycles (`hy_reset_test`).

`ej_ready` must never depend combinationally on `ej_valid` (DECISIONS 38); the `hy_sink_driver` implements always / random / bursty / never modes with a **registered** random bit.

---

## 2. Running one test

```bash
./scripts/run_uvm.sh --sim vcs   --test hy_smoke_test
./scripts/run_uvm.sh --sim xrun  --test hy_oscillate_test --load 400 --cycles 300
./scripts/run_uvm.sh --sim questa --test hy_nodrain_test --policy 1 --det 1
```

| Argument | Meaning |
|---|---|
| `--test <name>` | UVM test name (`+UVM_TESTNAME=`), see §5 for the catalogue |
| `--policy 0\|1` | `SWITCH_POLICY` (0 drain-and-switch, 1 no-drain) |
| `--det 0\|1\|2` | `DET_VARIANT` (no-drain: `oe_fixed` / `oe_par` / `oe_x21`) |
| `--thigh/--tlow/--dual/--ewma/--xy-only` | hysteresis point (default: HYSTERoC 160/96 dual/EWMA) |
| `--load/--cycles/--pat` | workload plusargs `+HY_LOAD=/+HY_CYCLES=/+HY_PAT=` |
| `--timeout <n>` | `+HY_TIMEOUT=` hard stop in cycles (default 3 000 000) |
| `--campaign` | run the full test catalogue and summarise PASS/FAIL per test |

**Pass criterion:** the UVM report server severity count (`UVM_ERROR + UVM_FATAL = 0`) ⇒ `*** HYSTERoC TEST PASSED ***`. Hard timeouts are enforced by a forked watcher in `hy_base_test.run_phase`.

**DUT parameterisation:** `tb/env/test.sv` exposes every `noc_top` parameter as a top-level parameter and echoes it into the test via `uvm_config_db`, so *one compiled snapshot* can be re-elaborated per point (e.g. `xrun -generic_top`). `hy_cfg` carries the echo plus the checker switches (`chk_scoreboard`, `chk_route`, `chk_protocol`, `chk_latency`, `chk_models`, `cov_en`, `expect_drain_timeout`, `require_inflight_switch`).

### 2.1 Workload plusargs (honoured by most workload tests)

```
+HY_LOAD=<pm>      offered load, flits/node/cycle × 1000    (e.g. 400 = 0.4 flits/node/cycle)
+HY_CYCLES=<n>     cycles per phase
+HY_PAT=<n>        0 uniform  1 hotspot  2 transpose  3 complement  4 neighbour  5 bit-rev
+HY_ALLPAIRS=1     zero-load test over all 240 flows
+HY_TIMEOUT=<n>    hard stop in cycles (default 3 000 000)
+HY_LO/HY_HI/HY_STEP/HY_DWELL   hy_compare_test load sweep
```

---

## 3. Environment blocks

| File | Class | Role |
|---|---|---|
| `tb/iface/hy_if.sv` | `hy_if` | DUT-facing interface (§1). Also exposes the router-output taps (`link_out_*`) for the hop monitor and `sva_errors` for the protocol checker's report phase |
| `tb/pkg/hy_types.svh` | — (pure functions) | network constants, `hy_flit_t`, ready-mode enums, header build/parse helpers; no UVM → unit-testable alone |
| `tb/pkg/hy_item.svh` | `hy_item` | the driven transaction (packet descriptor) |
| `tb/pkg/hy_cfg.svh` | `hy_cfg` | config object (§2): per-node sink modes, bubble percentages, DUT echo, checker switches |
| `tb/pkg/hy_route.svh` | — | UVM-side route references (mirrors `route_compute`, including the `oe_*` variants) |
| `tb/env/hy_sb_core.svh` | `hy_sb_core`, `hy_pkt_rec` | scoreboard core: packet reassembly, per-flow queueing, integrity and ordering bookkeeping |
| `tb/pkg/hy_traffic.svh` | — | traffic-pattern helpers (uniform/hotspot/transpose/…) |
| `tb/env/hy_model.svh` | `hy_model` | **the** cycle-exact reference model of the whole control path — EWMA, hysteresis, FSM, drain detector, statistics — fed with the *observed* DUT inputs, outputs compared every cycle |
| `tb/pkg/hy_sequencer.svh` | `hy_vseqr` | virtual sequencer (runs the virtual sequences) |
| `tb/agents/hy_inject_driver.svh` | `hy_inject_driver` | 16 flit-exact injection drivers; models source bubbles and the busy/ack bits |
| `tb/agents/hy_monitors.svh` | `hy_inject_monitor`, `hy_eject_monitor` | per-port monitors (accepted flits, ejection events) |
| `tb/agents/hy_sink_driver.svh` | `hy_sink_driver` | per-node ready drivers (registered random, bursty or never) |
| `tb/agents/hy_hop_monitor.svh` | `hy_hop_monitor` | hop-level observer on `link_out_*`: every flit hop is checked against the **legal-output set** for the mode observed in that cycle (including the no-drain `oe_*` variants) |
| `tb/env/hy_scoreboard.svh` | `hy_scoreboard` | delivery exactly once, right node, right content (integrity word), per-source order |
| `tb/env/hy_protocol_checker.svh` | `hy_protocol_checker` | deep protocol rules (mode-change legality, drain/timeout invariants, in-flight switch requirement for `hy_nodrain_test`) + the `sva_errors` report |
| `tb/env/hy_coverage.svh` | `hy_coverage` | functional coverage: mode × threshold-band × pattern |
| `tb/agents/hy_agents.svh` | `hy_inject_agent`, `hy_eject_agent` | agent wrappers |
| `tb/env/hy_env.svh` | `hy_env` | instantiates agents/checkers/model, wires TLM analysis ports |
| `tb/env/hy_sva.sv` | `hy_sva` | concurrent assertions on `hy_if` (§4) |
| `tb/env/test.sv` | `test` | testbench top: clock (100 MHz `#5`), POR, DUT, SVA, `uvm_config_db`, `run_test()` |
| `tb/env/testbench.sv` | — | include list (compile order) |

Sequences (`tb/seq/`): `hy_item_seq` (packet item), `hy_traffic_seq` (random traffic), `hy_load_ctl`; virtual sequences in `hy_vseq_lib.svh` (`hy_vseq_single`, `hy_vseq_zero_load`, `hy_vseq_all_pairs`, `hy_vseq_flow`, `hy_vseq_pattern/uniform/hotspot/transpose`, `hy_vseq_phased`, `hy_vseq_oscillate`, `hy_vseq_patterns_all`, `hy_vseq_lengths`, `hy_vseq_loopback`, `hy_vseq_backpressure`, `hy_vseq_watchdog`, `hy_vseq_reset`, `hy_vseq_compare`).

---

## 4. Assertion catalogue (`hy_sva`)

Stateless / short-history properties only; the deep checks live in `hy_model` + `hy_protocol_checker`. A failure increments `vif.sva_errors` (UVM does not count SVA `$error`s itself) and is reported in the check phase. `SWITCH_POLICY` selects the rule set at elaboration:

| Assertion | Rule |
|---|---|
| `a_blk_implies_busy` | `inject_block_req → transition_busy` |
| `a_state_legal` | `ctrl_state ≠ 7` |
| `a_timeout_is_error` | `drain_timeout ≡ (state == ERROR)` |
| `a_mode_change_only_after_switch` *(policy 0)* | `active_mode` changed only in a SWITCH cycle with `network_drained` |
| `a_mode_change_sources_idle` *(0)* | `active_mode` changed only with all sources empty |
| `a_start_only_from_run` *(0)* | `transition_busy` rose only from RUN with `req_mode != active_mode` |
| `a_block_rises_from_run` *(0)* | `inject_block_req` rose only from RUN |
| `a_release_in_resume` *(0)* | `inject_block_req` fell only in RESUME |
| `a_error_sticky` *(0)* | `ERROR → ERROR` (until reset) |
| `a_no_inject_in_drain_switch` *(0)* | `inj_valid == 0` during DRAIN or SWITCH |
| `a_nd_run_only` *(1)* | no-drain: `ctrl_state == RUN`, block/busy/timeout never asserted |
| `a_nd_mode_follows_req` *(1)* | no-drain: `active_mode == $past(req_mode)` |
| `a_drained_means_empty` *(both)* | `network_drained → (&router_empty && &src_empty)` |
| `a_no_internal_errors` *(both)* | `!proto_err_any && !alloc_err_any` |
| `a_no_x_control` *(both)* | no X/Z on control and handshake outputs |
| `a_ej_hold` *(both, ×16)* | a stalled ejection output holds data/type until accepted |

---

## 5. Test catalogue (`tb/tests/hy_tests_pkg.sv`)

| Test | Purpose |
|---|---|
| `hy_smoke_test` | one packet, corner to corner (0 → 15) |
| `hy_zero_load_test` | latency is exactly `distance + 4` into an empty network (`+HY_ALLPAIRS=1` for all 240 flows) |
| `hy_all_pairs_test` | every node → every other node |
| `hy_flow_test` | 200 packets of one flow (ordering, integrity) |
| `hy_uniform_test` / `hy_hotspot_test` / `hy_transpose_test` / `hy_pattern_test` | pattern × load/cycles via plusargs (`+HY_PAT=`) |
| `hy_phased_test` | low → high → low load (guaranteed band crossings) |
| `hy_oscillate_test` | square-wave load — many threshold crossings; the ping-pong exerciser |
| `hy_nodrain_test` | oscillate + *requires* a mode change with flits in flight (policy 1) |
| `hy_compare_test` | one configuration over a load sweep (latency/throughput/switch comparison) |
| `hy_patterns_all_test` | every pattern at a heavy load |
| `hy_lengths_test` | packet lengths 2..16 (beyond the 4-flit contract) |
| `hy_loopback_test` | src == dst (beyond the contract) |
| `hy_bubble_test` | gaps inside packets + 60%-ready random sinks |
| `hy_backpressure_test` | sink behaviour changes while traffic runs |
| `hy_watchdog_test` | a source never acknowledges → ERROR state required (policy 0 only; reported as skipped under policy 1) |
| `hy_reset_test` | reset mid-traffic, twice |
| `hy_regress_test` | all of the above, sequenced in one simulation |

---

## 6. Adding a test

1. Add a sequence (or a virtual sequence) in `tb/seq/`.
2. Add a test class in `tb/tests/hy_tests_pkg.sv` extending `hy_base_test`; override `configure(hy_cfg)` *before* the environment is built (checker switches, sink modes) and `run_body()` to start sequences on `env.vseqr`.
3. Read knobs with `plus("HY_LOAD", 100)` etc. — never hard-code them into the test.
4. If the test provokes the watchdog, set `cfg.expect_drain_timeout = 1`; if it must observe an in-flight no-drain switch, set `cfg.require_inflight_switch = 1`.
5. Run it through `scripts/run_uvm.sh`; the UVM severity count is the verdict.

## 7. Sign-off checklist (before claiming a design point clean)

- [ ] `hy_smoke_test`, `hy_zero_load_test`, `hy_all_pairs_test` green (data path + routing legality)
- [ ] `hy_oscillate_test` green on *both* policies and, for the claim, on the conventional-hybrid parameterisation (`--dual 0 --ewma 0`)
- [ ] `hy_nodrain_test` green (in-flight switch observed) under `SWITCH_POLICY=1`
- [ ] `hy_watchdog_test` green under `SWITCH_POLICY=0` **and** reports `ERROR` (not a hang)
- [ ] `hy_reset_test` + `hy_regress_test` green
- [ ] Coverage: mode × band bins hit (especially the *inside-the-band* bin with an oscillating signal, which is the entire point of the work)
- [ ] `performance_monitor` numbers saved per configuration for the experiment table ([experiments.md](experiments.md))
