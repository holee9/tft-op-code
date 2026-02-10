# i.MX8 Linux System - aSi TFT Leak Reduction

This directory contains i.MX8 Linux BSP and application specifications.

## Directory Structure

### docs/
i.MX8 specific documentation.

- Device tree configurations
- GPIO pin mappings
- Power management settings
- Kernel module requirements

### specs/
Specifications for delivery to Yocto Project team.

- SPI driver requirements
- Device tree bindings
- Kernel configuration requirements
- Userspace interface specifications

## Platform Information

- **SoC**: NXP i.MX8 series
- **OS**: Linux (Yocto Project based)
- **Compiler**: ARM GCC toolchain
- **Interface**: SPI master to FPGA

## Key Responsibilities

1. **SPI Master Control**: Communicate with FPGA via SPI
2. **Power Management**: Control panel power rails
3. **Temperature Monitoring**: Read temperature sensors
4. **Logging**: Store leakage measurement data

## Dependencies

- FPGA firmware specifications (see `../docs/design/fpga_panel_control_spec.md`)
- SPI protocol definition (see `../docs/common/spi/`)
