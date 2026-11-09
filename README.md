# HYSTERoC — Hysteresis-Based Dynamic Routing Control for FPGA NoCs

**An FPGA-oriented Network-on-Chip (NoC) router that switches dynamically between deterministic XY routing and adaptive Odd–Even routing with a congestion-forecast hysteresis controller, designed to reduce the ping-pong effect under fluctuating congestion while preserving deadlock safety.**

| | |
|---|---|
| Design status | Research prototype (V6). RTL is Verilog-2001, synthesisable; the full UVM environment is the primary verification vehicle. |
| Mesh | 4×4 (16 wormhole routers, 5 ports each), 16-bit flits, 4-flit packets |
| Control | EWMA congestion filter → global max → dual-threshold hysteresis → drain-and-switch (or no-drain) global controller |
| Verification | UVM 1.2 testbench (`tb/`), concurrent SVA (`hy_sva`), reference model of the control path, functional coverage, hop-level legality monitors |
| Paper/reports | Start at [docs/experiments.md](docs/experiments.md) (evaluation plan) and [DECISIONS.md](DECISIONS.md) (design-decision log) |

---

## Table of contents

1. [Quick start](#1-quick-start)
2. [What HYSTERoC does (and why)](#2-what-hysteroc-does-and-why)
3. [Architecture](#3-architecture)
4. [Configuration matrix (one DUT, all ablation points)](#4-configuration-matrix-one-dut-all-ablation-points)
5. [Traffic, packets and the measurement model](#5-traffic-packets-and-the-measurement-model)
6. [Verification](#6-verification)
7. [Repository layout](#7-repository-layout)
8. [Documentation map](#8-documentation-map)
9. [Limitations and open issues](#9-limitations-and-open-issues)
10. [Citation and licence](#10-citation-and-licence)
11. [Agent notes](#11-agent-notes)

---

## 1. Quick start

### 1.1 RTL lint (no simulator licence needed)

```bash
# iverilog is enough to elaborate the whole design after the restructure
iverilog -g2005-sv -I rtl/common \
  -o /tmp/hysteroc.vvp \
  rtl/router/input_fifo.v rtl/control/ewma_filter.v rtl/control/congestion_estimator.v \
  rtl/control/hysteresis_controller.v rtl/router/round_robin_arbiter.v rtl/router/route_compute.v \
  rtl/router/route_select.v rtl/router/packet_tracker.v rtl/router/switch_allocator.v \
  rtl/router/crossbar.v rtl/router/noc_router.v rtl/control/drain_detector.v \
  rtl/control/transition_controller.v rtl/control/mode_switch_direct.v rtl/top/noc_top.v \
  rtl/bench/performance_monitor.v rtl/bench/traffic_generator.v rtl/bench/packet_sink.v
```

Or with Verilator (if installed):

```bash
verilator --lint-only --top-module noc_top -I rtl/common \
  $(cat sim/hysteroc.f | grep '\.v$') $(cat sim/hysteroc_bench.f | grep '\.v$') 2>&1 | head -50
```

### 1.2 UVM smoke test (commercial simulator)

```bash
# Pick one: VCS, Xcelium, or Questa (see scripts/run_uvm.sh --help)
./scripts/run_uvm.sh --sim vcs   --test hy_smoke_test
./scripts/run_uvm.sh --sim xrun  --test hy_smoke_test
./scripts/run_uvm.sh --sim questa --test hy_smoke_test
```

Every run is parameterised (DUT variant + workload + plusargs) — see [docs/verification.md](docs/verification.md) §2 and `scripts/run_uvm.sh --help`.

### 1.3 `make` shortcuts

```bash
make lint          # iverilog elaboration of the full design
make sim-rtl       # compile-only smoke (needs a UVM-capable simulator to link)
make help          # list targets
```

---

## 2. What HYSTERoC does (and why)

### 2.1 The problem

Conventional hybrid adaptive routing copies a single adaptive decision into a mode switch the moment congestion crosses **one** threshold. Because the congestion signal is noisy (per-router FIFO occupancies bounce cycle-to-cycle), the network oscillates: XY → adaptive → XY → adaptive … Each switch costs a drain, stalls injection, and moves no packets. We call this **ping-pong**; it is the primary failure mode of threshold-switched routing under fluctuating load.

### 2.2 The idea

HYSTERoC treats the mode-switching problem as **hysteresis (Schmitt-trigger) control with a filtered signal**:

```
neighbour FIFO occupancy ── max ── mapped 0..255 ── EWMA filter ── global max
        ── dual-threshold hysteresis (TLOW < THIGH) ── request ── global
           drain-and-switch FSM ── active_mode (broadcast, one writer)
```

* **EWMA filtering (α = 1/8)** removes high-frequency occupancy ripple so the controller reacts to *sustained* congestion, not single-cycle spikes.
* **Dual thresholds (TLOW = 96, THIGH = 160 by default)** create a dead band. Once adaptive, the controller only returns to XY when congestion has clearly fallen below TLOW; once in XY, it only leaves when congestion clearly rises above THIGH. A signal oscillating inside the band causes **zero** switches.
* **Drain-and-switch (SWITCH_POLICY = 0, the paper baseline):** a mode change only happens when the network is provably empty *and* every source has acknowledged. The mode register has exactly one writer, and the FSM re-checks quiescence in the switching cycle itself. This makes mode changes provably safe with respect to in-flight packets.
* **No-drain variant (SWITCH_POLICY = 1):** both the deterministic and adaptive modes derive from the *same Odd–Even turn set* (`oe_fixed`/`oe_par` ∈ `oe_cand`), so a mode change can happen with traffic in flight — the next header simply uses the new mode. Deterministic Odd–Even is the drop-in replacement for XY in that configuration. This variant is the more FPGA-friendly design point (no global drain, no injection stalls); the cost is that the non-adaptive mode is no longer plain XY.

### 2.3 What is being claimed (and what is not)

| Claim status | Statement |
|---|---|
| **Supported by design** | Ping-pong is eliminated when the filtered congestion signal oscillates inside (TLOW, THIGH); the number of switches is bounded by the number of band crossings, not by signal ripple. |
| **Baseline to compare** | Single-threshold control on the raw (unfiltered) congestion value — the conventional hybrid design point (`DUAL=0, USE_EWMA=0, THRESH_SINGLE=128`). HYSTERoC must beat it on switches and ping-pong events *at equal or better* latency/throughput. |
| **Not claimed** | End-to-end deadlock freedom of the whole router is **not** proved in this repository. Deadlock safety of the adaptive Odd–Even turn set is argued from Chiu's turn model and checked by CDG analysis (`docs/architecture.md` §3); a formal whole-design proof is future work. See [DECISIONS.md](DECISIONS.md). |

---

## 3. Architecture

Full details: [docs/architecture.md](docs/architecture.md). This section is the orientation a new reader needs.

### 3.1 One picture

```
 16× source ── inj ──► ┌─────────────────────────────────┐
                        │ 16× noc_router (5-port wormhole)│── ej ──► 16× sink
                        │   input_fifo → packet_tracker   │  (neighbour-FIFO occupancy
                        │   → route_compute → route_select│   sideband feeds congestion
                        │   → switch_allocator → crossbar │   estimate + adaptive choice)
                        └────┬───────────┬────────┬───────┘
                 EWMA(E)    │           │        │  router_empty
                            ▼           ▼        ▼
            max of 16 E  ──► hysteresis_controller ──► req_mode ──► transition_controller
                 ──────────────────────────▲        (the ONLY writer of active_mode)
                            congestion_estimator (per router, max neighbour occupancy)
                            ewma_filter (per router, S_next = S − S/8 + C)
                            drain_detector (router_empty ∧ src_empty ∧ ¬link_active)
                            traffic_generator / packet_sink (bench, per node)
                            performance_monitor (observer: switches, dwell, ping-pong,
                                                   drain length, latency/throughput)
```

### 3.2 Router (`rtl/router/noc_router.v`)

Five ports — North (0), South (1), East (2), West (3), Local (4). One instance of `input_fifo → packet_tracker → route_compute → route_select` per input port, joined by a `switch_allocator` (five round-robin arbiters, packet-level reservation) and a `crossbar`.

Key interface rules (see the module header for the exact statement):

* `in_ready` derives from **registered** FIFO state only — it never depends on `pop` or on any combinational path through the router. A full FIFO therefore does *not* accept a push in the same cycle as a pop.
* `out_valid` is asserted only in a cycle in which `out_ready` is 1, so `valid && !ready` never occurs on an output link.
* Neighbour occupancy is registered once (one cycle old) so the congestion estimate and the adaptive route choice see the **same** value.
* The router is **type-driven**: `packet_tracker` tracks the reserved output between header and tail. There is no per-port flit counter — the fixed 4-flit packet length is a *source contract*, checked by the sinks and the UVM scoreboard, not by the router (rationale in [DECISIONS.md](DECISIONS.md)).

### 3.3 Routing modes

| Mode | Rule | Where |
|---|---|---|
| **XY (deterministic)** | X dimension first, then Y. Local at the destination. | `route_compute.xy_port` |
| **Adaptive Odd–Even** | Among the Odd–Even-legal candidates (`oe_cand`), pick the neighbour FIFO with the lowest occupancy; ties break East → South → West → North. | `route_compute.oe_cand` + `route_select` |
| **Deterministic Odd–Even (no-drain policy)** | `oe_fixed` (Local > East > West > vertical) or `oe_par` (source-parity variant): always a member of `oe_cand`, so the no-drain mode switch keeps every turn legal. | `route_compute` (DET_VARIANT) |

`route_select` also implements an optional **relative margin** (`ADAPT_MARGIN`, v6): in adaptive mode, keep the deterministic port unless another candidate is at least `m` flits emptier. Default 0 = off.

### 3.4 Congestion estimation and hysteresis (`rtl/control/`)

* `congestion_estimator`: `max_occ = max` over *available* cardinal directions of the registered neighbour FIFO occupancy; boundary directions are excluded by masking to 0, and `any_avail` distinguishes "no direction available" from "available and empty". The occupancy is mapped through the frozen table `0,64,128,192,255`.
* `ewma_filter`: fixed-point EWMA, `S_next = S − (S>>3) + C`, `E = S>>3`. `S` is 11 bit, so `S` can never overflow for `C ≤ 255` and there is no signed arithmetic. `enable = 0` freezes the state (used outside RUN).
* `hysteresis_controller`: the dual/single-threshold request logic. With `DUAL=1` the state is `req_mode` itself: `value ≥ THIGH ⇒ ADAPT`, `value ≤ TLOW ⇒ XY`, hold otherwise. With `DUAL=0` it degrades to the single-threshold baseline. `enable = 0` holds the request (the transition FSM freezes it outside RUN so a request cannot be retargeted mid-transition).
* `drain_detector`: `network_drained` = (all routers empty ∧ all sources empty ∧ no link active) held for `CONFIRM` consecutive cycles; it drops in the same cycle as any activity.
* `transition_controller`: the drain-and-switch FSM — `RUN → QUIESCE_REQUEST → WAIT_ACK → DRAIN → SWITCH → RESUME → RUN`, with a sticky `ERROR` state on watchdog expiry (`WATCHDOG_LIMIT_CYCLES`). `active_mode` is written **only** in SWITCH, only when `network_drained` is still true in that cycle; if quiescence is lost, the FSM returns to DRAIN without switching. A timeout never forces a switch.
* `mode_switch_direct`: the no-drain alternative (same ports); `active_mode` follows `req_mode` one cycle later, unconditionally safe because both modes share the Odd–Even turn set.

### 3.5 Top level (`rtl/top/noc_top.v`)

Sixteen routers, node `n = 4·y + x` (x East, y South). Link wiring: out-N of (x,y) → in-S of (x,y−1), out-S → in-N of (x,y+1), out-E → in-W of (x+1,y), out-W → in-E of (x−1,y); boundary inputs tied to `valid=0`, boundary `out_ready=0`. The global max of the 16 EWMA estimates and of the 16 unfiltered congestion values feed the hysteresis controller; its request feeds the transition controller, whose `active_mode` is broadcast to every router. All DUT parameters are top-level (spec §12.1) so one compiled snapshot can be re-elaborated per configuration (see §4).

### 3.6 Bench and measurement (`rtl/bench/` — simulation only)

* `traffic_generator`: per-node packet source (creation queue + ready/valid injection, honours the injection-block protocol, 5 traffic patterns, xorshift32 PRNG).
* `packet_sink`: per-node ejection endpoint with integrity checking (header/destination, body/tail ordering, `tail = hdr ^ body1 ^ body2 ^ 16'hC3A5`).
* `performance_monitor`: an **observer only** — it never drives the network. It counts completed switches, dwell time, ping-pong events (dwell ≤ 50/100/200/400 cycles), blocked periods (drain length), packet latency statistics (mean/std/max), throughput, and a progress watchdog (longest stall). Everything is windowed to the measurement window.

### 3.7 Packet format and link rules

```
flit = { flit_type[1:0], data[15:0] }        // 18 bits
packet = header, body, body, tail             // exactly 4 flits (source contract)
header = { dst_x[15:14], dst_y[13:12], src_x[11:10], src_y[9:8], pid[7:0] }
types: 00 header, 01 body, 10 tail, 11 reserved
valid held, data stable while stalled; ready never depends combinationally on valid
```

---

## 4. Configuration matrix (one DUT, all ablation points)

`noc_top` parameters (defaults) — see `docs/architecture.md` §5 for the full catalogue:

| Parameter | Default | Meaning |
|---|---|---|
| `XY_ONLY` | 0 | 1 = pin the network to XY, controller never leaves RUN (reference) |
| `DUAL` | 1 | dual-threshold hysteresis (1) vs single threshold (0) |
| `USE_EWMA` | 1 | hysteresis input = EWMA-filtered value (1) vs raw congestion (0) |
| `THIGH` / `TLOW` | 160 / 96 | hysteresis band edges (require `TLOW < THIGH`) |
| `THRESH_SINGLE` | 128 | single-threshold baselines (raw occupancy ≥ 2 → adaptive) |
| `SAMPLE_PERIOD` | 1 | EWMA update period (cycles) |
| `WATCHDOG_LIMIT_CYCLES` | 1000 | drain watchdog limit (≥ 4) |
| `POST_RESUME_MASK_CYCLES` | 0 | cycles the hysteresis stays frozen after a switch |
| `DRAIN_CONFIRM` | 2 | consecutive quiet cycles before `network_drained` rises |
| `SWITCH_POLICY` | 0 | 0 = drain-and-switch (XY ↔ adaptive Odd–Even, paper baseline); 1 = no-drain (deterministic Odd–Even ↔ adaptive Odd–Even) |
| `DET_VARIANT` | 0 | no-drain only: 0 = `oe_fixed`, 1 = `oe_par` (source-parity), 2 = `oe_x21` |
| `ADAPT_MARGIN` | 0 | v6 relative bias toward the deterministic port in adaptive mode (0 = off) |
| `ENABLE_STATS` | 1 | 0 removes all transition counters (resource-overhead comparison) |

The four canonical comparison points:

| # | Point | Parameters | What it isolates |
|---|---|---|---|
| 1 | **HYSTERoC** | `DUAL=1, USE_EWMA=1, THIGH=160, TLOW=96` | the proposed design |
| 2 | Ablation B | `DUAL=1, USE_EWMA=0` | dual-threshold on the *raw* signal (value of filtering alone) |
| 3 | Ablation A | `DUAL=0, USE_EWMA=1, THRESH_SINGLE=128` | single threshold on the *filtered* signal (value of hysteresis alone) |
| 4 | Conventional hybrid | `DUAL=0, USE_EWMA=0, THRESH_SINGLE=128` | the baseline HYSTERoC must beat |
| 0 | XY-only reference | `XY_ONLY=1` | routing with no adaptive mode at all |

In the UVM flow these are elaborated as *top-level parameters of the testbench* (`tb/env/test.sv`), so a single compiled snapshot can be re-elaborated per point with simulator generic overrides (e.g. `xrun -generic_top`).

---

## 5. Traffic, packets and the measurement model

### 5.1 Offered load

Per node: `offered load = 4 × rate / 65536` flits per cycle (1.0 = one flit every cycle, i.e. a 4-flit packet every 4 cycles). Patterns: **uniform** (any other node), **hotspot** (a fraction to one hot node), **transpose** ((x,y)→(y,x)), **complement** ((x,y)→(3−x,3−y)), **neighbour** ((x,y)→(x+1 mod 4, y)). UVM workloads additionally use bit-reversal and all-pattern sweeps.

### 5.2 Latency definitions

* **Total latency** — packet creation → tail accepted (includes queuing at the source, hence includes the transition-induced waiting).
* **Network latency** — header first offered → tail accepted.
* Latencies are 16-bit modulo values; runs are sized so that ≥ 65536 cycles never occurs.

### 5.3 Switching metrics (`performance_monitor`)

* **Completed switch** — the cycle `active_mode` changes.
* **Ping-pong event** — a completed switch whose predecessor was ≤ `T_PP` cycles earlier (T_PP = 50/100/200/400 counted simultaneously), both inside the measurement window.
* **Dwell time** — cycles between two consecutive completed switches (sum/count/min/max).
* **Blocked period / drain length** — cycles with `inject_block_req` asserted, from first assertion to release (the release cycle is also the first cycle of the new mode).
* **Injection-blocked fraction** — windowed blocked cycles / measurement cycles.

### 5.4 How to reproduce an experiment

An experiment = (DUT configuration) × (workload) × (measurement window). The UVM testbench logs every test's configuration echo (`cfg: policy=…`) and the standard UVM `RESULT` line; the `performance_monitor` numbers are printed by the bench testbench. The full evaluation protocol (workload matrix, sweep ranges, statistics to report, and the exact plusargs) is in [docs/experiments.md](docs/experiments.md).

---

## 6. Verification

Full description: [docs/verification.md](docs/verification.md).

### 6.1 Structure

| Component | Location | Checks |
|---|---|---|
| Agents (inject / eject / hop) | `tb/agents/` | drive sources & sinks (always / random / bursty / never ready), monitor every port, record per-flit hops |
| Sequences | `tb/seq/` | directed single-packet, zero-load, all-pairs, flow, pattern, phased (low→high→low), oscillate (square-wave load), backpressure, watchdog, in-run reset |
| Scoreboard | `tb/env/hy_scoreboard.svh` | delivery exactly once, right node, right content (integrity word), per-source order |
| Hop legality monitor | `tb/agents/hy_hop_monitor.svh` | every flit hop departs on a *legal* Odd–Even/XY output for the observed mode |
| Protocol checker + reference model | `tb/env/hy_protocol_checker.svh`, `hy_model.svh` | cycle-exact model of EWMA, hysteresis, transition FSM, drain detector and statistics; mode-change/drain/watchdog rules |
| Coverage | `tb/env/hy_coverage.svh` | mode × threshold-band × pattern bins |
| SVA | `tb/env/hy_sva.sv` | mode change only in a SWITCH cycle with `network_drained`; ERROR stickiness; no injection during DRAIN/SWITCH; no X on control outputs; output hold under stall |
| Tests | `tb/tests/hy_tests_pkg.sv` | 19 tests: smoke, zero-load, all-pairs, flow, uniform/hotspot/transpose/pattern, phased, oscillate, nodrain, compare, patterns-all, lengths, loopback, bubble, backpressure, watchdog, reset, regress |

### 6.2 Running tests

```bash
./scripts/run_uvm.sh --sim xrun --test hy_oscillate_test --load 400 --cycles 300
./scripts/run_uvm.sh --sim vcs  --test hy_nodrain_test  --policy 1 --det 1
./scripts/run_uvm.sh --sim xrun --test hy_watchdog_test
./scripts/run_uvm.sh --sim xrun --test hy_regress_test
```

Pass/fail is the UVM severity count (0 `UVM_ERROR`/`UVM_FATAL` ⇒ `*** HYSTERoC TEST PASSED ***`). DUT parameters are selectable at compile/run time via `--policy/--det/--thigh/--tlow/--dual/--ewma/--xy-only` (see `scripts/run_uvm.sh --help`).

---

## 7. Repository layout

```
HYSTERoC/
├── README.md                  this file — orientation, quick start, claims
├── DECISIONS.md               design-decision log (why; what was rejected)
├── CHANGELOG.md               version history
├── LICENSE                    code licence (academic / research use)
├── Makefile                   lint / compile convenience targets
│
├── rtl/                       synthesizable Verilog-2001 (__EXCEPT__ bench/)
│   ├── common/hysteroc.vh     frozen shared constants ("spec" values)
│   ├── router/                input_fifo, packet_tracker, route_compute,
│   │                          route_select, round_robin_arbiter,
│   │                          switch_allocator, crossbar, noc_router
│   ├── control/               congestion_estimator, ewma_filter,
│   │                          hysteresis_controller, drain_detector,
│   │                          transition_controller, mode_switch_direct
│   ├── top/noc_top.v          4×4 mesh + global control
│   ├── bench/                 SIMULATION ONLY: traffic_generator,
│   │                          packet_sink, performance_monitor
│   └── design.sv              RTL include list (-I rtl/common -I rtl/router …)
│
├── tb/                        UVM testbench (SystemVerilog, UVM 1.2)
│   ├── iface/hy_if.sv         the single DUT-facing interface
│   ├── pkg/                   hy_pkg package + types/config/route/traffic helpers
│   ├── agents/                inject/eject/hop agents, monitors, drivers
│   ├── seq/                   base sequences + virtual-sequence library
│   ├── env/                   env, scoreboard, protocol checker, model,
│   │                          coverage, SVA, testbench top (test.sv)
│   └── tests/hy_tests_pkg.sv  19 tests + hy_base_test
│
├── sim/                       compile file lists (hysteroc.f, hysteroc_bench.f,
│                              hysteroc_tb.f) consumed by scripts/ and CI
├── scripts/run_uvm.sh         VCS / Xcelium / Questa driver → runs any test
├── docs/                      architecture / verification / experiments / fpga
├── fpga/                      synthesis notes + constraints stub for the add-on
│                              FPGA top (no vendor project is in this repo)
└── .github/workflows/ci.yml   Verilator lint + iverilog elaboration + docs check
```

> The old flat `rtl/` + `uvm/` layout was restructured (see [CHANGELOG.md](CHANGELOG.md), v6.1); the only tool-visible change is the include directories (now `-I rtl/common -I rtl/router …` instead of `-I rtl -I uvm`), and the header `hysteroc_defs.vh` was renamed to `hysteroc.vh`. All file *contents* are unchanged.

---

## 8. Documentation map

| Document | Contents |
|---|---|
| [docs/architecture.md](docs/architecture.md) | Module catalogue with per-module interface & invariants, the Odd–Even turn table, deadlock-safety argument, global control path, parameter reference, FPGA notes |
| [docs/verification.md](docs/verification.md) | UVM environment walkthrough, checker/assertion catalogue, how to run one test, how to add a test, coverage and sign-off checklist |
| [docs/experiments.md](docs/experiments.md) | Evaluation protocol: configuration points, workloads, sweep ranges, metrics and statistics to report, reproducibility checklist |
| [docs/fpga.md](docs/fpga.md) | Synthesis notes for the FPGA deployment (what scales, what to keep in BRAM, timing considerations, the `ENABLE_STATS=0` overhead point) |
| [DECISIONS.md](DECISIONS.md) | Decision log numbered to match the in-code references (e.g. "DECISIONS 38", "DECISIONS 41") — read this before changing a "frozen" choice |
| [CHANGELOG.md](CHANGELOG.md) | What changed per version/feature-batch |

---

## 9. Limitations and open issues

These are explicit, not hidden:

1. **Deadlock freedom is argued, not proved.** The adaptive Odd–Even turn set is known to be deadlock-free under Chiu's two turn rules for the *turn model*; the whole-router result additionally depends on the allocator, FIFO policy and the mode-switch policy, which are verified by directed tests, SVA and the progress watchdog, but there is no formal proof in this repository.
2. **The 4-flit packet length is a contract, not a router-enforced property.** The router is type-driven and never counts flits; a longer packet (the UVM `hy_lengths_test` sends up to 16 flits) is routed correctly because reservation is header→tail, but a *malformed* stream is detected by the sinks/scoreboard, not by the router.
3. **Congestion signal = neighbour FIFO occupancy only.** No link-utilisation, queue-depth-weighted, or latency-based signal is implemented; `congestion_estimator`'s max-over-available-directions table is frozen by spec and changing it invalidates the threshold calibration.
4. **Global control registers are not X-protected in hardware.** `noc_top.active_mode` is broadcast; X-sources are checked in simulation (`hy_sva` `a_no_x_control`) but there is no hardware X-propagation immunity.
5. **The FPGA top is not in this repo.** `fpga/` documents the integration path (pin-out, clocking, constraints stub), but no checked-in vendor constraints or bitstream exist yet. See [docs/fpga.md](docs/fpga.md).
6. **Measurement aliasing.** Latencies are modulo 2¹⁶ cycles; the bench testbenches are sized to stay far below that, but a run that exceeds it silently aliases.

---

## 10. Citation and licence

This is an academic research prototype. If you use it in a paper or thesis, cite the project and the specific configuration point you evaluated (the ablation matrix in §4 exists so that results are comparable across papers).

The RTL and testbench in this repository are released under the licence in [LICENSE](LICENSE) — research and educational use; discuss redistribution or commercial use with the repository owner.

---

## 11. Agent notes

This repo is maintained with AI agents. The operational conventions that agents must follow (branch discipline, frozen-value rules, `make lint && make docs` before committing, never commit to `main`) are in [AGENTS.md](AGENTS.md), which is the single source of truth for agent instructions — this section only points there.

This is a research codebase with a long spec → RTL → UVM history (feature batches U1–U5+ in the UVM tree, RTL V6 spec v5). The structure of this repository — including the odd legacy names that remain (`hy_*` UVM classes, `hysteroc.vh`, the `V6` version stamps) — is kept deliberately so that the in-code spec references (`spec 6.4`, `DECISIONS 41`, …) stay resolvable. See [DECISIONS.md](DECISIONS.md) for the full lineage.
