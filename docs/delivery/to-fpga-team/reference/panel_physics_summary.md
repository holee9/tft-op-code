# Panel Physics Summary

## FPGA Team Delivery Package - Reference Document

**Document**: reference/panel_physics_summary.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Overview

This document summarizes the physical characteristics of the a-Si TFT FPD panel relevant to FPGA implementation.

---

## 2. Panel Specifications

### 2.1 Basic Specifications

| Parameter | Value | Unit |
|-----------|-------|------|
| **Model** | R1717AS01.3 | - |
| **Technology** | a-Si:H TFT + PIN Diode | - |
| **Resolution** | 2048 x 2048 | pixels |
| **Pixel Pitch** | 140 | µm |
| **Active Area** | 286.7 x 286.7 | mm |
| **Fill Factor** | > 80 | % |

### 2.2 Electrical Characteristics

| Parameter | Value | Unit |
|-----------|-------|------|
| **VGH (Gate High)** | +15 | V |
| **VGL (Gate Low)** | -5 | V |
| **V_PD (Normal)** | -1.5 | V |
| **V_PD (Idle)** | -0.2 | V |
| **V_COL (Normal)** | -1.0 | V |
| **V_COL (Idle)** | -0.2 | V |

---

## 3. Dark Current Mechanism

### 3.1 Leakage Current in a-Si TFT

```
Dark Current Source: Gate-Insulator Interface

                 ┌─────────────────┐
                 │   Gate Electrode│
                 └────────┬────────┘
                          │
                 ┌────────▼────────┐
                 │  Gate Insulator │  ← Traps cause leakage
                 └────────┬────────┘
                          │
    ┌─────────────────────┼─────────────────────┐
    │                     │                     │
    ▼                     ▼                     ▼
┌───────┐             ┌─────────┐           ┌───────┐
│Source │────────────▶│  Channel│──────────▶│ Drain │
│(n+)   │             │  (a-Si) │           │ (n+)  │
└───────┘             └─────────┘           └───────┘
```

### 3.2 Dark Current vs Idle Time

```
Dark Current (DN) vs Idle Time:

    D(t) = D0 + k(T) × t

Where:
    D(t)  = Dark signal at idle time t
    D0    = Initial dark offset
    k(T)  = Temperature-dependent drift rate
    t     = Idle time in minutes

Example at 25°C:
    k(25°C) = 1.0 DN/min
    After 60 minutes idle:
    D(60) = D0 + 1.0 × 60 = D0 + 60 DN
```

---

## 4. Temperature Dependence (Arrhenius Model)

### 4.1 Arrhenius Equation

```
Drift Rate vs Temperature:

    k(T) = k_ref × exp[(E_A/k_B) × (1/T - 1/T_ref)]

Where:
    k(T)   = Drift rate at temperature T
    k_ref  = Reference drift rate at T_ref
    E_A    = Activation energy (0.45 eV for a-Si:H)
    k_B    = Boltzmann constant (8.617×10⁻⁵ eV/K)
    T      = Temperature in Kelvin
    T_ref  = Reference temperature (298.15 K = 25°C)
```

### 4.2 Drift Rate Look-Up Table

| Temperature (°C) | Drift Rate (DN/min) |
|------------------|---------------------|
| 15 | 0.50 |
| 20 | 0.75 |
| 25 | 1.00 |
| 30 | 1.80 |
| 35 | 3.20 |
| 40 | 5.80 |

### 4.3 Maximum Idle Time (t_max)

```
t_max = ΔD_max / k(T)

Where:
    t_max   = Maximum allowable idle time (seconds)
    ΔD_max  = Maximum allowable dark increase (50 DN)
    k(T)    = Drift rate at current temperature

Example at 25°C:
    t_max = 50 DN / (1.0 DN/min) × 60 sec/min
    t_max = 3000 seconds = 50 minutes
```

---

## 5. Bias Voltage Impact

### 5.1 Bias Modes and Dark Current

| Mode | V_PD | V_COL | Dark Current |
|------|------|-------|--------------|
| NORMAL_BIAS | -1.5 V | -1.0 V | Normal operation |
| IDLE_LOW_BIAS | -0.2 V | -0.2 V | Reduced by ~90% |
| SLEEP_BIAS | 0 V | 0 V | Minimal |

### 5.2 Mode Switching Requirements

```
Switching Time Impact:

FPGA must complete bias switch within 10 µs to avoid:

1. Ghost images (incomplete switching)
2. Charge injection (slow transitions)
3. Panel stress (intermediate voltages)
```

---

## 6. Dummy Scan Physics

### 6.1 Storage Node Reset

```
Storage Node (at pixel):

           V_PD (Bias)
              │
              ▼
    ┌─────────┴─────────┐
    │   Photodiode      │
    │  (PIN Junction)   │
    └─────────┬─────────┘
              │
         Storage Node ◄─── Reset Pulse
         (Charge accumulates)
              │
              ▼
         ┌─────┴─────┐
         │   TFT     │
         │  (Switch) │
         └───────────┘
```

### 6.2 Dummy Scan Operation

```
Purpose: Prevent charge accumulation during idle

Operation:
    For each row i = 0 to 2047:
        1. Turn ON gate_i
        2. Apply reset pulse (1 µs)
        3. Settle (100 µs)
        4. Turn OFF gate_i

Effect:
    - All storage nodes reset to baseline
    - No photocharge accumulation
    - Eliminates dark current drift
```

---

## 7. Timing Constraints from Physics

### 7.1 Minimum Timing Requirements

| Parameter | Minimum Value | Physical Reason |
|-----------|---------------|-----------------|
| Reset Pulse | 1 µs | Complete charge evacuation |
| Settle Time | 100 µs | Charge redistribution |
| Row Read Time | 50 µs | 2048 pixels @ 20 MHz |
| Bias Switch | 10 µs | DAC settling time |

### 7.2 Maximum Timing Constraints

| Parameter | Maximum Value | Physical Reason |
|-----------|---------------|-----------------|
| Integration Time | 65535 ms | Saturation avoidance |
| Dummy Period | 65535 sec | Register limit |
| Idle Time (L1→L2) | 10 min | Dark current limit |

---

## 8. Failure Modes

### 8.1 Timing-Related Failures

| Failure | Cause | Effect |
|---------|-------|--------|
| Image lag | Insufficient reset | Ghost image |
| Dark increase | Long idle time | Elevated baseline |
| Non-uniformity | Row timing mismatch | Banding |
| Saturation | Long integration | Clip at maximum |

### 8.2 FPGA Mitigation

| Failure | FPGA Mitigation |
|---------|-----------------|
| Image lag | Ensure reset pulse ≥ 1 µs |
| Dark increase | Implement L2/L3 idle modes |
| Non-uniformity | Calibrate row timing |
| Saturation | Limit integration time |

---

## 9. Design Implications

### 9.1 For FPGA Implementation

1. **Reset Pulse Width**: Must be ≥ 1 µs for complete charge evacuation
2. **Settle Time**: 100 µs minimum for charge redistribution
3. **Bias Switching**: < 10 µs to avoid intermediate states
4. **Dummy Scan**: All 2048 rows must be reset sequentially

### 9.2 Critical Timing Paths

```
Most Critical:
1. Reset pulse generation (1 µs ± 10%)
2. Bias mode switching (< 10 µs)
3. Row address change (50 µs period)

Less Critical:
1. Integration timer (ms range)
2. Dummy period (seconds range)
```

---

## 10. References

| Document | Location |
|----------|----------|
| Full Panel Spec | `../../panel/spec/` |
| Physics Analysis | `../../panel/physics/` |
| Implementation Plan | `../../reference/latest/` |

---

## 11. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial summary |

---

**End of Document**
