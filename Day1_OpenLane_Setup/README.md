# Day 1: OpenLane Environment Setup

## Objective

Before any RTL can be pushed through synthesis, placement, or routing, a working OpenLane environment is required. This day covers setting up **two** independent environments capable of running OpenLane:

1. A local **VirtualBox** virtual machine (offline-capable)
2. A cloud-based **GitHub Codespace** (browser-based, no local install)

Both were brought to a fully working state — confirmed by successfully running a design through the flow. **The VirtualBox environment is the primary machine used for the actual PnR (Place-and-Route) flow in later days; the Codespace serves as a verified backup environment.**

## Environment / Tools

- Oracle VirtualBox (local virtualization)
- Ubuntu (guest OS, via a pre-built `.vdi` disk image)
- Docker (containerized OpenLane tools)
- GitHub Codespaces (cloud dev environment)
- OpenLane v0.21 (local) / OpenLane (via `ghcr.io/the-openroad-project/openlane`, cloud)

## Workflow

```text
Download VDI / Codespace repo
   ↓
Set up virtualization (VirtualBox) or cloud container (Codespaces)
   ↓
Resolve Docker image/alias issues
   ↓
Launch OpenLane
   ↓
Ready for RTL synthesis
```

### Local (VirtualBox) setup

- Downloaded the workshop's `.vdi` disk image and loaded it into VirtualBox as an existing virtual hard disk.
- Configured the VM with 4096 MB RAM and multiple CPUs, using the existing `openlane.vdi`.
- Booted the VM into a pre-configured Ubuntu desktop with the OpenLane repo already present under `~/Desktop/work/tools/openlane_working_dir/`.

### Cloud (Codespaces) setup

- Created a GitHub account and opened the `vsdip/vsd-openlane` repository.
- Used **"Create codespace on main"** to spin up a cloud Ubuntu environment with OpenLane, Magic, and the SKY130 PDK pre-installed via a devcontainer setup.
- Confirmed the container's Docker installation with `docker --version`.

## Issues Faced

**Problem:** Running `make mount` (local VM) failed with:
```
Unable to find image 'efabless/openlane:current' locally
docker: Error response from daemon: manifest for efabless/openlane:current not found
```

**Cause:** The Docker image tag referenced by the workshop's Makefile (`efabless/openlane:current`) no longer exists on Docker Hub — Efabless had changed/retired that specific tag.

**Resolution:** Investigation showed the `docker` command on this VM was actually **aliased** to a fixed `docker run` invocation pointing at a specific, already-downloaded image tag: `efabless/openlane:v0.21`. Running `docker` (bypassing arguments that were being swallowed by the alias) revealed this. Using `\docker pull efabless/openlane:v0.21` (the backslash bypasses shell aliases) confirmed the correct image was already present locally (`Status: Image is up to date`). Running the bare `docker` alias afterward correctly launched an interactive shell inside the OpenLane v0.21 container.

**Problem:** The Codespace's browser interface repeatedly got stuck on "Setting up your codespace" despite the codespace showing as **Active** in the GitHub dashboard.

**Cause:** Browser-side connection/caching issue, not a build failure — confirmed because the codespace remained "Active" throughout.

**Resolution:** Opening the same Codespace in an **Incognito/Private browser window** resolved the connection issue immediately.

## Screenshots

------------------ to be added

## Key Takeaway

A correctly configured environment is a prerequisite for everything downstream — every later stage (synthesis, floorplanning, placement) depends on OpenLane's tools (Yosys, OpenROAD, Magic) being reachable inside a working container with the SKY130 PDK properly linked. Environment issues (like a stale Docker tag, or a broken shell alias) can look like tool failures but are actually infrastructure problems — worth debugging systematically (checking `docker images`, log output, and aliases) rather than assuming the tools themselves are broken.
