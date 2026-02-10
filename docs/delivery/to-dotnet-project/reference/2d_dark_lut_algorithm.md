# 2D Dark LUT Algorithm

## .NET Project Team Delivery Package - Reference Document

**Document**: reference/2d_dark_lut_algorithm.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Overview

The 2D Dark LUT provides temperature and idle-time dependent correction factors for dark current drift.

---

## 2. LUT Structure

```
Dimensions:
  - Temperature axis: 15-40°C (26 steps, 1°C each)
  - Idle time axis: 0-60 min (11 steps, 6 min each)

Data format:
  LUT[temp, idle] = correction_factor (DN)

Example value:
  LUT[25, 30] = 12.5  # At 25°C, 30 min idle → 12.5 DN correction
```

---

## 3. Interpolation Algorithm

```csharp
public float GetCorrectionFactor(float temperature, uint idleTimeSeconds)
{
    // Clamp to range
    temperature = Math.Clamp(temperature, 15, 40);
    idleTimeSeconds = Math.Clamp(idleTimeSeconds, 0, 3600);

    // Calculate indices (0-25, 0-10)
    float tempIdx = temperature - 15;
    float idleIdx = idleTimeSeconds / 360f; // 3600 sec = 60 min

    int t0 = (int)tempIdx;
    int t1 = Math.Min(t0 + 1, 25);
    float tFrac = tempIdx - t0;

    int i0 = (int)idleIdx;
    int i1 = Math.Min(i0 + 1, 10);
    float iFrac = idleIdx - i0;

    // Bilinear interpolation
    float v00 = LutData[t0, i0];
    float v01 = LutData[t0, i1];
    float v10 = LutData[t1, i0];
    float v11 = LutData[t1, i1];

    float v0 = v00 * (1 - iFrac) + v01 * iFrac;
    float v1 = v10 * (1 - iFrac) + v11 * iFrac;

    return v0 * (1 - tFrac) + v1 * tFrac;
}
```

---

## 4. Application to Image

```
For each pixel:
  corrected_value = raw_value - LUT(temperature, idle_time)
```

---

## 5. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial document |

---

**End of Document**
