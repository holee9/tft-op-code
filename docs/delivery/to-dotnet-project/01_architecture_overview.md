# Architecture Overview

## .NET Project Team Delivery Package

**Document**: 01_architecture_overview.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Solution Structure

```
TftLeakage.App.sln
├── src/
│   ├── TftLeakage.App/            # Main WPF application
│   │   ├── Views/                 # XAML views
│   │   ├── ViewModels/            # MVVM view models
│   │   ├── Services/              # Business logic
│   │   ├── Models/                # Data models
│   │   └── Converters/            # Value converters
│   ├── TftLeakage.Core/           # Shared library
│   │   ├── Interfaces/            # Service interfaces
│   │   ├── Communication/         # Network client
│   │   ├── Processing/            # Image processing
│   │   └── Configuration/          # Config management
│   └── TftLeakage.Tests/          # Unit tests
└── docs/                          # Documentation
```

---

## 2. Key Design Patterns

### 2.1 MVVM Pattern

```
View (XAML) ←→ ViewModel ←→ Model
     ↑              ↓
  Commands      Services
```

### 2.2 Dependency Injection

```csharp
// Program.cs
var services = new ServiceCollection();
services.AddSingleton<ICommunicationService, TcpCommunicationService>();
services.AddSingleton<IImageProcessor, ImageProcessor>();
services.AddSingleton<ILutService, DarkLutService>();

var serviceProvider = services.BuildServiceProvider();
```

---

## 3. Service Interfaces

### 3.1 Communication Service

```csharp
public interface ICommunicationService
{
    Task ConnectAsync(string host, int port);
    Task DisconnectAsync();
    Task SendCommandAsync(byte[] command);
    Task<byte[]> ReceiveResponseAsync();
    event Action<byte[]>? DataReceived;
}
```

### 3.2 Image Processing Service

```csharp
public interface IImageProcessor
{
    ushort[] ApplyDarkCorrection(ushort[] frameData, ushort[] darkFrame);
    ushort[] Apply2dDarkLut(ushort[] frameData, float temperature, float idleTime);
    ushort[] ApplyBadPixelCorrection(ushort[] frameData, bool[] badPixelMap);
}
```

### 3.3 LUT Service

```csharp
public interface ILutService
{
    float GetLutValue(float temperature, float idleTime);
    void LoadLutFromFile(string path);
    void LoadLutFromBytes(byte[] lutData);
}
```

---

## 4. Component Interaction

```
MainViewModel
    │
    ├──┬─> CaptureService ──> CommunicationService ──> i.MX8
    │
    ├──┼─> ImageProcessor ──> Raw Frame ──> Corrected Frame
    │
    ├──┼─> LutService ──> Temperature ──> Correction Factor
    │
    └──┴─> StorageService ──> File System
```

---

## 5. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial architecture |

---

**End of Document**
