# Decisions log — HYSTERoC

Every non-trivial design choice recorded here, numbered to match the in-code references. **Fix numbers are stable** — never renumber; a code comment `"DECISIONS 38"` must keep pointing at the same decision. Add new decisions with the next free number.

The originating spec document ("spec v5") is the upstream source of truth for the FROZEN values below; this log explains *why* the spec chose what it chose and what was considered and rejected, where that history survived in the source headers.

---

## 1. Network and packet format

### D1. 4×4 mesh, 2-bit coordinates, node `n = 4·y + x` — FROZEN
A 4×4 mesh keeps the whole network tractable for exhaustive checks (X/Y route tables, candidate-set emptiness) while still exhibiting multi-hop congestion. 2-bit coordinates suffice. x grows East, y grows South (the natural reading order of the mesh).

### D2. Flit = `{type[1:0], data[15:0]}`; packet = exactly 4 flits — FROZEN
Header + 2 body + tail gives the checker a second body flit to verify ordering and the integrity word (`tail = hdr ^ body1 ^ body2 ^ 16'hC3A5`) without a length field. Rejected: variable-length packets with a length in the header (more state in every router, no benefit for the routing question). See D31 for why the router itself does not enforce the length.

### D3. Header field order `{dst_x, dst_y, src_x, src_y, pid}` — FROZEN
Destination first so the numerically most significant bits are the routing decision; source follows (used by the Odd–Even source-column rule and the parity variants); pid last (unused by the DUT, used by the scoreboard key `{src, pid}`).

### D4. Port ids N=0, S=1, E=2, W=3, Local=4 — FROZEN, and separate from tie-break order
Id assignment is an encoding; the *tie-break priority* (East, South, West, North in adaptive mode) is a different decision (the DUT's spec freezes both, and `route_select` keeps them distinct in code and comments).

## 2. Router datapath

### D5. Input FIFO: depth 4, first-word-fall-through, `ready_in = (count != DEPTH)` — FROZEN
`ready_in` depends only on registered `count` — never on `pop` — so a full FIFO does not accept a push in the same cycle as a pop. Rejected: "pop-aware ready" (accept when full and popping): it creates a combinational path from downstream backpressure into the upstream ready, which complicates the global timing and the X-propagation argument, for one cycle of throughput that a depth-4 FIFO rarely needs. Illegal push/pop are ignored, never corrupting state.

### D6. Router outputs: `out_valid` only together with `out_ready` — FROZEN
`out_valid` is asserted **only** in a cycle in which `out_ready` is 1, so `valid && !ready` never occurs on a router output. Consequence: the "hold while stalled" rule is trivially satisfied at the router, and the sink-side hold rule is the only visible hold obligation. The hop monitor checks this (H4).

### D7. Packet-level reservation, round-robin per output, pointer advances only on accepted grant
The allocator (`switch_allocator` + five `round_robin_arbiter`s) reserves an output from the header transfer to the tail; the arbiter's pointer moves **only** when the transfer actually happened (a stalled grant is re-offered). Round-robin gives starvation freedom with no weight logic and is the natural counterpart to the E/S/W/N tie-break (which is for *route choice*, not arbitration).

### D8. The router is type-driven; no per-port flit counter
`packet_tracker` holds `locked`/`locked_port` between the header and the tail, using the flit *type* bits. Saving 2 flip-flops per port × 80 ports was not the primary goal — the real reason is that the packet length is a *source contract* (D2): the router never needs to count, and a longer packet (UVM `hy_lengths_test` sends up to 16 flits) is routed correctly by the same header→tail reservation. Malformed streams are detected by the sinks/scoreboard, not the router. See README §9 item 2.

### D9. Neighbour occupancy is registered once (one cycle old)
`noc_router.occ_q` registers the sideband; the congestion estimate and the route choice see the same value. This removes the occupancy comparators from the input-to-register path (timing) and from the control-to-data path (no torn decisions). The cost is one cycle of staleness in both consumers, accepted deliberately.

### D10. Boundary directions excluded by masking to 0, with a separate `any_avail` flag
An unavailable direction contributes 0 to `max_occ`, which is identical to excluding it (0 is the minimum) — but the two cases are *not* the same, so `any_avail` distinguishes "no direction available" from "available and empty" (e.g. a corner router's estimate). Illegal occupancy values 5..7 (never produced by `input_fifo`) are clamped to 4 to keep the congestion map monotonic and bounded.

## 3. Congestion estimation and hysteresis

### D11. Congestion signal = max neighbour FIFO occupancy, mapped 0,64,128,192,255 — FROZEN
A *local, distributed* estimate: each router looks only at the FIFOs it would send into. The 5-level map gives the EWMA and the thresholds a fixed scale (occupancy 2 ↔ 128 ↔ the default single threshold). Rejected: link-utilisation or latency-based signals (need extra state and a different interface, and would confound the ablation: the claim is about *switching policy*, not about signal quality).

### D12. EWMA with α = 1/8, `S = 8·E`, `S_next = S − (S>>3) + C` — FROZEN
A shift-only filter (no multiplications) with an 11-bit state that cannot overflow for `C ≤ 255` and needs no signed arithmetic. From reset the state converges exactly: rising from below it lands on `8·C`; falling from above it settles at `8·C + 7` while `E` settles exactly at `C`. `SAMPLE_PERIOD = 1` synthesises it with no period counter (16 flip-flops × 16 routers saved); higher periods exist for sensitivity runs. `enable = 0` freezes state and counter.

### D13. Global signal = max of the 16 per-router EWMA estimates (and max of the 16 raw values)
A single network-wide controller needs a single input; the max is the conservative choice (it tracks the *worst* local congestion, which is what an adaptive mode is for). Rejected: the mean (a single hotspot would be diluted), and per-region controllers (a much larger design; see D25 for how the no-drain policy reuses one global request without paying the drain).

### D14. Dual thresholds THIGH=160 / TLOW=96 (DEFAULT), plus a single-threshold 128 for the baselines
The band is the dead zone: with a signal oscillating inside (TLOW, THIGH) the controller switches *zero* times. TLOW < THIGH is enforced at compile time. The single-threshold point (128 = mapped occupancy 2) exists **only** as the controlled baseline so that the ablation matrix (`docs/experiments.md` §1) varies one axis at a time.

### D15. `req_mode` is a request; `active_mode` has exactly one writer
`hysteresis_controller.req_mode` never changes a mode by itself (`mode_change_request` is advisory). The transition FSM decides when a switch is allowed. `enable = 0` holds the request outside RUN so it cannot be retargeted mid-transition.

## 4. Mode-switch policies

### D16. Drain-and-switch is the baseline policy (SWITCH_POLICY = 0)
A mode change requires demonstrated quiescence: injection blocked, every source acknowledged, the network drained, and quiescence *re-checked in the switching cycle*. This makes the mode switch provably safe with respect to in-flight packets by construction (there are none) and isolates the switching-policystudy from the routing-safety question. The cost — injection stalls during transitions — is measured, not hidden (`performance_monitor` blocked-period metrics).

### D17. FSM rule set: latched target, level acks, re-checked quiescence, watchdog never forces a switch
`RUN → QUIESCE_REQUEST → WAIT_ACK → DRAIN → SWITCH → RESUME → RUN`, with a sticky ERROR (until reset) if the drain is not seen within `WATCHDOG_LIMIT_CYCLES`. `active_mode` is written only in SWITCH and only when `network_drained` is still true; if quiescence is lost, the FSM returns to DRAIN *without* switching. RESUME lasts 1 + `POST_RESUME_MASK_CYCLES` cycles with EWMA and hysteresis frozen. A timeout NEVER forces a switch — a misbehaving source stalls the network safely (the watchdog test expects ERROR, not a hang).

### D18. No-drain policy (SWITCH_POLICY = 1) is safe only because both modes share the Odd–Even turn set
`mode_switch_direct` lets `active_mode` follow `req_mode` one cycle later with traffic in flight. The safety argument is that the switch is a change of *preference* within one turn set: `oe_fixed`/`oe_par`/`oe_x21` are always members of `oe_cand`, and packet-level reservation means a packet holding an output keeps it. If either mode ever produced a non-Odd–Even turn, the no-drain policy would be unsafe — `hy_hop_monitor`'s legality check is the regression net for exactly that.

### D19. Deterministic Odd–Even variants: `oe_fixed` (v2), `oe_par` (v3, source parity), `oe_x21` (v5)
Each was added to displace the naive "deterministic = XY" choice in the no-drain policy: `oe_fixed` (fixed Local > E > W > vertical priority) is the baseline; `oe_par` spreads the deterministic mode over both dimension orders by source parity; `oe_x21` specifically balances the column-1/column-2 vertical links that `oe_fixed` overloads (complement traffic: link load 4 → 2). All are members of `oe_cand` by construction.

### D20. v6 relative margin `ADAPT_MARGIN` (default 0 = off)
In adaptive mode, keep the deterministic port unless another candidate is at least `m` flits emptier. A stability knob between "always adaptive" and "always deterministic-route"; off by default so the baseline numbers are the pure min-occupancy policy.

## 5. Bench and measurement

### D21. `traffic_generator`: open-loop creation, closed-loop injection, drops counted
Creation is open-loop (rate-controlled, destination by pattern); injection is ready/valid with a start rule that always finishes a started packet. A full creation queue drops (counted; offered = created + dropped — the metric stays honest under overload). PRNG is xorshift32 seeded from `SEED`/`NODE`, never zero.

### D22. Packet timestamps use a shared `now` counter; latency is modulo 2¹⁶
Total latency (creation → tail) and network latency (header offered → tail) are 16-bit modulo differences. Queued-but-not-admitted packets are *not* part of `src_empty` (they wait for the block) but keep their creation time — so the transition-induced waiting is visible in total latency. Runs are sized far below 65536 cycles.

### D23. `packet_sink.ej_ready` is a registered random bit
`ej_ready` must never depend combinationally on `ej_valid` — otherwise a ready/valid loop is possible, and the timing/X arguments get much harder. Always / random / bursty / never modes are driven from a registered bit.

### D24. `performance_monitor` is an observer only, and windowed
It never drives the network; the only output that drives anything is `now`. Windowed quantities count only inside `warmup..warmup+measure−1`. Ping-pong is defined on *completed* switches with both switches inside the window (T_PP = 50/100/200/400 counted simultaneously). The progress watchdog is explicitly empirical evidence, not a proof — it is reported as such.

### D25. One global controller, not per-region control
Per-region mode control would need a mode-agreement protocol between adjacent regions and a different safety argument for the no-drain policy. A single global request (D13, D15) is the minimal design that can answer the research question; the file structure (`transition_controller` as a separate module with a single-writer `active_mode`) is chosen so that a per-region variant would be a drop-in later.

### D26. Statistics are first-class outputs, but gated off for the overhead comparison
`ENABLE_STATS = 0` zeroes all thirteen counters/stamps. The FSM never uses them. The FPGA overhead comparison (README §9, `docs/fpga.md`) uses the gated-off number; simulation runs keep them on.

## 6. Verification decisions

### D27. UVM environment first, unit testbenches second
The reference model (`hy_model`) is a cycle-exact re-implementation of the *control path*, fed with the observed DUT inputs and compared every cycle — so a control regression fails in the cycle it is introduced, not 5000 cycles later in a latency distribution. The data path is checked independently (scoreboard, hop monitor, latency auditors). Per-block testbenches existed upstream of this snapshot and are superseded by this environment; its input-file list is in `sim/hysteroc_tb.f`.

### D28. Interface timing: drive after the edge (NBA), sample in the active region
No clocking blocks — the same rule runs on every simulator. `ej_ready` independence (D23, D38 in the old numbering) is part of the contract.

### D29. The 16 sources live inside `hy_if`
`src_empty[n] = !(src_busy[n] | src_force_busy[n])`, `inject_block_ack[n] = inject_block_req & src_empty[n]`. This lets 16 independent drivers own one bit each without a contended interface, and gives tests a one-bit knob (`src_force_busy`) to provoke the watchdog.

### D30. Inherent X-propagation is checked, not hardened
`hy_sva.a_no_x_control` asserts no X/Z on control and handshake outputs in simulation. There is deliberately no hardware X-immunity — it would cost area for a research prototype and is documented as a limitation (README §9).

## 7. Recorded in-code references (kept stable)

These are the numbers the code actually cites (from the pre-restructure snapshot). Their entries are reconstructed above; the in-code citations are the anchors.

| Citation in code | Subject | Entry above |
|---|---|---|
| DECISIONS 36 | neighbour sideband registered one cycle old (`noc_top`, `noc_router`) | D9 |
| DECISIONS 38 | ready never depends on valid; `ej_ready` registered; H4 hold rule | D6, D23 |
| DECISIONS 41 | `src_empty` = no admitted packet queued, no injection in progress | D29 |
| DECISIONS 56 | packet start rule shared by `traffic_generator` and the UVM inject driver | D21 |
| DECISIONS 58 | zero-load latency = distance + 4 (4-flit packet into an empty network) | D2, D32* |

\* D58's "distance + 4" is the latency consequence of the 4-flit contract (D2): the farthest node is 6 hops away and the network adds the packet length; the test asserts it exactly.
