# FPGA Firmware - aSi TFT Leak Reduction

This directory contains FPGA firmware for controlling aSi TFT panel leakage current.

## Directory Structure

### rtl/
Register Transfer Level (RTL) design source code.

- **core/**: Core control logic and state machines
- **spi/**: SPI slave interface for host communication
- **bias_mux/**: Bias voltage multiplexer control
- **timing_gen/**: Timing generator for panel sequences
- **dummy_scan/**: Dummy scan pattern generator

### sim/
Simulation environment and testbenches.

- **tb/**: Testbench files for verification
- **scripts/**: Simulation scripts and Makefiles

### synth/
Synthesis scripts and constraints.

- Vivado project files
- Timing constraints (SDC)
- Synthesis reports

### scripts/
Utility scripts for development and deployment.

- Build scripts
- Programming scripts
- Verification utilities

## Development Tools

- **Simulation**: ModelSim/Questa
- **Synthesis**: Vivado (Xilinx)
- **Version Control**: Git

## Quick Start

```bash
# Run simulation
cd sim
./run_simulation.sh

# Synthesize design
cd synth
./synthesize.sh
```
