# SPI Protocol Documentation

This directory contains SPI communication protocol documentation.

## Documents

- **Register Map**: `../../design/spi_register_map.md`
- **Protocol Specification**: *To be added*

## Quick Reference

| Parameter | Value |
|-----------|-------|
| Mode | SPI Mode 0 (CPOL=0, CPHA=0) |
| Clock | Max 10 MHz |
| Data Width | 8 bits |
| Byte Order | MSB first |

## Command Format

- **Read**: `[CMD_READ][ADDR][DUMMY][DATA...]`
- **Write**: `[CMD_WRITE][ADDR][DATA...]`
