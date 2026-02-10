# FPGA Team Delivery Package

**Delivery Package Version**: 1.0
**Date**: 2026-02-10
**Target**: FPGA Development Team

---

## Overview

This package contains complete specifications for implementing the FPGA control logic for a-Si TFT FPD panel driving. The FPGA is responsible for real-time timing generation, bias voltage control, and data acquisition.

### Target FPGA

| Parameter | Value |
|-----------|-------|
| **Device** | Xilinx Artix-7 35T FGG484 |
| **Speed Grade** | -1 or higher |
| **Tools** | Vivado 2025.2+ |
| **Clock** | 100 MHz main oscillator |

### System Role

```
FPGA sits between i.MX8 (MCU) and aSi TFT Panel:

i.MX8 <--SPI--> FPGA <--LVDS/Bias--> Panel
       (Master)    (Slave)           (Gate Driver)
```

---

## Document Structure

| Document | Description |
|----------|-------------|
| `00_project_overview.md` | Project goals, system overview, responsibilities |
| `01_requirements.md` | Functional and performance requirements |
| `02_interfaces.md` | SPI, LVDS, Bias Control, ADC interfaces |
| `03_timing_specifications.md` | Row/column timing, frame timing |
| `04_register_map.md` | Complete SPI register address map |
| `05_acceptance_criteria.md` | Validation and test requirements |
| `reference/panel_physics_summary.md` | Panel physical characteristics |

---

## Quick Start

1. Read `00_project_overview.md` for system context
2. Review `02_interfaces.md` for pin definitions
3. Implement modules per `03_timing_specifications.md`
4. Use `04_register_map.md` for SPI slave implementation
5. Validate against `05_acceptance_criteria.md`

---

## Key Requirements Summary

| Category | Requirement |
|----------|-------------|
| **Panel Resolution** | 2048 x 2048 |
| **Row Read Time** | 50 µs per row |
| **Frame Time** | ~102 ms for full frame |
| **SPI Mode** | Mode 0, max 10 MHz |
| **Bias Switching** | < 10 µs transition time |
| **Resource Target** | < 72% LUT, < 100% PLL/MMCM |

---

## Delivery Checklist

The FPGA team must deliver:

- [ ] RTL source code (SystemVerilog recommended)
- [ ] Testbench for all modules
- [ ] Simulation reports (Questa/ModelSim)
- [ ] Synthesis reports (Vivado)
- [ ] Timing analysis results
- [ ] Bitstream file
- [ ] Register access test utility

---

## Communication

- **Primary Contact**: [To be assigned]
- **Review Meetings**: Weekly
- **Issue Tracking**: GitHub Issues

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-10 | Initial delivery package |
