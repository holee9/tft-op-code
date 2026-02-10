# System Design Documentation

System design specifications and architecture documents.

## Contents

This directory contains the following design documents (moved from `.moai/design/`):

| Document | Description |
|----------|-------------|
| `control_sequence_flows.md` | State machine and control flow definitions |
| `fpga_panel_control_spec.md` | FPGA firmware requirements and specifications |
| `imx8_dotnet_control_spec.md` | i.MX8 and .NET application interface |
| `spi_register_map.md` | SPI register address map and bit definitions |

## Design Overview

The system consists of three main components:

1. **FPGA Firmware**: Real-time panel control and measurement
2. **i.MX8 Linux System**: High-level control and data logging
3. **.NET Application**: User interface and data visualization

## Design Principles

- **Separation of Concerns**: Each subsystem has clear responsibilities
- **Real-time Performance**: FPGA handles timing-critical operations
- **Flexibility**: Configurable parameters via SPI
- **Testability**: Built-in self-test and debug modes

## Related Documents

- Panel Specifications: `../panel/spec/`
- Common Protocols: `../common/`
- Delivery Specs: `../delivery/`
