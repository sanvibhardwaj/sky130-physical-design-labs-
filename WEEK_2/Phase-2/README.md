# Phase 2 — Toolchain Understanding (Devcontainer Deep Dive)

Studied `.devcontainer/Dockerfile` and `.devcontainer/install-openroad.sh` to understand what tools power the flow, where they come from, and how the flow itself is orchestrated end to end.

---
## Task 2.1: Toolchain Mapping (Devcontainer Deep Dive)

### Objective
Study `.devcontainer/Dockerfile` and `.devcontainer/install-openroad.sh` to identify every major tool used in the RTL-to-GDS flow, where each one comes from, and what stage of the flow it serves.

### Files Studied
- `.devcontainer/Dockerfile`
- `.devcontainer/install-openroad.sh`

### Toolchain Table

| Tool | Installed From | Purpose in Flow | Stage Used |
|---|---|---|---|
| **OpenROAD** | Prebuilt binary, downloaded via `wget` from a DigitalOcean Spaces CDN (`vsd-labs...openroad-linux-x64.tar.gz`), extracted and installed to `/opt/openroad/bin` with a symlink at `/usr/local/bin/openroad`. Shared libraries registered via `ld.so.conf.d`. Not compiled from source. | Core P&R engine — floorplanning, placement, CTS, routing, timing/power analysis | Floorplan → Placement → CTS → Routing → STA |
| **Yosys** | Prebuilt binary from the OSS CAD Suite GitHub release (`YosysHQ/oss-cad-suite-build`), pulled via `curl`, extracted to `/opt/oss-cad-suite` | RTL synthesis — converts Verilog into a gate-level netlist | Synthesis |
| **TritonCTS** | Bundled inside the OpenROAD binary — no separate install step | Clock tree synthesis — builds clock distribution network, minimizes skew | CTS |
| **FastRoute** | Bundled inside the OpenROAD binary — no separate install step | Global router — plans coarse wire paths before detailed routing | Global Routing |
| **OpenSTA** | Bundled inside the OpenROAD binary — no separate install step | Static timing analysis — checks setup/hold timing, reports WNS/TNS | Post-placement/CTS/routing STA |
| **KLayout** | `.deb` package downloaded directly from klayout.org (v0.30.9-1), installed via `apt-get install .../klayout.deb` | GDSII viewer/editor — visual layout inspection, DRC/LVS | Sign-off / Layout inspection |
| **Python** | Ubuntu `apt` package manager (`python3`, `pip`, `numpy`, `pandas`, `matplotlib`, `jinja2`, `xlsxwriter`) | Scripting glue — flow automation and report generation | Used throughout the flow |
| **Make** | `apt` package manager | Build automation — runs Makefile targets that invoke the flow | Environment setup / flow invocation |
| **Git** | `apt` package manager (`git`, `git-lfs`) | Version control — clones repos, pulls large tracked files via LFS | Environment setup |

### Key Observations

**Install pattern:** Every tool in this setup is installed as a prebuilt binary or package — nothing is compiled from source. OpenROAD comes from a custom CDN, Yosys from OSS CAD Suite's GitHub releases, KLayout as a `.deb`, and the rest via `apt`. This keeps container build time fast, but it makes the setup dependent on those external URLs staying live and unchanged.

**Bundled vs. standalone tools:** TritonCTS, FastRoute, and OpenSTA are not separate installs — they're compiled into the OpenROAD binary itself and invoked as internal commands within the OpenROAD shell, not as standalone executables.

**Fragility risk:** This same "external URL dependency" pattern caused the `efabless/openlane:current` Docker tag issue debugged on Day 1 — a reminder that prebuilt-binary pipelines trade build speed for a dependency on third-party hosting staying available.

### Key Takeaway
Understanding where each tool comes from isn't just bookkeeping — it explains why the environment behaves the way it does. Knowing that OpenROAD's CTS/routing/STA capabilities are bundled (not separate tools) clarifies why a single `openroad` command handles multiple flow stages, and knowing the install sources helps debug future "tool not found" or version-mismatch issues by pointing straight to where a tool actually came from.

---
## Task 2.2: Flow Architecture Explanation

### Objective
Explaining , in my own words, how OpenROAD-flow-scripts (ORFS) automates the RTL-to-GDSII flow — what it automates, how the Makefiles orchestrate it, where synthesis hands off to physical design, where timing is checked, and where the final GDS is produced.

### Structured Explanation

### 1. What ORFS automates
Before I understood ORFS, I thought each PD stage (synthesis, floorplan, placement, CTS, routing) had to be run as a separate manual step, feeding the output of one tool into the next by hand.Instead of manually running every tool and stage, ORFS provides scripts and Makefiles that coordinate the different steps. ORFS basically removes that manual handoff.The flow starts with an RTL design and automatically takes it through synthesis, floorplanning, placement, clock tree synthesis, routing, timing analysis, and finally GDS generation.

In simple terms, ORFS acts as a framework that connects different EDA tools and controls the order in which they are executed.

### 2. How Makefiles orchestrate the flow
Makefiles are more like a checklist that decides the order things run in and makes sure nothing runs out of order. There's a top-level Makefile with targets like `synth`, `floorplan`, `place`, `cts`, `route`, and a separate `config.mk` where design-specific stuff goes — things like the design name, clock period, which platform (I used sky130hd), utilization %, etc. When I run something like `make cts`, it checks that placement has already finished, because CTS needs placement's output to exist first. The Makefile itself doesn't run Yosys or OpenROAD commands directly — it calls a Tcl script for that stage, and that script is what actually talks to the tool. So Makefile = sequencing/dependencies, Tcl script = actual tool commands.

### 3. Where synthesis ends and physical design begins
Synthesis ends the moment Yosys produces a gate-level netlist — a purely logical description mapped to standard cells from the target library, with no physical coordinates or layout information yet. Physical design begins at floorplanning, where the die and core area are defined and initial placement locations for macros and I/O pins are set. From that point on, every stage (placement, CTS, routing) is assigning real x/y coordinates and physical wiring to what was, until then, just a logical netlist.

### 4. Where timing is checked
Timing is checked at multiple points, not just once at the end, because timing estimates get more accurate as more physical information becomes available:
- **Post-synthesis** — a rough estimate based on the gate-level netlist alone, before any physical layout exists
- **Post-placement** — using estimated (not yet extracted) parasitics based on approximate wire lengths
- **Post-CTS** — now includes the real clock tree structure and its actual delays/skew
- **Post-routing (signoff STA)** — the final and most accurate check, using RC parasitics extracted from the actual routed wires

OpenSTA is the engine performing all of these checks, and it's this progression that lets timing violations get caught and fixed early rather than only being discovered after the full flow completes.

### 5. Where GDS is produced
The GDSII file — the actual manufacturable mask layout — is generated at the very end of the flow, after routing is complete and the design has passed DRC (design rule checks) and LVS (layout-vs-schematic) clean. This final step is a "stream-out," where the routed layout is written into the GDSII format that a fab can actually use to produce photomasks. Everything before this point (synthesis, placement, CTS, routing) builds toward this file; nothing downstream of GDS generation modifies the design further — it's the final deliverable of the flow.

### Key Takeaway
ORFS isn't just a script that runs tools back-to-back — it's a dependency-driven pipeline where each stage's Makefile target both depends on and validates the previous stage's output, and timing is treated as a continuously-refined check rather than a single gate at the end. Understanding this structure makes it much easier to debug: if a `make` target fails, the fix usually lies in the stage just before it, not in the target itself.
