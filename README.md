# aSi TFT Leak Reduction Project

**aSi TFT(비정질 실리콘 박막 트랜지스터) 기반 Flat Panel Detector의 누설 전류(Leakage Current) 감소 시스템**

[![Status](https://img.shields.io/badge/Status-In%20Progress-yellow)](https://github.com/your-repo/issues)
[![FPGA](https://img.shields.io/badge/FPGA-Xilinx%20Artix--7-blue)](https://www.xilinx.com/products/specialty/iot/artix-7.html)
[![Linux](https://img.shields.io/badge/Linux-Yocto%20BSP-orange)](https://www.yoctoproject.org/)
[![.NET](https://img.shields.io/badge/.NET-8.0-purple)](https://dotnet.microsoft.com/download/dotnet/8.0)

---

## Table of Contents

- [Project Overview](#project-overview)
- [System Architecture](#system-architecture)
- [Component Status](#component-status)
- [Implementation Roadmap](#implementation-roadmap)
- [Evaluation Metrics](#evaluation-metrics)
- [Directory Structure](#directory-structure)
- [Getting Started](#getting-started)

---

## Project Overview

### Goal

**Idle 상태에서 발생하는 aSi TFT 패널의 암전류(Dark Current) 드리프트를 실시간 제어로 최소화**

### Key Features

| Feature | Description | Status |
|---------|-------------|--------|
| **Temperature Compensation** | Arrhenius 모델 기반 온도 의존성 보상 | <!-- Status: Pending -->: :black_circle: |
| **Dummy Scan** | Idle 주기 중 더미 스캔으로 전하 축적 방지 | <!-- Status: Pending -->: :black_circle: |
| **Real-time Monitoring** | FPGA 기반 실시간 측정 및 제어 | <!-- Status: In Progress -->: :yellow_circle: |
| **Dynamic Bias Control** | VGH/VGL 바이어스 전압 동적 제어 | <!-- Status: Pending -->: :black_circle: |

### Target Panel

| Parameter | Value | Description |
|-----------|-------|-------------|
| **Panel Model** | R1717AS01.3 | 3072 x 3072 aSi TFT FPD |
| **Pixel Pitch** | 140 &micro;m | 픽셀 간격 |
| **Technology** | a-Si:H TFT + PIN Diode | 수소화 비정질 실리콘 |
| **Effective Area** | 427.8 x 427.8 mm | 유효 영역 |

---

## System Architecture

```
+----------------+     SPI      +----------------+     LVDS    +----------------+
|                | <---------> |                | <---------> |                |
|  i.MX8 Linux   |             |   FPGA (Xilinx |             |  aSi TFT Panel |
|  (Yocto BSP)   |             |   Artix-7)     |             |  R1717AS01.3   |
+----------------+             +----------------+             +----------------+
       |                              |                               |
       | Ethernet                    | Bias Control                  |
       v                              v                               v
+----------------+             +----------------+             +----------------+
|  .NET App      |             |  Gate Driver   |             |  Bias Mux /    |
|  (Windows)     |             |  (VGH/VGL)     |             |  Dummy Scan    |
+----------------+             +----------------+             +----------------+
```

### Architecture Layers

| Layer | Component | Responsibility |
|-------|-----------|-----------------|
| **Host Application** | .NET 8 (Windows) | UI, 데이터 시각화, 2D Dark LUT 보간 |
| **Edge Controller** | i.MX8 Linux (Yocto) | 온도 모니터링, t_max 계산, FPGA SPI 제어 |
| **Real-time Control** | FPGA (Artix-7) | 행 타이밍 생성, VGH/VGL 제어, ADC 데이터 버퍼링 |
| **Physical Layer** | Gate Driver + Panel | 바이어스 전압 공급, 픽셀 스캔 |

---

## Component Status

### 1. Panel (패널)

| Item | Status | Description |
|------|--------|-------------|
| **Specifications** | <!-- Status: Complete -->: :green_circle: | 전기적 특성, 타이밍 요구사항 정의 완료 |
| **Physics Model** | <!-- Status: Complete -->: :green_circle: | 누설 전류 메커니즘 분석 완료 |
| **Characterization** | <!-- Status: Pending -->: :black_circle: | 실제 패널 측정 데이터 필요 |

**Key Parameters**:
- VGH (Gate High): +15 V (TFT ON)
- VGL (Gate Low): -5 V (TFT OFF, Deep depletion)
- VPD (Photodiode Bias): -2.0 V (Normal), -0.5 V (Idle)

**Documents**: `docs/panel/spec/`, `docs/panel/physics/`

---

### 2. FPGA (실시간 제어)

| Module | Status | Description |
|--------|--------|-------------|
| **RTL Design** | <!-- Status: In Progress -->: :yellow_circle: | 코어 로직 설계 중 |
| **SPI Slave** | <!-- Status: Pending -->: :white_circle: | 호스트 통신 인터페이스 |
| **Timing Generator** | <!-- Status: Pending -->: :white_circle: | 3072 x 76 &micro;s 행 타이밍 |
| **Bias Mux Control** | <!-- Status: Pending -->: :white_circle: | VGH/VGL/PD 바이어스 MUX |
| **Simulation TB** | <!-- Status: Pending -->: :white_circle: | 검증 벤치 작성 예정 |
| **Synthesis** | <!-- Status: Not Started -->: :black_circle: | Vivado 합성 준비 |

**Target FPGA**: Xilinx Artix-7 35T FGG484

**Tools**: ModelSim/Questa (Simulation), Vivado (Synthesis)

**Documents**: `fpga/rtl/`, `fpga/sim/`, `docs/design/fpga_panel_control_spec.md`

---

### 3. i.MX8 Linux (엣지 컨트롤러)

| Component | Status | Description |
|-----------|--------|-------------|
| **BSP Setup** | <!-- Status: Pending -->: :black_circle: | Yocto Project 기반 빌드 |
| **SPI Driver** | <!-- Status: Pending -->: :black_circle: | FPGA 통신용 SPI 마스터 드라이버 |
| **Temperature Service** | <!-- Status: Pending -->: :black_circle: | 1 Hz 온도 폴링 서비스 |
| **Arrhenius Calculator** | <!-- Status: Pending -->: :black_circle: | t_max(T) 계산 모듈 |
| **Idle State Machine** | <!-- Status: Pending -->: :black_circle: | L1/L2/L3 상태 전환 |

**Platform**: NXP i.MX8 Plus (Cortex-A53 + Cortex-M4)

**Framework**: .NET 8.0

**Documents**: `imx8/docs/`, `imx8/specs/`, `docs/design/imx8_dotnet_control_spec.md`

---

### 4. Host Application (호스트 애플리케이션)

| Module | Status | Description |
|--------|--------|-------------|
| **UI Framework** | <!-- Status: Not Started -->: :black_circle: | WPF/WinUI3 미정 |
| **2D Dark LUT** | <!-- Status: Pending -->: :black_circle: | 온도 x 노출시간 LUT 보간 |
| **Image Processing** | <!-- Status: Pending -->: :black_circle: | Dark 보정, Blind 픽셀 보정 |
| **Communication** | <!-- Status: Pending -->: :black_circle: | Ethernet 프로토콜 |

**Technology**: .NET 8, C#, Windows 11

**Documents**: `docs/design/imx8_dotnet_control_spec.md`

---

### 5. Common Protocols (공통 프로토콜)

| Protocol | Status | Description |
|----------|--------|-------------|
| **SPI Register Map** | <!-- Status: Complete -->: :green_circle: | 레지스터 주소 맵 정의 |
| **Control Flows** | <!-- Status: Complete -->: :green_circle: | 상태 머신 정의 |
| **Vendor Cooperation** | <!-- Status: Complete -->: :green_circle: | 벤더 협조 가이드 |

**Documents**: `docs/design/spi_register_map.md`, `docs/design/control_sequence_flows.md`, `docs/reference/vendor/Vendor_Cooperation_Protocol_Guide.md`

---

## Implementation Roadmap

### Phase 1: Foundation (Week 1-4)

| Task | Owner | Deliverable |
|------|-------|-------------|
| Panel spec finalization | HW Team | Complete electrical spec |
| FPGA RTL core design | FPGA Team | Timing, SPI, Bias Mux modules |
| i.MX8 BSP build | SW Team | Bootable Yocto image |
| SPI driver implementation | SW Team | Working SPI communication |

### Phase 2: Integration (Week 5-8)

| Task | Owner | Deliverable |
|------|-------|-------------|
| FPGA simulation | FPGA Team | Verified RTL with testbench |
| Arrhenius calculator | SW Team | Working t_max(T) calculation |
| Host UI mockup | App Team | Basic UI framework |
| Panel characterization | HW Team | Measured E_A, k(T) data |

### Phase 3: Validation (Week 9-12)

| Task | Owner | Deliverable |
|------|-------|-------------|
| Full system integration | All | End-to-end working system |
| Performance testing | QA Team | Dark reduction % measurement |
| Long-term stability | QA Team | 48hr idle test results |
| Documentation | All | Complete user manuals |

---

## Evaluation Metrics

### Performance Targets

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| **Dark Frame Residual** | &le; 30% | Dark subtraction efficiency |
| **Idle Performance Degradation** | &lt; 10% | Normal vs Idle exposure comparison |
| **Temperature Range** | 15~35 &deg;C | Environmental chamber test |
| **Response Time** | &lt; 100 ms | Control loop latency |
| **t_max Accuracy** | &plusmn;5% | Calculated vs measured |

### Quality Gates

| Gate | Criteria | Status |
|------|----------|--------|
| **TRUST 5 - Tested** | 85%+ coverage | <!-- Status: Pending -->: :black_circle: |
| **TRUST 5 - Readable** | Code review pass | <!-- Status: Pending -->: :black_circle: |
| **TRUST 5 - Secured** | OWASP compliance | <!-- Status: Pending -->: :black_circle: |
| **LSP Clean** | Zero errors, zero type errors | <!-- Status: Pending -->: :black_circle: |

---

## Directory Structure

```
TFT-Leak-plan/
├── README.md                  # This file
├── CLAUDE.md                  # Development guidelines
├── docs/                      # All documentation
│   ├── design/                # System design specifications
│   │   ├── fpga_panel_control_spec.md
│   │   ├── imx8_dotnet_control_spec.md
│   │   ├── spi_register_map.md
│   │   └── control_sequence_flows.md
│   ├── panel/                 # Panel specifications
│   │   ├── spec/              # Electrical characteristics
│   │   ├── physics/           # Leakage current analysis
│   │   └── characterization/   # Measurement data
│   ├── common/                # Shared protocols
│   │   ├── spi/               # SPI protocol docs
│   │   └── protocol/          # Communication specs
│   ├── reference/             # Research & validation
│   │   ├── latest/            # Latest v3.1R documents
│   │   ├── implementation/    # Implementation plans by version
│   │   ├── analysis/          # Technical analysis reports
│   │   └── research/          # Academic papers
│   └── delivery/              # External team specs
├── fpga/                      # FPGA firmware
│   ├── rtl/                   # RTL source code
│   │   ├── core/              # Core control logic
│   │   ├── spi/               # SPI slave interface
│   │   ├── bias_mux/          # Bias voltage MUX
│   │   ├── timing_gen/        # Timing generator
│   │   └── dummy_scan/        # Dummy scan engine
│   ├── sim/                   # Testbenches & scripts
│   ├── synth/                 # Synthesis scripts
│   └── scripts/               # Utility scripts
├── imx8/                      # i.MX8 Linux BSP
│   ├── docs/                  # Platform documentation
│   └── specs/                 # Delivery specifications
└── .moai/                     # MoAI project configuration
```

---

## Getting Started

### Prerequisites

| Category | Tool | Version |
|----------|------|---------|
| **FPGA** | Vivado | 2025.2 |
| **FPGA** | ModelSim/Questa | Latest |
| **Linux** | Yocto Project | 4.0+ |
| **Application** | .NET | 8.0 |
| **IDE** | Visual Studio | 2022 |

### Quick Start

```bash
# 1. Clone repository
git clone https://github.com/your-org/TFT-Leak-plan.git
cd TFT-Leak-plan

# 2. Read key documents
cat docs/design/fpga_panel_control_spec.md
cat docs/design/imx8_dotnet_control_spec.md
cat docs/reference/latest/aSi_TFT_Leakage_Implementation_Plan_v3_1_Panel_Integrated.md

# 3. FPGA Simulation (when available)
cd fpga/sim
./run_simulation.sh

# 4. Build i.MX8 BSP (when available)
cd imx8
source setup-env.sh
bitbake tft-leak-image
```

### First Steps for Each Team

**FPGA Team**:
1. Review `docs/design/fpga_panel_control_spec.md`
2. Set up Vivado project for Artix-7 35T
3. Implement timing generator (3072 x 76 &micro;s)

**i.MX8 Team**:
1. Review `docs/design/imx8_dotnet_control_spec.md`
2. Build Yocto BSP for i.MX8 Plus
3. Implement SPI master driver

**Host App Team**:
1. Review `docs/design/imx8_dotnet_control_spec.md`
2. Create .NET 8 WPF/WinUI3 project
3. Design UI mockup for temperature monitoring

---

## Reference Documents

| Document | Location | Description |
|----------|----------|-------------|
| Implementation Plan v3.1 | `docs/reference/latest/` | Complete technical plan |
| Validation Report | `docs/reference/latest/v3_1_Validation_Report_5rounds.md` | 5-round validation results |
| Cross-Validation Summary | `docs/reference/analysis/cross_validation_summary.md` | External source verification |
| FPGA Specification | `docs/design/fpga_panel_control_spec.md` | FPGA requirements |
| i.MX8 Specification | `docs/design/imx8_dotnet_control_spec.md` | Edge controller requirements |

---

## Contributing

This project uses **MoAI (Modular AI)** framework for development. See [CLAUDE.md](./CLAUDE.md) for development guidelines.

### Development Workflow

1. Create feature branch: `git checkout -b feature/your-feature`
2. Follow TRUST 5 quality principles
3. Ensure LSP clean before commit
4. Create PR with description

---

## License

Copyright (c) 2025. All rights reserved.

---

**Version**: 1.0
**Last Updated**: 2026-02-10
**Status`: :yellow_circle: In Progress
