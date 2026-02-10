# Data Models

## .NET Project Team Delivery Package

**Document**: 03_data_models.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Core Data Models

### 1.1 FrameData

```csharp
/// <summary>
/// Complete frame data with metadata
/// </summary>
public record FrameData
{
    /// <summary>
    /// 16-bit pixel data (2048 x 2048)
    /// </summary>
    public ushort[] PixelData { get; init; } = Array.Empty<ushort>();

    /// <summary>
    /// Frame metadata
    /// </summary>
    public FrameMetadata Metadata { get; init; } = new();

    /// <summary>
    /// Frame width in pixels
    /// </summary>
    public int Width => 2048;

    /// <summary>
    /// Frame height in pixels
    /// </summary>
    public int Height => 2048;

    /// <summary>
    /// Total pixel count
    /// </summary>
    public int PixelCount => Width * Height;

    /// <summary>
    /// Converts to byte array (big-endian)
    /// </summary>
    public byte[] ToBytes()
    {
        byte[] result = new byte[PixelCount * 2];
        for (int i = 0; i < PixelCount; i++)
        {
            result[i * 2] = (byte)(PixelData[i] >> 8);
            result[i * 2 + 1] = (byte)(PixelData[i] & 0xFF);
        }
        return result;
    }

    /// <summary>
    /// Creates from byte array
    /// </summary>
    public static FrameData FromBytes(byte[] data, FrameMetadata metadata)
    {
        int pixelCount = data.Length / 2;
        ushort[] pixelData = new ushort[pixelCount];
        for (int i = 0; i < pixelCount; i++)
        {
            pixelData[i] = (ushort)((data[i * 2] << 8) | data[i * 2 + 1]);
        }
        return new FrameData { PixelData = pixelData, Metadata = metadata };
    }
}
```

### 1.2 FrameMetadata

```csharp
/// <summary>
/// Metadata attached to each captured frame
/// </summary>
public record FrameMetadata
{
    /// <summary>
    /// Capture timestamp (UTC)
    /// </summary>
    public DateTimeOffset Timestamp { get; init; } = DateTimeOffset.UtcNow;

    /// <summary>
    /// Panel temperature at capture (°C)
    /// </summary>
    public float Temperature { get; init; }

    /// <summary>
    /// Idle time before capture (seconds)
    /// </summary>
    public uint IdleTimeBeforeCapture { get; init; }

    /// <summary>
    /// Idle mode before capture
    /// </summary>
    public IdleMode IdleModeBeforeCapture { get; init; }

    /// <summary>
    /// Whether this is a warm-up frame
    /// </summary>
    public bool IsWarmupFrame { get; init; }

    /// <summary>
    /// Current drift rate (DN/min)
    /// </summary>
    public float DriftRate { get; init; }

    /// <summary>
    /// Frame sequence number
    /// </summary>
    public uint SequenceNumber { get; init; }

    /// <summary>
    /// Serializes to binary format
    /// </summary>
    public byte[] ToBytes()
    {
        using var ms = new MemoryStream();
        using var writer = new BinaryWriter(ms);
        writer.Write(Timestamp.ToUnixTimeMilliseconds());
        writer.Write(Temperature);
        writer.Write(IdleTimeBeforeCapture);
        writer.Write((byte)IdleModeBeforeCapture);
        writer.Write(IsWarmupFrame);
        writer.Write(DriftRate);
        writer.Write(SequenceNumber);
        return ms.ToArray();
    }
}
```

---

## 2. Configuration Models

### 2.1 AppSettings

```csharp
/// <summary>
/// Application settings
/// </summary>
public class AppSettings
{
    public const string FileName = "appsettings.json";

    /// <summary>
    /// Connection settings
    /// </summary>
    public ConnectionSettings Connection { get; set; } = new();

    /// <summary>
    /// Display settings
    /// </summary>
    public DisplaySettings Display { get; set; } = new();

    /// <summary>
    /// Processing settings
    /// </summary>
    public ProcessingSettings Processing { get; set; } = new();

    /// <summary>
    /// Logging settings
    /// </summary>
    public LoggingSettings Logging { get; set; } = new();
}

public class ConnectionSettings
{
    public string Host { get; set; } = "192.168.1.100";
    public int Port { get; set; } = 5000;
    public int DataPort { get; set; } = 5001;
    public int TimeoutMs { get; set; } = 5000;
    public int RetryCount { get; set; } = 3;
}

public class DisplaySettings
{
    public string ColorMap { get; set; } = "Grayscale";
    public int MinValue { get; set; } = 0;
    public int MaxValue { get; set; } = 16383;
    public bool AutoScale { get; set; } = true;
    public bool ShowHistogram { get; set; } = true;
}

public class ProcessingSettings
{
    public bool EnableDarkCorrection { get; set; } = true;
    public bool Enable2dLut { get; set; } = true;
    public bool EnableBadPixelCorrection { get; set; } = true;
    public string DarkFramePath { get; set; } = "calibration/dark.bin";
    public string BadPixelMapPath { get; set; } = "calibration/badpixels.bin";
}

public class LoggingSettings
{
    public string LogLevel { get; set; } = "Information";
    public bool LogToFile { get; set; } = true;
    public string LogPath { get; set; } = "logs";
}
```

---

## 3. Communication Models

### 3.1 MessageHeader

```csharp
/// <summary>
/// Network message header
/// </summary>
public struct MessageHeader
{
    public const uint Magic = 0xA5A5A5A5;
    public const int Size = 24;

    public uint MagicNumber;      // 0xA5A5A5A5
    public uint Length;           // Payload length
    public uint MessageType;      // See MessageType enum
    public uint Sequence;         // Sequence number
    public ulong Timestamp;       // Unix timestamp (ms)

    public static MessageHeader Read(BinaryReader reader)
    {
        return new MessageHeader
        {
            MagicNumber = reader.ReadUInt32(),
            Length = reader.ReadUInt32(),
            MessageType = reader.ReadUInt32(),
            Sequence = reader.ReadUInt32(),
            Timestamp = reader.ReadUInt64()
        };
    }

    public void Write(BinaryWriter writer)
    {
        writer.Write(MagicNumber);
        writer.Write(Length);
        writer.Write(MessageType);
        writer.Write(Sequence);
        writer.Write(Timestamp);
    }
}
```

### 3.2 MessageType Enum

```csharp
/// <summary>
/// Message type identifiers
/// </summary>
public enum MessageType : uint
{
    // System messages
    Ping = 0x0001,
    Pong = 0x0002,

    // Status messages
    GetStatus = 0x0010,
    StatusResponse = 0x0011,

    // Frame messages
    StartFrame = 0x0020,
    FrameData = 0x0021,
    FrameComplete = 0x0022,
    AbortFrame = 0x0023,

    // Control messages
    SetBiasMode = 0x0030,
    GetTemperature = 0x0040,
    TemperatureResponse = 0x0041,

    // Error messages
    Error = 0x00FF
}
```

---

## 4. Calibration Models

### 4.1 DarkLutData

```csharp
/// <summary>
/// 2D Dark LUT data structure
/// </summary>
public class DarkLutData
{
    /// <summary>
    /// Temperature range (°C)
    /// </summary>
    public float MinTemperature { get; set; } = 15f;

    public float MaxTemperature { get; set; } = 40f;

    /// <summary>
    /// Idle time range (seconds)
    /// </summary>
    public uint MinIdleTime { get; set; } = 0;

    public uint MaxIdleTime { get; set; } = 6000;

    /// <summary>
    /// Temperature steps
    /// </summary>
    public int TemperatureSteps { get; set; } = 26; // 15-40°C

    /// <summary>
    /// Idle time steps
    /// </summary>
    public int IdleTimeSteps { get; set; } = 11; // 0-60 min

    /// <summary>
    /// LUT data [temperature, idle_time] = correction_factor
    /// </summary>
    public float[,] LutData { get; set; } = new float[26, 11];

    /// <summary>
    /// Gets interpolated correction factor
    /// </summary>
    public float GetCorrectionFactor(float temperature, uint idleTimeSeconds)
    {
        // Clamp to range
        temperature = Math.Clamp(temperature, MinTemperature, MaxTemperature);
        idleTimeSeconds = Math.Clamp(idleTimeSeconds, MinIdleTime, MaxIdleTime);

        // Calculate indices
        float tempIdx = (temperature - MinTemperature) / (MaxTemperature - MinTemperature) * (TemperatureSteps - 1);
        float idleIdx = (float)idleTimeSeconds / MaxIdleTime * (IdleTimeSteps - 1);

        int tempIdx0 = (int)tempIdx;
        int tempIdx1 = Math.Min(tempIdx0 + 1, TemperatureSteps - 1);
        float tempFrac = tempIdx - tempIdx0;

        int idleIdx0 = (int)idleIdx;
        int idleIdx1 = Math.Min(idleIdx0 + 1, IdleTimeSteps - 1);
        float idleFrac = idleIdx - idleIdx0;

        // Bilinear interpolation
        float v00 = LutData[tempIdx0, idleIdx0];
        float v01 = LutData[tempIdx0, idleIdx1];
        float v10 = LutData[tempIdx1, idleIdx0];
        float v11 = LutData[tempIdx1, idleIdx1];

        float v0 = v00 * (1 - idleFrac) + v01 * idleFrac;
        float v1 = v10 * (1 - idleFrac) + v11 * idleFrac;

        return v0 * (1 - tempFrac) + v1 * tempFrac;
    }
}
```

### 4.2 BadPixelMap

```csharp
/// <summary>
/// Bad pixel map for correction
/// </summary>
public class BadPixelMap
{
    private readonly bool[,] _badPixels;

    public BadPixelMap(int width, int height)
    {
        Width = width;
        Height = height;
        _badPixels = new bool[width, height];
    }

    public int Width { get; }
    public int Height { get; }

    /// <summary>
    /// Marks a pixel as bad
    /// </summary>
    public void MarkBad(int x, int y)
    {
        if (x >= 0 && x < Width && y >= 0 && y < Height)
            _badPixels[x, y] = true;
    }

    /// <summary>
    /// Checks if pixel is bad
    /// </summary>
    public bool IsBad(int x, int y)
    {
        if (x >= 0 && x < Width && y >= 0 && y < Height)
            return _badPixels[x, y];
        return false;
    }

    /// <summary>
    /// Loads from file
    /// </summary>
    public static BadPixelMap Load(string path)
    {
        // Format: binary, 1 bit per pixel
        byte[] data = File.ReadAllBytes(path);
        int width = BitConverter.ToInt32(data, 0);
        int height = BitConverter.ToInt32(data, 4);
        var map = new BadPixelMap(width, height);

        int byteIdx = 8;
        for (int y = 0; y < height; y++)
        {
            for (int x = 0; x < width; x++)
            {
                int bytePos = 8 + (y * width + x) / 8;
                int bitPos = (y * width + x) % 8;
                if (bytePos < data.Length)
                    map._badPixels[x, y] = (data[bytePos] & (1 << bitPos)) != 0;
            }
        }
        return map;
    }
}
```

---

## 5. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial data models |

---

**End of Document**
