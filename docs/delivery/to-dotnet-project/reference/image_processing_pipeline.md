# Image Processing Pipeline

## .NET Project Team Delivery Package - Reference Document

**Document**: reference/image_processing_pipeline.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Processing Pipeline

```
Raw Frame
    │
    ▼
┌─────────────────┐
│  Bad Pixel      │
│  Correction     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Dark Frame     │
│  Subtraction    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  2D Dark LUT    │
│  Correction     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Normalization  │
│  (Auto Scale)   │
└────────┬────────┘
         │
         ▼
    Display Image
```

---

## 2. Bad Pixel Correction

```csharp
// Replace bad pixels with average of neighbors
ushort CorrectBadPixel(ushort[] data, int x, int y, int width)
{
    if (!badPixelMap.IsBad(x, y))
        return data[y * width + x];

    // Average of 4 neighbors
    int idx = y * width + x;
    ushort sum = 0;
    int count = 0;

    if (x > 0) { sum += data[idx - 1]; count++; }
    if (x < width - 1) { sum += data[idx + 1]; count++; }
    if (y > 0) { sum += data[idx - width]; count++; }
    if (y < height - 1) { sum += data[idx + width]; count++; }

    return (ushort)(sum / count);
}
```

---

## 3. Dark Frame Subtraction

```csharp
ushort[] ApplyDarkSubtraction(ushort[] frame, ushort[] dark)
{
    ushort[] result = new ushort[frame.Length];

    for (int i = 0; i < frame.Length; i++)
    {
        int value = frame[i] - dark[i];
        result[i] = (ushort)Math.Clamp(value, 0, 65535);
    }

    return result;
}
```

---

## 4. 2D Dark LUT Correction

```csharp
ushort[] ApplyLutCorrection(ushort[] frame, float temp, uint idleTime, float lutFactor)
{
    ushort[] result = new ushort[frame.Length];

    for (int i = 0; i < frame.Length; i++)
    {
        int corrected = frame[i] - (int)lutFactor;
        result[i] = (ushort)Math.Clamp(corrected, 0, 65535);
    }

    return result;
}
```

---

## 5. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial document |

---

**End of Document**
