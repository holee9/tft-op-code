# Idle State Machine

## Yocto Project Team Delivery Package - Reference Document

**Document**: reference/idle_state_machine.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Overview

The idle state machine manages panel bias modes based on idle time and temperature to minimize dark current drift.

---

## 2. State Definitions

### 2.1 L1: Normal Idle

| Parameter | Value |
|-----------|-------|
| **Bias Mode** | NORMAL_BIAS |
| **V_PD** | -1.5 V |
| **V_COL** | -1.0 V |
| **Dummy Scan** | Disabled |
| **Power** | Normal |
| **Entry Condition** | Power-on, or frame capture complete |
| **Exit Condition** | t_idle > 10 min OR t_idle > 0.8 × t_max |

### 2.2 L2: Low-Bias Idle

| Parameter | Value |
|-----------|-------|
| **Bias Mode** | IDLE_LOW_BIAS |
| **V_PD** | -0.2 V |
| **V_COL** | -0.2 V |
| **Dummy Scan** | Enabled, 60 sec period |
| **Power** | Reduced |
| **Entry Condition** | t_idle > 10 min OR t_idle > 0.8 × t_max |
| **Exit Condition** | Frame capture OR t_idle > 100 min |

### 2.3 L3: Deep Sleep

| Parameter | Value |
|-----------|-------|
| **Bias Mode** | SLEEP_BIAS |
| **V_PD** | 0 V |
| **V_COL** | 0 V |
| **Dummy Scan** | Disabled |
| **Power** | Minimal |
| **Entry Condition** | t_idle > 100 min |
| **Exit Condition** | Frame capture (requires wake) |

---

## 3. State Diagram

```
                          POWER ON
                               │
                               ▼
                    ┌─────────────────────┐
                    │       L1 IDLE        │
                    │  ┌───────────────┐   │
                    │  │ NORMAL_BIAS   │   │
                    │  │ No Dummy Scan │   │
                    │  └───────────────┘   │
                    └─────────┬───────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
    t_idle < 10 min          │              t_idle > 10 min
    AND                       │              OR
    t_idle < 0.8×t_max        │              t_idle > 0.8×t_max
           │                  │                  │
           │                  │                  ▼
           │             Frame Request    ┌─────────────────────┐
           │                  │           │       L2 IDLE        │
           │◄─────────────────┴───────────│  ┌───────────────┐   │
           │                                  │  │ IDLE_LOW_BIAS │   │
           │                                  │  │ Dummy: 60s    │   │
           │                                  │  └───────────────┘   │
           │                                  └─────────┬───────────┘
           │                                            │
           │                                 t_idle < 100  │
           │                                 AND capture  │
           │                                            │
           │                                            │ t_idle > 100 min
           │                                            │
           │                                            ▼
           │                                  ┌─────────────────────┐
           │                                  │       L3 SLEEP       │
           │                                  │  ┌───────────────┐   │
           │◄─────────────────────────────────│  │ SLEEP_BIAS    │   │
           │    Wake + Capture                 │  │ Minimal Power │   │
           │                                  │  └───────────────┘   │
           │                                  └─────────────────────┘
           │
           ▼
    ┌─────────────────────┐
    │   ACTIVE CAPTURE    │
    │  - Reset timer      │
    │  - Capture frame    │
    │  - Return to L1     │
    └─────────────────────┘
```

---

## 4. Transition Conditions

### 4.1 Condition Table

| From | To | Condition | Action |
|------|-----|-----------|--------|
| Any | L1 | Frame capture request | Reset idle timer, set NORMAL_BIAS |
| L1 | L2 | t_idle > 10 min OR t_idle > 0.8×t_max | Set IDLE_LOW_BIAS, enable dummy scan |
| L2 | L1 | Frame capture request | Reset idle timer, set NORMAL_BIAS |
| L2 | L3 | t_idle > 100 min | Set SLEEP_BIAS, disable dummy scan |
| L3 | L1 | Wake + frame capture | Initialize, set NORMAL_BIAS |

### 4.2 Timing Parameters

| Parameter | Value | Unit |
|-----------|-------|------|
| L1 timeout | 10 | min |
| L1 t_max threshold | 0.8 × t_max | - |
| L2 timeout | 100 | min |
| Dummy scan period | 60 | sec |

---

## 5. C# Implementation

### 5.1 State Machine Class

```csharp
public enum IdleMode
{
    L1_Normal = 0,
    L2_LowBias = 1,
    L3_DeepSleep = 2
}

public class IdleStateMachine
{
    private IdleMode _currentMode = IdleMode.L1_Normal;
    private DateTimeOffset _lastResetTime = DateTimeOffset.UtcNow;
    private readonly IFpgaClient _fpga;
    private readonly ArrheniusCalculator _calculator;

    public IdleMode CurrentMode => _currentMode;
    public uint IdleTimeSeconds => (uint)(DateTimeOffset.UtcNow - _lastResetTime).TotalSeconds;

    public void ResetIdleTimer()
    {
        _lastResetTime = DateTimeOffset.UtcNow;
        _ = TransitionToMode(IdleMode.L1_Normal);
    }

    public async Task UpdateAsync(float temperature)
    {
        // Calculate t_max at current temperature
        uint tMax = _calculator.CalculateTMax(temperature);
        uint tIdle = IdleTimeSeconds;

        // Determine next state
        IdleMode nextMode = _currentMode;
        if (_currentMode == IdleMode.L1_Normal)
        {
            if (tIdle > 600 || tIdle > (tMax * 8 / 10)) // 10 min or 80% t_max
            {
                nextMode = IdleMode.L2_LowBias;
            }
        }
        else if (_currentMode == IdleMode.L2_LowBias)
        {
            if (tIdle > 6000) // 100 minutes
            {
                nextMode = IdleMode.L3_DeepSleep;
            }
        }

        // Transition if needed
        if (nextMode != _currentMode)
        {
            await TransitionToMode(nextMode);
        }
    }

    private async Task TransitionToMode(IdleMode newMode)
    {
        BiasMode biasMode = newMode switch
        {
            IdleMode.L1_Normal => BiasMode.NormalBias,
            IdleMode.L2_LowBias => BiasMode.IdleLowBias,
            IdleMode.L3_DeepSleep => BiasMode.SleepBias,
            _ => BiasMode.NormalBias
        };

        await _fpga.SetBiasModeAsync(biasMode);

        if (newMode == IdleMode.L2_LowBias)
        {
            await _fpga.EnableDummyScanAsync(60); // 60 sec period
        }
        else
        {
            await _fpga.DisableDummyScanAsync();
        }

        _currentMode = newMode;
    }
}
```

---

## 6. Frame Capture Handling

### 6.1 Capture Flow

```
Capture Request:
    1. Check current mode and idle time
    2. Determine if warm-up is needed
    3. Reset idle timer
    4. Transition to L1 (if not already)
    5. Start frame capture

Warm-Up Decision:
    IF current_mode == L2 AND t_idle > 30 minutes:
        Require warm-up frame
    ELSE IF current_mode == L3:
        Require warm-up frame + longer stabilization
    ELSE:
        Normal capture
```

### 6.2 Warm-Up Frame Handling

```csharp
public async Task<FrameResult> StartFrameCaptureAsync()
{
    bool needWarmup = _currentMode switch
    {
        IdleMode.L2_LowBias => IdleTimeSeconds > 1800, // 30 min
        IdleMode.L3_DeepSleep => true,
        _ => false
    };

    ResetIdleTimer();
    await _fpga.StartFrameCaptureAsync();

    if (needWarmup)
    {
        // Capture warm-up frame first
        await Task.Delay(150); // Wait for warm-up frame
        // Warm-up frame is discarded after correction
    }

    // Main clinical frame
    return await GetFrameDataAsync();
}
```

---

## 7. Bias Mode Register Values

| Mode | Register Value | Description |
|------|----------------|-------------|
| NORMAL_BIAS | 0x00 | V_PD=-1.5V, V_COL=-1.0V |
| IDLE_LOW_BIAS | 0x01 | V_PD=-0.2V, V_COL=-0.2V |
| SLEEP_BIAS | 0x02 | V_PD=0V, V_COL=0V |

### 7.1 SPI Write Sequence

```csharp
// Write bias mode register
spi.Write(new byte[] {
    0x01,          // WRITE command
    0x02,          // BIAS_SELECT register
    (byte)mode     // Mode value
});
```

---

## 8. Timing Diagrams

### 8.1 L1 → L2 Transition

```
MCU                    FPGA
 │                       │
 │ Write BIAS_SELECT    │──► [Store 0x01]
 │ [0x01]                │     [Start bias switch]
 │                       │     [BIAS_UPDATE_PENDING=1]
 │                       │
 │ Poll STATUS_REG       │◄─── [BIAS_MODE_READY=0]
 │                       │     [Switching in progress]
 │                       │
 │ ... < 10 µs ...       │     [DAC settling]
 │                       │
 │ Poll STATUS_REG       │◄─── [BIAS_MODE_READY=1]
 │                       │     [Switch complete]
 │                       │
 │ Write DUMMY_PERIOD    │──► [Set 60 sec]
 │ [60 sec]              │
 │ Write DUMMY_CONTROL   │──► [Enable dummy scan]
 │ [0x81]                │     [DUMMY_ACTIVE=1]
```

### 8.2 Frame Capture from Idle

```
State Machine Flow:

Any Idle State
       │
       ▼
┌──────────────────┐
│ Capture Request  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐     No      ┌─────────────┐
│ Need Warm-up?    │────────────►│ Start Frame │
└────────┬─────────┘             └──────┬──────┘
         │ Yes                           │
         ▼                               │
┌──────────────────┐                    │
│ Warm-up Frame    │                    │
│ (Discard later)  │                    │
└────────┬─────────┘                    │
         │                               │
         ▼                               ▼
┌──────────────────────────────────────────────┐
│             Clinical Frame                   │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
            ┌─────────────┐
            │ Return to L1 │
            └─────────────┘
```

---

## 9. Reference Documents

| Document | Location |
|----------|----------|
| Control Flows | `../../design/control_sequence_flows.md` |
| SPI Register Map | `../../design/spi_register_map.md` |
| i.MX8 Control Spec | `../../design/imx8_dotnet_control_spec.md` |

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial state machine document |

---

**End of Document**
