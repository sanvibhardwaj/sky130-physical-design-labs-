# Week 4 — Phase 2: Prepare the ORFS Design Environment

## Objective
Set up a working OpenROAD Flow Scripts (ORFS) design environment for `user_project_wrapper`: organize RTL source files, write a `config.mk` configuration, and resolve tool-path issues so that synthesis runs without missing modules.

---

## 1. Environment Discovered

The local ORFS installation was found at:
```
~/Desktop/vsd-scl180-orfs/orfs/flow
```
containing multiple platform folders: `gf180`, `scl180fs120`, `sky130hd`, `sky130hs`. **`sky130hd`** was selected as the target platform, consistent with prior work in this environment.

[Insert screenshot: ORFS `flow/designs` folder listing showing platform folders]

---

## 2. Design Folder Structure Created

```
flow/designs/sky130hd/user_project_wrapper/     ← config.mk + constraint.sdc
flow/designs/sky130hd/src/user_project_wrapper/ ← RTL source files
```

Commands used:
```bash
mkdir -p ~/Desktop/vsd-scl180-orfs/orfs/flow/designs/sky130hd/user_project_wrapper
mkdir -p ~/Desktop/vsd-scl180-orfs/orfs/flow/designs/sky130hd/src/user_project_wrapper
```

RTL files copied into the source folder:
```bash
cp ~/Desktop/vsdsquadron-soc/caravel/verilog/rtl/__user_project_wrapper.v <src_folder>/
cp ~/Desktop/vsdsquadron-soc/caravel/verilog/rtl/debug_regs.v <src_folder>/
cp ~/Desktop/vsdsquadron-soc/caravel/verilog/rtl/defines.v <src_folder>/
```

**Note:** `defines.v` was not part of the original Phase 1 dependency list — it was discovered later during synthesis (see Section 5) as a missing macro-definition file required by the wrapper.

---

## 3. Final `config.mk`

**Path:** `flow/designs/sky130hd/user_project_wrapper/config.mk`

```makefile
export DESIGN_NAME = user_project_wrapper
export PLATFORM    = sky130hd

export VERILOG_FILES = ./designs/sky130hd/src/user_project_wrapper/defines.v ./designs/sky130hd/src/user_project_wrapper/__user_project_wrapper.v ./designs/sky130hd/src/user_project_wrapper/debug_regs.v

export SDC_FILE = ./designs/sky130hd/user_project_wrapper/constraint.sdc

export SKIP_INCREMENTAL_REPAIR = 1

export DIE_AREA  = 0 0 3000 3000
export CORE_AREA = 30 30 2970 2970
```

[Insert screenshot: `cat config.mk` output]

### Key settings explained
| Variable | Purpose |
|---|---|
| `DESIGN_NAME` | Top-level module name — must exactly match `module user_project_wrapper` in the RTL |
| `PLATFORM` | Target manufacturing technology (sky130hd standard-cell library) |
| `VERILOG_FILES` | All RTL sources, in dependency order — `defines.v` listed **first** since it defines macros used by the wrapper |
| `SDC_FILE` | Path to the timing constraints file (filled in during Phase 3) |
| `DIE_AREA` / `CORE_AREA` | Explicit chip/core dimensions in microns — set manually after the default utilization-based sizing proved too small for the design's I/O pin count (see Phase 4 notes) |
| `SKIP_INCREMENTAL_REPAIR` | Skips a specific resizer repair pass during global routing that was causing a routing failure (see Phase 4 debugging notes) |

---

## 4. Tool Path Issues and Fixes

Both Yosys and OpenROAD were installed on the system but not in the file paths ORFS expected (`tools/install/<tool>/bin/...`). Fixed using wrapper scripts pointing to the real installations.

**Yosys** (installed via oss-cad-suite at `/opt/oss-cad/bin/yosys`):
```bash
mkdir -p ~/Desktop/vsd-scl180-orfs/orfs/tools/install/yosys/bin
cat > ~/Desktop/vsd-scl180-orfs/orfs/tools/install/yosys/bin/yosys << 'EOF'
#!/bin/bash
export LD_LIBRARY_PATH=/opt/oss-cad/lib:$LD_LIBRARY_PATH
exec /opt/oss-cad/bin/yosys "$@"
EOF
chmod +x ~/Desktop/vsd-scl180-orfs/orfs/tools/install/yosys/bin/yosys
```

**OpenROAD** (installed system-wide at `/usr/local/bin/openroad`):
```bash
mkdir -p ~/Desktop/vsd-scl180-orfs/orfs/tools/install/OpenROAD/bin
cat > ~/Desktop/vsd-scl180-orfs/orfs/tools/install/OpenROAD/bin/openroad << 'EOF'
#!/bin/bash
exec /usr/local/bin/openroad "$@"
EOF
chmod +x ~/Desktop/vsd-scl180-orfs/orfs/tools/install/OpenROAD/bin/openroad
```

---

## 5. Missing Macro Discovery (`MPRJ_IO_PADS`)

Initial synthesis attempts failed with:
```
ERROR: Unimplemented compiler directive or undefined macro `MPRJ_IO_PADS
```
Traced to `caravel/verilog/rtl/defines.v`:
```verilog
`define MPRJ_IO_PADS_1 19
`define MPRJ_IO_PADS_2 19
`define MPRJ_IO_PADS (`MPRJ_IO_PADS_1 + `MPRJ_IO_PADS_2)
```
This file was missing from the original Phase 1 dependency list, since it defines a macro rather than a module — Phase 1's hierarchy trace only looked for module instantiations. It was added to `VERILOG_FILES`, **ordered first** so the macro is defined before the wrapper uses it.

[Insert screenshot: synthesis log showing successful Verilog-2005 frontend read of all three files]

---

## 6. Sanity Check — Synthesis

```bash
cd ~/Desktop/vsd-scl180-orfs/orfs/flow
make DESIGN_CONFIG=./designs/sky130hd/user_project_wrapper/config.mk synth
```

**Result:** Synthesis completed successfully — Yosys read all RTL files, resolved the `MPRJ_IO_PADS` macro, mapped the design to sky130 standard cells, and produced `1_synth.odb`.

[Insert screenshot: successful synthesis completion — "End of script", elapsed time, no errors]

---

## Summary

- **What was done:** Created an ORFS design folder for `user_project_wrapper`, organized RTL sources, wrote `config.mk`, and resolved tool-path mismatches for Yosys and OpenROAD via wrapper scripts.
- **Why:** ORFS requires an explicit, correctly-ordered file list and resolvable tool paths before any flow stage can run.
- **Key files:** `config.mk`, wrapper scripts for `yosys` and `openroad`.
- **Key concepts:** environment variables (`export`), Makefile-driven flow configuration, macro/`` `define `` resolution order, symlink vs. wrapper-script tool redirection.
- **Debugging highlight:** discovered and fixed a missing macro-definition dependency (`defines.v`) that Phase 1's module-only hierarchy trace had missed.
- **Final output:** Working `config.mk` and a clean synthesis run producing `1_synth.odb`.
