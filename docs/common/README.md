# Common Documentation

Documentation shared across all subsystems.

## Contents

### spi/
SPI communication protocol between i.MX8 and FPGA.

- **Register Map**: FPGA register definitions and bit mappings
- **Protocol Specification**: SPI transaction format and timing
- **Command Reference**: Available SPI commands and responses

### protocol/
Additional protocols and interfaces.

- **Debug Interface**: UART debug protocol
- **Logging Format**: Data logging structure
- **Error Handling**: Error codes and recovery procedures

## Quick Reference

### SPI Configuration

| Parameter | Value |
|-----------|-------|
| Mode | 0 (CPOL=0, CPHA=0) |
| Clock | Max 10 MHz |
| Data Width | 8 bits |
| Byte Order | MSB first |

### Register Access

- **Read**: `[CMD_READ][ADDR][DUMMY][DATA...]`
- **Write**: `[CMD_WRITE][ADDR][DATA...]`

## Related Documents

- FPGA Design: `../design/fpga_panel_control_spec.md`
- i.MX8 Driver: `../delivery/to-yocto-project/`
