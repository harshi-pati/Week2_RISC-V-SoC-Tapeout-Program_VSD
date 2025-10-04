
# VSDBabySoC – RISC-V SoC with PLL & DAC

## Table of Contents

1. [Project Overview](#project-overview)
2. [Integrated Modules](#integrated-modules)
3. [Project Structure](#project-structure)
4. [Setup & Cloning](#setup--cloning)
5. [TLV to Verilog Conversion](#tlv-to-verilog-conversion)
6. [Pre-Synthesis Simulation](#pre-synthesis-simulation)
7. [RTL Synthesis](#rtl-synthesis)
8. [Post-Synthesis Simulation](#post-synthesis-simulation)
9. [Waveform & Output Verification](#waveform--output-verification)
10. [Simulation Flow](#simulation-flow)


---

## Project Overview

**VSDBabySoC** is an open-source, compact **System-on-Chip (SoC)** integrating:

* **RVMYTH RISC-V processor core**
* **PLL (Phase-Locked Loop)** for clock generation
* **10-bit DAC** for analog output

**Objective:** Verify SoC functionality using pre-synthesis RTL simulations, synthesize using Yosys, and perform post-synthesis simulations to confirm functional correctness.

---

## Integrated Modules

| Module           | Description          | Inputs                                     | Outputs                          |
| ---------------- | -------------------- | ------------------------------------------ | -------------------------------- |
| **vsdbabysoc.v** | Top-level SoC module | reset, PLL & DAC controls                  | CLK, OUT (DAC analog), RV_TO_DAC |
| **rvmyth.v**     | RISC-V core          | CLK, reset                                 | 10-bit digital OUT               |
| **avsdpll.v**    | PLL module           | VCO_IN, ENb_CP, ENb_VCO, REF               | CLK                              |
| **avsddac.v**    | DAC module           | D[9:0], VREFH, VREFL                       | OUT (analog)                     |
| **clk_gate.v**   | Clock gating         | free_clk, func_en, pwr_en, gating_override | gated_clk                        |

**Key Notes:**

* RVMYTH outputs `RV_TO_DAC` connected to DAC input
* PLL provides stable clock for CPU & SoC timing
* DAC converts 10-bit digital signals to analog output using VREFH/VREFL

---

## Project Structure

**Setup and Prepare Project Directory**
Clone or set up the directory structure as follows:
```txt
VSDBabySoC/
├── src/
│   ├── include/
│   │   ├── sandpiper.vh
│   │   └── other header files...
│   ├── module/
│   │   ├── vsdbabysoc.v      # Top-level module integrating all components
│   │   ├── rvmyth.v          # RISC-V core module
│   │   ├── avsdpll.v         # PLL module
│   │   ├── avsddac.v         # DAC module
│   │   └── testbench.v       # Testbench for simulation
└── output/
└── compiled_tlv/         # Holds compiled intermediate files if needed
```


**Required Source Files:**

* Modules: `vsdbabysoc.v`, `rvmyth.tlv`, `avsddac.v`, `avsdpll.v`, `clk_gate.v`
* Headers: `sp_verilog.vh`, `sp_default.vh`, `sandpiper.vh`, `sandpiper_gen.vh`

---

## Setup & Cloning

```bash
cd vlsi
git clone https://github.com/manili/VSDBabySoC.git
cd VSDBabySoC/
```

* Create output directories:

```bash
mkdir -p output/pre_synth_sim output/post_synth_sim output/synthesized
```

* Ensure **Icarus Verilog** and **GTKWave** are installed.

---

## TLV to Verilog Conversion

Convert `rvmyth.tlv` (TL-Verilog) to standard Verilog:

```bash
# Install python3-venv (if not already installed)
sudo apt update
sudo apt install python3-venv python3-pip

cd vlsi/VSDBabySoC/

# Setup Python virtual environment
python3 -m venv sp_env
source sp_env/bin/activate

# Install SandPiper
pip install pyyaml click sandpiper-saas

# Convert TLV → Verilog
cd src/module/
sandpiper-saas -i rvmyth.tlv -o rvmyth.v --bestsv --noline -p verilog --outdir .
deactivate
```

---

## Pre-Synthesis Simulation

Compile and simulate RTL using Icarus Verilog:

```bash
iverilog -o output/pre_synth_sim/pre_synth_sim.out \
  -DPRE_SYNTH_SIM \
  -I src/include -I src/module \
  src/module/testbench.v

cd output/pre_synth_sim/
./pre_synth_sim.out
gtkwave pre_synth_sim.vcd
```
<img width="1280" height="800" alt="Image" src="https://github.com/user-attachments/assets/44c08b94-3b7a-440c-a458-ec10ea7c7e86" />
  
---

## RTL Synthesis

**Tool:** Yosys

```bash
yosys
```

Commands inside Yosys shell:

```tcl
read_verilog  -sv -I src/include/ -I src/module/ src/module/vsdbabysoc.v src/module/clk_gate.v src/module/rvmyth.v

read_liberty -lib /home/harshithapati/sky130RTLDesignAndSynthesisWorkshop/lib/sky130_fd_sc_hd__tt_025C_1v80.lib

read_liberty -lib src/lib/avsddac.lib

read_liberty -lib src/lib/avsdpll.lib

read_liberty -lib src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib

synth -top vsdbabysoc

write_verilog vsdbabysoc.synth.v

abc -liberty src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib

show  vsdbabysoc
```

<img width="1280" height="800" alt="Image" src="https://github.com/user-attachments/assets/4017f105-abb9-4d56-8b5d-93fe4270027a" />

Copy Standard Cell Models for Simulation

```
cp ../sky130RTLDesignAndSynthesisWorkshop/my_lib/verilog_model/primitives.v src/module/
cp ../sky130RTLDesignAndSynthesisWorkshop/my_lib/verilog_model/sky130_fd_sc_hd.v src/module/

```

---

## Post-Synthesis Simulation

Compile gate-level netlist with standard cells:

```bash
iverilog -o output/post_synth_sim/post_synth_sim.out
-DPOST_SYNTH_SIM
-I src/include/ -I src/module/
src/module/testbench.v

cd output/post_synth_sim/
./post_synth_sim.out
gtkwave post_synth_sim.vcd
```
<img width="1280" height="800" alt="Image" src="https://github.com/user-attachments/assets/f7578b8e-0eae-4127-8427-892631fbabae" />

**Goal:** Verify synthesized netlist behaves like RTL.


---

## Waveform & Output Verification

* Compare pre-synthesis and post-synthesis waveforms:

  * `CLK` stability
  * `RV_TO_DAC` correctness
  * DAC analog output follows CPU digital data
* Use GTKWave → Data Format → Analog → Step to observe DAC behavior
  
**Pre-Synthesis:**

<img width="1280" height="800" alt="Image" src="https://github.com/user-attachments/assets/1597e729-534c-4287-a506-04018b5d1679" />
<img width="1280" height="800" alt="Image" src="https://github.com/user-attachments/assets/da4fc344-18e4-4823-980b-a21e1e6eea8f" />

**Post-Synthesis:**

<img width="1280" height="800" alt="Image" src="https://github.com/user-attachments/assets/d14e383b-ef18-44c0-b2a2-8473eddcc77e" />

---

## Simulation Flow

### 1. Pre-Synthesis Simulation (`pre_synth_sim`)
- **Purpose:** Verify RTL functionality before synthesis.
- **Input:** RTL source files (`.v` / `.sv`) and header files (`.vh`).
- **Characteristics:**
  - Uses **behavioral models**.
  - Timing is **idealized** (no gate delays).
  - Faster simulation.
- **Typical files:** `testbench.v`, `avsddac.v`, `rvmyth.v`.

---

### 2. Post-Synthesis Simulation (`post_synth_sim`)
- **Purpose:** Verify synthesized netlist functionality and check timing.
- **Input:** Synthesized netlist + standard cell models (`gls_model/sky130_fd_sc_hd.v`).
- **Characteristics:**
  - Includes **realistic gate delays**.
  - Slower than pre-synthesis simulation.
  - Ensures synthesis did not introduce functional changes.
- **Typical files:** `testbench.rvmyth.post-routing.v`.

