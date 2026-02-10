# FPGA RTL Source Code

Register Transfer Level (RTL) design source files.

## Directory Structure

| Directory | Description |
|-----------|-------------|
| `core/` | Core control logic and state machines |
| `spi/` | SPI slave interface for host communication |
| `bias_mux/` | Bias voltage multiplexer control |
| `timing_gen/` | Timing generator for panel sequences |
| `dummy_scan/` | Dummy scan pattern generator |

## Design Guidelines

- Follow SystemVerilog coding standards
- Use synchronous reset where possible
- All registers on positive clock edge
- Document all external interfaces

## Status

*RTL development in progress*
