# SKY130 Physical Design Labs

A hands-on implementation of an **RTL-to-GDSII physical design flow** using [OpenLane](https://github.com/The-OpenROAD-Project/OpenLane) and the open-source [SKY130 PDK](https://github.com/google/skywater-pdk).

This repository documents actual tool execution, environment setup, debugging, and physical-design observations made while running real designs (`spm`, `picorv32a`) through the OpenLane flow — not just lecture notes.

**Primary environment:** the PnR flow itself is run locally, inside a VirtualBox Ubuntu VM with OpenLane/Docker set up on the machine directly. A GitHub Codespace was separately set up and verified as a working backup/alternative environment. This GitHub repository is used only to **document and upload** results after each run — it does not host the actual flow execution.

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

Each stage above takes the design one step closer to a manufacturable layout. This repository tracks progress through that flow, one stage at a time, using two working environments: a local VirtualBox/Docker setup and a cloud-based GitHub Codespace.


## Tools & Technologies

- **OpenLane** — automated RTL-to-GDSII flow, orchestrating the tools below
- **SKY130 PDK** — Google/SkyWater's open-source 130nm process design kit
- **Yosys** — RTL synthesis (Verilog → gate-level netlist)
- **OpenROAD** — floorplanning, placement, CTS, routing, static timing analysis
- **Magic** — VLSI layout viewing/editing, DRC, GDSII generation
- **Tcl** — scripting language used to control the flow (`flow.tcl` and related scripts)
- **Docker** — packages OpenLane's tools into a reproducible container
- **Linux (Ubuntu)** — host OS, run both locally (VirtualBox) and in the cloud (GitHub Codespaces)



## References

- OpenLane documentation: https://openlane.readthedocs.io
- SKY130 PDK: https://github.com/google/skywater-pdk