# Week 3, Phase 2: Run All Standalone Tests

Part of: **Block-Level Verification of VSDSquadron SoC**.
The Makefile flow, setup and glossary are in `Phase1_README.md`. This file covers the remaining standalone tests, their results, and how the failures were debugged.

---

## 1. What this phase is about

I repeated the Phase 1 process for every block under `tests-standalone`:

- ran `make clean` and `make`,
- recorded PASS / FAIL,
- compared each Makefile with the spi_master one,
- debugged the tests that failed, using the logs and waveforms.

### Result at a glance

| Test | Result |
|---|---|
| spi_master | PASS |
| gpio_mgmt | PASS |
| mem | PASS |
| uart | PASS |
| timer | FAIL (timeout). Passes when the testbench time limit is raised |
| irq | FAIL (timeout). Narrowed down, cause not resolved |
| debug | FAIL (timeout). Passes when `debug_in` is set to 0 |

### Extra words used here

| Word | Simple meaning |
|---|---|
| **Marker** | A small number the firmware writes so the testbench knows how far the test has progressed |
| **Timeout** | The testbench's stopwatch. If the test does not finish in time, it prints "Failed" |
| **GPIO** | A pin that can be turned ON and OFF |
| **UART** | A serial link that sends data one bit at a time on one wire |
| **IRQ** | Interrupt request: a signal that says "stop what you are doing and handle this now" |
| **ISR** | Interrupt service routine: the function the CPU runs when an interrupt arrives |

---

## 2. Routine used for every test

```bash
cd <test>
ls
diff ../spi_master/Makefile Makefile      # what is different from spi_master?
make clean
make 2>&1 | tee run.log                   # show output and save it
```

Then: record PASS / FAIL, find the line in the testbench that decides it, and look at the waveform when needed.

---

## 3. Standalone test result table

| tests-standalone | status (sky130) | Notes |
|---|---|---|
| GPIO Mgmt | **PASS** | Finished at about 0.94 ms |
| mem | **PASS** | Finished at about 9.16 ms |
| uart | **PASS** | Finished at about 3.77 ms |
| timer | **FAIL** | Timeout at about 1.25 ms. PASS after raising the time limit (section 6) |
| irq | **FAIL** | Timeout at about 1.25 ms. Cause not resolved (section 7) |
| debug | **FAIL** | Timeout at about 1.5 ms. PASS with `debug_in` set to 0 (section 8) |
| spi_master | **PASS** | Finished at about 5.03 ms (see Phase 1) |

---

## 4. Makefile differences from spi_master

Checked with `diff`. The Makefiles are almost identical, so the flow from Phase 1 applies to every test.

How to read `diff`: `116c116` means line 116 changed. Lines starting with `<` come from the first file (spi_master), lines starting with `>` from the second.

**gpio_mgmt** (`diff ../spi_master/Makefile Makefile`):

```diff
116c116
<       sed -i 's/^@1000/@0000/' $@
---
>       sed -i 's/^@10000000/@00000000/' $@
```

Its firmware is placed at a different address, so the `sed` line that fixes the `.hex` address marker is different.

**irq**:

```diff
57c57
<   SOURCE_FILES  := $(FIRMWARE_PATH)/start_caravel_vexriscv.s
---
>   SOURCE_FILES  := $(FIRMWARE_PATH)/start_caravel_vexriscv.s $(FIRMWARE_PATH)/isr.c
```

It also compiles `isr.c`, the interrupt service routine.

**uart, debug:** their build output shows the same `@10000000 -> @00000000` fix as gpio_mgmt.

---

## 5. Tests that passed

### gpio_mgmt

The testbench README describes it as the simplest "wake-up call" test: the firmware toggles the single `gpio` pin, first a few times quickly (so it can be seen in simulation), then slowly (so it would blink an LED on a board). The testbench prints `Blink.` for each toggle.

```verilog
// gpio_mgmt_tb.v: PASS and FAIL messages
$display("Blink.");                                              // lines 87 to 111
$display("Monitor: Test Mgmt GPIO (RTL) Passed");                // PASS
$display("Monitor: Timeout, Test Mgmt GPIO (RTL) Failed");       // timeout FAIL
```

![gpio_mgmt PASS](screenshots/image3.jpeg)

### mem

Writes to memory and reads back, in two ways: byte write / byte read, and byte write / word read. Each sub-test prints `passed`.

![mem PASS](screenshots/image4.jpeg)

### uart

The firmware sends a message through the UART pin and the testbench listens and checks it. This output shows the whole `make` flow in order: `objcopy`, `sed`, `iverilog`, `vvp`, the hex being loaded, the waveform file opening, the PASS line, and the cleanup.

![uart PASS](screenshots/image5.jpeg)

The compiler prints warnings that `reg_debug_1` and `reg_debug_2` are redefined in `defs.h`. They appear in passing builds too (uart), and the build continues.

---

## 6. Failed test 1: timer (testbench time limit too short)

**Symptom.** Finishing at 1,249,987,500 ps:

```text
Monitor: Timeout, Test GPIO (RTL) Failed
timer_tb.v:63: $finish called at 1249987500 (1ps)
```

The message says "GPIO" although this is the timer test. The wording appears to be reused from the gpio test.

![timer FAIL](screenshots/image6.jpeg)

**What the testbench waits for** (`timer_tb.v`):

```verilog
initial begin
    $dumpfile("timer.vcd");
    $dumpvars(0, timer_tb);

    // Repeat cycles of 1000 clock edges as needed to complete testbench
    repeat (50) begin
        repeat (1000) @(posedge clock);
        $display("+1000 cycles");
    end
    $display("Monitor: Timeout, Test GPIO (RTL) Failed");
    $finish;
end

assign checkbits = la_output[37:32];
```

`repeat (50)` blocks of `repeat (1000)` = 50,000 clock cycles. With a 25 ns clock this is about 1.25 ms, which matches the finish time.

**Waveform.**

![timer waveform](screenshots/image7.jpeg)

| Signal | Observation | Meaning |
|---|---|---|
| `clock`, `RSTB` | Running, released | Not a clock or reset problem |
| `checkbits` | Became `0A` (start marker) and stayed there | The firmware never reached the next checkpoint |
| `countbits` | `DCBA86A7`, `DCBA7547`, `DCBA63E7`... counting down | The timer works and the CPU is alive |

**Hypothesis.** The firmware is working but needs slightly more time than the testbench allows.

**Test (on a backup copy).** Changed one line in `timer_tb.v`:

```diff
-    repeat (50) begin
+    repeat (200) begin
```

Result: **PASS**.

![timer PASS after raising limit](screenshots/image8.jpeg)

**Cause confirmed.** The passing run finished at 1,250,587,500 ps, which is 600 ns (24 clock cycles) after the original timeout. The firmware needed about 24 more cycles than the testbench allowed. This is a testbench time-budget problem, not a design bug.

---

## 7. Failed test 2: irq (narrowed down, not resolved)

**Symptom.** Same 50 x 1000 limit as timer:

```text
Monitor: Timeout, Test IRQ (RTL) Failed
irq_tb.v:63: $finish called at 1249987500 (1ps)
```

![irq FAIL](screenshots/image9.jpeg)

**What the test should do** (`irq.c`, shortened):

```c
volatile uint16_t flag = 0;

void main()
{
    reg_la1_data = 0x5;                  // Signal start of test
    reg_la0_data = 0;
    flag = 0;

    irq_setmask(0);
    irq_setie(1);
    asm volatile ("csrs mie, %0" :: "r"(1u << 11));   // enable MEIE  <-- hand-added line
    irq_setmask(irq_getmask() | (1 << TIMER0_INTERRUPT));

    /* Configure timer for a single-shot countdown */
    reg_timer0_config = 0;
    reg_timer0_data = 3000;
    reg_timer0_update = 1;
    reg_timer0_irq_en = 1;
    reg_timer0_config = 1;

    // Loop, waiting for the interrupt
    while (flag == 0) {
        reg_la0_data = 0x10000;
    }
    reg_la0_data = 0x40000;
    reg_la1_data = 0xA;                  // TB expects PASS milestone
    reg_la1_data = 0xc;                  // Signal end of test
}
```

**What the testbench watches** (`irq_tb.v`):

```verilog
wire [3:0] status;
wire [3:0] checkbits;
assign checkbits = la_output[19:16];
```

`0x10000` is bit 16 of `reg_la0_data`, so `checkbits = 1` means the firmware is in the wait loop. After the interrupt, `0x40000` makes `checkbits = 4`.

**Waveform 1.** `status` reached `5` at about 250 us (the firmware started), but `checkbits` stayed `0` for the whole run. The firmware never reached the wait loop, so this is not the same cause as timer.

![irq waveform status](screenshots/image10.jpeg)

**Experiment (on a backup copy).** Commented out the hand-added line `asm volatile ("csrs mie, ...")`. Then `checkbits` became `1` at about 800 us, so the firmware reached the wait loop. The test still failed, because the interrupt was never handled.

![irq waveform with the line commented out](screenshots/image11.jpeg)

**Checked `isr.c`** (shortened). It reads the pending interrupts, acknowledges the timer, and sets the same `flag` that `irq.c` waits on, so the handler logic is not the cause:

```c
extern volatile uint16_t flag;      // defined in irq.c

void isr(void)
{
    unsigned int irqs = irq_pending();

    if (irqs & (1 << TIMER0_INTERRUPT)) {
        #ifdef CSR_TIMER0_EV_PENDING_ADDR
            reg_timer0_ev_pending = 1;      // acknowledge the timer interrupt
        #endif
        flag = 1;                            // tell the wait loop the alarm rang
        irq_setmask(irq_getmask() & ~(1 << TIMER0_INTERRUPT));
        return;
    }
}
```

**Status: FAIL, not resolved.** Remaining suspects: the startup file may not call `isr()`, or the timer interrupt may not fire.

---

## 8. Failed test 3: debug (testbench input froze the CPU)

**Symptom.** The limit is `repeat (60)` x 1000 cycles:

```text
Executing SRAM write
Debug Test started
Executing SRAM read
Monitor: Timeout, Test Debug (RTL) Failed
debug_tb.v:82: $finish called at 1499987500 (1ps)
```

![debug FAIL](screenshots/image12.jpeg)

**What the firmware does** (`debug.c`, shortened). Shows a start marker, runs two short delay loops, then shows an end marker. The interrupt and UART lines in the file are commented out.

```c
void main()
{
    int j;

    // Start test
    reg_la0_data = 0xa0000000;
    for (j = 0; j < 500; j++);

    // Allow transmission to complete before signalling that the program has ended.
    for (j = 0; j < 160; j++);
    reg_la0_data = 0xab000000;
}
```

**What decides PASS** (`debug_tb.v`). The testbench reads `checkbits` (top 16 bits). `A000` is the start marker and `AB00` is the end marker. It also sets `debug_in = 1` about 1 us after the start ("Debug mode") and leaves it high:

```verilog
assign checkbits = la_output[31:16];

initial begin
    debug_in = 1'b0;
    #1000
    debug_in = 1'b1;                      // Debug mode
    // the long waits and the irq lines after this are commented out
end

always @(checkbits) begin
    if (checkbits == 16'hA000) begin
        $display("Debug Test started");
    end
    else if (checkbits == 16'hAB00) begin
        $display("Monitor: Test Debug (RTL) passed");
        $finish;
    end
end
```

**Waveform (limit raised to 6 ms).**

![debug waveform](screenshots/image13.jpeg)

| Signal | Observation | Meaning |
|---|---|---|
| `checkbits` | `A000`, never `AB00` | Firmware started, never finished |
| `flash_clk`, `flash_csb` | Active at the start, then quiet | The CPU stopped fetching its program |
| `debug_in` | High the whole time | Held high by the testbench |

Raising the time limit 4 times did not help, so this is not a time-limit problem. The CPU is stuck, not slow.

**Test (on a backup copy).** Changed one line in `debug_tb.v`:

```diff
-    debug_in = 1'b1;
+    debug_in = 1'b0;
```

Result: **PASS**, finishing at about 0.72 ms.

![debug PASS with debug_in = 0](screenshots/image14.jpeg)

**Conclusion.** Holding `debug_in` high from the start stops the firmware from finishing. Not known: why the pin does this, and whether the original testbench was meant to hold it high. `debug_in` is the chip's debug input, so this may be intended debug behaviour.

---

## 9. Summary of diagnostics

| Test | Original result | Change made (backup copy) | Diagnostic result | Conclusion |
|---|---|---|---|---|
| timer | FAIL (timeout) | `repeat (50)` -> `repeat (200)` | PASS | Time limit about 24 cycles too short |
| irq | FAIL (timeout) | Commented out the `csrs mie` line | Still FAIL (reaches wait loop) | Not resolved |
| debug | FAIL (timeout) | `debug_in = 1'b1` -> `1'b0` | PASS | Held-high `debug_in` stops the firmware |

Diagnostic changes were made on backup copies (`*.orig`) and the originals were restored afterwards. The repository files were not changed to force a PASS.

---

## 10. How I debugged (the method)

1. Read the failure message (timeout, mismatch, or build error).
2. Read the firmware and testbench to know the expected sequence of markers.
3. Open the waveform and find the **last thing that happened**.
4. Check whether the CPU is **alive** (signals still moving) or **stuck** (flat). Alive suggests slow, stuck suggests a hang.
5. Make **one** change on a backup copy, re-run, compare, and restore the original.

---

## 11. Block diagrams (hand-drawn)

One diagram per standalone test, in `diagrams/`:

| Test | File |
|---|---|
| spi_master | `diagrams/spi_master_flow.jpg` |
| gpio_mgmt | `diagrams/gpio_mgmt_flow.jpg` |
| mem | `diagrams/mem_flow.jpg` |
| uart | `diagrams/uart_flow.jpg` |
| timer | `diagrams/timer_flow.jpg` |
| irq | `diagrams/irq_flow.jpg` |
| debug | `diagrams/debug_flow.jpg` |

Each shows: Makefile -> Compilation -> Testbench -> DUT -> Simulation -> Result (PASS / FAIL), with the real file names and commands for that test.

---

## 12. Open points

- **irq:** still fails. The interrupt never finishes the wait loop. Next checks: does the startup file call `isr()`, and does the timer interrupt fire.
- **debug:** passes only with `debug_in = 0`. The reason the held-high pin stops the firmware is not confirmed.
