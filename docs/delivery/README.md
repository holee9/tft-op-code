# Delivery Documentation

**Version**: 1.0
**Date**: 2026-02-10
**Status**: Active

---

## Overview

This directory contains delivery packages for external development teams. Each team has an independent package with complete specifications for their responsibilities.

### Project Context

The TFT Leakage Reduction Project manages a-Si TFT panel dark current through:

1. **FPGA Team** - Real-time timing generation and bias control
2. **Yocto Project Team** - Linux BSP for i.MX8 Plus (MCU)
3. **.NET Application Team** - Windows Host application

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TFT Leakage Reduction System                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐         SPI          ┌─────────────┐            │
│  │    FPGA     │◄────────────────────►│   i.MX8      │            │
│  │   (Artix-7) │                      │   (Linux)    │            │
│  └──────┬──────┘                      └──────┬──────┘            │
│         │                                    │                   │
│         │ LVDS/Bias                         │ Ethernet          │
│         ▼                                    ▼                   │
│  ┌─────────────┐                    ┌─────────────┐            │
│  │ aSi TFT     │                    │    Host PC   │            │
│  │   Panel     │                    │   (.NET App) │            │
│  └─────────────┘                    └─────────────┘            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Directory Structure

### to-fpga-team/

Specifications for FPGA development team implementing the panel control logic.

| Document | Description |
|----------|-------------|
| `README.md` | FPGA team delivery guide |
| `00_project_overview.md` | Project goals, system overview, responsibilities |
| `01_requirements.md` | Functional and performance requirements |
| `02_interfaces.md` | SPI, LVDS, Bias Control, ADC interfaces |
| `03_timing_specifications.md` | Row/column timing, frame timing |
| `04_register_map.md` | Complete SPI register address map (copied from design/) |
| `05_acceptance_criteria.md` | Validation and test requirements |
| `reference/panel_physics_summary.md` | Panel physical characteristics |

**Key Deliverables**:
- RTL source code (SystemVerilog)
- Testbenches and simulation reports
- Synthesis reports
- Bitstream file

---

### to-yocto-project/

Specifications for Yocto Project Linux BSP team building the i.MX8 Plus runtime.

| Document | Description |
|----------|-------------|
| `README.md` | Yocto team delivery guide |
| `00_project_overview.md` | Platform overview and architecture |
| `01_platform_specifications.md` | i.MX8 Plus hardware details |
| `02_spi_driver_requirements.md` | SPI master driver requirements |
| `03_device_tree_configuration.md` | Complete Device Tree source |
| `04_kernel_config.md` | Kernel configuration fragment |
| `05_userspace_interface.md` | Userspace API (/dev/spidevX, etc.) |
| `06_temperature_monitoring.md` | Temperature sensor and Arrhenius model |
| `07_ethernet_communication.md` | Ethernet setup and protocol |
| `08_acceptance_criteria.md` | Validation requirements |
| `reference/arrhenius_model_summary.md` | Arrhenius equation and LUT |
| `reference/idle_state_machine.md` | L1/L2/L3 state machine |

**Key Deliverables**:
- Yocto layer (meta-tft-leakage)
- Device Tree source files
- Kernel configuration fragment
- systemd service files

---

### to-dotnet-project/

Specifications for .NET application team developing the Windows Host UI.

| Document | Description |
|----------|-------------|
| `README.md` | .NET team delivery guide |
| `00_project_overview.md` | Application goals and architecture |
| `01_architecture_overview.md` | .NET 8 architecture and patterns |
| `02_api_specification.md` | HAL API and service interfaces |
| `03_data_models.md` | Data model definitions |
| `04_communication_protocol.md` | Ethernet protocol with i.MX8 |
| `05_ui_requirements.md` | UI wireframes and workflows |
| `06_configuration_schema.md` | JSON/YAML configuration schema |
| `07_acceptance_criteria.md` | Validation requirements |
| `reference/2d_dark_lut_algorithm.md` | 2D Dark LUT interpolation |
| `reference/image_processing_pipeline.md` | Image correction pipeline |

**Key Deliverables**:
- .NET 8 solution source code
- Unit tests (>80% coverage)
- User documentation
- Installer (MSI)

---

## Version Management

See `DELIVERY_VERSION.md` for version history and compatibility matrix.

### Current Versions

| Package | Document Version | Date |
|---------|------------------|------|
| to-fpga-team | 1.0 | 2026-02-10 |
| to-yocto-project | 1.0 | 2026-02-10 |
| to-dotnet-project | 1.0 | 2026-02-10 |

---

## Cross-Team Dependencies

### Communication Protocols

| From → To | Protocol | Document |
|-----------|----------|----------|
| FPGA → i.MX8 | SPI Slave | `to-fpga-team/04_register_map.md` |
| i.MX8 → Host | Ethernet | `to-yocto-project/07_ethernet_communication.md` |
| Host → i.MX8 | Ethernet | `to-dotnet-project/04_communication_protocol.md` |

### Shared Data

| Data | Source | Consumer |
|------|--------|----------|
| SPI Register Map | Design | FPGA Team |
| Arrhenius Model | Design | Yocto Team |
| State Machine | Design | Yocto Team |
| 2D Dark LUT | Design | .NET Team |

---

## Acceptance Process

Each team must:

1. Review their delivery package
2. Implement according to specifications
3. Validate against acceptance criteria
4. Deliver source code and documentation
5. Participate in integration testing

---

## Contact Information

| Role | Contact |
|------|---------|
| Project Coordinator | [To be assigned] |
| FPGA Team Lead | [To be assigned] |
| Yocto Team Lead | [To be assigned] |
| .NET Team Lead | [To be assigned] |

---

## Related Documents

- **Design Specifications**: `../design/` - Complete system design
- **Reference Documents**: `../reference/latest/` - Implementation plan and validation
- **Panel Specifications**: `../panel/spec/` - Panel electrical characteristics

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial delivery package structure |
