# .NET Project Team Delivery Package

**Delivery Package Version**: 1.0
**Date**: 2026-02-10
**Target**: .NET Application Team (Windows)

---

## Overview

This package contains specifications for the Windows Host .NET application that provides user interface, data visualization, and communication with the i.MX8 Linux backend.

### Target Platform

| Parameter | Value |
|-----------|-------|
| **Framework** | .NET 8 |
| **Language** | C# 12 |
| **OS** | Windows 10/11 |
| **Architecture** | x64 |

### System Role

```
Host .NET App provides:
├── User Interface (WPF/WinUI3)
├── Data Visualization (2D Dark LUT)
├── Image Processing (Dark correction)
├── Network Communication (Ethernet to i.MX8)
└── Configuration Management
```

---

## Document Structure

| Document | Description |
|----------|-------------|
| `00_project_overview.md` | Project goals and architecture overview |
| `01_architecture_overview.md` | .NET 8 architecture and patterns |
| `02_api_specification.md` | HAL API and service interfaces |
| `03_data_models.md` | Data model definitions |
| `04_communication_protocol.md` | Ethernet protocol with i.MX8 |
| `05_ui_requirements.md` | UI wireframes and workflows |
| `06_configuration_schema.md` | JSON/YAML configuration schema |
| `07_acceptance_criteria.md` | Validation requirements |
| `reference/2d_dark_lut_algorithm.md` | 2D Dark LUT interpolation |
| `reference/image_processing_pipeline.md` | Image correction pipeline |

---

## Quick Start

1. Read `00_project_overview.md` for system context
2. Review `01_architecture_overview.md` for design patterns
3. Implement API per `02_api_specification.md`
4. Create UI per `05_ui_requirements.md`
5. Validate against `07_acceptance_criteria.md`

---

## Key Requirements Summary

| Category | Requirement |
|----------|-------------|
| **Framework** | .NET 8, C# 12 |
| **UI Framework** | WPF or WinUI3 |
| **Communication** | TCP/UDP to i.MX8 |
| **Image Processing** | 2D Dark LUT, dark correction |
| **Configuration** | JSON/YAML settings |

---

## Interface to i.MX8

The .NET app communicates with i.MX8 via Ethernet:

```
Host .NET App          i.MX8 Linux
├── TCP (Port 5000) ──→ Control commands
├── UDP (Port 5001) ──→ Frame data
└── HTTP/REST ────────→ Optional API
```

---

## Delivery Checklist

The .NET team must deliver:

- [ ] Source code (.NET 8 solution)
- [ ] Project files (.csproj)
- [ ] Unit tests (>80% coverage)
- [ ] User documentation
- [ ] Configuration file templates
- [ ] Installation script

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
