# FPGA integration

The synthesised design is `rtl/top/noc_top.v` and its router/control submodules
(Verilog-2001, no vendor primitives). Simulation-only blocks in `rtl/bench/`
(`traffic_generator`, `packet_sink`, `performance_monitor`) must be replaced by
real endpoints.

See [../docs/fpga.md](../docs/fpga.md) for the integration notes (resource
shape, BRAM mapping, timing, bring-up order). A constraints stub for a Xilinx
flow is in that document; no vendor project (Vivado/Quartus) is checked in.
