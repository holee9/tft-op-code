# SPI Register Map Specification
## FPGA-MCU Communication Interface

**Version**: 1.0
**Created**: 2026-02-10
**Interface**: SPI Slave (Mode 0, up to 10 MHz)

---

## 1. Overview

This document defines the complete register map for FPGA-MCU SPI communication, including control registers, status registers, and configuration parameters for a-Si TFT FPD panel control.

### 1.1 Communication Protocol

| Parameter | Value |
|-----------|-------|
| SPI Mode | Mode 0 (CPOL=0, CPHA=0) |
| Max Clock Frequency | 10 MHz |
| Data Width | 8 bits per transfer |
| Byte Order | MSB first |
| Chip Select | Active low |

---

## 2. Register Map Summary

| Address | Name | Access | Bit Width | Reset Value | Description |
|---------|------|--------|-----------|-------------|-------------|
| 0x00 | CTRL_REG | RW | 8 | 0x00 | Control Register |
| 0x01 | STATUS_REG | RO | 8 | 0x01 | Status Register |
| 0x02 | BIAS_SELECT | RW | 2 | 0x00 | Bias Mode Selection |
| 0x03 | DUMMY_PERIOD_L | RW | 8 | 0x3C | Dummy Scan Period Low Byte |
| 0x04 | DUMMY_PERIOD_H | RW | 8 | 0x00 | Dummy Scan Period High Byte |
| 0x05 | DUMMY_CONTROL | RW | 8 | 0x00 | Dummy Scan Control |
| 0x06 | ROW_START_L | RW | 8 | 0x00 | ROI Start Row Low Byte |
| 0x07 | ROW_START_H | RW | 8 | 0x00 | ROI Start Row High Byte |
| 0x08 | ROW_END_L | RW | 8 | 0xFF | ROI End Row Low Byte |
| 0x09 | ROW_END_H | RW | 8 | 0x07 | ROI End Row High Byte |
| 0x0A | COL_START_L | RW | 8 | 0x00 | ROI Start Column Low Byte |
| 0x0B | COL_START_H | RW | 8 | 0x00 | ROI Start Column High Byte |
| 0x0C | COL_END_L | RW | 8 | 0xFF | ROI End Column Low Byte |
| 0x0D | COL_END_H | RW | 8 | 0x07 | ROI End Column High Byte |
| 0x0E | INTEGRATION_TIME_L | RW | 8 | 0x64 | Integration Time Low Byte (ms) |
| 0x0F | INTEGRATION_TIME_H | RW | 8 | 0x00 | Integration Time High Byte (ms) |
| 0x10 | TIMING_CONFIG_0 | RW | 8 | 0x0A | Row Clock Divider |
| 0x11 | TIMING_CONFIG_1 | RW | 8 | 0x05 | Column Clock Divider |
| 0x12 | TIMING_CONFIG_2 | RW | 8 | 0x01 | Read Mode Select |
| 0x13 | TIMING_CONFIG_3 | RW | 8 | 0x00 | Timing Adjustments |
| 0x14 | INTERRUPT_MASK | RW | 8 | 0x00 | Interrupt Enable Mask |
| 0x15 | INTERRUPT_STATUS | RO/W1C | 8 | 0x00 | Interrupt Status |
| 0x16 | ERROR_CODE | RO | 8 | 0x00 | Error Status |
| 0x17 | FIRMWARE_VERSION | RO | 8 | 0x10 | Firmware Version (BCD) |
| 0x18 - 0x1F | RESERVED | - | - | - | Reserved |
| 0x20 - 0x3F | TEST_PATTERN | RW | 256 | 0x00 | Test Pattern Memory |

---

## 3. Register Definitions

### 3.1 Control Register (0x00) - CTRL_REG

```
Bit 7: FRAME_START
  0 = No action
  1 = Start frame capture (self-clearing)

Bit 6: FRAME_RESET
  0 = No action
  1 = Reset frame capture logic (self-clearing)

Bit 5: DUMMY_ENABLE
  0 = Disable dummy scan
  1 = Enable dummy scan mode

Bit 4: ADC_ENABLE
  0 = Disable ADC
  1 = Enable ADC

Bit 3: TEST_MODE
  0 = Normal operation
  1 = Test pattern mode

Bit 2: BIAS_UPDATE_PENDING
  0 = Bias mode stable
  1 = Bias update in progress (read-only)

Bit 1: FIFO_RESET
  0 = No action
  1 = Reset data FIFO (self-clearing)

Bit 0: SOFT_RESET
  0 = No action
  1 = Soft reset all modules except PLL (self-clearing)
```

### 3.2 Status Register (0x01) - STATUS_REG

```
Bit 7: FRAME_BUSY
  0 = Frame capture idle
  1 = Frame capture in progress

Bit 6: DUMMY_ACTIVE
  0 = Dummy scan inactive
  1 = Dummy scan in progress

Bit 5: BIAS_MODE_READY
  0 = Bias switching in progress
  1 = Bias mode settled and ready

Bit 4: FIFO_EMPTY
  0 = FIFO has data
  1 = FIFO empty

Bit 3: FIFO_FULL
  0 = FIFO not full
  1 = FIFO full (data may be lost)

Bit 2: ADC_READY
  0 = ADC busy
  1 = ADC ready for readout

Bit 1: IDLE_STATE
  0 = Active mode
  1 = Idle mode (no frame capture)

Bit 0: POWER_GOOD
  0 = Power supply fault
  1 = Power supply OK
```

### 3.3 Bias Select Register (0x02) - BIAS_SELECT

```
Bits [1:0]: BIAS_MODE
  00 = NORMAL_BIAS
      V_PD = -1.5 V
      V_COL = -1.0 V
      Usage: Normal readout operation

  01 = IDLE_LOW_BIAS
      V_PD = -0.2 V
      V_COL = -0.2 V
      Usage: L2 idle mode (minimize leakage)

  10 = SLEEP_BIAS
      V_PD = 0 V
      V_COL = 0 V
      Usage: L3 deep sleep

  11 = RESERVED (do not use)
```

### 3.4 Dummy Scan Period (0x03-0x04) - DUMMY_PERIOD

16-bit value representing dummy scan period in seconds.

```
Register 0x03: DUMMY_PERIOD_L (Bits [7:0])
Register 0x04: DUMMY_PERIOD_H (Bits [15:8])

Default: 0x003C = 60 seconds
Range: 0x0001 to 0xFFFF (1 to 65535 seconds)
```

### 3.5 Dummy Scan Control (0x05) - DUMMY_CONTROL

```
Bit 7: DUMMY_AUTO_MODE
  0 = Manual trigger only
  1 = Automatic periodic mode

Bit 6: DUMMY_TRIGGER
  0 = No action
  1 = Trigger single dummy scan (self-clearing)

Bit 5: RESET_ONLY
  0 = Reset + Read
  1 = Reset only (no ADC readout)

Bit 4: SCAN_ALL_ROWS
  0 = ROI rows only
  1 = All rows (0 to 2047)

Bits [3:0]: DUMMY_ROW_DELAY
  Delay between row resets in microseconds × 10
  Default: 0x0A = 100 µs delay
```

### 3.6 ROI Configuration (0x06-0x0D)

#### Row Start (0x06-0x07)
```
16-bit value: First row to read (0 to 2047)
Default: 0x0000
```

#### Row End (0x08-0x09)
```
16-bit value: Last row to read (0 to 2047)
Default: 0x07FF = 2047
```

#### Column Start (0x0A-0x0B)
```
16-bit value: First column to read (0 to 2047)
Default: 0x0000
```

#### Column End (0x0C-0x0D)
```
16-bit value: Last column to read (0 to 2047)
Default: 0x07FF = 2047
```

### 3.7 Integration Time (0x0E-0x0F) - INTEGRATION_TIME

16-bit value representing integration time in milliseconds.

```
Register 0x0E: INTEGRATION_TIME_L (Bits [7:0])
Register 0x0F: INTEGRATION_TIME_H (Bits [15:8])

Default: 0x0064 = 100 ms
Range: 0x0001 to 0xFFFF (1 to 65535 ms)
```

### 3.8 Timing Configuration (0x10-0x13)

#### Row Clock Divider (0x10) - TIMING_CONFIG_0
```
Bits [7:0]: ROW_CLK_DIV
  Row clock = 100 MHz / (2 × ROW_CLK_DIV)
  Default: 0x0A = 5 MHz row clock
```

#### Column Clock Divider (0x11) - TIMING_CONFIG_1
```
Bits [7:0]: COL_CLK_DIV
  Column clock = 100 MHz / (2 × COL_CLK_DIV)
  Default: 0x05 = 10 MHz column clock
```

#### Read Mode Select (0x12) - TIMING_CONFIG_2
```
Bits [1:0]: READ_MODE
  00 = Normal read (sequential)
  01 = Interlaced read
  10 = Skipping read (every Nth pixel)
  11 = Binning mode (2×2)

Bits [7:2]: BINNING_FACTOR (when READ_MODE = 11)
  N = 2 to 8 (2×2 to 8×8 binning)
```

#### Timing Adjustments (0x13) - TIMING_CONFIG_3
```
Bits [3:0]: RESET_PULSE_WIDTH
  Reset pulse width in 10 ns units
  Default: 0x64 = 1 µs

Bits [7:4]: SETTLE_DELAY_ADJ
  Settle time adjustment in 10 µs units
  Default: 0x00 = no adjustment
```

### 3.9 Interrupt Registers (0x14-0x15)

#### Interrupt Mask (0x14) - INTERRUPT_MASK
```
Bit 7: FRAME_COMPLETE_EN
  0 = Disabled
  1 = Enable frame complete interrupt

Bit 6: DUMMY_COMPLETE_EN
  0 = Disabled
  1 = Enable dummy scan complete interrupt

Bit 5: BIAS_READY_EN
  0 = Disabled
  1 = Enable bias ready interrupt

Bit 4: FIFO_THRESHOLD_EN
  0 = Disabled
  1 = Enable FIFO threshold interrupt

Bit 3: ERROR_EN
  0 = Disabled
  1 = Enable error interrupt

Bits [2:0]: Reserved
```

#### Interrupt Status (0x15) - INTERRUPT_STATUS (Read/Clear)

```
Bit 7: FRAME_COMPLETE
  0 = No interrupt
  1 = Frame capture complete (write 1 to clear)

Bit 6: DUMMY_COMPLETE
  0 = No interrupt
  1 = Dummy scan complete (write 1 to clear)

Bit 5: BIAS_READY
  0 = No interrupt
  1 = Bias mode ready (write 1 to clear)

Bit 4: FIFO_THRESHOLD
  0 = No interrupt
  1 = FIFO reached threshold (write 1 to clear)

Bit 3: ERROR
  0 = No interrupt
  1 = Error occurred (write 1 to clear)

Bits [2:0]: Reserved
```

### 3.10 Error Code Register (0x16) - ERROR_CODE

```
Bits [7:0]: ERROR_CODE
  0x00 = No error
  0x01 = Timeout error
  0x02 = FIFO overflow
  0x03 = FIFO underflow
  0x04 = Bias switch timeout
  0x05 = ADC fault
  0x06 = Invalid register value
  0x07 = SPI communication error
  0x08 - 0xFF = Reserved
```

### 3.11 Firmware Version (0x17) - FIRMWARE_VERSION

```
Bits [7:4]: MAJOR_VERSION
Bits [3:0]: MINOR_VERSION

Example: 0x10 = Version 1.0
```

---

## 4. SPI Transaction Formats

### 4.1 Write Transaction

```
Sequence:
1. Master asserts CS_N LOW
2. Master sends: [CMD_BYTE] [ADDR_BYTE] [DATA_BYTE]
3. Master deasserts CS_N HIGH

CMD_BYTE = 0x01 (WRITE command)
ADDR_BYTE = Register address (0x00 to 0x3F)
DATA_BYTE = Data to write

Example: Write BIAS_SELECT to IDLE_LOW_BIAS
CS_N LOW → 0x01 0x02 0x01 → CS_N HIGH
```

### 4.2 Read Transaction

```
Sequence:
1. Master asserts CS_N LOW
2. Master sends: [CMD_BYTE] [ADDR_BYTE]
3. Master sends dummy byte (0x00)
4. Slave returns data while master sends dummy
5. Master deasserts CS_N HIGH

CMD_BYTE = 0x02 (READ command)
ADDR_BYTE = Register address (0x00 to 0x3F)

Example: Read STATUS_REG
CS_N LOW → 0x02 0x01 0x00 → [DATA] → CS_N HIGH
```

### 4.3 Burst Read Transaction

```
Sequence:
1. Master asserts CS_N LOW
2. Master sends: [CMD_BYTE] [START_ADDR]
3. Master sends dummy bytes, slave returns sequential data
4. Master deasserts CS_N HIGH

CMD_BYTE = 0x03 (BURST_READ command)
START_ADDR = Starting register address

Example: Read registers 0x00 to 0x03
CS_N LOW → 0x03 0x00 0x00 0x00 0x00 0x00 → [0x00] [0x01] [0x02] [0x03] → CS_N HIGH
```

---

## 5. Default State Values

### 5.1 Power-On Default Configuration

```
Register:Value (Hex)
────────────────────────────────────
CTRL_REG:        0x00
STATUS_REG:      0x01 (POWER_GOOD=1)
BIAS_SELECT:     0x00 (NORMAL_BIAS)
DUMMY_PERIOD:    0x003C (60 sec)
ROW_START:       0x0000
ROW_END:         0x07FF (2047)
COL_START:       0x0000
COL_END:         0x07FF (2047)
INTEGRATION_TIME: 0x0064 (100 ms)
TIMING_CONFIG_0: 0x0A (5 MHz row)
TIMING_CONFIG_1: 0x05 (10 MHz col)
TIMING_CONFIG_2: 0x00 (Normal read)
TIMING_CONFIG_3: 0x00 (No adjust)
INTERRUPT_MASK:  0x00 (All disabled)
```

### 5.2 Idle Mode Configurations

#### L1 Normal Idle Mode
```
BIAS_SELECT:     0x00 (NORMAL_BIAS)
DUMMY_CONTROL:   0x00 (Disabled)
TIMING_CONFIG_0: 0x0A
TIMING_CONFIG_1: 0x05
```

#### L2 Low-Bias Idle Mode
```
BIAS_SELECT:     0x01 (IDLE_LOW_BIAS)
DUMMY_PERIOD:    0x003C (60 sec)
DUMMY_CONTROL:   0x81 (Auto mode, reset only, all rows)
TIMING_CONFIG_0: 0x0A
TIMING_CONFIG_1: 0x05
```

#### L3 Deep Sleep Mode
```
BIAS_SELECT:     0x02 (SLEEP_BIAS)
DUMMY_CONTROL:   0x00 (Disabled)
All clocks:      Gated
ADC:             Disabled
```

---

## 6. Timing Diagrams

### 6.1 Bias Mode Switch Sequence

```
Master                  FPGA
│                       │
│ Write BIAS_SELECT     │──► [Store new bias mode]
│ [01]                  │     [Start bias switch]
│                       │     [Set BIAS_UPDATE_PENDING=1]
│                       │
│ Poll STATUS_REG       │◄─── [BIAS_MODE_READY=0]
│                       │     [Switching in progress]
│                       │
│ ... < 10 µs ...       │     [DAC settling]
│                       │
│ Poll STATUS_REG       │◄─── [BIAS_MODE_READY=1]
│                       │     [Switch complete]
│                       │
```

### 6.2 Frame Capture Sequence

```
Master                  FPGA
│                       │
│ Write CTRL_REG        │──► [Store FRAME_START=1]
│ [80]                  │     [Start frame capture]
│                       │     [Set FRAME_BUSY=1]
│                       │
│ Poll STATUS_REG       │◄─── [FRAME_BUSY=1]
│                       │     [Capture in progress]
│                       │
│ ... ~250 ms ...       │     [Reading 2048 rows]
│                       │
│ Poll STATUS_REG       │◄─── [FRAME_BUSY=0]
│                       │     [Frame complete]
│                       │     [Set FRAME_COMPLETE int]
│                       │
│ Read FIFO data        │──► [Get pixel data]
│                       │
```

---

## 7. Error Handling

### 7.1 Error Detection

| Error | Detection Method | Recovery Action |
|-------|------------------|-----------------|
| FIFO Overflow | FIFO_FULL bit + ERROR_CODE | FIFO_RESET command |
| Timeout | Watchdog timer | SOFT_RESET command |
| Bias Switch Fault | BIAS_MODE_READY timeout | Retry or error report |
| ADC Fault | ADC_READY stuck low | ADC_ENABLE toggle |
| SPI Protocol | CRC/parity check | Resend transaction |

### 7.2 Error Recovery Flowchart

```
Error Detected
      │
      ▼
Set ERROR_CODE register
      │
      ▼
Set ERROR interrupt
      │
      ▼
Wait for MCU acknowledge
      │
      ▼
┌─────────────────────┐
│ Is error recoverable?│
└─────────────────────┘
      │YES    │NO
      ▼       ▼
Auto-continue  Report to Host
```

---

## 8. Register Access Timing

| Parameter | Min | Typical | Max | Unit |
|-----------|-----|---------|-----|------|
| CS_N setup time | 10 | 20 | - | ns |
| CS_N hold time | 10 | 20 | - | ns |
| SCK to data valid | - | 50 | 100 | ns |
| Register write time | - | 1 | 2 | µs |
| Bias switch time | 1 | 5 | 10 | µs |
| Mode switch time | - | 100 | - | µs |

---

## 9. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial register map definition |

---

**End of Specification**
