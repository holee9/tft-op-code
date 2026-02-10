# Acceptance Criteria

## FPGA Team Delivery Package

**Document**: 05_acceptance_criteria.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Overview

This document defines the acceptance criteria for the FPGA implementation. All criteria must be met before the FPGA design is considered complete and ready for integration.

---

## 2. Functional Requirements Validation

### FR-1: Timing Generation

| ID | Test Case | Description | Pass Criteria |
|----|-----------|-------------|---------------|
| FR-1.1 | TB_TIM_001 | Sequential row address generation | Row address increments 0→2047 sequentially |
| FR-1.2 | TB_TIM_002 | Column address generation | Column address completes full range per row |
| FR-1.3 | TB_TIM_003 | Row clock frequency | Measured frequency: 5 MHz ±1% |
| FR-1.4 | TB_TIM_004 | Column clock frequency | Measured frequency: 10 MHz ±1% |
| FR-1.5 | TB_TIM_005 | ROI readout | Configurable start/end rows work correctly |
| FR-1.6 | TB_TIM_006 | Integration time | Configurable 1-65535 ms, ±1% accuracy |

### FR-2: Bias Control

| ID | Test Case | Description | Pass Criteria |
|----|-----------|-------------|---------------|
| FR-2.1 | TB_BIAS_001 | NORMAL_BIAS mode | V_PD=-1.5V, V_COL=-1.0V selected |
| FR-2.2 | TB_BIAS_002 | IDLE_LOW_BIAS mode | V_PD=-0.2V, V_COL=-0.2V selected |
| FR-2.3 | TB_BIAS_003 | SLEEP_BIAS mode | V_PD=0V, V_COL=0V selected |
| FR-2.4 | TB_BIAS_004 | Switching time | All mode switches < 10 µs |
| FR-2.5 | TB_BIAS_005 | Glitch-free switching | No glitches observed on bias outputs |
| FR-2.6 | TB_BIAS_006 | Status reporting | BIAS_MODE_READY bit set correctly |

### FR-3: Data Acquisition

| ID | Test Case | Description | Pass Criteria |
|----|-----------|-------------|---------------|
| FR-3.1 | TB_ADC_001 | 14-bit ADC interface | All 14 bits captured correctly |
| FR-3.2 | TB_ADC_002 | ADC clock generation | 20 MHz clock ±1% |
| FR-3.3 | TB_FIFO_001 | FIFO operation | No overflow/underflow in normal operation |
| FR-3.4 | TB_FIFO_002 | FIFO status flags | EMPTY and FULL flags assert correctly |
| FR-3.5 | TB_TEST_001 | Test pattern mode | Incrementing pattern output correctly |

### FR-4: Dummy Scan

| ID | Test Case | Description | Pass Criteria |
|----|-----------|-------------|---------------|
| FR-4.1 | TB_DUMMY_001 | Configurable period | 30-60 sec period configurable |
| FR-4.2 | TB_DUMMY_002 | Row reset sequence | All 2048 rows reset sequentially |
| FR-4.3 | TB_DUMMY_003 | No ADC readout | No data sent during dummy scan |
| FR-4.4 | TB_DUMMY_004 | Scan duration | Complete scan < 2.5 ms |
| FR-4.5 | TB_DUMMY_005 | Status reporting | DUMMY_ACTIVE flag asserts correctly |

### FR-5: SPI Interface

| ID | Test Case | Description | Pass Criteria |
|----|-----------|-------------|---------------|
| FR-5.1 | TB_SPI_001 | Mode 0 operation | Correct CPOL=0, CPHA=0 behavior |
| FR-5.2 | TB_SPI_002 | 10 MHz operation | Clean operation at 10 MHz |
| FR-5.3 | TB_SPI_003 | Register write | All registers write correctly |
| FR-5.4 | TB_SPI_004 | Register read | All registers read back correctly |
| FR-5.5 | TB_SPI_005 | Burst read | Sequential read works correctly |

### FR-6: Status and Interrupt

| ID | Test Case | Description | Pass Criteria |
|----|-----------|-------------|---------------|
| FR-6.1 | TB_STATUS_001 | Frame busy flag | Asserts during capture, deasserts after |
| FR-6.2 | TB_STATUS_002 | FIFO status flags | Correct status at all FIFO levels |
| FR-6.3 | TB_INT_001 | Frame complete interrupt | Asserts at frame end |
| FR-6.4 | TB_INT_002 | Dummy complete interrupt | Asserts after dummy scan |
| FR-6.5 | TB_INT_003 | Error interrupt | Asserts on fault conditions |

---

## 3. Performance Requirements Validation

### PR-1: Timing Accuracy

| Metric | Target | Measurement Method | Pass |
|--------|--------|-------------------|------|
| Row clock frequency | 5 MHz ±1% | Frequency counter | ✓ |
| Column clock frequency | 10 MHz ±1% | Frequency counter | ✓ |
| Reset pulse width | 1 µs ±10% | Oscilloscope | ✓ |
| Integration time | Configurable ±1% | Timer measurement | ✓ |

### PR-2: Response Time

| Metric | Max | Measurement Method | Pass |
|--------|-----|-------------------|------|
| Bias mode switch | 10 µs | Oscilloscope | ✓ |
| SPI register access | 2 µs | Logic analyzer | ✓ |
| Interrupt assertion | 1 µs | Logic analyzer | ✓ |

### PR-3: Throughput

| Metric | Target | Measurement Method | Pass |
|--------|--------|-------------------|------|
| Full frame readout | < 250 ms | Timer | ✓ |
| Row read time | 50 µs | Oscilloscope | ✓ |
| FIFO depth | 2048 pixels | Simulation | ✓ |

### PR-4: Resource Limits

| Resource | Target | Actual | Pass |
|----------|--------|--------|------|
| Slice LUTs | < 15,000 (72%) | ___ | TBD |
| Slice Registers | < 20,000 (48%) | ___ | TBD |
| BRAMs | < 25 (25%) | ___ | TBD |
| MMCM/PLL | ≤ 2 | ___ | TBD |

---

## 4. Test Procedures

### 4.1 Unit Tests

Each module must have dedicated unit testbench:

| Module | Testbench | Coverage Target |
|--------|-----------|-----------------|
| `timing_generator` | `tb_timing_generator` | 95% |
| `bias_mux_controller` | `tb_bias_mux` | 95% |
| `adc_controller` | `tb_adc_controller` | 95% |
| `dummy_scan_engine` | `tb_dummy_scan` | 95% |
| `spi_slave_interface` | `tb_spi_slave` | 95% |
| `data_fifo` | `tb_fifo` | 90% |

### 4.2 Integration Tests

| Test | Description | Pass Criteria |
|------|-------------|---------------|
| IT_001 | Full frame capture | Complete frame captured without error |
| IT_002 | Mode transition | L1→L2→L3 transitions work |
| IT_003 | ROI readout | Partial region read correctly |
| IT_004 | Dummy scan cycle | Complete dummy scan executes |
| IT_005 | SPI stress test | 1000+ read/write operations pass |

### 4.3 System Tests

| Test | Description | Pass Criteria |
|------|-------------|---------------|
| ST_001 | Power-on reset | All registers initialize correctly |
| ST_002 | Continuous operation | 100+ frames captured without error |
| ST_003 | Error recovery | System recovers from fault conditions |
| ST_004 | Thermal compliance | No thermal violations at 70°C ambient |

---

## 5. Simulation Requirements

### 5.1 Coverage Requirements

| Coverage Type | Target | Tool |
|---------------|--------|------|
| Code Coverage | 90% | Questa/ModelSim |
| Functional Coverage | 100% | Custom scoreboard |
| Transition Coverage | 85% | Questa/ModelSim |

### 5.2 Simulation Test List

```
Required Simulations:
├── 01_power_on_reset
├── 02_spi_register_access
├── 03_normal_frame_capture
├── 04_roi_readout
├── 05_bias_mode_switching
├── 06_dummy_scan_execution
├── 07_fifo_overflow_recovery
├── 08_interrupt_generation
├── 09_integration_time_config
└── 10_stress_test
```

---

## 6. Synthesis Requirements

### 6.1 Synthesis Results

| Metric | Requirement | Acceptance |
|--------|-------------|------------|
| Timing constraints | All met | ✓ No WNS/TNS violations |
| Resource utilization | Within budget | ✓ See PR-4 table |
| Power consumption | < 2W | ✓ XPower analysis |
| Critical warnings | Zero | ✓ Clean run |

### 6.2 Timing Analysis

```
Required Timing Reports:
├── Setup time analysis (all paths)
├── Hold time analysis (all paths)
├── Clock skew analysis
└── Path delay report (top 100)
```

---

## 7. Deliverables Checklist

### 7.1 Source Code

- [ ] SystemVerilog RTL source files
- [ ] Top-level module with all interfaces
- [ ] XDC constraint file
- [ ] Synthesis script (Tcl)
- [ ] Simulation testbenches
- [ ] Makefile/build script

### 7.2 Documentation

- [ ] Module hierarchy document
- [ ] Timing analysis report
- [ ] Resource utilization report
- [ ] Power analysis report
- [ ] Known issues/limitations

### 7.3 Test Results

- [ ] Simulation log files
- [ ] Coverage reports
- [ ] Synthesis reports
- [ ] Timing reports
- [ ] Test summary

---

## 8. Sign-Off Criteria

The FPGA design is considered complete when:

1. [ ] All functional requirements (FR) pass tests
2. [ ] All performance requirements (PR) met
3. [ ] Simulation coverage ≥ 90%
4. [ ] Synthesis timing clean (no violations)
5. [ ] Resource utilization within budget
6. [ ] All deliverables submitted
7. [ ] Code review completed
8. [ ] Integration testing passed

---

## 9. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial acceptance criteria |

---

## 10. Test Report Template

```markdown
## FPGA Test Report

**DUT**: fpga_panel_controller
**Date**: YYYY-MM-DD
**Tester**: Name
**Build**: Version X.Y

### Test Summary

| Category | Run | Passed | Failed | Blocked |
|----------|-----|--------|--------|---------|
| Unit Tests | 42 | 42 | 0 | 0 |
| Integration Tests | 5 | 5 | 0 | 0 |
| System Tests | 4 | 4 | 0 | 0 |
| **Total** | **51** | **51** | **0** | **0** |

### Coverage Summary

| Type | Coverage | Target | Status |
|------|----------|--------|--------|
| Code | 92.3% | 90% | ✓ |
| Functional | 100% | 100% | ✓ |
| Transition | 87.1% | 85% | ✓ |

### Resource Utilization

| Resource | Used | Available | Utilization |
|----------|------|-----------|-------------|
| Slice LUTs | 14,523 | 20,800 | 69.8% |
| Slice Registers | 9,876 | 41,600 | 23.7% |
| BRAMs | 18 | 100 | 18.0% |
| DSP48E1 | 4 | 80 | 5.0% |

### Timing Summary

| Clock | Frequency | WNS | TNS |
|-------|-----------|-----|-----|
| clk_100mhz | 100 MHz | 0.5 | 0 |
| clk_adc | 20 MHz | 1.2 | 0 |
| clk_row | 5 MHz | 2.5 | 0 |
| clk_col | 10 MHz | 1.8 | 0 |

### Sign-Off

[ ] Approved for integration
[ ] Requires rework
[ ] Failed - see issues

**Signature**: _________________
**Date**: _________________
```

---

**End of Document**
