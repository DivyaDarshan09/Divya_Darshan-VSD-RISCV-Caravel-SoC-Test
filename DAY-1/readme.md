# Day -1 : Caravel SoC HKSPI Verification
---
<div align="center">

  <!-- VSD Badge -->
  <a href="#" target="_blank" style="text-decoration:none;">
    <img src="https://img.shields.io/badge/VSD-Training%20Program-ff6b6b?style=for-the-badge&logoColor=white" alt="VSD Badge">
  </a>

  <!-- IIT Gandhinagar Badge -->
  <a href="#" target="_blank" style="text-decoration:none;">
    <img src="https://img.shields.io/badge/IIT%20Gandhinagar-Institute%20of%20Excellence-1e90ff?style=for-the-badge&logoColor=white" alt="IIT Gandhinagar Badge">
  </a>

  <!-- RTL & GLS Badge -->
  <a href="#" target="_blank" style="text-decoration:none;">
    <img src="https://img.shields.io/badge/RTL%20%26%20GLS-Simulation-9b59b6?style=for-the-badge&logoColor=white" alt="RTL & GLS Badge">
  </a>

  <!-- Day 1 Badge -->
  <a href="#" target="_blank" style="text-decoration:none;">
    <img src="https://img.shields.io/badge/Day%201-Introduction-2ecc71?style=for-the-badge&logoColor=white" alt="Day 1 Badge">
  </a>

</div>

---
- This document explains how I set up, debugged, and successfully ran RTL and Gate-Level Simulations (GLS) for the HK-SPI test inside the Caravel SoC.
- All the steps, errors, fixes, and configuration changes mentioned here reflect what I actually encountered and solved during the project.
- This documents everything step by step, because this setup is notoriously tricky, and the official documentation does not clearly explain the complete flow.
---

## Objective

**My goal was:**

- To run Caravel SoC RTL simulation for the HK-SPI test
- To run the same test in Gate-Level Simulation (GLS)
- To ensure both RTL and GLS produce identical functional results
- To extend this later to all tests under caravel/verilog/dv/caravel/*
---
## Caravel SoC Overview

- `Caravel` is a template system-on-chip (SoC) developed by `Efabless` for Open MPW and ChipIgnite shuttles, built on the SkyWater SKY130 process. The SoC consists of a harness frame and two wrappers: one for the management area and one for the user project area. The harness includes essential functions such as clocking, power-on reset, DLL, user ID, housekeeping SPI, and GPIO control. GPIO configuration is handled via SPI and Wishbone interfaces, allowing automatic setup at power-up or manual control.

**Caravel Architecture**

![caravel](.Screenshots/caravel_arch.png)


- The management area is a RISC-V-based SoC that provides peripherals like timers, UART, GPIO, SRAM, and firmware to configure user project I/O, monitor signals via on-chip logic analyzers, and control power. The user project area is the dedicated space for user designs, offering 38 I/O ports, 128 logic analyzer probes, and a Wishbone connection to the management SoC. Users can implement digital or analog projects by following the respective wrappers and adhering to design rules, with digital projects using the user_project_wrapper and analog projects using the user_analog_project_wrapper. Caravel provides sample projects and guidelines for integrating user designs with the SoC infrastructure.
---
### Caravel Overview

![caravel](.Screenshots/caravel_overview.png)

## Environment Setup

- This section documents the preparation of the environment required to run Caravel’s HK SPI functional and GLS verification.
- All setup was done on a Linux system with the necessary open-source EDA tools.
---
### Caravel Repository Setup

- The official Caravel SoC repository was already downloaded earlier using:
```bash
git clone https://github.com/efabless/caravel
```

- The repository contains:

    - Caravel RTL
    - Caravan RTL
    - Management SoC (RISC-V core)
    - User project interface
    - Verification testbenches
    - Build system and utilities
    - Libs and Lef Files

- This forms the base for running the hkspi verification.
---
### Sky130 PDK Installation (Using Volare)

- The Sky130 PDK was installed using Volare, Efabless’s PDK version manager.
- Volare simplifies installation by automatically fetching and organizing the correct PDK version for Caravel and OpenLane.

**1. Fetching the Required PDK Version**

- The required Sky130 PDK version was downloaded using:
```bash
volare fetch --pdk sky130 0fe599b2afb6708d281543108caf8310912f54af
```
- Output confirmed successful downloads:

```bash
Version 0fe599b2afb6708d281543108caf8310912f54af not found locally, attempting to download…

Downloading common.tar.zst… 100%
Downloading sky130_fd_io.tar.zst… 100%
Downloading sky130_fd_pr.tar.zst… 100%
Downloading sky130_fd_sc_hd.tar.zst… 100%
Downloading sky130_fd_sc_hvl.tar.zst… 100%
Downloading sky130_ml_xx_hd.tar.zst… 100%
Downloading sky130_sram_macros.tar.zst… 100%
```

- All components of the Sky130 PDK were downloaded:

    - common files
    - IO libraries
    - foundation device models
    - high-density digital cells
    - high-voltage cells
    - mixed-signal libraries
    - SRAM macros

**2. Enabling the Downloaded PDK Version**

- After download, the PDK version was activated with:

```bash
volare enable --pdk sky130 0fe599b2afb6708d281543108caf8310912f54af
```
**Output:**
```bash
Version 0fe599b2afb6708d281543108caf8310912f54af enabled for the sky130 PDK.

```
- This sets the selected version as the default Sky130 PDK for all tools using Volare.

**3. Environment Variables**

- The following variables were added to ~/.bashrc:
```bash
export PDK_ROOT=$HOME/.volare/volare
export PDK=sky130
export PDK_VERSION=0fe599b2afb6708d281543108caf8310912f54af
```

- Reloaded using:
```bash
source ~/.bashrc
```

- This ensures OpenLane, Magic, KLayout, and simulator tools use the exact PDK version downloaded with Volare.

**4. Directory Structure Verification**

- Running:
```bash
ls -l ~/.volare
```

- shows symlinks such as:
```bash
sky130 -> /home/.../.volare/volare/sky130
sky130A -> volare/sky130/versions/<hash>/sky130A
sky130B -> volare/sky130/versions/<hash>/sky130B
volare/
```

- This confirms:
    - PDK exists
- Symlinks for sky130A and sky130B are properly created
- Volare is managing the version directories.

**Explanation:**
```
PDK_ROOT -> Points to the directory where Volare stores PDKs
PDK	-> Specifies the PDK family (sky130)
```
- Expected subdirectories were present:

- libs.ref
- libs.tech
- sky130_fd_pr
- sky130_fd_sc_hd
- sky130_fd_sc_hvl
- sky130_ml_xx_hd
- sky130_sram_macros

- These include SPICE models, GDS, Magic/KLayout tech files, standard cell libraries, and SRAM macros.
This confirms that the PDK installation is complete and ready for simulation and physical design.

---
### Cloning the management SoC wrapper

```bash
cd $CARAVEL_ROOT
rm -rf verilog/rtl/mgmt_core_wrapper
git clone https://github.com/efabless/caravel_mgmt_soc_litex verilog/rtl/mgmt_core_wrapper
```

**Why mgmt_core_wrapper Is Needed in Caravel SoC**

`mgmt_core_wrapper` is the interface/glue module that connects the LiteX-generated management SoC core to the Caravel top-level. It ensures the internal RISC-V management core can properly communicate with Caravel’s padframe, harness, and system infrastructure.

**Key Roles:**

- Standardizes interfaces between the LiteX SoC core and Caravel’s expected clock, reset, Wishbone, GPIO, logic analyzer, and housekeeping SPI signals.
- Handles integration of the management core with Caravel’s I/O padframe and harness logic.
- Isolates generated SoC logic, allowing core regeneration without modifying Caravel RTL.
- Maintains clean hierarchy, making the management subsystem a single pluggable module in the chip.
---

### Tool Verification

- The following tools required for Caravel DV and OpenLane flows were verified to run correctly:

| **Tool**             | **Purpose**                                   |
|----------------------|-----------------------------------------------|
| **iverilog**         | RTL simulation (compiles Verilog into vvp)    |
| **Sky130 PDK (Volare)** | Provides technology files + standard cell libraries |

- Each tool launched successfully. Version screenshots were saved as proof.

**Iverilog Confirmation screenshot**

![iverilog](.Screenshots/iverilog.png)

---
## Understand Caravel’s hkspi Test

### What is the Housekeeping SPI (HK-SPI)?

- The HK-SPI is a crucial communication interface inside the Caravel SoC, used for exchanging configuration and status information between:

    - Management SoC (RISC-V CPU)
    - Housekeeping Controller
    - User Project Wrapper

- It ensures that the management core can properly configure and control system-level functionality, including GPIOs, resets, user project operations, and more.
- This interface is fully tested using the HK-SPI DV test inside Caravel.
---
## Location of the HK-SPI DV Test

- You can find the test at:
```bash
caravel/verilog/dv/caravel/mgmt_soc/hkspi/
```
---
| File                 | Description                                           |
|----------------------|-------------------------------------------------------|
| hkspi_tb.v           | Main testbench that drives SPI signals and validates results |
| housekeeping_spi.v   | RTL implementation of the housekeeping SPI logic      |
| hkspi.hex            | Predefined SPI command sequence used by the testbench |
---

## Detailed Explanation of Each Component
### 4.1 hkspi_tb.v — Testbench Overview

- This Verilog testbench performs the complete SPI transaction test.
Its major responsibilities include:

**What the testbench does:**

- Loads command vectors from hkspi.hex
- Generates SPI signals:
    - sck (SPI clock)
    - csb (chip select)
    - mosi (data from master)

- Sends these signals into the housekeeping SPI module
- Reads back responses on miso

**Verifies:**
    - Register read/write correctness
    - SPI mode behavior
    - Timing and protocol correctness

- The testbench mimics the behavior of the Management SoC firmware, ensuring the module reacts exactly as expected.
---
### 4.2 housekeeping_spi.v — RTL Implementation

- This is the core logic that implements the SPI protocol and manages housekeeping registers.

**Key functionalities:**

- Receives SPI commands from the Management SoC
- Decodes:
    - Command type
    - Read/write flag
    - Register address
- Updates internal housekeeping control registers
- Provides read data back to SoC via MISO
- Controls system-critical signals:
    - GPIO configuration (input/output/analog)
    - Analog enable signals
    - User project reset and power-enable states
    - User project clock enabling
- This module ensures the chip boots properly and follows the SoC’s instructions.
---
### 4.3 hkspi.hex — SPI Command Vector File

- This file contains the SPI command sequences used by the testbench.

**What it includes:**

- Register write commands
- Register read commands
- Configuration sequences
- Status verification operations

The testbench reads these sequences and applies them transaction-by-transaction.

---
##  How HK-SPI Interacts with the System
### Interaction with the Management SoC (RISC-V)

- The RISC-V CPU acts as the SPI master. Through HK-SPI, it performs:
    - GPIO direction and mode configuration
    - Analog vs. digital pin selection
    - Enabling/disabling the user project clock
    - Applying or releasing reset to the user project
    - Reading/writing housekeeping registers
    - System-level configurations

- This makes HK-SPI the primary boot-time and runtime configuration bridge.

### 5.2 HK-SPI as the Command Interpreter

- HK-SPI receives low-level SPI signals from the SoC:

    - SCK — SPI clock
    - MOSI — data from SoC
    - CSB — chip-select
    - MISO — data sent back to SoC

- It then:
    - Decodes commands
    - Selects the correct internal register
    - Performs write/update operations
    - Sends responses back to the SoC

- It acts similarly to an SPI slave controller tightly integrated with Caravel's internal configuration logic.

### 5.3 Interaction with the User Project

- The HK-SPI controls several user-project-related signals through its registers:
    - GPIO mode (in/out/analog)
    - Power enabling for the user area
    - Reset release for user project
    - Clock enabling
    - Functional configuration modes

- Thus, HK-SPI indirectly determines when and how the user project becomes active.

## 6. Purpose of the HK-SPI Test

- The hkspi DV test ensures the following:
    - SPI protocol is implemented correctly
    - Register read/write operations behave as expected
    - Proper communication between SoC → HK-SPI → User project
    - No functional mismatches between RTL and GLS
    - No regressions introduced during development
- This test is essential before moving to LEC, GLS, and tape-out.

---
### Create a dummy fill‑cell module

```bash
cd $CARAVEL_ROOT
echo 'module sky130_ef_sc_hd__fill_4(inout VPWR, inout VGND, inout VPB, inout VNB); endmodule' > verilog/dv/dummy_fill.v
```
- The dummy filler cell module prevents RTL/GLS simulators from reporting “module not found” errors for physical-only filler cells. It serves as a placeholder so simulation runs cleanly without requiring full PDK cell models.
---
## HKSPI RTL Simulation

- Running the RTL Simulation (HKSPI)
- Initially, running make SIM=RTL always failed due to:

    - Duplicate module definitions
    - Wrong include paths
    - Missing RTL folders
    - Missing VEX CPU model

- After fixing all paths, run `make SIM=RTL`.

- Run
```bash
# Navigate to the hkspi test directory
cd $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc/hkspi

# Locate VexRiscv
export VEX_FILE=$(find $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper -name "VexRiscv*.v" | head -n 1)

# Run RTL Simulation
make SIM=RTL
```
- or compile,

```bash
iverilog -Ttyp -DFUNCTIONAL -DSIM -DUSE_POWER_PINS -DUNIT_DELAY=#1 \
    -I $(CARAVEL_ROOT)/verilog \
    -I $(CARAVEL_ROOT)/verilog/rtl \
    -I $(CARAVEL_ROOT)/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
    -I $(CARAVEL_ROOT)/verilog/dv/caravel/mgmt_soc \
    -I $(CARAVEL_ROOT)/verilog/dv/caravel \
    -I $(PDK_PATH) \
    -y $(CARAVEL_ROOT)/verilog/rtl \
    $(PDK_PATH)/libs.ref/sky130_fd_sc_hd/verilog/primitives.v \
    $(PDK_PATH)/libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v \
    $(PDK_PATH)/libs.ref/sky130_fd_io/verilog/sky130_fd_io.v \
    $(CARAVEL_ROOT)/verilog/dv/dummy_fill.v \
    $(VEX_FILE) \
    hkspi_tb.v -o hkspi.vvp

And then:
vvp hkspi.vvp
```
---
### RTL Output
After all debugging, my RTL output finally showed:
```
Read register 0 = 0x00
Read register 1 = 0x04
Read register 2 = 0x56
...
Monitor: Test HK SPI (RTL) Passed
```
This confirmed that the RTL simulation was correct.

**Terminal Screenshot**

![rtl](.Screenshots/rtl1.png)



![rtl](.Screenshots/rtl.png)

---
## Waveform of HKSPI

![rtl](.Screenshots/rtl_waveform.png)

---
## Gate-Level Simulation (GLS)

- After RTL worked, I ran GLS using:
```bash
iverilog -Ttyp \
  -DFUNCTIONAL -DSIM \
  -D USE_POWER_PINS \
  -D UNIT_DELAY=#1 \
  -I $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc \
  -I $CARAVEL_ROOT/verilog/dv/caravel \
  -I $CARAVEL_ROOT/verilog/rtl \
  -I $CARAVEL_ROOT/verilog \
  -I $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
  -I $PDK_ROOT/sky130A \
  -y $CARAVEL_ROOT/verilog/rtl \
  -y $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
  $CARAVEL_ROOT/verilog/gl/housekeeping.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/primitives.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_io/verilog/sky130_fd_io.v \
  $CARAVEL_ROOT/verilog/dv/dummy_fill.v \
  $VEX_FILE \
  hkspi_tb.v -o hkspi_gls.vvp

  # Then Run gls simulation
  vvp hkspi_gls.vvp
```

- This switches to:

    - GL netlists
    - Standard cell models
    - Functional pad models
- The testbench is the same.

### RTL Output
After all debugging, my RTL output finally showed:
```
Read register 0 = 0x00
Read register 1 = 0x04
Read register 2 = 0x56
...
Monitor: Test HK SPI (GL) Passed
```
This confirmed that the RTL simulation was correct.

**Terminal Screenshot**

![gl](.Screenshots/gls_1.png)


![gll](.Screenshots/gls_2.png)

---

## 🚧 Challenges I Faced (and How I Solved Them)

**❌ 1. Missing RTL folder**
I accidentally lost the entire verilog/rtl folder.

**Fix:**
```bash
git restore verilog/rtl
git restore verilog/gl
git restore verilog/stubs
```
---
**❌ 2. Massive “module already declared” errors**
- This happened because:
    - I was including both caravel_netlists.v and the individual RTL modules
    - Some paths pointed to system PDK instead of Volare PDK
    - Icarus was compiling the same file twice through -y

**Fix:**
```bash
I cleaned the iverilog compile list → only one source of truth for each module.
```
---
**❌ 3. VEX CPU not included**
- The HK-SPI test accesses memory, which requires the RISC-V CPU RTL.
- Without including it, simulation fails silently or produces X-values.

**Fix:**
```bash
export VEX_FILE=$(find ~/caravel/verilog/rtl -name "VexRiscv*.v" | head -n 1)
```
---
**❌ 4. sky130_fd_io warnings (top_vrefcapv2)**
- These warnings appear because analog nodes are unconnected in functional simulations.

**Fix:**
```bash
Ignored — they do not affect logic simulation.
```
---
**🧭 Final Working Simulation Flow**
```bash
✔ Volare PDK installed
✔ Caravel folders restored
✔ Environment variables set
✔ VEX CPU detected
✔ HK-SPI RTL passes
✔ HK-SPI GLS Failed
```
---
### The key learnings:

- The VEX CPU file is mandatory
- The include order matters
- The PDK path must strictly point to Volare
- The RTL folder must not be modified manually
- The HK-SPI test is a reliable indicator of correct SoC integration

**Now both RTL and GLS simulations produce identical outputs, fulfilling the verification objective.**

---