# FPGA integration notes

`noc_top` and its router/control submodules are synthesisable Verilog-2001 with no vendor primitives, no latches and no SystemVerilog constructs in `rtl/` (the include lists are `.sv` file shells only). The FPGA top is **not** in this repository; this document is the integration path.

## 1. What to synthesise and what to replace

| Synthesise | Replace |
|---|---|
| `rtl/router/*`, `rtl/control/*`, `rtl/top/noc_top.v`, `rtl/common/hysteroc.vh` | the sources/sinks: `rtl/bench/traffic_generator.v`, `rtl/bench/packet_sink.v` |
| | the observer: `rtl/bench/performance_monitor.v` (keep it for bring-up; remove or `ENABLE_STATS`-gate it for the overhead comparison) |

## 2. Resource shape (expectations)

* The fabric cost is dominated by 16 routers × (5 FIFOs + 5× `packet_tracker` + 5× `route_compute` + 5× `route_select` + 5× arbiter + crossbar). The FIFOs are the only candidates for BRAM: depth 4 × 18 bit is small enough to map to distributed RAM or LUTs — **do not** burn a block RAM per FIFO without measuring; 80 FIFOs × 1 BRAM is not viable on most parts.
* `ENABLE_STATS = 0` removes all thirteen 32-bit transition counters and stamps (tens of flip-flops × 13). Use it for the FPGA overhead number and keep it at 1 for simulation runs.
* `SAMPLE_PERIOD = 1` (default) synthesises the EWMA update every cycle with **no** period counter (16 flip-flops/router saved per router).
* Thresholds are `parameter`s, so the comparators fold to constants — the hysteresis controller is a handful of gates once synthesised.

## 3. Timing

The neighbour-occupancy sideband is registered once per router (`occ_q`), so the occupancy comparison fan-in does not sit on the input-to-register path. The global max-of-16 tree (`global_congestion`/`global_raw`) is a single combinational reduction feeding one hysteresis comparator — closure on a single clock is expected at NoC-local frequencies; the transition FSM is a 7-state controller whose transitions are all register-to-register. If `global_congestion` becomes critical at high frequency, register `max16` outputs once (the EWMA/`enable` freeze semantics already tolerate an extra cycle of the request signal? — **no**: document any such change in [DECISIONS.md](../DECISIONS.md) first, because `hy_model` is cycle-exact).

## 4. Constraints stub

A minimal Xilinx-style constraints skeleton (adjust clock period and pin names to your board):

```tcl
create_clock -period 10.000 -name sysclk [get_ports clk]
set_property PACKAGE_PIN <CLK_PIN> [get_ports clk]
set_property PACKAGE_PIN <RSTN_PIN> [get_ports rst_n]
# 16 local data/type/valid/ready ports: 16x18 bits + handshake.
# Consider an Ethernet-AXI-Stream / custom serdes wrapper for the 16 local ports
# instead of direct pins on large meshes.
set_false_path -from [get_ports rst_n]
```

## 5. Bring-up order

1. Synthesise `noc_top` alone, then `+ performance_monitor` + a `traffic_generator`/`packet_sink` wrapper. Run the UVM tests on the same parameterisation in simulation *first* (the `hy_model` is cycle-exact and will catch integration mis-wires in one cycle).
2. Check the mesh wiring again against the direction rule (out-N → in-S, …) — the `noc_top` header is the reference.
3. Confirm the mode never changes with traffic in flight under `SWITCH_POLICY=0` (the SVA `a_mode_change_only_after_switch` exists for exactly this), then move to the no-drain point.
