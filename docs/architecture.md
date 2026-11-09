# Architecture — HYSTERoC

Full technical reference for the design. If you only read one doc after the README, read this one.

Conventions: **FROZEN** = pinned by the spec (changing it invalidates the threshold calibration and the checked test vectors); **DEFAULT** = tunable parameter.

---

## 1. Network model

| Property | Value | Source |
|---|---|---|
| Mesh | 4×4 = 16 routers | `hysteroc.vh` `HY_MESH_DIM` / `HY_NUM_ROUTERS` (FROZEN) |
| Coordinates | 2 bit each, 0..3; x grows **East**, y grows **South** | spec 3.1 (FROZEN) |
| Node id | `n = 4·y + x` | used everywhere (flattening `16n+:16` etc.) |
| Ports per router | 5: N=0, S=1, E=2, W=3, Local=4 | `HY_PORT_*` (FROZEN) |
| Flit | 18 bit = `{flit_type[1:0], data[15:0]}`; types: 00 header, 01 body, 10 tail, 11 reserved | spec 4.1 (FROZEN) |
| Packet | header + 2 body + tail, **exactly 4 flits** | spec 4.2 (FROZEN — a source contract, not router-enforced, see §9 of README) |
| Header | `{dst_x[15:14], dst_y[13:12], src_x[11:10], src_y[9:8], pid[7:0]}` | spec 4.3 (FROZEN) |
| Input FIFO | depth 4, first-word-fall-through, `ready_in = (count != DEPTH)` | `input_fifo`, spec 5.2 (FROZEN) |
| Link rule | `valid` held and `data` stable while `valid && !ready`; `ready` never depends combinationally on `valid` | spec 3.2 |
| Router output rule | `out_valid` is asserted **only** in a cycle where `out_ready` is 1 (so `valid && !ready` never occurs on a router output) | `noc_router` header |

Because the flit type travels with the data, the router is **type-driven**: `packet_tracker` learns the output reservation from the header flit and holds it until the tail. No per-port flit counter exists.

---

## 2. Module catalogue

### 2.1 `rtl/router/`

#### `input_fifo` (`input_fifo.v`)
Synchronous FIFO holding `{flit_type, data}`. `ready_in = (count != DEPTH)` depends **only** on registered `count` — never on `pop` — so a full FIFO does not accept a push in the same cycle as a pop (no combinational path back through the router). Illegal push (full) / pop (empty) are ignored, never corrupting state. Simultaneous push+pop at 1..DEPTH−1 entries leaves `count` unchanged and advances both pointers. Storage is not reset; `valid_out` gates its use (first-word-fall-through: `data_out`/`type_out` always show the head).

#### `packet_tracker` (`packet_tracker.v`)
One instance per input port. Parses the header (`dst_x/dst_y/src_x/src_y` — `src_y` and the pid are decoded but unused by the router) and holds `locked`/`locked_port` (one-hot output) from the header transfer until the tail. Emits `req_valid`/`req_new`/`req_port` to the allocator; `proto_err` on protocol violations. Per-port state is exactly `locked` + `locked_port`.

#### `route_compute` (`route_compute.v`)
Purely combinational. Inputs: router `(x, y)`, header `(dst_x, dst_y)` and `(src_x, src_y)` — note **no incoming-direction input** (spec 5.4). Outputs are 5-bit one-hot port masks:

* `xy_port` — deterministic XY: X first, then Y; Local at the destination. Exactly one bit.
* `oe_cand` — the legal adaptive Odd–Even candidate set (1 or 2 bits).
* `oe_fixed` — one member of `oe_cand` by fixed priority Local > East > West > vertical (deterministic Odd–Even, DET_VARIANT 0).
* `oe_par` — `oe_fixed`, except even-parity sources (`src_x ⊕ src_y = 0`) go vertical-first wherever Odd–Even allows (DET_VARIANT 1).
* `oe_x21` — `oe_fixed`, except column-2 → column-1 packets go vertical-first (balances the column-1/2 vertical links, DET_VARIANT 2).

The Odd–Even **candidate table** (spec v5 §5.4; `ex = dst_x − x`, `ey = dst_y − y`):

```
ex==0, ey==0 : Local
ex==0, ey!=0 : vertical toward destination
ex >0, ey==0 : East
ex >0, ey!=0 : vertical if (x odd) OR (x == src_x);  East if (dst_x odd) OR (ex != 1)   [independent tests]
ex <0        : West; also vertical (if ey != 0) when x is even
```

Every candidate is a progress direction toward an in-range destination, so it cannot point outside the mesh — no separate boundary mask is needed for the 2-bit/4×4 case (exhaustively checked in `tb_route_compute` upstream of this repo snapshot). The set is never empty (checked exhaustively).

#### `route_select` (`route_select.v`)
Chooses one output port for a header flit.

| `active_mode` | Selection |
|---|---|
| XY (0) | `xy_port`, after masking out unavailable (boundary) ports; occupancy and `oe_cand` ignored |
| ADAPT (1) | the available `oe_cand` candidate whose neighbour input FIFO has the **lowest** occupancy; Local first; ties E → S → W → N |

Boundary rule: a cardinal direction with `avail_* = 0` is never selected in either mode; Local is always available. A single available candidate is selected regardless of its occupancy. `sel_valid = |sel_port`. Selection may change cycle-to-cycle while a header is stalled (it follows the occupancy sideband); that is safe because nothing is reserved until the header transfer is accepted and every candidate is legal.

Optional **v6 bias** (`ADAPT_MARGIN = m > 0`): in adaptive mode keep the deterministic port (`xy_port` input = det port, an Odd–Even candidate when `DET_OE ≥ 1`) unless another candidate's neighbour FIFO holds at least `m` fewer flits. This trades adaptivity for determinism/stability; default `m = 0` (off).

#### `round_robin_arbiter` (`round_robin_arbiter.v`)
Parameterised (`N`, default 5) round-robin arbiter for **one output**. Priority starts at `ptr`; the first requesting input in order `ptr, ptr+1, …` (wrapping) wins. The pointer moves **only** when `advance` says the granted transfer actually took place; on advance it moves to granted index + 1, making the winner lowest priority. `grant` is one-hot or zero, always a subset of `req`.

#### `switch_allocator` (`switch_allocator.v`)
Packet-level output allocation for one 5×5 router (five `round_robin_arbiter`s). Inputs: `req_valid/req_new/req_port` and `locked/locked_port` from the five trackers, plus `out_ready` from downstream. A packet that owns an output keeps it, whatever the stalls; a new arbitration is offered only when the output is free. Exposes `in_grant` (pop strobe to each input FIFO), `xbar_sel` (25-bit one-hot per output), `out_busy`, and `alloc_err` on inconsistency.

#### `crossbar` (`crossbar.v`)
5×5 AND-OR mux. `sel[o*5+i] = 1` connects input `i` to output `o`. `out_valid[o] = |sel[o]`; idle outputs carry all-zero data/type (keeps idle links quiet and makes "no select → no data" checkable). The allocator guarantees at most one select per output and never selects one input twice, so no flit is duplicated.

#### `noc_router` (`noc_router.v`)
Integration of the seven blocks above. The neighbour-occupancy sideband (`nbr_occ_in[3d+:3]`, direction d = N0 S1 E2 W3) is **registered once** (`occ_q`), so the congestion estimate and the route choice see the same one-cycle-old value, and the boundary availability bits ride along. `alloc_ready` masks `out_ready` with `{1'b1, avail_W, avail_E, avail_S, avail_N}` so a boundary output never transfers. Outputs: `fifo_occ[3p+:3]` (drain detection), `router_empty = ~|h_valid & ~|tr_locked`, `ewma_out`, `cong_now`, `max_occ_now`, `proto_err`, `alloc_err`.

### 2.2 `rtl/control/`

All modules are Verilog-2001, synchronous active-low reset, no latches. Inputs/outputs marked "unused" in a header are lint-silenced with `verilator lint_off UNUSEDSIGNAL`.

#### `congestion_estimator` (`congestion_estimator.v`)
Combinational. `max_occ = max` over **available** cardinal directions of the registered neighbour occupancy; an unavailable direction is masked to 0 (identical to excluding it, since 0 is the minimum), and `any_avail` distinguishes the two cases. `cong = map(max_occ)` through the FROZEN table `{0,64,128,192,255}`; an illegal occupancy 5..7 (never produced by `input_fifo`) is clamped to 4 so the map stays monotonic and bounded. Local is never an input.

#### `ewma_filter` (`ewma_filter.v`)
Fixed-point EWMA: `S = 8·E`, `S_next = S − (S>>3) + C`, `E = S>>3`, with `S` an 11-bit unsigned value. From reset with `C ≤ 255`, `S` never exceeds `8·255 = 2040`, so no overflow is possible; `S − (S>>3) ≥ 0` always, so no signed arithmetic is needed. `SAMPLE_PERIOD` (default 1) updates `S` once every N enabled cycles using the sample on that cycle; the period counter exists only when `SAMPLE_PERIOD > 1` (saves 16 flip-flops × 16 routers). `enable = 0` freezes both `S` and the counter. Behaviour on a constant input: rising from below, `S` climbs by ≥ 1 per tick and lands exactly on `8·C`; falling from above, it settles at `8·C + 7` while `E` settles exactly at `C`.

#### `hysteresis_controller` (`hysteresis_controller.v`)
`req_mode` is the **requested** mode; it never changes a mode by itself (`mode_change_request = req_mode != active_mode` is advisory only — the transition FSM decides when a switch is allowed). `enable = 0` holds `req_mode`.

| Parameters | Behaviour |
|---|---|
| `DUAL=1, USE_EWMA=1` | **HYSTERoC**: `value_ewma ≥ THIGH (160) → ADAPT`; `value_ewma ≤ TLOW (96) → XY`; hold in between |
| `DUAL=1, USE_EWMA=0` | **Ablation B**: same band on the raw value |
| `DUAL=0, USE_EWMA=1` | **Ablation A**: `value_ewma ≥ 128 → ADAPT`, else XY |
| `DUAL=0, USE_EWMA=0` | **Conventional hybrid**: `value_raw ≥ 128 → ADAPT`, else XY |

`TLOW < THIGH` is enforced at compile time (simulation `$finish` with an error otherwise). `req_mode` resets to XY.

#### `drain_detector` (`drain_detector.v`)
Per node: `router_empty[n]`, `src_empty[n]`, `link_active[n]`, all describing the **same** cycle. `drained_now = (&router_empty) & (&src_empty) & ~(|link_active)`. `network_drained` rises only after `CONFIRM` (default 2) consecutive quiet cycles and drops in the same cycle as any activity. Because injection is blocked before DRAIN is entered, once drained the network stays drained; the confirm window and the SWITCH re-check are extra conservatism.

#### `transition_controller` (`transition_controller.v`)
The global drain-and-switch FSM (`STATE` encodings: RUN 0, QUIESCE 1, WAIT 2, DRAIN 3, SWITCH 4, RESUME 5, ERROR 6):

```
RUN ── req != active ──► QUIESCE_REQUEST ──► WAIT_ACK ── all acks ──► DRAIN
   ▲                        │                     │                       │
   │                        ▼                     ▼                       ▼
   └─────────────────── RESUME ◄── SWITCH ◄── network_drained ─────────┘
                              (re-checked; if lost, back to DRAIN without switching)
   any of QUIESCE/WAIT/DRAIN past the watchdog ──► ERROR (sticky until reset)
```

* `active_mode` is written **only** in SWITCH, only when `network_drained` is true **in that same cycle**. Target mode is latched at the transition start and never changed; later `req_mode` changes are ignored until RUN.
* `inject_block_req = 1` in QUIESCE/WAIT/DRAIN/SWITCH/ERROR; `ewma_enable` and `hys_enable` are 1 **only** in RUN (EWMA frozen and request held during a transition and the post-resume mask).
* **Watchdog:** cycle 1 is the first cycle `inject_block_req = 1`. If `network_drained` has not been seen (in DRAIN) by cycle `WATCHDOG_LIMIT_CYCLES`, the FSM goes to ERROR. A drain seen exactly in cycle LIMIT is a success. ERROR keeps `inject_block_req = 1`, never switches, never resumes; only reset recovers. **A timeout never forces a switch** (requires `LIMIT ≥ 4`).
* Statistics (all zero when `ENABLE_STATS = 0`; the FSM does not use them): `cycle_cnt`, per-/cumulative `blocked`/`ackwait`/`drain` cycle counts, `transitions`, and `t_block/t_ack/t_drained/t_switch/t_resume` stamps of the last transition. Note `t_switch == t_resume` by construction (injection is released in RESUME, which is also the first cycle the new mode is visible).

#### `mode_switch_direct` (`mode_switch_direct.v`)
No-drain alternative with the **same ports** (so `noc_top` can instantiate either). `active_mode <= req_mode` one cycle later, at any time, with traffic in flight. Safety rationale — a header routed in *either* mode takes only Odd–Even-legal turns: the non-adaptive mode (`oe_fixed`/`oe_par`/`oe_x21`) is always a member of `oe_cand`, and the adaptive mode picks from that same set. A packet already holding an output keeps it (wormhole); only the next header uses the new mode. No injection block, no handshake, no watchdog: `inject_block_req`, `transition_busy`, `drain_timeout` are constants 0; `ewma_enable`/`hys_enable` are constants 1; `state_o` is 0 (RUN).

### 2.3 `rtl/top/`

#### `noc_top` (`noc_top.v`)
16 routers (`node n = 4·y + x`), flattened local ports (node n at `[16n+:16]` / `[2n+:2]` / `[n]`), plus the transition protocol with the sources (`inject_block_req`, per-source `inject_block_ack`, `src_empty`) and the observation outputs (mode, global congestion, drain/transition status, per-router FIFO occupancy `fifo_occ_all[15n+:15]`, per-router EWMA `ewma_all[8n+:8]`, error aggregation, statistics).

* Link wiring: out-N of (x,y) → in-S of (x,y−1); out-S → in-N of (x,y+1); out-E → in-W of (x+1,y); out-W → in-E of (x−1,y). Boundary inputs: `valid = 0`; boundary `out_ready = 0`. `out_ready` of a sender is driven by the receiver's `in_ready` (registered FIFO state only, so no combinational loop).
* Sideband: each router receives, for each direction d, the **current** occupancy of the neighbour FIFO that would receive its flits, and registers it.
* Global aggregation: `global_congestion = max16(EWMA)`, `global_raw = max16(raw)` (a function `max16`, spec 6.5).
* `XY_ONLY = 1` forces `req_mode = XY` — the controller never leaves RUN (reference point).
* `SWITCH_POLICY = 0` instantiates `transition_controller`; `= 1` instantiates `mode_switch_direct` with `DET_OE = 1 + DET_VARIANT`.

### 2.4 `rtl/bench/` (simulation only — do not synthesise)

#### `traffic_generator` (`traffic_generator.v`)
One packet source per node. `rate` (16 bit): a packet is created in a cycle if a 16-bit random is below `rate` — offered load per node = `4·rate/65536` flits/cycle (1.0 = one flit every cycle). Destination patterns: 0 uniform (any other node), 1 hotspot (fraction `hot_pct/256` to `hot_node`, rest uniform), 2 transpose, 3 complement, 4 neighbour (x+1 mod 4). Created packets enter a `2**QPTR_W` queue; a full queue **drops** (counted; offered = created + dropped). Injection FSM: while idle and `inject_block_req = 0`, the queue head is started and its four flits are offered, `valid` held and data stable until accepted; a started packet always finishes. `inject_block_ack = inject_block_req & idle`; `src_empty = idle` (queued-but-not-admitted packets are *not* part of `src_empty` — they wait for the block to be released, and their creation timestamps preserve the waiting in the latency numbers). Packet: `{hdr, ts_create, ts_inject, tail}` with `tail = hdr ^ ts_create ^ ts_inject ^ 16'hC3A5`. PRNG: xorshift32, seeded from `SEED` and `NODE`, never zero.

#### `packet_sink` (`packet_sink.v`)
One ejection endpoint per node. `ej_ready` is a **registered** random bit (probability `ready_thresh/256` per cycle; 255 = always) — it never depends combinationally on `ej_valid`. Every accepted flit is checked against the generator format: header destination = NODE, body/tail ordering, no reserved types, `tail == hdr ^ body1 ^ body2 ^ 16'hC3A5`. After a violation it resynchronises on the next header.

#### `performance_monitor` (`performance_monitor.v`)
Observer only; the sole output that drives anything is `now` (the shared free-running cycle counter, 0 in the first cycle after reset — wired to every `traffic_generator` and `packet_sink`). Measurement window: cycles `warmup .. warmup + measure − 1` (`meas_en` inside the window, `meas_done` sticky after it). Metrics (spec 12/13):
* *Switching (12.2):* completed switches, `req` toggles, `blk` starts; **ping-pong** = a switch whose predecessor was ≤ T_PP cycles earlier (T_PP counted simultaneously at 50/100/200/400); dwell sum/count/min/max.
* *Transition cost (12.3):* blocked periods (start = first `inject_block_req = 1` cycle, end = last such cycle, release cycle included), drain length stats, windowed blocked cycles → **injection-blocked fraction** = `w_blocked / measure`.
* *Packets/throughput:* count, sum, sum-of-squares and max of total and network latency (mean/std derived); flits accepted at sources/sinks in the window; whole-run totals; in-flight = injected − delivered.
* *Progress watchdog:* the longest run of consecutive cycles with flits in flight and no flit delivered (`STALL_LIMIT` default 2000). A stall detection is **empirical** evidence, not a proof.

---

## 3. Deadlock-safety argument

**Claim scope:** the adaptive Odd–Even *turn set* is deadlock-free under Chiu's turn model, and the *implementation choices* (below) preserve that property through mode switches. **This is not a whole-design formal proof** (see README §9).

1. **Odd–Even turn rules.** A turn is illegal if it can participate in a cyclic channel-dependency cycle. The two Chiu rules used by `route_compute` (§2.1 table): eastbound traffic may only turn vertical on an odd column (or at the source column), and westbound traffic may only turn vertical on an even column; the destination-column exception (ex == 1, dst_x odd) covers the final hop. The candidate set is acyclic under these rules, and it is minimal (every candidate reduces `|ex| + |ey|`), so every route terminates.
2. **Drain-and-switch (POLICY 0).** The mode register changes only when the network is empty, so no packet is ever routed by two different turn sets. XY itself is trivially deadlock-free.
3. **No-drain (POLICY 1).** Both modes draw from the *same* turn set (`oe_fixed`/`oe_par`/`oe_x21` ∈ `oe_cand`), so a mode change mid-flight is a change of *preference* within a safe set, not of the set itself. A packet holding an output keeps it (wormhole reservation is header→tail), so the switch cannot redirect a packet that already owns resources.
4. **Allocation.** Packet-level reservation (header→tail) plus round-robin arbitration means a held output cannot be stolen, so the channel-dependency graph of the *implementation* has no extra edges beyond the turn set.

What is **not** proved: freedom from livelock under pathological adversarial scheduling, and liveness of the drain handshake when a source misbehaves (the watchdog covers it by refusing to switch — the network stalls safely rather than deadlocking).

---

## 4. Parameter reference (`noc_top`)

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `XY_ONLY` | bit | 0 | 1 = force XY, never switch (reference) |
| `DUAL` | bit | 1 | 1 dual-threshold, 0 single-threshold |
| `USE_EWMA` | bit | 1 | 1 hysteresis on EWMA value, 0 on raw |
| `THIGH` | 8 bit | 160 | high threshold (ADAPT at ≥) |
| `TLOW` | 8 bit | 96 | low threshold (XY at ≤) |
| `THRESH_SINGLE` | 8 bit | 128 | single threshold (= mapped occupancy 2) |
| `SAMPLE_PERIOD` | int | 1 | EWMA update period |
| `WATCHDOG_LIMIT_CYCLES` | int | 1000 | drain watchdog (≥ 4) |
| `POST_RESUME_MASK_CYCLES` | int | 0 | hysteresis freeze after a switch |
| `ENABLE_STATS` | bit | 1 | 0 removes all transition counters |
| `DRAIN_CONFIRM` | int | 2 | quiet cycles before `network_drained` |
| `SWITCH_POLICY` | bit | 0 | 0 drain-and-switch (paper baseline), 1 no-drain |
| `DET_VARIANT` | int | 0 | no-drain deterministic mode: 0 `oe_fixed`, 1 `oe_par`, 2 `oe_x21` |
| `ADAPT_MARGIN` | int | 0 | v6 relative bias of adaptive toward the deterministic port (0 = off) |

The UVM testbench (`tb/env/test.sv`) mirrors this parameter list as top-level parameters so one snapshot can be re-elaborated per point.

---

## 5. Verification architecture (summary)

The UVM environment (`tb/`) is itself a small architecture worth orienting in before writing a test. The full walkthrough is [verification.md](verification.md); the key idea: the **reference model of the control path** (`hy_model`) is a cycle-exact SystemVerilog re-implementation of `ewma_filter + congestion_estimator + hysteresis_controller + transition/mode-switch + drain_detector + stats`, run in lockstep with the DUT on the same inputs. Every global control output is compared every cycle, so control-path regressions fail in the exact cycle they are introduced. The data path is checked independently (scoreboard, hop monitor, queue/latency auditors).

---

## 6. FPGA integration (summary)

`noc_top` synthesises as-is; the only integration obligation is that the bench blocks (`rtl/bench/`) are **not** synthesised — the real system replaces them with real sources/sinks. `ENABLE_STATS = 0` removes the 32-bit transition counters for the resource-overhead comparison (the counters are otherwise free-running and cost flip-flops/plus LUTs). Timing: the critical path of the design is the neighbour-occupancy comparison fan-in (5 ports × 4 directions × 16 routers) — registering the sideband in each router (already done) is what keeps this off the clock path. See [fpga.md](fpga.md).
