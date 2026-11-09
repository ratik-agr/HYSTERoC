# Changelog — HYSTERoC

Format: newest first. The repository layout after **v6.1** is the reorganised tree
described in [README.md](README.md) §7 (`rtl/{common,router,control,top,bench}`,
`tb/{iface,pkg,agents,seq,env,tests}`, `sim/`, `scripts/`, `docs/`).

---

## v6.1 — Repository restructure + full documentation

- **Layout:** flat `rtl/` + `uvm/` reorganised into `rtl/` (router / control /
  top / bench / common) and `tb/` (iface / pkg / agents / seq / env / tests).
  All file *contents* are unchanged; only locations and the include
  directories (`-I rtl/common -I rtl/router …` instead of `-I rtl -I uvm`)
  changed. The shared header `hysteroc_defs.vh` was renamed to
  `hysteroc.vh`, and `rtl/design.sv`'s include list was corrected
  (it omitted `noc_router.v`).
- **Documentation:** `README.md` (orientation, quick start, claims, limitations),
  `docs/architecture.md`, `docs/verification.md`, `docs/experiments.md`,
  `docs/fpga.md`, `DECISIONS.md` (design-decision log numbered to match the
  in-code references), this changelog, `LICENSE`.
- **Build/run:** `sim/hysteroc.f` / `hysteroc_bench.f` / `hysteroc_tb.f` file
  lists, `scripts/run_uvm.sh` (VCS / Xcelium / Questa driver → 19 tests at
  every configuration point), `Makefile` (`lint`, `sim-rtl`).
- **CI:** `.github/workflows/ci.yml` — Verilator lint, iverilog elaboration,
  README/docs integrity.

## v6 — ADAPT_MARGIN (relative bias toward the deterministic port)

- `route_select`: optional v6 bias. With `ADAPT_MARGIN = m > 0`, adaptive mode
  keeps the deterministic port unless another Odd–Even candidate is at least
  `m` flits emptier. Default 0 (off). Top-level `ADAPT_MARGIN` parameter on
  `noc_top` (only effective when the deterministic port is an Odd–Even
  candidate, i.e. no-drain / `DET_OE ≥ 1`).

## v5 — `oe_x21` deterministic route, source-column Odd–Even rule exists

- `route_compute.oe_x21` (DET_VARIANT 2): `oe_fixed`, except column-2 →
  column-1 packets go vertical first (balances the column-1/2 vertical links
  that `oe_fixed` overloads; complement traffic link load 4 → 2).
- The `x == src_x` exception in the Odd–Even candidate table (eastbound
  vertical turn allowed at the source column).

## v4 — (history superseded by the restructured docs)

*No per-feature entries survive the snapshot for v4 and earlier; see
`DECISIONS.md` and the per-module `spec` references in the RTL headers.*

## v3 — Source-parity deterministic route

- `route_compute.oe_par` (DET_VARIANT 1): packets from even-parity sources
  (`src_x ⊕ src_y = 0`) take the vertical port first wherever Odd–Even allows;
  others use `oe_fixed`. Spreads the deterministic mode over both dimension
  orders, always within `oe_cand`.

## v2 — No-drain policy with deterministic Odd–Even

- `SWITCH_POLICY = 1` (`mode_switch_direct`): the non-adaptive mode becomes
  `oe_fixed` (a member of `oe_cand`) and `active_mode` follows `req_mode` one
  cycle later with traffic in flight — no drain, no injection block, no
  watchdog. The injection-block handshake and watchdog become unused.

## v1 — Drain-and-switch baseline

- The original HYSTERoC design point: XY ↔ adaptive Odd–Even with the global
  drain-and-switch FSM (`transition_controller`), EWMA-filtered congestion
  estimate, and dual-threshold hysteresis. Reference configurations and the
  conventional single-threshold hybrid baseline are defined against this point.
