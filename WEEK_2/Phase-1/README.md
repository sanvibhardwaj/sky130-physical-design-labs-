# Phase 1 — ORFS Execution in GitHub Codespaces

## Overview

This document records the execution of the OpenROAD RTL-to-GDS flow inside GitHub Codespaces, covering repository setup (Task 1.1) and a full flow run (Task 1.2).

**Repository forked:** [vsdip/vsd-scl180-orfs](https://github.com/vsdip/vsd-scl180-orfs)

**Note on PDK used:** The original repository is configured for the licensed **SCL180 (FS120)** PDK, which requires access from a private/licensed source. As the SCL180 PDK link was not available at the time of this run, the mentor advised switching to the open-source **Sky130hd** PDK, which is already bundled with the ORFS (OpenROAD-Flow-Scripts) submodule included in the repository — no manual PDK download or copy steps were required. All results below reflect the **Sky130hd / `gcd`** design run.

---

## Task 1.1 — Repository Setup

**Steps performed:**
1. Forked the repository to a personal GitHub account.
2. Launched GitHub Codespaces from the fork (**Code → Codespaces → Create codespace on main**).
3. Allowed the devcontainer to build fully (OpenROAD, Yosys, KLayout, and noVNC preinstalled).
4. Verified the environment was ready by checking core tool versions.

**Evidence:**

#### Screenshot — Devcontainer Build Success
![Devcontainer Build](WEEK_2\Phase-1\images/devcontainer_build.png)

#### Screenshot — Tool Version Checks
```bash
openroad -version
yosys -V
python3 --version
make --version
```
![Tool Versions](WEEK_2\Phase-1\images/02_tool_versions.png)

---

## Task 1.2 — Run Testcase in Cloud (Sky130hd / `gcd`)

**Steps performed:**
1. Navigated to `orfs/flow/` inside the Codespace.
2. Confirmed the `sky130hd` platform was already available under `orfs/flow/platforms/` — no PDK setup needed.
3. Ran the full RTL-to-GDS flow:
   ```bash
   cd orfs/flow
   make DESIGN_CONFIG=./designs/sky130hd/gcd/config.mk
   ```
4. Allowed the flow to run to completion without interruption.

### Stage-by-Stage Evidence

| Stage | Log file checked | Result |
|---|---|---|
| **Synthesis** | `logs/sky130hd/gcd/base/1_synth.log` | Completed — 441 library cells linked, `1_synth.odb`/`.sdc` written, 0.85s |
| **Floorplan** | `logs/sky130hd/gcd/base/2_1_floorplan.log` | Completed — design area 2441 µm², 43% utilization, 1.97s |
| **Placement** | `logs/sky130hd/gcd/base/3_5_place_dp.log` | Completed — design area 3518 µm², 62% utilization, 0.81s |
| **CTS** | `logs/sky130hd/gcd/base/4_1_cts.log` | Completed — design area 3995 µm², 71% utilization, 25.87s |
| **Routing** | `logs/sky130hd/gcd/base/5_2_route.log` | Completed — **0 antenna, 0 net, 0 pin violations**, 52.24s |
| **Final GDS** | `results/sky130hd/gcd/base/6_final.gds` | Generated — 840,414 bytes |
| **Final Timing** | `reports/sky130hd/gcd/base/6_finish.rpt` | **WNS: -2.19 ns | TNS: -92.21 ns** |

**Evidence screenshots:**

#### Synthesis Completion Proof
![Synthesis Completion](./screenshots/03_synthesis_completion.png)

#### Floorplan Stage Log Snippet
![Floorplan Log](./screenshots/04_floorplan_log.png)

#### Placement Completion Proof
![Placement Completion](./screenshots/05_placement_completion.png)

#### CTS Log Snippet
![CTS Log](./screenshots/06_cts_log.png)

#### Routing Completion Proof
![Routing Completion](./screenshots/07_routing_completion.png)

#### Final GDS Generation Proof
![Final GDS Proof](./screenshots/08_final_gds_proof.png)

#### Final Timing Report Snippet (WNS/TNS)
![Timing Report](./screenshots/09_timing_report.png)

---

## Run Summary

| Metric | Value |
|---|---|
| Design | `gcd` |
| Platform | `sky130hd` |
| Total runtime | **152 seconds** (~2.5 min) |
| Peak memory | 631 MB |
| Final WNS | -2.19 ns |
| Final TNS | -92.21 ns |
| Routing DRC/antenna/pin violations | 0 |

## Errors Faced and Resolution

| Issue | Resolution |
|---|---|
| SCL180 PDK link unavailable (licensed foundry IP, not publicly distributable) | Mentor advised switching the target platform to **Sky130hd**, an open-source PDK already bundled with the ORFS submodule in the repository. No PDK download/copy was required — the flow ran directly using `make DESIGN_CONFIG=./designs/sky130hd/gcd/config.mk`. |

No other errors were encountered; all pipeline stages (synthesis → floorplan → placement → CTS → routing → GDS export) completed cleanly on the first run.

---

## Conclusion

The full RTL-to-GDS flow was successfully executed end-to-end inside GitHub Codespaces using the Sky130hd open-source PDK, producing a final GDS file and timing reports with zero routing violations. Total flow runtime was 152 seconds.
