# Yocto Project Team Delivery Package

**Delivery Package Version**: 1.0
**Date**: 2026-02-10
**Target**: Yocto Project BSP Team

---

## Overview

This package contains specifications for the Yocto Project Linux BSP development on NXP i.MX8 Plus platform. The BSP provides the runtime environment for the TFT leakage control application.

### Target Platform

| Parameter | Value |
|-----------|-------|
| **SoC** | NXP i.MX8 Plus |
| **Cores** | Cortex-A53 (quad) + Cortex-M4 |
| **Yocto Version** | 4.0+ (Kirkstone or later) |
| **Framework** | .NET 8.0 |

### System Role

```
Yocto BSP provides:
├── Linux kernel with SPI driver
├── .NET 8 runtime
├── System services (systemd)
├── Network stack (Ethernet)
└── Hardware access (GPIO, I2C, SPI)
```

---

## Document Structure

| Document | Description |
|----------|-------------|
| `00_project_overview.md` | Project goals and platform overview |
| `01_platform_specifications.md` | i.MX8 Plus hardware details |
| `02_spi_driver_requirements.md` | SPI master driver for FPGA |
| `03_device_tree_configuration.md` | Device Tree settings |
| `04_kernel_config.md` | Kernel configuration fragment |
| `05_userspace_interface.md` | Userspace API (/dev/spidevX) |
| `06_temperature_monitoring.md` | Temperature sensor requirements |
| `07_ethernet_communication.md` | Ethernet setup |
| `08_acceptance_criteria.md` | Validation requirements |
| `reference/arrhenius_model_summary.md` | Arrhenius model for t_max |
| `reference/idle_state_machine.md` | L1/L2/L3 state machine |

---

## Quick Start

1. Read `00_project_overview.md` for system context
2. Review `01_platform_specifications.md` for hardware details
3. Implement SPI driver per `02_spi_driver_requirements.md`
4. Configure Device Tree per `03_device_tree_configuration.md`
5. Apply kernel config per `04_kernel_config.md`
6. Validate against `08_acceptance_criteria.md`

---

## Key Requirements Summary

| Category | Requirement |
|----------|-------------|
| **SPI Driver** | Mode 0, 10 MHz, /dev/spidevX |
| **Device Tree** | GPIO, SPI, Ethernet nodes |
| **Kernel Config** | SPI master, GPIO, Ethernet |
| **Services** | systemd service for .NET app |
| **Temperature** | 1 Hz polling via I2C |

---

## Interface to FPGA

The i.MX8 acts as SPI master to the FPGA (slave):

```
i.MX8 Plus          FPGA
├── ECSPI2 ──────→ SPI Slave
│   ├── MOSI ────→ MOSI
│   ├── MISO ←──── MISO
│   ├── SS   ────→ CS_N
│   └── SCLK ────→ SCLK
```

---

## Delivery Checklist

The Yocto team must deliver:

- [ ] Yocto recipe for TFT leakage image
- [ ] Device Tree source files (.dts/.dtsi)
- [ ] Kernel configuration fragment (.cfg)
- [ ] SPI driver (mainline or custom)
- [ ] systemd service files
- [ ] Test utilities

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
