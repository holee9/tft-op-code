# Control Sequence Flows Specification
## State Machine and Protocol Flow Definitions

**Version**: 1.0
**Created**: 2026-02-10

---

## 1. Overview

This document defines the complete control flow sequences for the a-Si TFT FPD system, including initialization, normal operation, idle mode transitions, and recovery procedures.

### 1.1 Flow Diagram Conventions

```
Notation:
┌───────┐  State/Process rectangle
│ ROUND │  Decision diamond
│       │  Connector/Arrow
└───────┘
→  Direction of flow
⇒  Immediate transition
⇏  Delayed/conditional transition
```

---

## 2. System Initialization Flow

### 2.1 Power-On Sequence

```
┌─────────────────────────────────────────────────────────────────────┐
│                     POWER-ON RESET SEQUENCE                         │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                        ┌───────────────┐
                        │ Power Applied │
                        │   (3.3V, 1.8V)│
                        └───────┬───────┘
                                │
                                ▼
                        ┌───────────────┐
                        │ Wait 100ms    │
                        │ Clock Stable   │
                        └───────┬───────┘
                                │
                                ▼
        ┌───────────────────────────────────────────┐
        │                                           │
        ▼                                           ▼
┌───────────────┐                           ┌───────────────┐
│ FPGA Reset    │                           │ MCU Boot      │
│ - Load Bitstream│                           │ - Linux Boot   │
│ - Initialize PLL│                           │ - .NET Start   │
│ - Set Default Regs│                          │ - Load Config  │
└───────┬───────┘                           └───────┬───────┘
        │                                             │
        └─────────────────────┬───────────────────────┘
                              │
                              ▼
                        ┌───────────────┐
                        │ SPI Handshake │
                        │ - Read VERSION│
                        │ - Verify Comm │
                        └───────┬───────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
            ┌───────────────┐       ┌───────────────┐
            │  Comm OK?     │       │ Temp Sensor   │
            │      │        │       │ Calibration   │
            │   YES │ NO    │       └───────────────┘
            │      │  │             │
            │      ▼  ▼             │
            │   ┌─────┐             │
            │   │ERROR│             │
            │   └─────┘             │
            │      │                │
            └──────┼────────────────┘
                   │
                   ▼
            ┌───────────────┐
            │ Enter L1 Mode │
            │ (Normal Idle) │
            └───────────────┘
```

### 2.2 Initialization Register Sequence

```
Step  Register    Value   Description
────────────────────────────────────────────
1     0x00        0x01    SOFT_RESET
2     Wait        10ms    Reset settling
3     0x00        0x10    ADC_ENABLE
4     0x02        0x00    NORMAL_BIAS
5     0x03-0x04   0x003C  DUMMY_PERIOD = 60 sec
6     0x05        0x00    DUMMY disable
7     0x06-0x0D   ROI     Full panel (0-2047)
8     0x0E-0x0F   0x0064  INTEGRATION = 100ms
9     0x10        0x0A    ROW_CLK_DIV
10    0x11        0x05    COL_CLK_DIV
11    0x14        0x80    Enable FRAME_COMPLETE int
12    Verify STATUS_REG[0] = 1  (POWER_GOOD)
```

---

## 3. Idle Mode State Machine

### 3.1 Complete State Diagram

```
                          ┌────────────────┐
                          │   POWER OFF    │
                          └────────────────┘
                                   │
                                   │ Power On
                                   ▼
                          ┌────────────────┐
                          │ INITIALIZATION │
                          └───────┬────────┘
                                  │
                                  │ Init Complete
                                  ▼
        ┌─────────────────────────────────────────────────┐
        │                   L1 NORMAL IDLE                │
        │  ┌───────────────────────────────────────────┐ │
        │  │ Bias: NORMAL (-1.5V, -1.0V)               │ │
        │  │ Dummy Scan: DISABLED                       │ │
        │  │ Timeout: t_idle > 10 min OR t_idle > t_max │ │
        │  └───────────────────────────────────────────┘ │
        └───────────────────────┬─────────────────────────┘
                                │
                ┌───────────────┴───────────────┐
                │                               │
                │ t_idle < 10 min                │ t_idle > 10 min
                │ AND                           │ OR
                │ No capture request             │ t_idle > t_max
                │                               │
                │                               ▼
                │                    ┌─────────────────────┐
                │                    │   L2 LOW-BIAS IDLE  │
                │  ┌───────────────────────────────────┐│
                │  │ Bias: IDLE_LOW (-0.2V, -0.2V)    ││
                │  │ Dummy Scan: ENABLED (60s period) ││
                │  │ Timeout: t_idle > 100 min        ││
                │  └───────────────────────────────────┘│
                │                    └─────────┬───────────┘
                │                              │
                │              ┌───────────────┴───────────┐
                │              │                           │
                │              │ Capture request           │ t_idle > 100 min
                │              │                           │
                │              │                           ▼
                │              │                ┌─────────────────────┐
                │              │                │   L3 DEEP SLEEP     │
                │              │  ┌───────────────────────────┐│
                │              │  │ Bias: SLEEP (0V, 0V)       ││
                │              │  │ Dummy Scan: DISABLED      ││
                │              │  │ Clocks: GATED             ││
                │              │  │ Wake: External interrupt  ││
                │              │  └───────────────────────────┘│
                │              │                └─────────┬───────────┘
                │              │                          │
                └──────────────┴──────────────────────────┘
                               │
                               │ Capture Request
                               │ (from any mode)
                               ▼
                    ┌─────────────────────┐
                    │   ACTIVE CAPTURE    │
                    │  - Reset Timer      │
                    │  - Capture Frame    │
                    │  - Return to L1     │
                    └─────────────────────┘
```

### 3.2 Mode Transition Conditions Table

| Current Mode | Next Mode | Condition | Action |
|--------------|-----------|-----------|--------|
| Any | L1 | Capture request | Reset idle timer, set NORMAL_BIAS |
| L1 | L2 | t_idle > 10 min OR t_idle > t_max(T) | Set IDLE_LOW_BIAS, enable dummy scan |
| L2 | L1 | Capture request | Reset idle timer, set NORMAL_BIAS |
| L2 | L3 | t_idle > 100 min | Set SLEEP_BIAS, gate clocks |
| L3 | L1 | Wake interrupt | Reinitialize, set NORMAL_BIAS |

---

## 4. Frame Capture Sequence

### 4.1 Normal Capture Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    NORMAL FRAME CAPTURE FLOW                        │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Capture Request │
                    │ (From Host/MCU) │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Check Idle Time │
                    │ t_idle > 30min? │
                    └────────┬────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
               NO                        YES
                │                         │
                ▼                         ▼
        ┌───────────────┐       ┌──────────────────┐
        │ Normal Capture│       │ Warm-up Required │
        └───────┬───────┘       └────────┬─────────┘
                │                        │
                ▼                        ▼
    ┌───────────────────┐    ┌──────────────────────┐
    │ FPGA Write:       │    │ Step 1: Warm-up Frame│
    │ CTRL_REG[7] = 1   │    │ - Reset + Integration │
    │ (FRAME_START)     │    │ - Dark read only      │
    └─────────┬─────────┘    └──────────┬───────────┘
              │                         │
              ▼                         ▼
    ┌───────────────────┐    ┌──────────────────────┐
    │ Wait Frame Done   │    │ Store Dark Reference │
    │ Poll STATUS[7] = 0│    │ For correction       │
    └─────────┬─────────┘    └──────────┬───────────┘
              │                         │
              ▼                         ▼
    ┌───────────────────┐    ┌──────────────────────┐
    │ Read Frame Data   │    │ Step 2: Clinical Frame│
    │ From FIFO         │    │ - Reset + Integration │
    │ Attach Metadata   │    │ - Full exposure       │
    └─────────┬─────────┘    └──────────┬───────────┘
              │                         │
              ▼                         ▼
    ┌───────────────────┐    ┌──────────────────────┐
    │ Send to Host      │    │ Send Both Frames     │
    │ + Metadata        │    │ with metadata tags   │
    └───────────────────┘    └──────────────────────┘
```

### 4.2 Metadata Assembly

```csharp
// Frame metadata structure for each captured frame
FrameMetadata {
    // Timing
    timestamp: u64          // Unix timestamp (ms)
    sequence_number: u32    // Frame sequence number

    // Thermal State
    temperature: f32        // Panel temperature (°C)
    drift_rate: f32         // Calculated k(T) (DN/min)

    // Idle History
    idle_time_before: u32   // Idle time before capture (s)
    idle_mode_before: u8    // L1/L2/L3 mode

    // Frame Type
    is_warmup_frame: bool   // True if warm-up/dark frame
    frame_type: u8          // 0=DARK, 1=LAG1, 2=LAG2, 3=CLINICAL

    // Quality Indicators
    bias_mode_ready: bool   // Bias was settled
    fifo_overflow: bool     // FIFO overflow occurred
}
```

---

## 5. Mode Transition Sequences

### 5.1 L1 to L2 Transition

```
┌─────────────────────────────────────────────────────────────────────┐
│                    L1 → L2 TRANSITION SEQUENCE                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Transition Event │
                    │ - t_idle > 10min │
                    │ - OR t_idle > t_max│
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Calculate t_max │
                    │ k(T) = Arrhenius│
                    └────────┬────────┘
                             │
                             ▼
        ┌───────────────────────────────────────────┐
        │  Step 1: Change Bias Mode                │
        │  FPGA Write: BIAS_SELECT = 0x01           │
        │  (IDLE_LOW_BIAS: V_PD=-0.2V, V_COL=-0.2V) │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────────┐
        │  Step 2: Wait Bias Settled               │
        │  Poll STATUS[5] = 1 (BIAS_MODE_READY)    │
        │  Timeout: 10 µs                          │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────────┐
        │  Step 3: Configure Dummy Scan             │
        │  FPGA Write: DUMMY_PERIOD = 60 seconds   │
        │  FPGA Write: DUMMY_CONTROL = 0x81        │
        │  (Auto mode, reset only, all rows)       │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────────┐
        │  Step 4: Verify L2 Entry                  │
        │  STATUS[6] = 1 (DUMMY_ACTIVE)            │
        │  STATUS[1] = 1 (IDLE_STATE)              │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
                    ┌─────────────────┐
                    │    L2 ACTIVE    │
                    │  Monitoring     │
                    └─────────────────┘
```

### 5.2 L2 to L3 Transition

```
┌─────────────────────────────────────────────────────────────────────┐
│                    L2 → L3 TRANSITION SEQUENCE                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Transition Event │
                    │ t_idle > 100min  │
                    └────────┬────────┘
                             │
                             ▼
        ┌───────────────────────────────────────────┐
        │  Step 1: Stop Dummy Scan                 │
        │  FPGA Write: DUMMY_CONTROL = 0x00        │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────────┐
        │  Step 2: Change to Sleep Bias            │
        │  FPGA Write: BIAS_SELECT = 0x10          │
        │  (SLEEP_BIAS: V_PD=0V, V_COL=0V)         │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────────┐
        │  Step 3: Gate Clocks (Optional)           │
        │  Disable row/column clocks               │
        │  Reduce power consumption                 │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
                    ┌─────────────────┐
                    │    L3 ACTIVE    │
                    │  Deep Sleep     │
                    └─────────────────┘
```

### 5.3 L3 to L1 Transition (Wake)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    L3 → L1 WAKE SEQUENCE                             │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Wake Event      │
                    │ (External/Timer) │
                    └────────┬────────┘
                             │
                             ▼
        ┌───────────────────────────────────────────┐
        │  Step 1: Enable Clocks                    │
        │  Re-enable row/column clocks              │
        │  Wait for PLL lock                        │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────────┐
        │  Step 2: Change to Normal Bias           │
        │  FPGA Write: BIAS_SELECT = 0x00          │
        │  (NORMAL_BIAS: V_PD=-1.5V, V_COL=-1.0V)  │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────────┐
        │  Step 3: Warm-up Frames Required?        │
        │  t_idle > 30min → YES                     │
        └───────────────────┬───────────────────────┘
                            │
                ┌───────────┴───────────┐
                │                       │
               NO                      YES
                │                       │
                ▼                       ▼
        ┌───────────────┐    ┌──────────────────┐
        │ Ready for     │    │ 2-Frame Warm-up  │
        │ Normal Capture│    │ Sequence Required│
        └───────────────┘    └──────────────────┘
```

---

## 6. Error Recovery Sequences

### 6.1 SPI Communication Error

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SPI ERROR RECOVERY SEQUENCE                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Error Detected   │
                    │ - CRC mismatch  │
                    │ - Timeout       │
                    │ - Invalid response│
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Retry Counter++ │
                    │ Count < 3?      │
                    └────────┬────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
               YES                        NO
                │                         │
                ▼                         ▼
        ┌───────────────┐        ┌──────────────────┐
        │ Resend Command│        │ Set ERROR_CODE   │
        │ Wait response │        │ Notify MCU       │
        └───────────────┘        │ Request recovery │
                                 └──────────────────┘
```

### 6.2 FIFO Overflow Error

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FIFO OVERFLOW RECOVERY                           │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ FIFO_FULL = 1   │
                    │ Data may be lost│
                    └────────┬────────┘
                             │
                             ▼
        ┌───────────────────────────────────────────┐
        │  Immediate Action:                        │
        │  1. Stop current capture                  │
        │  2. FIFO_RESET via CTRL_REG[1]            │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────────┐
        │  Root Cause Analysis:                     │
        │  - Host read too slow?                    │
        │  - Integration time too long?            │
        │  - Clock frequency mismatch?             │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
                    ┌─────────────────┐
                    │ Adjust Config   │
                    │ or Notify Host  │
                    └─────────────────┘
```

### 6.3 Bias Switch Timeout

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BIAS SWITCH TIMEOUT RECOVERY                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ BIAS_READY = 0  │
                    │ After 10 µs     │
                    └────────┬────────┘
                             │
                             ▼
        ┌───────────────────────────────────────────┐
        │  Retry Counter < 3?                      │
        └───────────────────┬───────────────────────┘
                            │
                ┌───────────┴───────────┐
                │                       │
               YES                      NO
                │                       │
                ▼                       ▼
        ┌───────────────┐        ┌──────────────────┐
        │ Re-write      │        │ Set ERROR_CODE   │
        │ BIAS_SELECT   │        │ = 0x04           │
        │ Wait again    │        │ Fall back to     │
        └───────────────┘        │ Previous mode    │
                                 └──────────────────┘
```

---

## 7. Timing Diagrams

### 7.1 Complete Capture Cycle

```
MCU                    FPGA                    Panel
 │                       │                       │
 │─── WRITE REG ────────►│                       │
 │   (FRAME_START)       │                       │
 │                       │─── RESET_ALL ────────►│
 │                       │─── GATE ON ──────────►│
 │                       │                       │
 │                       │─── INTEGRATE ────────►│ (100ms)
 │                       │                       │
 │                       │─── GATE OFF ─────────►│
 │                       │─── ROW_CLK ──────────►│
 │                       │─── READ_ROW(0) ──────►│
 │                       │◄─── ADC_DATA ─────────│
 │                       │                       │
 │◄─── FIFO_DATA ────────│                       │
 │                       │─── ROW_CLK ──────────►│
 │                       │─── READ_ROW(1) ──────►│
 │                       │◄─── ADC_DATA ─────────│
 │                       │                       │
 │   ... (repeat 2048 rows) ...                  │
 │                       │                       │
 │                       │─── READ_ROW(2047) ───►│
 │                       │◄─── ADC_DATA ─────────│
 │                       │                       │
 │─── STATUS_REG ────────►│                       │
 │   (Check FRAME_BUSY=0)│                       │
 │                       │                       │
```

### 7.2 L2 Dummy Scan Cycle

```
MCU                    FPGA                    Panel
 │                       │                       │
 │ (Background timer, 60s period)               │
 │                       │                       │
 │                       │─── DUMMY_TRIGGER ────►│
 │                       │─── GATE_i ON ───────►│
 │                       │─── RESET_PULSE ─────►│ (i=0)
 │                       │─── GATE_i OFF ──────►│
 │                       │─── WAIT 100µs        │
 │                       │                       │
 │                       │─── GATE_i+1 ON ─────►│
 │                       │─── RESET_PULSE ─────►│ (i=1)
 │                       │─── GATE_i+1 OFF ────►│
 │                       │─── WAIT 100µs        │
 │                       │                       │
 │   ... (repeat all 2048 rows, ~2ms total) ... │
 │                       │                       │
 │                       │─── DUMMY_COMPLETE ────│
 │◄─── STATUS_REG (DUMMY_ACTIVE=0)             │
 │                       │                       │
 │ (Back to low-power waiting for next period)  │
 │                       │                       │
```

---

## 8. Configuration Sequences

### 8.1 ROI Configuration

```
Sequence to configure region of interest:

1. Disable capture (if active):
   Write CTRL_REG = 0x00

2. Set row boundaries:
   Write ROW_START_L = (row_start & 0xFF)
   Write ROW_START_H = ((row_start >> 8) & 0xFF)
   Write ROW_END_L = (row_end & 0xFF)
   Write ROW_END_H = ((row_end >> 8) & 0xFF)

3. Set column boundaries:
   Write COL_START_L = (col_start & 0xFF)
   Write COL_START_H = ((col_start >> 8) & 0xFF)
   Write COL_END_L = (col_end & 0xFF)
   Write COL_END_H = ((col_end >> 8) & 0xFF)

4. Verify configuration:
   Read back ROW_START, ROW_END, COL_START, COL_END

5. Enable capture:
   Write CTRL_REG = 0x90 (FRAME_START | ADC_ENABLE)
```

### 8.2 Integration Time Configuration

```
Sequence to change integration time:

1. Ensure IDLE state:
   Poll STATUS_REG[7] = 0 (FRAME_BUSY = 0)

2. Write new integration time:
   Write INTEGRATION_TIME_L = (time_ms & 0xFF)
   Write INTEGRATION_TIME_H = ((time_ms >> 8) & 0xFF)

3. Apply on next capture:
   New integration time takes effect on next FRAME_START
```

---

## 9. Diagnostic Sequences

### 9.1 Built-In Self Test (BIST)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BIST EXECUTION SEQUENCE                          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Enter Test Mode │
                    │ CTRL_REG[3] = 1 │
                    └────────┬────────┘
                             │
                             ▼
        ┌───────────────────────────────────────────┐
        │  Test 1: Register Loopback               │
        │  - Write pattern to all registers        │
        │  - Read back and verify                   │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────────┐
        │  Test 2: Counter Pattern                 │
        │  - Enable test pattern output            │
        │  - Verify incrementing data sequence     │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────────┐
        │  Test 3: Timing Generator                 │
        │  - Verify row/column clock frequencies    │
        │  - Check reset pulse width                │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
                    ┌─────────────────┐
                    │ Exit Test Mode  │
                    │ Report Results  │
                    └─────────────────┘
```

### 9.2 Temperature Compensation Calibration

```
┌─────────────────────────────────────────────────────────────────────┐
│           TEMPERATURE CALIBRATION SEQUENCE                          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Set T = 25°C    │
                    │ (Thermal chamber)│
                    └────────┬────────┘
                             │
                             ▼
        ┌───────────────────────────────────────────┐
        │  Capture dark frames at various idle times│
        │  t_idle = 0, 10, 30, 60, 120, 240 min     │
        └───────────────────┬───────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────────┐
        │  Fit data to: D(t) = D0 + k(T) × t       │
        │  Extract k(25°C) = reference drift rate   │
        └───────────────────┬───────────────────────┘
                            │
                             ▼
        ┌───────────────────────────────────────────┐
        │  Repeat at T = 15, 20, 30, 35, 40°C      │
        │  Build k(T) lookup table                  │
        └───────────────────────────────────────────┘
```

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial flow definitions |

---

**End of Specification**
