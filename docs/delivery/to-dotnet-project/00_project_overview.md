# Project Overview

## .NET Project Team Delivery Package

**Document**: 00_project_overview.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Project Goal

**Main Objective**: Develop a Windows .NET 8 application that provides user interface and data visualization for the TFT leakage control system.

### Application Responsibilities

1. **User Interface** - Control panel and visualization
2. **Network Communication** - TCP/UDP to i.MX8 Linux backend
3. **Image Processing** - 2D Dark LUT interpolation and dark correction
4. **Data Management** - Frame storage and calibration data
5. **Configuration** - System settings management

---

## 2. System Architecture

### 2.1 Three-Tier Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Presentation Layer                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │ Main Window  │  │ Image View   │  │ Status Panel │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     Business Logic Layer                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │2D Dark LUT   │  │Frame Processor│  │Config Manager│        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      Data Access Layer                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │Comm Service  │  │Storage Service│  │HAL Interface │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Communication Flow

```
┌─────────────┐         TCP/UDP         ┌─────────────┐
│             │◄──────────────────────►│             │
│  .NET App   │      Port 5000/5001      │  i.MX8 Linux │
│  (Windows)  │                         │  (Backend)   │
└─────────────┘                         └─────────────┘
```

---

## 3. UI Framework Selection

### 3.1 Options

| Framework | Pros | Cons | Recommendation |
|-----------|------|------|----------------|
| **WPF** | Mature, stable | Windows only | Recommended |
| **WinUI3** | Modern, Fluent | Newer, learning curve | Alternative |

### 3.2 Recommended Stack

- **Framework**: WPF (.NET 8)
- **MVVM**: CommunityToolkit.Mvvm
- **DI**: Microsoft.Extensions.DependencyInjection
- **Logging**: Serilog
- **Config**: Microsoft.Extensions.Configuration

---

## 4. Main Features

| Feature | Description | Priority |
|---------|-------------|----------|
| **Dashboard** | System status display | Critical |
| **Image View** | Frame visualization | Critical |
| **Capture Control** | Start/stop capture | Critical |
| **Temperature Display** | Real-time temperature | High |
| **Calibration** | Dark frame management | High |
| **Settings** | Configuration UI | Medium |
| **Logging** | Event viewer | Medium |

---

## 5. Related Documents

| Document | Description |
|----------|-------------|
| Architecture Details | `01_architecture_overview.md` |
| API Specification | `02_api_specification.md` |
| Data Models | `03_data_models.md` |
| Communication Protocol | `04_communication_protocol.md` |
| UI Requirements | `05_ui_requirements.md` |

---

## 6. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial document |

---

**End of Document**
