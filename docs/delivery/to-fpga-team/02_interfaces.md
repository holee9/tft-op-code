# Interfaces

## FPGA Team Delivery Package

**Document**: 02_interfaces.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Interface Overview

The FPGA provides four primary interfaces:

1. **SPI Slave Interface** - Communication with i.MX8 MCU
2. **LVDS Panel Interface** - Row/column control to panel
3. **Bias Control Interface** - Bias voltage MUX control
4. **ADC Interface** - Data acquisition from panel

---

## 2. SPI Slave Interface

### 2.1 Electrical Specifications

| Parameter | Value | Notes |
|-----------|-------|-------|
| **Mode** | Mode 0 (CPOL=0, CPHA=0) | Clock idle low, sample on rising edge |
| **Max Clock Frequency** | 10 MHz | Must support up to 10 MHz |
| **Data Width** | 8 bits | Per transfer |
| **Byte Order** | MSB first | Most significant bit first |
| **Chip Select** | Active low | CS_N = 0 selects FPGA |

### 2.2 Signal Definitions

| Signal | Direction | Width | Description |
|--------|-----------|-------|-------------|
| `spi_sclk` | Input | 1 | SPI clock from MCU |
| `spi_mosi` | Input | 1 | Master Out Slave In |
| `spi_miso` | Output | 1 | Master In Slave Out (tri-state when CS_N=1) |
| `spi_cs_n` | Input | 1 | Chip Select (active low) |

### 2.3 Pin Assignment (Recommended)

| Signal | FPGA Pin Group | I/O Standard |
|--------|----------------|--------------|
| `spi_sclk` | HR Bank 15 | LVCMOS18 |
| `spi_mosi` | HR Bank 15 | LVCMOS18 |
| `spi_miso` | HR Bank 15 | LVCMOS18 |
| `spi_cs_n` | HR Bank 15 | LVCMOS18 |

### 2.4 Timing Diagram

```
SPI Write Transaction:
          __    __    __    __    __    __
CS_N  ___|  |__|  |__|  |__|  |__|  |__|  |_____
            \     \     \     \     \     \
SCLK _______/^\___/^\___/^\___/^\___/^\___/^\____
                \     \     \     \     \     \
MOSI ____________X____X____X____X_________________
                CMD   ADDR  DATA
              (0x01) (0x00) (0xFF)

SPI Read Transaction:
          __    __    __    __    __    __    __    __
CS_N  ___|  |__|  |__|  |__|  |__|  |__|  |__|  |__|  |___
            \     \     \     \     \     \     \     \
SCLK _______/^\___/^\___/^\___/^\___/^\___/^\___/^\___
                \     \     \     \     \     \     \
MOSI ____________X____X____X__________________________
                CMD   ADDR  DUMMY
              (0x02) (0x00) (0x00)

MISO ______________XXXXXXXXXXXXXXXX___________________
                         DUMMY  DATA
                       (0x00)  (0xFF)
```

### 2.5 Protocol Details

See `04_register_map.md` for complete SPI protocol and register definitions.

---

## 3. LVDS Panel Interface

### 3.1 Signal Group A: Row/Column Address

| Signal | Direction | Width | Description |
|--------|-----------|-------|-------------|
| `row_addr[11:0]` | Output | 12 | Row address (0-2047) |
| `col_addr[11:0]` | Output | 12 | Column address (0-2047) |

**Timing**: Address must be stable for minimum 100 ns before clock enable.

### 3.2 Signal Group B: Clock Enables

| Signal | Direction | Width | Description |
|--------|-----------|-------|-------------|
| `row_clk_en` | Output | 1 | Row clock enable (pulse per row) |
| `col_clk_en` | Output | 1 | Column clock enable (pulse per pixel) |

**Pulse Width**: Minimum 50 ns.

### 3.3 Signal Group C: Gate Control

| Signal | Direction | Width | Description |
|--------|-----------|-------|-------------|
| `gate_sel` | Output | 1 | Gate select for current row |
| `reset_pulse` | Output | 1 | Reset pulse to storage node |

**Pulse Width**: 1-10 µs (configurable).

### 3.4 Electrical Specifications

| Parameter | Min | Typical | Max | Unit |
|-----------|-----|---------|-----|------|
| **Output Voltage (High)** | 2.4 | 3.3 | 3.6 | V |
| **Output Voltage (Low)** | 0 | 0.2 | 0.4 | V |
| **Rise Time** | - | 5 | 20 | ns |
| **Fall Time** | - | 5 | 20 | ns |
| **Drive Strength** | - | 12 | - | mA |

### 3.5 LVDS Link (Data Path)

| Signal | Direction | Width | Description |
|--------|-----------|-------|-------------|
| `lvds_data_out[13:0]` | Output | 14 | ADC data to MCU (LVDS) |
| `lvds_data_clk` | Output | 1 | LVDS clock for data |
| `lvds_data_valid` | Output | 1 | Data valid strobe |

**Clock Frequency**: DDR, up to 500 Mbps.

---

## 4. Bias Control Interface

### 4.1 Signal Definitions

| Signal | Direction | Width | Description |
|--------|-----------|-------|-------------|
| `bias_mode_sel[1:0]` | Output | 2 | Bias mode select |
| `bias_update_req` | Output | 1 | Request bias update |
| `bias_ack` | Input | 1 | Bias change acknowledged |

### 4.2 Bias Mode Encoding

| Code | Mode | V_PD | V_COL |
|------|------|------|-------|
| 00 | NORMAL_BIAS | -1.5 V | -1.0 V |
| 01 | IDLE_LOW_BIAS | -0.2 V | -0.2 V |
| 10 | SLEEP_BIAS | 0 V | 0 V |
| 11 | RESERVED | - | - |

### 4.3 Bias Switch Timing

```
FPGA                    Bias MUX
 │                       │
 │─── BIAS_MODE_SEL ─────►│──► [Mode Change]
 │    [01]                │
 │                       │
 │─── BIAS_UPDATE_REQ ───►│──► [Start Switching]
 │                       │
 │                       │    [DAC Settling]
 │                       │    < 10 µs
 │                       │
 │◄─── BIAS_ACK ──────────│──► [Switch Complete]
 │                       │
 │─── Set BIAS_READY ◄────│    [Update Status]
```

**Requirements**:
- Maximum switching time: 10 µs
- Glitch-free transitions
- Acknowledge must be received before proceeding

---

## 5. ADC Interface

### 5.1 Signal Definitions

| Signal | Direction | Width | Description |
|--------|-----------|-------|-------------|
| `adc_start` | Output | 1 | Start ADC conversion |
| `adc_clk` | Output | 1 | ADC clock (up to 20 MHz) |
| `adc_data[13:0]` | Input | 14 | ADC data output |
| `adc_busy` | Input | 1 | ADC busy flag |
| `adc_valid` | Input | 1 | ADC data valid flag |

### 5.2 Timing Diagram

```
FPGA                    ADC
 │                       │
 │─── ADC_START ─────────►│──► [Start Conversion]
 │                       │
 │   . . . ADC_CLK . . .  │    [Clocking ADC]
 │                       │
 │◄─── ADC_BUSY = 1 ──────│    [Converting]
 │                       │
 │◄─── ADC_VALID ─────────│    [Data Ready]
 │                       │
 │◄─── ADC_DATA[13:0] ────│    [14-bit Data]
 │                       │
```

### 5.3 ADC Timing Parameters

| Parameter | Min | Typical | Max | Unit |
|-----------|-----|---------|-----|------|
| **Clock Frequency** | 1 | 10 | 20 | MHz |
| **Conversion Time** | - | 500 | - | ns |
| **Data Hold Time** | 10 | 20 | - | ns |
| **Busy Latency** | - | 3 | 5 | clocks |

---

## 6. Data Output Interface (to MCU)

### 6.1 Signal Definitions

| Signal | Direction | Width | Description |
|--------|-----------|-------|-------------|
| `data_out[31:0]` | Output | 32 | Pixel data output |
| `data_valid` | Output | 1 | Data valid strobe |
| `data_ready` | Input | 1 | MCU ready for data |

### 6.2 Data Format

```
data_out[31:0] Format:
┌────────────────────────────────────────────────────────────┐
│ Bits 31-16: Pixel Data (14 bits, MSB aligned)              │
│ Bits 15-8:  Row Address (8 bits)                           │
│ Bits 7-0:   Column Address (8 bits)                        │
└────────────────────────────────────────────────────────────┘
```

### 6.3 Flow Control

```
FPGA                    MCU
 │                       │
 │─── DATA_VALID ────────►│
 │                       │
 │◄─── DATA_READY ────────│
 │                       │
 │─── DATA_OUT[31:0] ────►│
 │                       │
```

---

## 7. Clock and Reset

### 7.1 Clock Specifications

| Signal | Frequency | Source | Purpose |
|--------|-----------|--------|---------|
| `clk_100mhz` | 100 MHz | External oscillator | Main system clock |

### 7.2 Reset Specifications

| Signal | Active Level | Minimum Width | Description |
|--------|--------------|---------------|-------------|
| `rst_n` | Low | 100 ns | Global reset |

### 7.3 Generated Clocks (Internal)

| Clock | Frequency | Source | Purpose |
|-------|-----------|--------|---------|
| `clk_adc` | 20 MHz | PLL | ADC sampling |
| `clk_row` | 1-10 MHz | PLL | Row timing |
| `clk_col` | 1-10 MHz | PLL | Column timing |

---

## 8. Interrupt Interface

### 8.1 Signal Definition

| Signal | Direction | Width | Description |
|--------|-----------|-------|-------------|
| `interrupt` | Output | 1 | Active high interrupt to MCU |

### 8.2 Interrupt Sources

| Source | Priority | Condition |
|--------|----------|-----------|
| Frame Complete | High | STATUS_REG[FRAME_COMPLETE] = 1 |
| Dummy Complete | Medium | STATUS_REG[DUMMY_COMPLETE] = 1 |
| FIFO Threshold | Medium | STATUS_REG[FIFO_THRESHOLD] = 1 |
| Error | Critical | STATUS_REG[ERROR] = 1 |

---

## 9. Top-Level Module Template

```systemverilog
module fpga_panel_controller #(
    parameter ROWS = 2048,
    parameter COLS = 2048,
    parameter ADC_BITS = 14
)(
    // Clock and Reset
    input  logic clk_100mhz,
    input  logic rst_n,

    // SPI Slave Interface
    input  logic spi_sclk,
    input  logic spi_mosi,
    output logic spi_miso,
    input  logic spi_cs_n,

    // Bias Control Interface
    output logic [1:0] bias_mode_sel,
    output logic bias_update_req,
    input  logic bias_ack,

    // Row/Column Control to Panel
    output logic [11:0] row_addr,
    output logic [11:0] col_addr,
    output logic row_clk_en,
    output logic col_clk_en,
    output logic gate_sel,
    output logic reset_pulse,

    // ADC Interface
    output logic adc_start,
    output logic adc_clk,
    input  logic [ADC_BITS-1:0] adc_data,
    input  logic adc_busy,
    input  logic adc_valid,

    // Data Output (to MCU)
    output logic [31:0] data_out,
    output logic data_valid,
    input  logic data_ready,

    // Interrupt
    output logic interrupt
);

    // Module implementation here

endmodule
```

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial interface specification |

---

**End of Document**
