# Week 4 — Phase 1: Wrapper Dependency Analysis

## Objective
Analyze `__user_project_wrapper.v` from the VSDSquadron SoC repository to identify all instantiated modules, trace the design hierarchy, and locate the RTL source files required before running synthesis.

**File analyzed:** `caravel/verilog/rtl/__user_project_wrapper.v`

---

## 1. Module Hierarchy / Dependency Tree

```
user_project_wrapper
    ├── user_project_la_example     (conditional — only if `ifdef LA_TESTING is defined)
    ├── user_project_gpio_example   (conditional — only if `ifdef GPIO_TESTING is defined)
    └── debug_regs                  (unconditional — always included)
            → leaf module, no further sub-instantiations
```

---

## 2. RTL Files Required (default synthesis run)

| # | Module | File Path | Included by default? |
|---|--------|-----------|----------------------|
| 1 | `user_project_wrapper` | `caravel/verilog/rtl/__user_project_wrapper.v` | Yes (top module) |
| 2 | `debug_regs` | `caravel/verilog/rtl/debug_regs.v` | Yes |
| 3 | `defines.v` (macro definitions, e.g. `MPRJ_IO_PADS`) | `caravel/verilog/rtl/defines.v` | Yes (required dependency, not a module — see Section 5) |
| 4 | `user_project_la_example` | Not traced (conditional, `LA_TESTING`) | No, by default |
| 5 | `user_project_gpio_example` | Not traced (conditional, `GPIO_TESTING`) | No, by default |

For a normal RTL-to-GDS run (no testing flags defined), **files 1, 2, and 3** are required.

---

## 3. Explanation of Module Hierarchy

- **`user_project_wrapper`** is the top-level module. It mostly defines the pin interface (Wishbone slave ports, Logic Analyzer signals, GPIOs, power pins) and wires together whatever user logic is placed inside it.
- **`debug_regs`** is instantiated unconditionally inside the wrapper. It implements two 32-bit registers (`debug_reg_1`, `debug_reg_2`) that can be read/written over the Wishbone bus — a small scratchpad register block for debug/observability. It is a **leaf module**: only register declarations and an `always` block with `if/else` conditions; no further sub-modules.
- **`user_project_la_example`** and **`user_project_gpio_example`** are only instantiated inside `` `ifdef LA_TESTING `` and `` `ifdef GPIO_TESTING `` blocks respectively. Since these flags are not defined during a normal flow run, these modules are excluded from the design by the Verilog preprocessor before synthesis ever sees them.
- Other conditional blocks found in the file (`` `ifdef USE_POWER_PINS ``, `` `ifdef SIM ``) are **not module instantiations** — they only conditionally add power pins to the wrapper's own port list, or add a dummy `assign io_out = io_in;` statement for simulation. Excluded from the dependency tree since they are not sub-modules.

---

## 4. Method Used

1. Opened `__user_project_wrapper.v` in VS Code and manually scanned for instantiation patterns: `module_type instance_name ( ... );`
2. Distinguished true module instantiations from port declarations (`input`/`output`/`inout`) and simple `assign` statements, which are not dependencies.
3. Separated modules into **unconditional** (always included) and **conditional** (`` `ifdef ``-gated) categories.
4. Used a terminal search to locate the definition file for the unconditional module:
   ```bash
   cd ~/Desktop/vsdsquadron-soc
   grep -rn "module debug_regs" .
   ```
   Output:
   ```
   ./caravel/verilog/rtl/debug_regs.v:2:module debug_regs (
   ```
5. Opened `debug_regs.v` and confirmed it contains only registers and combinational/sequential logic — no further instantiations, confirming it as a leaf module.

---

## 5. Follow-up Finding: Missing Macro Dependency

During synthesis in Phase 2, a compile error revealed a dependency **missed** in the initial module-only trace:
```
ERROR: Unimplemented compiler directive or undefined macro `MPRJ_IO_PADS
```
This macro (used in `input [`MPRJ_IO_PADS-1:0] io_in`) is defined in `defines.v`, not in a module — so it was not caught by module-instantiation tracing alone. Located via:
```bash
grep -n "MPRJ_IO_PADS" caravel/verilog/rtl/defines.v
```
```
`define MPRJ_IO_PADS_1 19
`define MPRJ_IO_PADS_2 19
`define MPRJ_IO_PADS (`MPRJ_IO_PADS_1 + `MPRJ_IO_PADS_2)
```
**Lesson:** Phase 1 dependency analysis must also check for `` `define `` macros used (via `` `include `` or macro references), not just `module` instantiations.

[Insert screenshot: grep search results for MPRJ_IO_PADS]

---

## 6. Debugging Notes

- **Issue:** VS Code's "Search across files" (`Ctrl+Shift+F`) initially returned no results / showed "You have not opened or specified a folder."
- **Cause:** Only a single file was open in VS Code, not the full repository folder, so search was scoped to just that one file.
- **Resolution:** Used `File → Open Folder` to open the repo root (`vsdsquadron-soc`) properly. As a reliable fallback, used the terminal command `grep -rn "module <name>" .` from the repo root instead of relying on the VS Code UI.

[Insert screenshot: terminal grep confirming debug_regs.v location]

---

## Summary

Phase 1 identified three required RTL files for the default (non-testing) synthesis configuration: `__user_project_wrapper.v`, `debug_regs.v`, and `defines.v`. The conditional testing modules (`user_project_la_example`, `user_project_gpio_example`) are documented but excluded from the default dependency set, as they are only included when their respective `` `ifdef `` flags are explicitly defined.
