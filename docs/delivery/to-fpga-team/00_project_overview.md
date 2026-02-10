# Project Overview

## FPGA Team Delivery Package

**Document**: 00_project_overview.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Project Goal

**Main Objective**: Implement FPGA control logic for a-Si TFT (amorphous Silicon Thin Film Transistor) Flat Panel Detector panel driving to minimize dark current drift during idle states.

### Problem Statement

a-Si TFT panels exhibit dark current drift when idle, causing image quality degradation. The FPGA must:

1. Generate precise row/column timing for 2048x2048 panel readout
2. Control bias voltage MUX for Normal/Idle/Sleep modes
3. Execute periodic dummy scans during L2 idle mode
4. Provide SPI slave interface for MCU communication

---

## 2. Target Panel Specification

| Parameter | Value | Description |
|-----------|-------|-------------|
| **Panel Model** | R1717AS01.3 | a-Si TFT FPD |
| **Resolution** | 2048 x 2048 | Active pixels |
| **Pixel Pitch** | 140 µm | Pixel spacing |
| **Effective Area** | 286.7 x 286.7 mm | Active area |
| **Technology** | a-Si:H TFT + PIN Diode | Hydrogenated amorphous silicon |

### Electrical Characteristics

| Signal | Value | Description |
|--------|-------|-------------|
| **VGH (Gate High)** | +15 V | TFT ON voltage |
| **VGL (Gate Low)** | -5 V | TFT OFF voltage |
| **V_PD (Normal)** | -1.5 V | Photodiode bias (normal) |
| **V_PD (Idle)** | -0.2 V | Photodiode bias (idle) |
| **V_COL (Normal)** | -1.0 V | Column bias (normal) |
| **V_COL (Idle)** | -0.2 V | Column bias (idle) |

---

## 3. System Architecture

### 3.1 Hardware Block Diagram

```
+----------------+     SPI      +----------------+     LVDS     +----------------+
|                | <---------> |                | <----------->|                |
|  i.MX8 Plus    |             |   FPGA         |             |  aSi TFT Panel |
|  (MCU)         |   Master    |   Artix-7 35T  |   Gate/Row/  |  R1717AS01.3    |
|                |             |                |   Col/Data   |                |
+----------------+             +--------+-------+             +----------------+
       |                              |
       |                              |
       v                              v
+----------------+             +----------------+
|  Temperature   |             |   Bias MUX /   |
|  Sensor (NTC)  |             |   Gate Driver  |
+----------------+             +----------------+
```

### 3.2 FPGA Module Hierarchy

```
fpga_panel_controller (Top)
├── spi_slave_interface          # SPI slave, register file
│   ├── register_file            # 0x00-0x3F registers
│   └── command_decoder          # Read/write logic
├── timing_generator             # Row/column timing
│   ├── row_clock_pll           # Row address clock
│   ├── col_clock_pll           # Column clock
│   ├── reset_pulse_generator   # Storage node reset
│   └── sequencer_fsm           # Main state machine
├── bias_mux_controller          # Bias mode control
│   ├── bias_register           # Current bias mode
│   └── output_driver           # Glitch-free switching
├── dummy_scan_engine            # L2 idle dummy scan
│   ├── periodic_timer          # 30-60 sec timer
│   └── row_reset_controller    # All-row reset sequence
├── adc_controller               # ADC interface
│   ├── adc_interface           # ADC protocol
│   ├── data_fifo               # Data buffering
│   └── output_formatter        # Data formatting
└── status_monitor               # Status flags
    └── interrupt_generator     # Event notification
```

---

## 4. FPGA Responsibilities

### 4.1 Primary Functions

| Function | Description | Priority |
|----------|-------------|----------|
| **Timing Generation** | Generate row/column addresses and clocks | Critical |
| **Bias Control** | Switch bias modes per MCU command | Critical |
| **Data Acquisition** | Read ADC and buffer pixel data | Critical |
| **Dummy Scan** | Execute periodic dummy scan in L2 idle | High |
| **SPI Communication** | Register read/write interface | High |
| **Status Reporting** | Report FPGA state to MCU | Medium |

### 4.2 What FPGA Does NOT Handle

- Image processing algorithms (Host SW)
- Temperature sensor reading (MCU)
- High-level state machine decisions (MCU)
- Arrhenius t_max calculation (MCU)

---

## 5. Timing Requirements Summary

### 5.1 Frame Capture Timing

| Phase | Duration | Description |
|-------|----------|-------------|
| **Reset All** | 10 µs | All rows reset simultaneously |
| **Integration** | 100 ms | Photocharge accumulation |
| **Readout** | 102.4 ms | 2048 rows × 50 µs/row |
| **Total** | ~212 ms | Full frame capture time |

### 5.2 Timing Parameters

| Parameter | Min | Typical | Max | Unit |
|-----------|-----|---------|-----|------|
| **Reset Pulse Width** | 1 | 5 | 10 | µs |
| **Settling Time** | 50 | 100 | 200 | µs |
| **Row Read Time** | 30 | 50 | 100 | µs |
| **Bias Switch Time** | 1 | 5 | 10 | µs |
| **Dummy Reset Time** | 50 | 100 | 200 | µs |

---

## 6. Interface Summary

### 6.1 Input Signals

| Signal | Direction | Width | Description |
|--------|-----------|-------|-------------|
| `clk_100mhz` | Input | 1 | 100 MHz system clock |
| `rst_n` | Input | 1 | Active low reset |
| `spi_sclk` | Input | 1 | SPI clock |
| `spi_mosi` | Input | 1 | SPI master data |
| `spi_cs_n` | Input | 1 | SPI chip select (active low) |
| `adc_data[13:0]` | Input | 14 | ADC data output |
| `adc_busy` | Input | 1 | ADC busy flag |
| `adc_valid` | Input | 1 | ADC data valid |
| `bias_ack` | Input | 1 | Bias change acknowledged |

### 6.2 Output Signals

| Signal | Direction | Width | Description |
|--------|-----------|-------|-------------|
| `spi_miso` | Output | 1 | SPI slave data |
| `row_addr[11:0]` | Output | 12 | Row address (0-2047) |
| `col_addr[11:0]` | Output | 12 | Column address (0-2047) |
| `row_clk_en` | Output | 1 | Row clock enable |
| `col_clk_en` | Output | 1 | Column clock enable |
| `gate_sel` | Output | 1 | Gate select signal |
| `reset_pulse` | Output | 1 | Reset pulse to storage node |
| `bias_mode_sel[1:0]` | Output | 2 | Bias mode select |
| `bias_update_req` | Output | 1 | Request bias update |
| `adc_start` | Output | 1 | Start ADC conversion |
| `adc_clk` | Output | 1 | ADC clock |
| `data_out[31:0]` | Output | 32 | Data output to MCU |
| `data_valid` | Output | 1 | Data valid flag |

---

## 7. Resource Budget

| Resource | Used | Available | Utilization | Status |
|----------|------|-----------|-------------|--------|
| **Slice LUTs** | ~15,000 | 20,800 | 72% | Within budget |
| **Slice Registers** | ~10,000 | 41,600 | 24% | OK |
| **BRAMs** | ~20 | 100 | 20% | OK |
| **DSP48E1** | ~5 | 80 | 6% | OK |
| **MMCM/PLL** | 2 | 2 | 100% | Critical |

---

## 8. Development Tools

| Tool | Version | Purpose |
|------|---------|---------|
| **Vivado** | 2025.2+ | Synthesis, P&R, Bitstream |
| **Questa/ModelSim** | Latest | Simulation |
| **Tcl** | - | Build scripts |

---

## 9. Related Documents

| Document | Location |
|----------|----------|
| Interface Details | `02_interfaces.md` |
| Timing Specifications | `03_timing_specifications.md` |
| Register Map | `04_register_map.md` |
| Acceptance Criteria | `05_acceptance_criteria.md` |
| Panel Physics | `reference/panel_physics_summary.md` |

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial document |

---

**End of Document**
