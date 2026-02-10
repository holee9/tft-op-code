# Timing Specifications

## FPGA Team Delivery Package

**Document**: 03_timing_specifications.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Overview

This document defines all timing requirements for the FPGA panel controller, including row/column timing, frame timing, and bias switching timing.

---

## 2. Clock Architecture

### 2.1 Clock Tree

```
                     ┌─────────────┐
                     │ 100 MHz OSC │
                     └──────┬──────┘
                            │ clk_100mhz
                            ▼
                     ┌─────────────┐
                     │   BUFGMUX   │
                     └──────┬──────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
    ┌───────────┐   ┌───────────┐   ┌───────────┐
    │   PLL     │   │   PLL     │   │   PLL     │
    │ ADC Clock │   │ Row Clock │   │ Col Clock │
    └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
          │               │               │
          ▼               ▼               ▼
    ┌───────────┐   ┌───────────┐   ┌───────────┐
    │ clk_adc   │   │ clk_row   │   │ clk_col   │
    │ 20 MHz    │   │ 5 MHz     │   │ 10 MHz    │
    └───────────┘   └───────────┘   └───────────┘
```

### 2.2 Clock Specifications

| Clock | Frequency | Source | Tolerance | Jitter |
|-------|-----------|--------|-----------|--------|
| `clk_100mhz` | 100 MHz | Oscillator | ±50 ppm | < 100 ps |
| `clk_adc` | 20 MHz | PLL | ±0.5% | < 200 ps |
| `clk_row` | 5 MHz | PLL | ±1% | < 200 ps |
| `clk_col` | 10 MHz | PLL | ±1% | < 200 ps |

### 2.3 Reset Timing

| Parameter | Value | Description |
|-----------|-------|-------------|
| Reset Assertion | ≥ 100 ns | Minimum `rst_n` pulse |
| Reset Release Time | < 1 µs | Time to exit reset state |
| PLL Lock Time | < 10 ms | Maximum PLL lock time |

---

## 3. Frame Capture Timing

### 3.1 Frame Capture State Machine

```
       ┌─────────┐
       │  IDLE   │
       └────┬────┘
            │ FRAME_START
            ▼
       ┌─────────┐
       │  RESET  │
       │  10 µs  │
       └────┬────┘
            │
            ▼
       ┌─────────┐
       │INTEGRATE│
       │ 100 ms  │
       └────┬────┘
            │
            ▼
       ┌─────────┐
       │ READOUT │◄─────────────────┐
       │~102.4ms│                   │
       └────┬────┘                   │
            │                       │
            ▼                       │
       ┌─────────┐                   │
       │  IDLE   │───────────────────┘
       └─────────┘
```

### 3.2 Frame Timing Budget

| Phase | Duration | Description |
|-------|----------|-------------|
| Reset Phase | 10 µs | All rows reset simultaneously |
| Integration | 100 ms | Photocharge accumulation |
| Row Readout | 50 µs/row | Read each row sequentially |
| Total Readout | 102.4 ms | 2048 rows × 50 µs |
| **Total Frame** | **~212 ms** | Full frame capture |

### 3.3 Row Readout Timing

```
Row Readout Sequence (50 µs total):

     ┌────────┐
     │ Row i  │
     └───┬────┘
         │
         ▼
   ┌───────────┐
   │ Gate ON   │ 1 µs
   └─────┬─────┘
         │
         ▼
   ┌───────────┐
   │ Settle    │ 5 µs
   └─────┬─────┘
         │
         ▼
   ┌───────────┐
   │ ADC Read  │ 40 µs (2048 cols @ 20 MHz)
   └─────┬─────┘
         │
         ▼
   ┌───────────┐
   │ Gate OFF  │ 1 µs
   └─────┬─────┘
         │
         ▼
   ┌───────────┐
   │ Row Next  │ 3 µs (overhead)
   └───────────┘
```

### 3.4 Row Timing Parameters

| Parameter | Min | Typical | Max | Unit |
|-----------|-----|---------|-----|------|
| Gate ON Time | 0.5 | 1 | 2 | µs |
| Settle Time | 3 | 5 | 10 | µs |
| ADC Read Time | 30 | 40 | 50 | µs |
| Gate OFF Time | 0.5 | 1 | 2 | µs |
| Row Overhead | 1 | 3 | 5 | µs |
| **Total Row Time** | 35 | 50 | 70 | µs |

---

## 4. Column Timing

### 4.1 Column Readout Timing

Within each row, 2048 columns are read sequentially:

```
Column Readout:
     ┌──────────────────────────────────────┐
     │  Column 0   Column 1  ... Column 2047 │
     │    20ns       20ns          20ns     │
     └──────────────────────────────────────┘
     Total: 2048 × 20ns = 40.96 µs
```

### 4.2 Column Timing Parameters

| Parameter | Value | Unit |
|-----------|-------|------|
| ADC Clock | 20 | MHz |
| Samples per Row | 2048 | pixels |
| Sample Period | 50 | ns |
| Total Column Time | 40.96 | µs |

---

## 5. Bias Switching Timing

### 5.1 Bias Mode Switch Sequence

```
Bias Switch Timing Diagram:

     ┌────────────────────────────────────────────┐
     │       BIAS_MODE_SEL Change                 │
     │  [00]              ┌───[01]───┐           │
     └────────────────────┘           └───────────┘
                                │
     ┌───────────────────────────┴───────────────┐
     │       BIAS_UPDATE_REQ                      │
     │              ┌────────────┐                │
     └──────────────┘            └────────────────┘
                                │
                                │    < 10 µs
                                │    (Switching)
                                │
     ┌───────────────────────────┴───────────────┐
     │       BIAS_ACK                             │
     │                        ┌──────────┐       │
     └────────────────────────┘          └───────┘
```

### 5.2 Bias Switching Parameters

| Parameter | Min | Typical | Max | Unit |
|-----------|-----|---------|-----|------|
| Mode Change to Update | 0 | 0 | 1 | µs |
| Switching Time | 1 | 5 | 10 | µs |
| Acknowledge Delay | 0 | 0 | 1 | µs |
| **Total Switch Time** | 1 | 5 | 10 | µs |

### 5.3 Bias Mode Timing Relationships

```
Bias Mode vs Frame Capture:

L1 (Normal Bias):
  - Frame capture: Normal timing
  - Idle: No action

L2 (Low Bias):
  - Frame capture: Switch to Normal → Capture → Return to Low
  - Idle: Dummy scan every 30-60 sec

L3 (Sleep):
  - Frame capture: Wake → Warm-up → Capture → Return to Sleep
  - Idle: Minimal power
```

---

## 6. Dummy Scan Timing (L2 Idle)

### 6.1 Dummy Scan Sequence

```
L2 Idle Dummy Scan (60 second period):

     ┌─────────────────────────────────────────────┐
     │         Normal L2 Idle (Low Bias)          │
     │              ~60 seconds                    │
     └──────────────────┬──────────────────────────┘
                        │
                        ▼
     ┌─────────────────────────────────────────────┐
     │         Dummy Scan Trigger                  │
     └──────────────────┬──────────────────────────┘
                        │
                        ▼
     ┌─────────────────────────────────────────────┐
     │   Row 0: Reset (1 µs) + Settle (100 µs)    │
     │   Row 1: Reset (1 µs) + Settle (100 µs)    │
     │   ...                                      │
     │   Row 2047: Reset (1 µs) + Settle (100 µs) │
     └─────────────────────────────────────────────┘
                    Total: ~2 ms
                        │
                        ▼
     ┌─────────────────────────────────────────────┐
     │         Return to L2 Idle                   │
     └─────────────────────────────────────────────┘
```

### 6.2 Dummy Scan Timing Parameters

| Parameter | Min | Typical | Max | Unit |
|-----------|-----|---------|-----|------|
| Period | 30 | 60 | 65535 | sec |
| Reset Pulse | 1 | 1 | 10 | µs |
| Settle Time | 50 | 100 | 200 | µs |
| Per Row Time | 51 | 101 | 210 | µs |
| **Total Scan Time** | 0.1 | 0.2 | 0.5 | sec |

---

## 7. SPI Interface Timing

### 7.1 SPI Mode 0 Timing

```
SPI Mode 0 (CPOL=0, CPHA=0):

     ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
SCK ┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘
     ┌───┬───┬───┬───┬───┬───┬───┬───┐
MOSI S7─S6─S5─S4─S3─S2─S1─S0───
     ────┬───┬───┬───┬───┬───┬───┬───
MISO ────X7─X6─X5─X4─X3─X2─X1─X0
         ↑   ↑               ↑   ↑
         CS_L  Setup    Sample   Hold
```

### 7.2 SPI Timing Parameters

| Parameter | Min | Max | Unit |
|-----------|-----|-----|------|
| SCK Frequency | - | 10 | MHz |
| CS Setup Time | 10 | - | ns |
| CS Hold Time | 10 | - | ns |
| SCK to MOSI Valid | - | 50 | ns |
| MOSI Hold | 10 | - | ns |
| MISO Valid | - | 100 | ns |

---

## 8. Timing Constraints (XDC Template)

```tcl
# ===================================================================
# Timing Constraints for fpga_panel_controller
# ===================================================================

# Primary Clock
create_clock -period 10.000 -name clk_100mhz [get_ports clk_100mhz]

# Generated Clocks (from PLL outputs)
create_generated_clock -name clk_adc \
    -source [get_pins pll/clk_in] \
    -multiply 1 -divide 5 \
    [get_ports adc_clk]

create_generated_clock -name clk_row \
    -source [get_pins pll/clk_in] \
    -multiply 1 -divide 20 \
    [get_ports row_clk_en]

create_generated_clock -name clk_col \
    -source [get_pins pll/clk_in] \
    -multiply 1 -divide 10 \
    [get_ports col_clk_en]

# Input Delay Constraints (SPI)
set_input_delay -clock clk_100mhz -max 2 [get_ports spi_*]
set_input_delay -clock clk_100mhz -min 0 [get_ports spi_*]

# Output Delay Constraints (Panel Interface)
set_output_delay -clock clk_100mhz -max 3 [get_ports {row_addr[*]}]
set_output_delay -clock clk_100mhz -max 3 [get_ports {col_addr[*]}]
set_output_delay -clock clk_100mhz -max 2 [get_ports gate_sel]
set_output_delay -clock clk_100mhz -max 2 [get_ports reset_pulse]

# ADC Interface
set_input_delay -clock clk_adc -max 5 [get_ports {adc_data[*]}]
set_input_delay -clock clk_adc -max 3 [get_ports adc_valid]

# False Paths (asynchronous resets)
set_false_path -from [get_ports rst_n] -to [all_registers]

# Multicycle Paths
set_multicycle_path -setup 2 -from [get_cells row_counter*] \
    -to [get_cells col_counter*]

# Minimize delay for interrupt
set_max_delay 5 -from [get_cells interrupt_gen*] \
    -to [get_ports interrupt]
```

---

## 9. Timing Margin Analysis

### 9.1 Row Timing Budget

| Element | Time | Margin |
|---------|------|--------|
| Gate ON | 1 µs | 0.5 µs |
| Settle | 5 µs | 2 µs |
| ADC Read | 40 µs | 0 µs (critical) |
| Gate OFF | 1 µs | 0.5 µs |
| Overhead | 3 µs | 1 µs |
| **Total** | **50 µs** | **4 µs (8%)** |

### 9.2 Frame Timing Budget

| Element | Time | Margin |
|---------|------|--------|
| Reset | 10 µs | 5 µs |
| Integration | 100 ms | 0 ms (user config) |
| Readout | 102.4 ms | 5 ms |
| **Total** | **~212 ms** | **5 ms (2.4%)** |

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial timing specification |

---

**End of Document**
