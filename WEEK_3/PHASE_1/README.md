# Week 3, Phase 1: Standalone Block Verification (spi_master)

Part of: **Block-Level Verification of VSDSquadron SoC**.
Phase 2 (all standalone tests) is in `Phase2_README.md`.

---

## 1. What this phase is about

Before a chip is built, its design (written in Verilog) is **simulated on a computer** and checked. This is called **verification**.

In Phase 1 I:

- ran one standalone test (`spi_master`) with `make`,
- decided whether it passed or failed,
- studied the Makefile to understand what `make` really does,
- looked at the waveform to see what the chip did.

**Result: spi_master = PASS.**

---

## 2. Words used in this report

| Word | Simple meaning |
|---|---|
| **SoC** | System on Chip: a whole small computer (CPU, memory, UART, GPIO...) on one chip |
| **Caravel** | Open-source chip frame. Its **management SoC** is a small RISC-V CPU system that controls the chip |
| **RTL** | The design written in Verilog (what we simulate) |
| **DUT** | Design Under Test: the design being checked |
| **Testbench** | A Verilog file that is *not* hardware. It drives the DUT and checks the results |
| **Firmware** | A small C program that runs on the chip's CPU during a test |
| **Simulator** | Program that imitates the chip over time. Here: Icarus Verilog (`iverilog` compiles, `vvp` runs) |
| **Makefile** | A recipe file. Typing `make` runs its commands automatically |
| **Waveform (.vcd)** | A recording of every signal against time. Viewed in GTKWave |
| **PASS / FAIL** | PASS: the design behaved as expected. FAIL: it did not, or the test never finished |

---

## 3. Setup

- **Machine:** Ubuntu inside a VirtualBox VM
- **Repository:**
  ```bash
  git clone https://github.com/vsdip/vsdsquadron-soc
  cd vsdsquadron-soc/caravel_mgmt_soc_litex/verilog/dv/tests-standalone
  ```
- **Tools:** Icarus Verilog (`iverilog`, `vvp`), RISC-V GCC 8.3.0 (`riscv64-unknown-elf-gcc`), GTKWave 3.4.0
- **Compiler name:** `riscv32-unknown-elf-gcc` was not found on my system. The Makefile does not use it. It uses the 64-bit compiler with 32-bit flags (`-march=rv32i -mabi=ilp32`), which is installed.
- **Paths:** the Makefile expects its files under `/home/vsduser/...`. These paths exist on my VM, so no change was needed.

---

## 4. Task 1: Run the SPI master test

```bash
cd spi_master
make clean
make
```

**Result: PASS.**

![spi_master PASS](C:\Users\sanvi\sky130-physical-design-labs-\WEEK_3\PHASE_1\image_pdfs\image1.pdf)

How to read the output:

| Output line | Meaning |
|---|---|
| `SPI value = 0x57 (should be 0x57)` | The testbench reads the SPI value and prints it next to the expected value |
| `+5000 cycles` | The testbench waits 5000 clock cycles between steps |
| `Monitor: Test SPI Master (RTL) Passed` | The verdict: every value matched |
| `spi_master_tb.v:179: $finish called at 5027837500 (1ps)` | The testbench ended the simulation after about 5 ms |

Files left in the folder after `make`: `spi_master.c` (firmware), `spi_master.hex`, `spi_master.lst`, `spi_master_tb.v` (testbench), `RTL-spi_master.vcd` (waveform).

---

## 5. Task 2: Understanding the Makefile

Key parts of the Makefile (shortened):

### 5.1 Settings at the top

```make
BLOCKS := $(notdir $(PWD))          # test name = folder name

CONFIG ?= standalone                # test without the full Caravel frame
SIM    ?= RTL                       # simulate the original Verilog

# Paths
DESIGNS ?= /home/vsduser
export PDK_ROOT ?= /home/vsduser/.ciel
export PDK      ?= sky130A
export CARAVEL_VERILOG_PATH ?= $(DESIGNS)/caravel/verilog
export CORE_VERILOG_PATH    ?= $(DESIGNS)/caravel_mgmt_soc_litex/verilog
export USER_PROJECT_VERILOG ?= $(DESIGNS)/verilog

export VIP_PATH      := $(CORE_VERILOG_PATH)/dv/vip
export FIRMWARE_PATH := $(CORE_VERILOG_PATH)/dv/firmware

# Toolchain
export GCC_PATH   ?= /home/vsduser/riscv64-unknown-elf-gcc-8.3.0-2019.08.0-x86_64-linux-ubuntu14/bin
export GCC_PREFIX ?= riscv64-unknown-elf

# CPU selection
CPU ?= vexriscv
CPUFLAGS_vexriscv := -march=rv32i -mabi=ilp32 -D__vexriscv__
```

| Setting | Meaning |
|---|---|
| `?=` | Use this value unless it is overridden from the command line |
| `:=` | Set the value immediately |
| `export` | Make the variable visible to the programs `make` starts |
| `PDK = sky130A` | The SkyWater 130 nm process kit |
| `GCC_PREFIX` | Compiler used for the firmware |
| `-march=rv32i -mabi=ilp32` | Generate 32-bit RISC-V code |

### 5.2 CPU selection and the main targets

```make
ifeq ($(CPU),vexriscv)
    LINKER_SCRIPT := $(FIRMWARE_PATH)/sections_vexriscv.lds
    SOURCE_FILES  := $(FIRMWARE_PATH)/start_caravel_vexriscv.s
    CPUFLAGS      := $(CPUFLAGS_vexriscv)
endif

export IVERILOG_DUMPER = fst        # waveform is written in FST format

all: $(BLOCKS).vcd $(BLOCKS).lst    # what plain "make" builds
hex: $(BLOCKS).hex                  # "make hex" builds only the firmware hex
```

- **Linker script (`.lds`):** tells the linker where code and data go in memory.
- **Startup file (`start_caravel_vexriscv.s`):** assembly code that runs first on the CPU and then calls `main()`.
- **`prepare-generated`:** creates a symlink (shortcut) so the linker script finds `regions.ld`.

### 5.3 Compile rule (iverilog)

```make
%.vvp: %_tb.v %.hex
ifeq ($(SIM),RTL)
  ifeq ($(CONFIG),caravel_user_project)
	iverilog -Ttyp -DFUNCTIONAL -DSIM -DUSE_POWER_PINS -DUNIT_DELAY=#1 \
	    -f$(VERILOG_PATH)/includes/includes.rtl.caravel \
	    -f$(USER_PROJECT_VERILOG)/includes/includes.rtl.$(CONFIG) \
	    -o $@ $<
  else
	iverilog -Ttyp -DFUNCTIONAL -DSIM -DUNIT_DELAY=#1 \
	    -y /home/vsduser/caravel/verilog/rtl \
	    -I /home/vsduser/caravel/verilog/rtl \
	    -f $(VERILOG_PATH)/includes/includes.rtl.$(CONFIG) \
	    -o $@ $(CARAVEL_PATH)/rtl/__user_project_wrapper.v $<
  endif
endif
```

For a standalone test the `else` branch is used.

| Symbol / option | Meaning |
|---|---|
| `%` | Wildcard. `%.vvp` means "any name ending in .vvp" |
| `$@` | The target being built (the `.vvp` file) |
| `$<` | The first prerequisite (the `_tb.v` testbench) |
| `-y <dir>` | Search this folder for module definitions |
| `-I <dir>` | Search this folder for `include` files |
| `-f <file>` | Read the list of design files from this file |
| `-D<NAME>` | Define a switch the Verilog code checks (`FUNCTIONAL`, `SIM`...) |
| `-o <file>` | Name of the compiled output |

### 5.4 Simulation rule (vvp) and clean

```make
%.vcd: %.vvp
ifeq ($(SIM),RTL)
	vvp $<
	mv $@ RTL-$@                     # waveform becomes RTL-spi_master.vcd
endif

clean:
	rm -f *.elf *.hex *.bin *.vvp *.vcd *.lst ...
	rm -rf "$(GEN_INCLUDE_DIR)"

.PHONY: clean hex all prepare-generated check-fw
```

`.PHONY` lists names that are commands, not files.

### 5.5 Build steps in order (pattern from the build outputs, shown for spi_master)

```text
riscv64-unknown-elf-gcc ... -o spi_master.elf start_caravel_vexriscv.s spi_master.c
riscv64-unknown-elf-objcopy -O verilog spi_master.elf spi_master.hex
sed -i 's/^@1000/@0000/' spi_master.hex          # fix the load address
iverilog ... -o spi_master.vvp ... spi_master_tb.v
vvp spi_master.vvp
mv spi_master.vcd RTL-spi_master.vcd
riscv64-unknown-elf-objdump -d -S spi_master.elf > spi_master.lst
rm spi_master.elf spi_master.vvp
```

Step summary:

| Step | Command | What it does |
|---|---|---|
| 1 | `gcc` | Compiles the firmware into `.elf` |
| 2 | `objcopy` + `sed` | Makes `.hex` and fixes its address marker so the firmware loads at address 0 of the simulated flash |
| 3 | `iverilog` | Compiles the testbench and design files into `.vvp` |
| 4 | `vvp` | Runs the simulation and writes the waveform |
| 5 | `mv` | Renames the waveform to `RTL-<test>.vcd` |
| 6 | `objdump` | Makes `.lst` (readable assembly listing) |
| 7 | `rm` | Deletes the temporary `.elf` and `.vvp` |

---

## 6. Verification flow

```text
make
 -> gcc:          test C firmware + startup file   -> .elf
 -> objcopy+sed:  .elf                             -> .hex (loaded into simulated flash)
 -> iverilog:     testbench (_tb.v) + design files -> .vvp
 -> vvp:          runs .vvp                        -> waveform (.vcd) + printed log
 -> testbench prints PASS or FAIL, then calls $finish
```

**How the testbench talks to the design:**

1. The testbench creates the clock, reset and power-up sequence and connects them to the DUT.
2. The `.hex` firmware is loaded into a simulated flash memory. The CPU fetches its program from it.
3. The firmware writes marker values to its output registers. They appear on the `la_output` wires, which the testbench reads (as `checkbits`).
4. The testbench compares what it sees with what it expects.

**How PASS / FAIL is decided:** not by the Makefile. The testbench prints the verdict with `$display`:

- **PASS:** the expected values or markers are seen, and "Passed" is printed.
- **FAIL:** a mismatch is printed, or the **timeout** block prints "Failed" because the test did not finish in time.

---

## 7. Waveform of spi_master

Opened with `gtkwave RTL-spi_master.vcd`, after adding signals from `spi_master_tb`.

![spi_master waveform](C:\Users\sanvi\sky130-physical-design-labs-\WEEK_3\PHASE_1\image_pdfs\image2.pdf)

| Signal | What it shows |
|---|---|
| `VDD1V8`, `power1` | Power on |
| `RSTB` | Reset released (high) |
| `clock` | Chip clock running (looks like a solid bar because it is zoomed out) |
| `flash_clk`, `flash_csb`, `flash_io0/1` | The CPU reading its program from flash |
| `spi_clk` | SPI activity, the block under test |

**Why the waveform helps debugging:** the PASS/FAIL message says *whether* something is wrong. The waveform shows *where and when* it went wrong.

---

## 8. Block diagram (hand-drawn)

`C:\Users\sanvi\sky130-physical-design-labs-\WEEK_3\PHASE_1\image_pdfs\spi_master_flow.jpg`

Shows: Makefile -> Compilation -> Testbench -> DUT -> Simulation -> Result (PASS / FAIL), with the real file names and commands of this test.

---


