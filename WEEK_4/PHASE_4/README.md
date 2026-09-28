# Week 4 — Phase 4: Run the RTL-to-GDS Flow

## Objective
Run the complete OpenROAD flow — floorplanning, placement, clock tree synthesis, routing, fill insertion, and final GDSII generation — for `user_project_wrapper` at a 100 MHz target, debugging each stage failure as it occurred.

---

## Stage-by-Stage Summary

| Stage | Command | Output | Status |
|---|---|---|---|
| Synthesis | `make ... synth` | `1_synth.odb` | ✅ |
| Floorplan | `make ... floorplan` | `2_floorplan.odb` | ✅ |
| Placement | `make ... place` | `3_place.odb` | ✅ |
| CTS | `make ... cts` | `4_cts.odb` | ✅ |
| Global + Detailed Routing | `make ... route` | `5_2_route.odb` | ✅ |
| Fill Insertion | (automatic, part of `route`) | `5_3_fillcell.odb` | ✅ |
| Final Merge / GDSII | `make ... finish` | `6_final.gds` | ✅ |

[Insert screenshot: final `Log` summary table showing all stages with elapsed time / memory]

---

## Stage 1 — Floorplanning

Ran cleanly on the first attempt. Sub-stages completed automatically:
- **`2_1_floorplan`** — die/core area sizing, I/O pin placement
- **`2_2_floorplan_macro`** — no macros found (pure standard-cell design), skipped
- **`2_3_floorplan_tapcell`** — 260 tapcells inserted (latch-up prevention)
- **`2_4_floorplan_pdn`** — power distribution network grid generated

[Insert screenshot: floorplan stage completion log]

---

## Stage 2 — Placement

### Issue: I/O pin count exceeded available die perimeter
```
[ERROR PPL-0024] Number of IO pins (637) exceeds maximum number of available positions (508).
Increase die area to 866.32um.
```

**Root cause:** every bit of a multi-bit bus (`la_data_in[127:0]`, `la_data_out[127:0]`, `la_oenb[127:0]`, `io_in[37:0]`, `io_out[37:0]`, `io_oeb[37:0]`, `analog_io[10:0]`, plus scalar signals) counts as an individual physical pin, totaling 637 — far more than the default utilization-based die size could fit around its edge.

**Fix:** explicit die/core sizing added to `config.mk` (see Phase 2), replacing `CORE_UTILIZATION`-based automatic sizing (the two methods are mutually exclusive and cannot both be set).

[Insert screenshot: PPL-0024 error]

---

## Stage 2 (continued) — Global Routing Failure: Undriven Output Ports

### Issue
```
[ERROR RSZ-0074] failed to build tree from gloubal routes: found route to 1 pins, expected 0
```

**Diagnosis process:**
1. Increasing die area repeatedly did not resolve the error — ruled out pure congestion as the cause.
2. Searched synthesis logs for "no driver" warnings:
```bash
grep -i "no driver" logs/sky130hd/user_project_wrapper/base/1_2_yosys.log
```
3. Found four signals declared as outputs but never driven in the default (non-testing) build configuration:
   - `la_data_out[127:0]`
   - `user_irq[2:0]`
   - `io_oeb[37:0]`
   - `wbs_dat_o_user[31:0]`

**Root cause (ties back to Phase 1):** these signals are only driven inside `` `ifdef LA_TESTING `` / `` `ifdef GPIO_TESTING `` conditional blocks in the RTL. Since these flags are correctly undefined in a normal build, the signals were left as dangling outputs with no driver — a category of dependency that Phase 1's module-only hierarchy trace did not surface, since it concerns signal-level connectivity rather than module instantiation.

**Fix:** explicitly tied off the unused outputs at the end of `__user_project_wrapper.v`:
```verilog
assign la_data_out = 128'b0;
assign user_irq = 3'b0;
assign io_oeb = 38'b0;
assign wbs_dat_o_user = 32'b0;
```

[Insert screenshot: tail of edited `__user_project_wrapper.v` showing the four tie-off assign statements]

**Verification before proceeding:**
```bash
grep -c "no driver" logs/sky130hd/user_project_wrapper/base/1_2_yosys.log
```
Confirmed output: `0` — zero undriven signals remained.

---

## Stage 2 (continued) — Global Routing Failure: Incremental Repair

### Issue
After fixing the undriven ports, the **identical** RSZ-0074 error persisted, even after increasing die area to 3000×3000 µm. This ruled out both previously-suspected causes (congestion and undriven nets).

**Diagnosis:** inspected `global_route.tcl` directly to find the exact failing command:
```bash
sed -n '90,145p' ~/Desktop/vsd-scl180-orfs/orfs/flow/scripts/global_route.tcl
```
Found the failure traced to a `repair_timing_helper` procedure calling `global_route -end_incremental` as part of a resizer-driven repair pass following placement/routing.

**Fix:** ORFS provides a documented flow variable to skip this specific step:
```makefile
export SKIP_INCREMENTAL_REPAIR = 1
```
(Reference: OpenROAD-flow-scripts FlowVariables documentation)

With this set, routing completed successfully.

[Insert screenshot: RSZ-0074 error before fix]
[Insert screenshot: final config.mk showing SKIP_INCREMENTAL_REPAIR = 1]

**Note for timing signoff:** since a timing-repair step was skipped, final timing results should be reviewed for any residual violations that this step would normally have addressed.

---

## Stage 3 — Detailed Routing and Fill

Completed successfully immediately after the incremental-repair fix, with no further errors.

**Routing results:**
```
Total wire length on LAYER met2 = 26693 um.
Total wire length on LAYER met3 = 19897 um.
Total wire length on LAYER met4 = 265 um.
Total wire length on LAYER met5 = 74 um.
Total number of vias = 3872.
[INFO ANT-0002] Found 0 net violations.
[INFO ANT-0001] Found 0 pin violations.
```
Zero antenna violations — confirms routing is manufacturing-safe with respect to charge accumulation on long metal wires.

**Fill insertion:**
```
[INFO DPL-0001] Placed 1033958 filler instances.
```
Filler cells were inserted to maintain uniform silicon density across the large empty die area surrounding the small logic core.

[Insert screenshot: detailed routing completion — wire length / via summary]
[Insert screenshot: fill insertion completion]

---

## Stage 4 — Final Database and GDSII Generation

```bash
make DESIGN_CONFIG=./designs/sky130hd/user_project_wrapper/config.mk finish
```

KLayout merged the routed DEF with the sky130 standard-cell GDS library to produce the final layout:
```
[INFO] All LEF cells have matching GDS/OAS cells
[INFO] No orphan cells in the final layout
cp results/sky130hd/user_project_wrapper/base/6_1_merged.gds results/sky130hd/user_project_wrapper/base/6_final.gds
```
"No orphan cells" confirms every cell instance in the design successfully matched a real physical GDS shape — no missing or mismatched library cells.

**Final artifact:** `results/sky130hd/user_project_wrapper/base/6_final.gds`

[Insert screenshot: final GDSII layout view in KLayout]

---

## Deliverables Checklist (per task requirements)

| Deliverable | File |
|---|---|
| Synthesized netlist | `1_synth.odb` |
| Floorplanned design | `2_floorplan.odb` |
| Placed design database | `3_place.odb` |
| Clock-tree inserted design | `4_cts.odb` |
| Routed database | `5_2_route.odb` |
| Final filled database | `5_3_fillcell.odb` |
| Final GDSII | `6_final.gds` |
| Timing reports | generated under `6_report` stage |

---

## Summary

- **What was done:** Ran the complete RTL-to-GDS flow from synthesis through final GDSII, debugging three distinct real issues along the way: I/O pin count vs. die size, undriven conditional output ports, and an incremental timing-repair failure during global routing.
- **Why:** Each fix was necessary to progress the design through a stage that would otherwise fail or produce an invalid database.
- **Key commands:** `make DESIGN_CONFIG=... <stage>` for each flow stage; `grep`/`sed` log analysis for root-causing failures; ORFS flow-variable documentation lookup for `SKIP_INCREMENTAL_REPAIR`.
- **Key concepts:** die/core area sizing, I/O pin-to-perimeter relationship, undriven-net diagnosis via synthesis logs, antenna violations, filler cells, GDS/LEF cell matching.
- **Final output:** Complete, verified `6_final.gds` — the manufacturable layout for `user_project_wrapper`, ready for gate-level verification preparation in later weeks.
