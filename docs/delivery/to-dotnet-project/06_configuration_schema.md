# Configuration Schema

## .NET Project Team Delivery Package

**Document**: 06_configuration_schema.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "TFT Leakage Application Configuration",
  "type": "object",
  "properties": {
    "Connection": {
      "type": "object",
      "properties": {
        "Host": { "type": "string", "format": "hostname" },
        "Port": { "type": "integer", "minimum": 1024, "maximum": 65535 },
        "DataPort": { "type": "integer", "minimum": 1024, "maximum": 65535 },
        "TimeoutMs": { "type": "integer", "minimum": 100, "maximum": 60000 }
      },
      "required": ["Host", "Port"]
    },
    "Display": {
      "type": "object",
      "properties": {
        "ColorMap": { "type": "string", "enum": ["Grayscale", "Rainbow", "Hot"] },
        "MinValue": { "type": "integer", "minimum": 0 },
        "MaxValue": { "type": "integer", "maximum": 65535 },
        "AutoScale": { "type": "boolean" }
      }
    },
    "Processing": {
      "type": "object",
      "properties": {
        "EnableDarkCorrection": { "type": "boolean" },
        "Enable2dLut": { "type": "boolean" },
        "DarkFramePath": { "type": "string" },
        "LutPath": { "type": "string" }
      }
    }
  }
}
```

---

## 2. Default Configuration

```json
{
  "Connection": {
    "Host": "192.168.1.100",
    "Port": 5000,
    "DataPort": 5001,
    "TimeoutMs": 5000
  },
  "Display": {
    "ColorMap": "Grayscale",
    "MinValue": 0,
    "MaxValue": 16383,
    "AutoScale": true
  },
  "Processing": {
    "EnableDarkCorrection": true,
    "Enable2dLut": true,
    "DarkFramePath": "calibration/dark.bin",
    "LutPath": "calibration/lut.bin"
  }
}
```

---

## 3. Configuration File Locations

| Platform | Path |
|----------|------|
| Windows | `%APPDATA%/TftLeakage/appsettings.json` |
| Development | `./appsettings.Development.json` |

---

## 4. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial schema |

---

**End of Document**
