# AGENTS.md — working with the HYSTERoC repository

Instructions for AI agents (and humans) editing this repo. Read this before
making changes. The human-facing docs are in [README.md](README.md) and
[docs/](docs/); this file is the operational companion.

---

## 1. What this repo is

HYSTERoC — an FPGA-oriented 4×4 mesh NoC router (16 wormhole routers, 5 ports
each) whose core research object is a **hysteresis-based routing-mode
controller**: EWMA-filtered congestion → dual thresholds (160/96) → global
drain-and-switch (or no-drain) FSM switching between deterministic XY and
adaptive Odd–Even routing, to kill the ping-pong effect under fluctuating load.

- **RTL:** Verilog-2001 only, synthesisable (except `rtl/bench/`), in `rtl/`.
- **Testbench:** SystemVerilog, UVM 1.2, in `tb/`. Primary verification vehicle.
- **Status:** research prototype (V6). There is **no formal deadlock proof**;
  deadlock safety is argued + tested. See [README.md](README.md) §9.

## 2. Repository layout (post-restructure; do not revert to flat)

```
rtl/common/hysteroc.vh        frozen shared constants ("spec" values)
rtl/router/                   input_fifo, packet_tracker, route_compute,
                              route_select, round_robin_arbiter,
                              switch_allocator, crossbar, noc_router
rtl/control/                  congestion_estimator, ewma_filter,
                              hysteresis_controller, drain_detector,
                              transition_controller, mode_switch_direct
rtl/top/noc_top.v             4x4 mesh + global control
rtl/bench/                    SIMULATION ONLY: traffic_generator, packet_sink,
                              performance_monitor  (never synthesise these)
rtl/design.sv                 RTL include list (-I rtl/common -I rtl/router ...)
tb/iface/hy_if.sv             the single DUT-facing interface
tb/pkg/                       hy_pkg package + hy_types/hy_cfg/hy_item/hy_route/hy_traffic
tb/agents/                    inject/eject/hop agents, monitors, sink driver
tb/seq/                       base sequences + virtual-sequence library
tb/env/                       env, scoreboard, model, protocol checker, coverage,
                              SVA (hy_sva), testbench top (test.sv)
tb/tests/hy_tests_pkg.sv      19 tests + hy_base_test
sim/                          compile file lists (hysteroc.f, hysteroc_bench.f, hysteroc_tb.f)
scripts/run_uvm.sh            VCS / Xcelium / Questa driver  → any test, any config point
docs/                         architecture / verification / experiments / fpga
fpga/                         FPGA integration pointer (no vendor project in repo)
.github/workflows/ci.yml      iverilog elaboration + Verilator lint + docs link check
```

**Include dirs (both flows):** RTL `-I rtl/common -I rtl/router -I rtl/control
-I rtl/top -I rtl/bench`; UVM additionally `-Itb/pkg -Itb/agents -Itb/seq
-Itb/env -Itb/tests`. Compile order: `hy_if.sv` → `hy_pkg.sv` →
`hy_tests_pkg.sv` → `testbench.sv` (`rtl/design.sv` for RTL-only builds).

## 3. Durable conventions (the rules that keep this repo coherent)

### 3.1 RTL

- **`rtl/` is Verilog-2001 except the include-list shells** (`design.sv`, the
  tb tops). No SystemVerilog in `rtl/`. No vendor primitives, no latches.
- **Frozen values live only in `rtl/common/hysteroc.vh`** and are cited as
  `spec §X.Y` in module headers. Never hard-code a frozen value in a module —
  use the `` `HY_* `` macro. TUNABLE thresholds are `noc_top` parameters.
- Every module carries the `// ===…===` header block with its contract
  (ports, rules, edge cases). **When you change behaviour, update the header
  in the same commit** — it is the spec of record for that module.
- Module-boundary rules you must preserve: `ready` never depends on `valid`
  (registered state only); `out_valid` only with `out_ready`; neighbour
  occupancy registered once (one cycle old); `active_mode` has exactly one
  writer (the transition FSM); `ej_ready` never depends on `ej_valid`.
- `ripple`/lint pragmas: silence unused signals with
  `/* verilator lint_off UNUSEDSIGNAL */` blocks exactly as the existing code
  does — do not delete them.

### 3.2 Verification (`tb/`)

- **Interface timing rule (non-negotiable):** drive with NBA right after
  `@(posedge clk)`; sample in the active region. No clocking blocks.
- **Every change needs a test**: new behaviour → a directed or workload test in
  `tb/tests/hy_tests_pkg.sv` (extend `hy_base_test`; read knobs via
  `plus("HY_LOAD", …)`, never hard-code them). `hy_reset_test`/`hy_regress_test`
  are the safety net — don't break them.
- The cycle-exact reference model `tb/env/hy_model.svh` compares control
  outputs **every cycle**. If you change the control path, update the model in
  the same commit or the "compare" checks will (correctly) fail.
- Pass criterion: UVM severity count = 0, i.e. `*** HYSTERoC TEST PASSED ***`.
  A hang or a watchdog ERROR when not expected is a failure, not a "result".

### 3.3 Decisions and docs

- **`DECISIONS.md` numbers are stable — never renumber.** Code cites them
  ("DECISIONS 38" etc.). A behaviour change to a FROZEN value or an accepted
  policy gets the *next* decision number and a citation in the code.
- Keep the docs current in the same PR as the code: the module header (§3.1),
  [docs/architecture.md](docs/architecture.md) if interfaces/rules change,
  [docs/verification.md](docs/verification.md) if the test catalogue/checkers
  change, [docs/experiments.md](docs/experiments.md) if metrics change.
- `CHANGELOG.md`: feature batches were lettered (U1–U5) upstream; keep the
  v1…v6 semantic thread (v2/v3/v5/v6 entries) and add new entries at the top.

### 3.4 Commits and branches

- **Never commit to `main` directly.** Branch (e.g. `feat/<name>`,
  `fix/<name>`, `chore/<name>`), commit, push, open a PR.
- Conventional-commit messages (`feat:`, `fix:`, `docs:`, `refactor:`).
- Run `make lint && make docs` before committing.

## 4. Commands

```bash
make lint          # iverilog elaboration of the full design (no licence) — always run
make docs          # README/docs relative-link check
./scripts/run_uvm.sh --sim vcs   --test hy_smoke_test                 # any test
./scripts/run_uvm.sh --sim xrun  --test hy_oscillate_test --load 400 --cycles 300
./scripts/run_uvm.sh --sim xrun  --test hy_nodrain_test  --policy 1 --det 1
./scripts/run_uvm.sh --help      # full option list (DUT points + plusargs)
```

For an all-open check use `make lint`; the UVM runs require VCS/Xcelium/Questa
(not installed in every environment — if unavailable, say so in your report
rather than claiming a pass).

### Configuration points (one DUT, every ablation)

`SWITCH_POLICY`/`DUAL`/`USE_EWMA`/`THIGH`/`TLOW`/`THRESH_SINGLE`/`DET_VARIANT`/
`ADAPT_MARGIN`: HYSTERoC = `DUAL=1 USE_EWMA=1 THIGH=160 TLOW=96 POLICY=0`;
conventional hybrid = `DUAL=0 USE_EWMA=0 THRESH_SINGLE=128`. Test params are
top-level on `tb/env/test.sv` (one snapshot, re-elaborated per point).

## 5. Orientation for new agents

1. [README.md](README.md) — claims, quick start, layout, limitations.
2. [docs/architecture.md](docs/architecture.md) — module-by-module invariants.
3. [docs/verification.md](docs/verification.md) — how to run/add a test.
4. [DECISIONS.md](DECISIONS.md) — why frozen choices are frozen.
5. `rtl/common/hysteroc.vh` — the constants everything keys off.

## 6. Known limitations (do not paper over them)

1. Deadlock freedom is argued, not proved — no formal proof in repo.
2. The 4-flit packet length is a *source contract*, not router-enforced.
3. `latency` values are 16-bit modulo (runs must stay < 65536 cycles).
4. X-propagation is checked in simulation (`a_no_x_control`) but not hardened in hardware.
5. The FPGA top is not in this repo (see `docs/fpga.md`).
