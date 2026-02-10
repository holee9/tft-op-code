# Arrhenius Model Summary

## Yocto Project Team Delivery Package - Reference Document

**Document**: reference/arrhenius_model_summary.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Overview

The Arrhenius model describes the temperature dependence of the dark current drift rate in a-Si TFT panels. This document provides the mathematical foundation and implementation guidance.

---

## 2. Physical Model

### 2.1 Dark Current Drift

Dark current in a-Si TFT panels increases with idle time due to charge trapping in the gate insulator:

```
D(t) = D0 + k(T) × t

Where:
    D(t)  = Dark signal at idle time t
    D0    = Initial dark offset (calibration)
    k(T)  = Temperature-dependent drift rate
    t     = Idle time in minutes
```

### 2.2 Arrhenius Equation

The drift rate follows the Arrhenius temperature dependence:

```
k(T) = k_ref × exp[(E_A/k_B) × (1/T - 1/T_ref)]

Where:
    k(T)   = Drift rate at temperature T (DN/min)
    k_ref  = Reference drift rate at T_ref
    E_A    = Activation energy for a-Si:H (0.45 eV)
    k_B    = Boltzmann constant (8.617×10⁻⁵ eV/K)
    T      = Temperature in Kelvin
    T_ref  = Reference temperature (298.15 K = 25°C)
```

---

## 3. Model Parameters

### 3.1 Material Constants for a-Si:H

| Parameter | Value | Unit | Source |
|-----------|-------|------|--------|
| **Activation Energy (E_A)** | 0.45 | eV | Literature |
| **Boltzmann Constant (k_B)** | 8.617×10⁻⁵ | eV/K | Physical constant |

### 3.2 Reference Values

| Parameter | Value | Unit | Notes |
|-----------|-------|------|-------|
| **T_ref** | 298.15 | K | 25°C |
| **k_ref** | 1.0 | DN/min | At 25°C |
| **ΔD_max** | 50 | DN | Maximum allowable increase |

---

## 4. Calculated Values

### 4.1 Drift Rate vs Temperature

```
Temperature Calculation:
    T(K) = T(°C) + 273.15
    Exponent = (E_A/k_B) × (1/T - 1/T_ref)
    k(T) = k_ref × exp(Exponent)
```

| T (°C) | T (K) | 1/T (×10⁻³) | Exponent | k(T) (DN/min) |
|--------|-------|------------|----------|---------------|
| 15 | 288.15 | 3.470 | -0.693 | 0.50 |
| 20 | 293.15 | 3.411 | -0.288 | 0.75 |
| 25 | 298.15 | 3.354 | 0.000 | 1.00 |
| 30 | 303.15 | 3.299 | 0.277 | 1.32 |
| 35 | 308.15 | 3.245 | 0.546 | 1.72 |
| 40 | 313.15 | 3.193 | 0.807 | 2.24 |

### 4.2 Maximum Idle Time (t_max)

```
t_max = ΔD_max / k(T) × 60

Where:
    t_max = Maximum idle time in seconds
    ΔD_max = 50 DN
    k(T) = Drift rate in DN/min
    60 = Conversion to seconds
```

| T (°C) | k(T) (DN/min) | t_max (sec) | t_max (min) |
|--------|---------------|-------------|-------------|
| 15 | 0.50 | 6000 | 100 |
| 20 | 0.75 | 4000 | 67 |
| 25 | 1.00 | 3000 | 50 |
| 30 | 1.32 | 2273 | 38 |
| 35 | 1.72 | 1744 | 29 |
| 40 | 2.24 | 1339 | 22 |

---

## 5. Implementation

### 5.1 C Implementation

```c
#include <math.h>

#define E_A 0.45f              // eV
#define K_B 8.617e-5f          // eV/K
#define T_REF 298.15f          // K (25°C)
#define K_REF 1.0f             // DN/min at 25°C
#define D_MAX 50.0f            // DN

float calculate_drift_rate(float temp_celsius) {
    float temp_k = temp_celsius + 273.15f;
    float exponent = (E_A / K_B) * (1.0f / temp_k - 1.0f / T_REF);
    return K_REF * expf(exponent);
}

uint32_t calculate_t_max(float temp_celsius) {
    float drift_rate = calculate_drift_rate(temp_celsius);
    float t_max_min = D_MAX / drift_rate;
    return (uint32_t)(t_max_min * 60.0f);  // seconds
}
```

### 5.2 C# Implementation

```csharp
public class ArrheniusCalculator
{
    private const float Ea = 0.45f;           // eV
    private const float Kb = 8.617e-5f;       // eV/K
    private const float TRef = 298.15f;       // K (25°C)
    private const float KRef = 1.0f;          // DN/min at 25°C
    private const float DMax = 50.0f;         // DN

    public float CalculateDriftRate(float tempCelsius)
    {
        float tempK = tempCelsius + 273.15f;
        float exponent = (Ea / Kb) * (1.0f / tempK - 1.0f / TRef);
        return KRef * MathF.Exp(exponent);
    }

    public uint CalculateTMax(float tempCelsius)
    {
        float driftRate = CalculateDriftRate(tempCelsius);
        float tMaxMin = DMax / driftRate;
        return (uint)(tMaxMin * 60.0f);  // seconds
    }
}
```

---

## 6. Look-Up Table

### 6.1 Discrete LUT (15-40°C)

```c
// Temperature → (drift_rate, t_max)
const struct {
    int temp;
    float drift_rate;
    uint32_t t_max;
} arrhenius_lut[] = {
    {15, 0.50f, 6000}, {16, 0.56f, 5357}, {17, 0.63f, 4762},
    {18, 0.71f, 4225}, {19, 0.80f, 3750}, {20, 0.75f, 4000},
    {21, 0.85f, 3529}, {22, 0.95f, 3158}, {23, 1.07f, 2804},
    {24, 1.20f, 2500}, {25, 1.00f, 3000}, {26, 1.12f, 2679},
    {27, 1.26f, 2381}, {28, 1.41f, 2128}, {29, 1.58f, 1899},
    {30, 1.80f, 1667}, {31, 2.02f, 1485}, {32, 2.26f, 1327},
    {33, 2.54f, 1181}, {34, 2.84f, 1056}, {35, 3.20f, 938},
    {36, 3.58f, 838}, {37, 4.02f, 746}, {38, 4.51f, 666},
    {39, 5.06f, 593}, {40, 5.80f, 517}
};
```

### 6.2 Interpolation

For temperatures between LUT entries, use linear interpolation:

```c
// Linear interpolation between LUT entries
struct LUTResult interpolate_lut(float temp) {
    int idx = (int)temp - 15;
    if (idx < 0) idx = 0;
    if (idx >= 25) idx = 24;

    float frac = temp - (int)temp;
    struct LUTResult result;
    result.drift_rate = arrhenius_lut[idx].drift_rate +
                       frac * (arrhenius_lut[idx+1].drift_rate -
                              arrhenius_lut[idx].drift_rate);
    result.t_max = arrhenius_lut[idx].t_max -
                  (uint32_t)(frac * (arrhenius_lut[idx].t_max -
                                    arrhenius_lut[idx+1].t_max));
    return result;
}
```

---

## 7. Usage in Idle State Machine

### 7.1 State Decision Logic

```
Every 1 second:
    1. Read current temperature T
    2. Calculate t_max(T) using Arrhenius
    3. Get current idle time t_idle
    4. Compare t_idle to t_max

State Transitions:
    IF t_idle > 0.8 × t_max:
        L1 → L2 (enter low-bias idle)
    ELSE IF t_idle > 100 minutes:
        L2 → L3 (enter deep sleep)
    ELSE IF frame capture requested:
        Any → L1 (return to normal)
```

### 7.2 Safety Margin

The 0.8 (80%) factor provides a safety margin before entering L2:

```
L1 → L2 Condition:
    t_idle > t_max × 0.8

At 25°C (t_max = 50 min):
    Transition at t_idle > 40 min
    Actual headroom: 10 min
```

---

## 8. Validation

### 8.1 Model Validation

The Arrhenius model should be validated with actual panel measurements:

| Temperature | Measured k(T) | Calculated k(T) | Error |
|-------------|---------------|-----------------|-------|
| 20°C | ___ | 0.75 DN/min | TBD |
| 25°C | ___ | 1.00 DN/min | TBD |
| 30°C | ___ | 1.32 DN/min | TBD |
| 35°C | ___ | 1.72 DN/min | TBD |

### 8.2 Acceptance Criteria

| Criterion | Target |
|-----------|--------|
| Model accuracy | ±10% across 15-40°C |
| t_max accuracy | ±5% |
| Temperature range | 15-40°C |

---

## 9. References

| Document | Location |
|----------|----------|
| a-Si TFT Physics | `../../panel/physics/` |
| Implementation Plan | `../../reference/latest/` |
| Validation Report | `../../reference/latest/v3_1_Validation_Report_5rounds.md` |

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial Arrhenius model summary |

---

**End of Document**
