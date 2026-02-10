# FPGA Panel Control Specification
## a-Si TFT FPD Panel Drive Implementation

**Version**: 1.0
**Created**: 2026-02-10
**Target FPGA**: Xilinx Artix-7 35T FGG484
**Panel**: a-Si TFT FPD 2048x2048

---

## 1. Overview

This specification defines the FPGA control logic for a-Si TFT Flat Panel Detector panel driving, including timing generation, bias voltage control, and idle state management.

### 1.1 Scope

- Row/Column timing generation for 2048x2048 panel
- Bias voltage MUX control (Normal/Idle modes)
- Dummy scan engine for L2 idle state
- SPI slave interface for MCU communication
- ADC control and data buffering

### 1.2 Exclusions

- Image processing algorithms (Host SW responsibility)
- Temperature sensor interface (MCU responsibility)
- High-level state machine (MCU responsibility)

---

## 2. Register Map

### 2.1 Register Address Map

| Address | Name | Bit Width | Access | Description |
|---------|------|-----------|--------|-------------|
| 0x00 | CTRL_REG | 8 | RW | Control Register |
| 0x01 | STATUS_REG | 8 | RO | Status Register |
| 0x02 | BIAS_SELECT | 2 | RW | Bias Mode Selection |
| 0x03 | DUMMY_PERIOD | 16 | RW | Dummy Scan Period (sec) |
| 0x04 | DUMMY_CONTROL | 8 | RW | Dummy Scan Control |
| 0x05 | ROW_START | 16 | RW | Starting Row (ROI) |
| 0x06 | ROW_END | 16 | RW | Ending Row (ROI) |
| 0x07 | COL_START | 16 | RW | Starting Column (ROI) |
| 0x08 | COL_END | 16 | RW | Ending Column (ROI) |
| 0x09 | INTEGRATION_TIME | 16 | RW | Integration Time (ms) |
| 0x0A | TIMING_CONFIG | 32 | RW | Timing Configuration |
| 0x0B - 0x0F | RESERVED | - | - | Reserved |

### 2.2 Control Register (0x00)

```
Bit 7: FRAME_START - Write 1 to start frame capture
Bit 6: FRAME_RESET - Write 1 to reset frame capture
Bit 5: DUMMY_ENABLE - Enable dummy scan mode
Bit 4: ADC_ENABLE - Enable ADC
Bit 3: TEST_MODE - Enable test pattern mode
Bit 2: BIAS_UPDATE_PENDING - Bias update in progress
Bit 1: FIFO_RESET - Reset data FIFO
Bit 0: SOFT_RESET - Soft reset all modules
```

### 2.3 Status Register (0x01)

```
Bit 7: FRAME_BUSY - Frame capture in progress
Bit 6: DUMMY_ACTIVE - Dummy scan active
Bit 5: BIAS_MODE_READY - Bias mode settled
Bit 4: FIFO_EMPTY - Data FIFO empty
Bit 3: FIFO_FULL - Data FIFO full
Bit 2: ADC_READY - ADC ready for readout
Bit 1: IDLE_STATE - System in idle state
Bit 0: POWER_GOOD - Power supply OK
```

### 2.4 Bias Select Register (0x02)

```
Bits [1:0]: BIAS_MODE
  00: NORMAL_BIAS - V_PD = -1.5V, V_COL = -1.0V
  01: IDLE_LOW_BIAS - V_PD = -0.2V, V_COL = -0.2V
  10: SLEEP_BIAS - V_PD = 0V, V_COL = 0V (minimal)
  11: RESERVED
```

---

## 3. Module Architecture

### 3.1 Top-Level Module

```systemverilog
module fpga_panel_controller #(
    parameter ROWS = 2048,
    parameter COLS = 2048,
    parameter ADC_BITS = 14
)(
    // Clock and Reset
    input  logic clk_100mhz,
    input  logic rst_n,

    // SPI Slave Interface (from MCU)
    input  logic spi_sclk,
    input  logic spi_mosi,
    output logic spi_miso,
    input  logic spi_cs_n,

    // Bias Control Outputs
    output logic [1:0] bias_mode_sel,    // 2-bit bias select
    output logic bias_update_req,        // Request bias update
    input  logic bias_ack,               // Bias change acknowledged

    // Row/Column Control to Panel
    output logic [11:0] row_addr,        // Row address
    output logic [11:0] col_addr,        // Column address
    output logic row_clk_en,             // Row clock enable
    output logic col_clk_en,             // Column clock enable
    output logic gate_sel,               // Gate select signal
    output logic reset_pulse,            // Reset pulse to storage node

    // ADC Interface
    output logic adc_start,
    output logic adc_clk,
    input  logic [ADC_BITS-1:0] adc_data,
    input  logic adc_busy,
    input  logic adc_valid,

    // Data Output (to Host via MCU)
    output logic [31:0] data_out,
    output logic data_valid,
    input  logic data_ready
);
```

### 3.2 Module Hierarchy

```
fpga_panel_controller
├── spi_slave_interface
│   ├── register_file
│   └── command_decoder
├── timing_generator
│   ├── row_clock_generator (PLL)
│   ├── col_clock_generator (PLL)
│   ├── reset_pulse_generator
│   └── sequencer_state_machine
├── bias_mux_controller
│   ├── bias_register
│   ├── mode_decoder
│   └── output_driver
├── dummy_scan_engine
│   ├── periodic_timer
│   ├── row_reset_controller
│   └── scan_supervisor
├── adc_controller
│   ├── adc_interface
│   ├── data_fifo
│   └── output_formatter
└── status_monitor
    ├── busy_detector
    └── error_handler
```

---

## 4. Timing Generation

### 4.1 Clock Domains

| Domain | Frequency | Source | Purpose |
|--------|-----------|--------|---------|
| CLK_100M | 100 MHz | Oscillator | Main system clock |
| CLK_ADC | 20 MHz | PLL | ADC sampling clock |
| CLK_ROW | 1-10 MHz | PLL | Row address clock |
| CLK_COL | 1-10 MHz | PLL | Column clock |

### 4.2 Timing Parameters

| Parameter | Value | Unit | Description |
|-----------|-------|------|-------------|
| T_RESET_PULSE | 1 | µs | Minimum reset pulse width |
| T_SETTLE | 100 | µs | Storage node settle time |
| T_INTEGRATION | 100 | ms | Normal integration time |
| T_READ_ROW | 50 | µs | Time to read one row |
| T_BIAS_SWITCH | 10 | µs | Bias switching time |
| T_DUMMY_RESET | 100 | µs | Dummy reset time per row |

### 4.3 Normal Read Sequence

```
State Transition Diagram:

IDLE → RESET_ALL → INTEGRATE → READ_ROW(n) → READ_ROW(n+1) → ... → READ_ROW(2047) → IDLE

Timing:
RESET_ALL:   10 µs  (all rows)
INTEGRATE:   100 ms (exposure time)
READ_ROW:    50 µs × 2048 = 102.4 ms total
```

### 4.4 Idle Mode Timing

**L1 (Normal Idle)**:
- No active signals
- Bias: NORMAL_BIAS
- Period: No periodic action

**L2 (Low-bias Idle)**:
- Dummy scan every T_DUMMY (30-60 sec)
- Bias: IDLE_LOW_BIAS
- Dummy scan duration: ~2 ms (all rows)

**L3 (Deep Sleep)**:
- Minimal signals
- Bias: SLEEP_BIAS
- All clocks stopped

---

## 5. Bias Voltage MUX Control

### 5.1 Bias Mode Definitions

| Mode | V_PD | V_COL | Purpose |
|------|------|-------|---------|
| NORMAL_BIAS | -1.5 V | -1.0 V | Normal readout operation |
| IDLE_LOW_BIAS | -0.2 V | -0.2 V | L2 idle (minimize leakage) |
| SLEEP_BIAS | 0 V | 0 V | L3 deep sleep |

### 5.2 Bias Control State Machine

```systemverilog
typedef enum logic [1:0] {
    NORMAL_BIAS = 2'b00,
    IDLE_LOW_BIAS = 2'b01,
    SLEEP_BIAS = 2'b10,
    RESERVED = 2'b11
} bias_mode_t;

module bias_mux_controller (
    input  logic clk,
    input  logic rst_n,
    input  bias_mode_t bias_mode,
    input  logic bias_update_req,
    output logic [1:0] bias_sel_out,
    output logic bias_ready
);
    // Bias switching with < 10 µs transition time
    // glitch-free MUX switching
endmodule
```

### 5.3 DAC Interface

For external DAC control (if needed):

```
DAC Register Values:
- V_PD = -1.5V → DAC_CODE_NORMAL = 0xCCC (assuming 12-bit DAC, 0-3.3V range)
- V_PD = -0.2V → DAC_CODE_IDLE = 0xE6E
- V_COL = -1.0V → DAC_CODE_COL_NORMAL = 0xDAA
- V_COL = -0.2V → DAC_CODE_COL_IDLE = 0xE6E
```

---

## 6. Dummy Scan Engine (L2 Idle)

### 6.1 Purpose

Periodic reset of storage nodes during L2 idle to prevent dark current accumulation.

### 6.2 Operation

```
Every T_DUMMY seconds (30-60 sec, configurable):
    for each row i = 0 to 2047:
        SET gate_i = ON
        WAIT 1 µs
        TRIGGER_RESET pulse (1 µs)
        WAIT 100 µs (settle)
        SET gate_i = OFF

    Total duration: ~2 ms
    No ADC readout (data not used)
```

### 6.3 Implementation

```systemverilog
module dummy_scan_engine (
    input  logic clk,
    input  logic rst_n,
    input  logic dummy_enable,
    input  logic [15:0] dummy_period_sec,
    output logic dummy_active,
    output logic row_reset_req,
    output logic [11:0] row_addr
);
    // 32-bit timer for 30-60 second period
    // Counter value = period * CLK_FREQ
    // Trigger row reset sequence when timer expires
endmodule
```

---

## 7. SPI Slave Interface

### 7.1 SPI Configuration

| Parameter | Value |
|-----------|-------|
| Mode | Mode 0 (CPOL=0, CPHA=0) |
| Clock Frequency | Max 10 MHz |
| Data Width | 8 bits |
| Byte Order | MSB first |

### 7.2 Protocol

```
Write Operation:
CS_N LOW
MOSI: [CMD(8)] [ADDR(8)] [DATA(8)]
CS_N HIGH

Read Operation:
CS_N LOW
MOSI: [CMD(8)] [ADDR(8)]
MISO: [DUMMY] [DATA(8)]
CS_N HIGH

CMD Codes:
0x01: WRITE_REG
0x02: READ_REG
0x03: BURST_READ
```

---

## 8. Reset Strategy

### 8.1 Reset Types

| Reset Type | Scope | Trigger |
|------------|-------|---------|
| Soft Reset | All logic except PLL | Register write (CTRL_REG[0]) |
| Hard Reset | All logic including PLL | External reset_n pin |
| FIFO Reset | Data FIFO only | Register write (CTRL_REG[1]) |

### 8.2 Reset Sequence

```
Power-On Reset:
1. Assert reset_n = LOW
2. Wait for clock stable (100 ms)
3. Deassert reset_n = HIGH
4. Initialize all registers to default values
5. Start timing generators
```

---

## 9. State Machine Design

### 9.1 Main Control State Machine

```systemverilog
typedef enum logic [2:0] {
    ST_IDLE = 3'b000,
    ST_RESET = 3'b001,
    ST_INTEGRATE = 3'b010,
    ST_READOUT = 3'b011,
    ST_DUMMY_SCAN = 3'b100,
    ST_BIAS_SWITCH = 3'b101,
    ST_ERROR = 3'b111
} state_t;

always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        state <= ST_IDLE;
    end else begin
        case (state)
            ST_IDLE: begin
                if (frame_start_req) state <= ST_RESET;
                else if (dummy_enable && dummy_timer_expire) state <= ST_DUMMY_SCAN;
                else if (bias_mode_change) state <= ST_BIAS_SWITCH;
            end

            ST_RESET: begin
                if (reset_done) state <= ST_INTEGRATE;
            end

            ST_INTEGRATE: begin
                if (integration_done) state <= ST_READOUT;
            end

            ST_READOUT: begin
                if (all_rows_read) state <= ST_IDLE;
            end

            ST_DUMMY_SCAN: begin
                if (all_rows_reset) state <= ST_IDLE;
            end

            ST_BIAS_SWITCH: begin
                if (bias_settled) state <= ST_IDLE;
            end

            default: state <= ST_IDLE;
        endcase
    end
end
```

---

## 10. Timing Constraints

### 10.1 XDC Constraints

```tcl
# Clock constraints
create_clock -period 10.000 -name clk_100mhz [get_ports clk_100mhz]
create_generated_clock -name clk_adc -source clk_100mhz -multiply 1 -divide 5 [get_pins adc_pll/clk_out]

# Input/Output delays
set_input_delay -clock clk_100mhz -max 2 [get_ports spi_*]
set_output_delay -clock clk_100mhz -max 3 [get_ports bias_*]

# False paths (asynchronous resets)
set_false_path -from [get_ports rst_n] -to [all_registers]

# Multicycle paths
set_multicycle_path -setup 2 -from [get_cells row_counter*] -to [get_cells col_counter*]
```

---

## 11. Resource Utilization Estimate

| Resource | Used | Available | Utilization |
|----------|------|-----------|-------------|
| Slice LUTs | ~15,000 | 20,800 | ~72% |
| Slice Registers | ~10,000 | 41,600 | ~24% |
| BRAMs | ~20 | 100 | ~20% |
| DSP48E1 | ~5 | 80 | ~6% |
| MMCM/PLL | ~2 | 2 | ~100% |

---

## 12. Verification Plan

### 12.1 Simulation Testbench

- Normal readout sequence
- Idle mode transitions (L1 → L2 → L3)
- Bias switching timing
- Dummy scan periodic operation
- SPI register read/write
- FIFO overflow/underflow conditions

### 12.2 Test Points

| Test | Description | Pass Criteria |
|------|-------------|---------------|
| TB_001 | Power-on reset | All registers initialized |
| TB_002 | SPI register write | Values written correctly |
| TB_003 | Bias mode switch | < 10 µs switching time |
| TB_004 | Dummy scan period | ±1% accuracy |
| TB_005 | Full frame readout | All rows read once |
| TB_006 | FIFO behavior | No overflow/underflow |

---

## 13. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial specification |

---

**End of Specification**
