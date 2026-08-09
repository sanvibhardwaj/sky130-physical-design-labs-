# sky130-physical-design-labs-

A hands-on implementation of an **RTL-to-GDSII physical design flow** using [OpenLane](https://github.com/The-OpenROAD-Project/OpenLane) and the open-source [SKY130 PDK](https://github.com/google/skywater-pdk).

This repository documents actual tool execution, environment setup, debugging, and physical-design observations made while running real designs (`spm`, `picorv32a`) through the OpenLane flow — not just lecture notes.

**Primary environment:** the PnR flow itself is run locally, inside a VirtualBox Ubuntu VM with OpenLane/Docker set up on the machine directly. A GitHub Codespace was separately set up and verified as a working backup/alternative environment. This GitHub repository is used only to **document and upload** results after each run — it does not host the actual flow execution.

## Overview

```text
RTL
 ↓
Synthesis
 ↓
Floorplanning
 ↓
Power Planning
 ↓
Placement
 ↓
CTS
 ↓
Routing
 ↓
Sign-off
 ↓
GDSII
```

Each stage above takes the design one step closer to a manufacturable layout. This repository tracks progress through that flow, one stage at a time, using two working environments: a local VirtualBox/Docker setup and a cloud-based GitHub Codespace.

## Progress

| Day   | Stage                          | Status        |
| ----- | ------------------------------ | -------------- |
| Day 1 | OpenLane Environment Setup     | ✅ Completed    |
| Day 2 | RTL Synthesis                  | ✅ Completed (as part of full-flow run) |
| Day 3 | Floorplanning & Power Planning | ✅ Completed    |
| Day 4 | Standard Cell Placement        | 🔜 In progress |
| Day 5 | Clock Tree Synthesis           | 🔜 Upcoming    |
| Day 6 | Routing                        | 🔜 Upcoming    |
| Day 7 | Sign-off / GDSII               | 🔜 Upcoming    |

## Tools & Technologies

- **OpenLane** — automated RTL-to-GDSII flow, orchestrating the tools below
- **SKY130 PDK** — Google/SkyWater's open-source 130nm process design kit
- **Yosys** — RTL synthesis (Verilog → gate-level netlist)
- **OpenROAD** — floorplanning, placement, CTS, routing, static timing analysis
- **Magic** — VLSI layout viewing/editing, DRC, GDSII generation
- **Tcl** — scripting language used to control the flow (`flow.tcl` and related scripts)
- **Docker** — packages OpenLane's tools into a reproducible container
- **Linux (Ubuntu)** — host OS, run both locally (VirtualBox) and in the cloud (GitHub Codespaces)

## What I am learning

- RTL-to-GDSII implementation, end to end
- Logic synthesis and technology mapping
- Standard-cell libraries and how cells get selected
- Floorplanning: core/die area, utilization, I/O placement
- Power distribution: rings, straps, and standard-cell power connections
- Standard-cell placement fundamentals
- Reading and interpreting tool logs (OpenROAD, Magic) to debug a flow
- Linux-based EDA workflows, including Docker-based tool environments
- Tcl-based flow automation

This is a learning/project repository — it documents a process still in progress, not a finished expert-level flow.

## Repository Structure

```text
SKY130-Physical-Design-Labs/
│
├── README.md
│
├── Day1_OpenLane_Setup/
│   ├── README.md
│   └── images/
│
├── Day2_Synthesis/
│   ├── README.md
│   └── images/
│
├── Day3_Floorplan_and_Powerplanning/
│   ├── README.md
│   └── images/
│
└── Day4_Placement/
    ├── README.md
    └── images/
```

## Physical Design Flow

Each stage feeds directly into the next:

- **Synthesis** turns Verilog RTL into a gate-level netlist built from real, pre-characterized standard cells.
- **Floorplanning** decides the chip's physical size and where I/O pins and major blocks sit.
- **Power Planning** builds the power/ground distribution grid every cell will connect to.
- **Placement** assigns exact physical (x, y) coordinates to every gate from synthesis.
- **CTS** builds a balanced clock distribution network so all sequential elements switch together.
- **Routing** draws the actual metal wires connecting placed cells.
- **Sign-off** runs DRC/LVS/STA checks to confirm the design is manufacturable and meets timing.
- **GDSII** is the final geometric file format sent to a foundry for fabrication.