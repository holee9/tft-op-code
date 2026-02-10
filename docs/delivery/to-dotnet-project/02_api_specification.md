# API Specification

## .NET Project Team Delivery Package

**Document**: 02_api_specification.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Communication API

### 1.1 ITftLeakageClient

```csharp
/// <summary>
/// Main client interface for TFT Leakage system communication
/// </summary>
public interface ITftLeakageClient : IDisposable
{
    /// <summary>
    /// Connects to the i.MX8 backend
    /// </summary>
    Task ConnectAsync(string host, int port = 5000, CancellationToken cancellationToken = default);

    /// <summary>
    /// Disconnects from the backend
    /// </summary>
    Task DisconnectAsync();

    /// <summary>
    /// Gets the current system status
    /// </summary>
    Task<SystemStatus> GetStatusAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Requests a frame capture
    /// </summary>
    Task<FrameResult> CaptureFrameAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Sets the bias mode
    /// </summary>
    Task SetBiasModeAsync(BiasMode mode, CancellationToken cancellationToken = default);

    /// <summary>
    /// Gets the current temperature
    /// </summary>
    Task<float> GetTemperatureAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Connection status event
    /// </summary>
    event Action<bool>? ConnectionChanged;

    /// <summary>
    /// Status update event
    /// </summary>
    event Action<SystemStatus>? StatusUpdated;

    /// <summary>
    /// Frame data event
    /// </summary>
    event Action<FrameData>? FrameReceived;
}
```

### 1.2 SystemStatus Model

```csharp
/// <summary>
/// System status from i.MX8 backend
/// </summary>
public record SystemStatus
{
    public float Temperature { get; init; }           // Panel temperature (°C)
    public uint IdleTimeSeconds { get; init; }        // Idle time (s)
    public IdleMode CurrentIdleMode { get; init; }    // L1/L2/L3
    public uint CalculatedTMax { get; init; }         // t_max at current T (s)
    public float CurrentDriftRate { get; init; }      // k(T) (DN/min)
    public bool WarmupRequired { get; init; }         // Warm-up needed flag
    public DateTimeOffset LastUpdate { get; init; }   // Update timestamp
}

public enum IdleMode : byte
{
    L1_Normal = 0,
    L2_LowBias = 1,
    L3_DeepSleep = 2
}
```

---

## 2. Image Processing API

### 2.1 IImageProcessor

```csharp
/// <summary>
/// Image processing interface for frame correction
/// </summary>
public interface IImageProcessor
{
    /// <summary>
    /// Applies dark frame subtraction
    /// </summary>
    ushort[] ApplyDarkCorrection(ushort[] frameData, ushort[] darkFrame);

    /// <summary>
    /// Applies 2D Dark LUT correction
    /// </summary>
    ushort[] Apply2dDarkLut(ushort[] frameData, float temperature, float idleTime);

    /// <summary>
    /// Applies bad pixel correction
    /// </summary>
    ushort[] ApplyBadPixelCorrection(ushort[] frameData, bool[] badPixelMap);

    /// <summary>
    /// Converts raw data to displayable image
    /// </summary>
    BitmapSource ConvertToImage(ushort[] frameData, int width, int height);

    /// <summary>
    /// Saves frame to file
    /// </summary>
    Task SaveFrameAsync(ushort[] frameData, string path, CancellationToken cancellationToken = default);
}
```

---

## 3. LUT Service API

### 3.1 IDarkLutService

```csharp
/// <summary>
/// 2D Dark LUT service for temperature/idle-time correction
/// </summary>
public interface IDarkLutService
{
    /// <summary>
    /// Gets correction factor for given temperature and idle time
    /// </summary>
    float GetCorrectionFactor(float temperature, float idleTime);

    /// <summary>
    /// Loads LUT from file
    /// </summary>
    Task LoadLutAsync(string path, CancellationToken cancellationToken = default);

    /// <summary>
    /// Loads LUT from embedded resource
    /// </summary>
    void LoadDefaultLut();

    /// <summary>
    /// Validates LUT data
    /// </summary>
    bool ValidateLut(byte[] lutData);

    /// <summary>
    /// LUT changed event
    /// </summary>
    event Action? LutUpdated;
}
```

---

## 4. Configuration API

### 4.1 IConfigurationService

```csharp
/// <summary>
/// Configuration management service
/// </summary>
public interface IConfigurationService
{
    /// <summary>
    /// Gets connection configuration
    /// </summary>
    ConnectionConfig GetConnectionConfig();

    /// <summary>
    /// Gets display configuration
    /// </summary>
    DisplayConfig GetDisplayConfig();

    /// <summary>
    /// Gets processing configuration
    /// </summary>
    ProcessingConfig GetProcessingConfig();

    /// <summary>
    /// Saves configuration
    /// </summary>
    Task SaveConfigAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Reloads configuration
    /// </summary>
    void ReloadConfig();

    /// <summary>
    /// Configuration changed event
    /// </summary>
    event Action? ConfigurationChanged;
}
```

### 4.2 Configuration Models

```csharp
public record ConnectionConfig
{
    public string Host { get; init; } = "192.168.1.100";
    public int Port { get; init; } = 5000;
    public int DataPort { get; init; } = 5001;
    public int TimeoutMs { get; init; } = 5000;
}

public record DisplayConfig
{
    public string ColorMap { get; init; } = "Grayscale";
    public int MinValue { get; init; } = 0;
    public int MaxValue { get; init; } = 16383;
    public bool AutoScale { get; init; } = true;
}

public record ProcessingConfig
{
    public bool EnableDarkCorrection { get; init; } = true;
    public bool Enable2dLut { get; init; } = true;
    public bool EnableBadPixelCorrection { get; init; } = true;
    public string DarkFramePath { get; init; } = "calibration/dark.bin";
    public string LutPath { get; init; } = "calibration/lut.bin";
}
```

---

## 5. Storage API

### 5.1 IStorageService

```csharp
/// <summary>
/// File storage service for frames and calibration data
/// </summary>
public interface IStorageService
{
    /// <summary>
    /// Saves frame data with metadata
    /// </summary>
    Task SaveFrameAsync(FrameData frame, CancellationToken cancellationToken = default);

    /// <summary>
    /// Loads frame data
    /// </summary>
    Task<FrameData> LoadFrameAsync(string path, CancellationToken cancellationToken = default);

    /// <summary>
    /// Saves dark frame
    /// </summary>
    Task SaveDarkFrameAsync(ushort[] data, CancellationToken cancellationToken = default);

    /// <summary>
    /// Loads dark frame
    /// </summary>
    Task<ushort[]> LoadDarkFrameAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Gets storage info
    /// </summary>
    Task<StorageInfo> GetStorageInfoAsync(CancellationToken cancellationToken = default);
}
```

---

## 6. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial API specification |

---

**End of Document**
