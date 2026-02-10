# UI Requirements

## .NET Project Team Delivery Package

**Document**: 05_ui_requirements.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Main Window Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ TFT Leakage Control                              [─][□][×]        │
├─────────────────────────────────────────────────────────────────┤
│ ┌─────────┬─────────────────────────────────────────────────┐ │
│ │         │                                                 │ │
│ │ Control │              Image Display                     │ │
│ │ Panel  │         (2048 x 2048 display)                 │ │
│ │         │                                                 │ │
│ │ [Start] │                                                 │ │
│ │ [Stop]  │                                                 │ │
│ │         │                                                 │ │
│ │ Status: │              Histogram / Info                  │ │
│ │ Ready   │                                                 │ │
│ └─────────┴─────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│ Menu | Capture | Calibration | Settings | Log | Help          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Screens

### 2.1 Dashboard (Main)

| Element | Description |
|---------|-------------|
| Image View | Frame display with pan/zoom |
| Histogram | Pixel value distribution |
| Status Bar | Connection status, temperature, mode |
| Control Buttons | Start/Stop capture, single capture |

### 2.2 Capture Dialog

| Element | Description |
|---------|-------------|
| Exposure Time | Integration time slider/input |
| Number of Frames | Multi-frame capture count |
| Save Location | File path for saving |
| Save Dark Frame | Option to save as dark reference |

### 2.3 Settings Dialog

| Element | Description |
|---------|-------------|
| Connection | i.MX8 IP address, ports |
| Display | Color map, min/max values |
| Processing | Enable/disable corrections |
| Calibration | Dark frame, bad pixel map paths |
| Logging | Log level, file path |

---

## 3. Controls

| Control | Action |
|---------|--------|
| Start Capture | Begin continuous capture |
| Stop Capture | Stop current capture |
| Single Frame | Capture one frame |
| Save Frame | Save current display |
| Load Dark | Load dark reference |
| Calibrate | Run calibration wizard |

---

## 4. Status Indicators

| Indicator | States |
|-----------|--------|
| Connection | Connected, Disconnected, Error |
| Capture | Idle, Capturing, Complete |
| Temperature | Numeric + color indicator |
| Idle Mode | L1 (green), L2 (yellow), L3 (red) |

---

## 5. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial UI requirements |

---

**End of Document**
