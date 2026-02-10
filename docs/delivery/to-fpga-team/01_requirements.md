# Requirements

## FPGA Team Delivery Package

**Document**: 01_requirements.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Functional Requirements

### FR-1: Timing Generation
The FPGA SHALL generate row and column timing signals for 2048x2048 panel readout.

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1.1 | Generate sequential row addresses from 0 to 2047 | Critical |
| FR-1.2 | Generate column addresses for each row read | Critical |
| FR-1.3 | Provide row clock enable signal (1-10 MHz) | Critical |
| FR-1.4 | Provide column clock enable signal (1-10 MHz) | Critical |
| FR-1.5 | Support ROI (Region of Interest) readout | High |
| FR-1.6 | Support configurable integration time (1-65535 ms) | High |

### FR-2: Bias Control
The FPGA SHALL control bias voltage MUX for Normal/Idle/Sleep modes.

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-2.1 | Support NORMAL_BIAS mode (V_PD=-1.5V, V_COL=-1.0V) | Critical |
| FR-2.2 | Support IDLE_LOW_BIAS mode (V_PD=-0.2V, V_COL=-0.2V) | Critical |
| FR-2.3 | Support SLEEP_BIAS mode (V_PD=0V, V_COL=0V) | Critical |
| FR-2.4 | Complete bias switching within 10 µs | Critical |
| FR-2.5 | Provide glitch-free MUX switching | High |
| FR-2.6 | Report bias ready status via SPI | High |

### FR-3: Data Acquisition
The FPGA SHALL acquire and buffer ADC data from panel readout.

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-3.1 | Interface with 14-bit ADC | Critical |
| FR-3.2 | Provide ADC clock (up to 20 MHz) | Critical |
| FR-3.3 | Buffer pixel data in FIFO | Critical |
| FR-3.4 | Report FIFO status (empty/full) | High |
| FR-3.5 | Support test pattern mode | Medium |

### FR-4: Dummy Scan (L2 Idle)
The FPGA SHALL execute periodic dummy scan during L2 idle mode.

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-4.1 | Support configurable dummy scan period (30-60 sec) | High |
| FR-4.2 | Reset all storage nodes sequentially | High |
| FR-4.3 | No ADC readout during dummy scan | High |
| FR-4.4 | Complete dummy scan within ~2 ms | Medium |
| FR-4.5 | Report dummy scan status via SPI | Medium |

### FR-5: SPI Slave Interface
The FPGA SHALL provide SPI slave interface for MCU communication.

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-5.1 | Support SPI Mode 0 (CPOL=0, CPHA=0) | Critical |
| FR-5.2 | Operate at up to 10 MHz clock | Critical |
| FR-5.3 | Support register read/write | Critical |
| FR-5.4 | Support burst read | High |
| FR-5.5 | Provide 8-bit data width | High |

### FR-6: Status and Interrupt
The FPGA SHALL report status and generate interrupts.

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-6.1 | Report frame busy status | Critical |
| FR-6.2 | Report FIFO status | High |
| FR-6.3 | Generate frame complete interrupt | High |
| FR-6.4 | Generate dummy scan complete interrupt | Medium |
| FR-6.5 | Generate error interrupt | Medium |

---

## 2. Performance Requirements

### PR-1: Timing Accuracy
| Requirement | Value | Tolerance |
|-------------|-------|-----------|
| Row clock frequency | 5 MHz | ±1% |
| Column clock frequency | 10 MHz | ±1% |
| Reset pulse width | 1 µs | ±10% |
| Integration time | Configurable | ±1% |

### PR-2: Response Time
| Requirement | Max Time |
|-------------|----------|
| Bias mode switch | 10 µs |
| SPI register access | 2 µs |
| Interrupt assertion | 1 µs |

### PR-3: Throughput
| Requirement | Value |
|-------------|-------|
| Full frame readout | < 250 ms |
| Row read time | 50 µs |
| FIFO depth | 2048 pixels |

### PR-4: Resource Limits
| Requirement | Target |
|-------------|--------|
| Slice LUTs | < 15,000 (72%) |
| Slice Registers | < 20,000 (48%) |
| BRAMs | < 25 (25%) |
| MMCM/PLL | ≤ 2 |

---

## 3. Interface Requirements

### IR-1: SPI Interface
| Parameter | Value |
|-----------|-------|
| Mode | Mode 0 (CPOL=0, CPHA=0) |
| Clock Frequency | Max 10 MHz |
| Data Width | 8 bits |
| Byte Order | MSB first |
| Chip Select | Active low |

### IR-2: ADC Interface
| Parameter | Value |
|-----------|-------|
| Data Width | 14 bits |
| Clock Frequency | Up to 20 MHz |
| Protocol | Manufacturer-specific (to be defined) |

### IR-3: Bias Control Interface
| Parameter | Value |
|-----------|-------|
| Mode Select | 2 bits |
| Update Request | 1 bit |
| Acknowledge | 1 bit |
| Switching Time | < 10 µs |

### IR-4: Panel Interface
| Signal | Description |
|--------|-------------|
| ROW_ADDR[11:0] | Row address (0-2047) |
| COL_ADDR[11:0] | Column address (0-2047) |
| ROW_CLK_EN | Row clock enable |
| COL_CLK_EN | Column clock enable |
| GATE_SEL | Gate select signal |
| RESET_PULSE | Reset pulse to storage node |

---

## 4. Constraint Requirements

### CR-1: Clock Domain
- Single main clock domain: 100 MHz
- Generated clocks via PLL: ADC (20 MHz), Row (1-10 MHz), Col (1-10 MHz)
- All clock domain crossings must be synchronized

### CR-2: Reset Strategy
- Asynchronous active-low reset
- Reset assertion time: ≥ 100 ns
- All registers initialize to known state

### CR-3: Power
- Target power consumption: < 2W
- Support clock gating for L3 sleep mode

### CR-4: Environmental
| Parameter | Min | Typical | Max |
|-----------|-----|---------|-----|
| Operating Temperature | 0°C | 25°C | 70°C |
| Supply Voltage (1.0V) | 0.95V | 1.0V | 1.05V |
| Supply Voltage (1.8V) | 1.71V | 1.8V | 1.89V |
| Supply Voltage (3.3V) | 3.14V | 3.3V | 3.47V |

---

## 5. Quality Requirements

### QR-1: Reliability
- Mean Time Between Failures (MTBF): > 10,000 hours
- No single point of failure in timing generation

### QR-2: Testability
- All registers readable/writeable via SPI
- Test pattern mode for validation
- Observable internal state for debug

### QR-3: Maintainability
- Clear module hierarchy
- Documented timing constraints
- Synthesis-friendly coding style

---

## 6. Safety Requirements

### SR-1: Fault Detection
- Detect FIFO overflow/underflow
- Detect bias switch timeout
- Detect ADC fault

### SR-2: Fault Recovery
- Report errors via SPI
- Soft reset via SPI command
- Automatic recovery for transient faults

### SR-3: Safe State
- On fault: Enter L1 idle mode
- On power loss: Preserve no state (no battery)

---

## 7. Compliance Requirements

### CM-1: Standards
- IEC 60601-1-2 (Medical electrical equipment)
- FCC Part 15 (EMC)

### CM-2: Coding Standards
- SystemVerilog IEEE 1800-2017
| Style | Requirement |
|-------|-------------|
| Naming | snake_case for signals, PascalCase for modules |
| Reset | Active-low async reset |
| Latches | No latches (use flip-flops only) |

---

## 8. Verification Requirements

### VR-1: Simulation
- Code coverage: > 90%
- Functional coverage: 100% of requirements
- Regression testing for all releases

### VR-2: Timing Analysis
- All paths meet timing
- No unconstrained paths
- Maximum clock frequency: 100 MHz

### VR-3: Synthesis
- Clean synthesis (no critical warnings)
- Resource utilization within budget
- Power analysis complete

---

## 9. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial requirements |

---

## 10. Requirements Traceability

| Requirement | Design Element | Test Case |
|-------------|----------------|-----------|
| FR-1.1 | timing_generator/row_counter | TB_TIMING_001 |
| FR-2.1 | bias_mux_controller | TB_BIAS_001 |
| FR-3.1 | adc_controller | TB_ADC_001 |
| FR-4.1 | dummy_scan_engine/timer | TB_DUMMY_001 |
| FR-5.1 | spi_slave_interface | TB_SPI_001 |

---

**End of Document**
