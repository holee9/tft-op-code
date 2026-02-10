# i.MX8 Plus .NET Control Specification
## MCU Control Software for TFT Leakage Management

**Version**: 1.0
**Created**: 2026-02-10
**Target Platform**: NXP i.MX8 Plus (Cortex-A53 + Cortex-M4)
**Framework**: .NET 8.0

---

## 1. Overview

This specification defines the .NET-based control software running on i.MX8 Plus for managing a-Si TFT FPD leakage through idle state control, temperature monitoring, and FPGA communication.

### 1.1 Scope

- Temperature monitoring (1 Hz polling)
- Arrhenius-based t_max calculation
- Idle state machine (L1/L2/L3)
- FPGA register access via SPI
- Frame metadata generation
- Host communication protocol

### 1.2 Exclusions

- Image processing (Host SW)
- Hardware-level ADC interface (Cortex-M4 HAL)

---

## 2. Architecture

### 2.1 System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     i.MX8 Plus Application                       │
├─────────────────────────────────────────────────────────────────┤
│  Cortex-A53 (Linux + .NET 8.0)                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  .NET Application Layer                                      ││
│  │  ┌───────────────────┐  ┌──────────────────────────────────┐││
│  │  │ TemperatureService│  │   IdleStateMachine               │││
│  │  │ - Polling (1 Hz)  │  │   - L1/L2/L3 transitions        │││
│  │  │ - Filtering       │  │   - Timer management            │││
│  │  └─────────┬─────────┘  └────────────┬─────────────────────┘││
│  │            │                          │                       ││
│  │  ┌─────────▼─────────┐  ┌───────────▼─────────────────────┐││
│  │  │ DriftModelService │  │   FpgaCommunicationService      │││
│  │  │ - Arrhenius calc  │  │   - SPI register access         │││
│  │  │ - t_max(T)        │  │   - Command serialization       │││
│  │  └─────────┬─────────┘  └────────────┬─────────────────────┘││
│  │            │                          │                       ││
│  │  ┌─────────▼──────────────────────────▼─────────────────────┐││
│  │  │              MetadataService                             │││
│  │  │ - Frame metadata assembly                                │││
│  │  │ - Temperature, idle_time history                          │││
│  │  └───────────────────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────────────────┘│
│                          │                                       │
│  ┌───────────────────────▼───────────────────────────────────────┐│
│  │              Inter-Core Communication (MU)                    ││
│  │  Messaging Unit between A53 and M4                           ││
│  └───────────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────────┤
│  Cortex-M4 (FreeRTOS)                                           │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Real-Time Layer                                            ││
│  │  - GPIO control                                             ││
│  │  - SPI/I2C/UART drivers                                     ││
│  │  - ADC sampling (temperature sensor)                        ││
│  │  - Hardware interrupt handling                              ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
         │                                    │
         ▼                                    ▼
    ┌─────────┐                          ┌─────────┐
    │  FPGA   │                          │  Host   │
    │(SPI IF) │                          │(USB/ETH)│
    └─────────┘                          └─────────┘
```

### 2.2 Module Dependencies

```
TftLeakageController (Main)
├── Core/
│   ├── TemperatureMonitor
│   ├── ArrheniusCalculator
│   ├── IdleStateMachine
│   └── SystemState
├── Hardware/
│   ├── FpgaSpiClient
│   ├── TemperatureSensor
│   └── GpioController
├── Services/
│   ├── MetadataService
│   ├── ConfigurationService
│   └── LoggingService
└── Protocols/
    ├── FpgaRegisterMap
    └── HostCommunicationProtocol
```

---

## 3. Data Models

### 3.1 System State

```csharp
namespace TftLeakage.Core.Models;

/// <summary>
/// System state including temperature, idle time, and mode
/// </summary>
public class SystemState
{
    /// <summary>
    /// Current panel temperature in Celsius
    /// </summary>
    public float Temperature { get; set; }

    /// <summary>
    /// Time elapsed since last reset/operation (seconds)
    /// </summary>
    public uint IdleTimeSeconds { get; set; }

    /// <summary>
    /// Current idle mode (L1, L2, L3)
    /// </summary>
    public IdleMode CurrentIdleMode { get; set; }

    /// <summary>
    /// Calculated maximum allowable idle time at current temperature (seconds)
    /// </summary>
    public uint CalculatedTMax { get; set; }

    /// <summary>
    /// Drift rate at current temperature (DN/min)
    /// </summary>
    public float CurrentDriftRate { get; set; }

    /// <summary>
    /// Whether warm-up frame is required before next exposure
    /// </summary>
    public bool WarmupRequired { get; set; }

    /// <summary>
    /// Last state transition timestamp
    /// </summary>
    public DateTimeOffset LastTransitionTime { get; set; }
}

/// <summary>
/// Idle mode levels
/// </summary>
public enum IdleMode
{
    /// <summary>
    /// L1: Normal idle (< 10 min), normal bias, no scan
    /// </summary>
    L1_Normal = 0,

    /// <summary>
    /// L2: Low-bias idle (10-100 min), low bias, periodic dummy scan
    /// </summary>
    L2_LowBias = 1,

    /// <summary>
    /// L3: Deep sleep (> 100 min), minimal bias, no scan
    /// </summary>
    L3_DeepSleep = 2
}
```

### 3.2 Frame Metadata

```csharp
namespace TftLeakage.Core.Models;

/// <summary>
/// Metadata attached to each frame
/// </summary>
public class FrameMetadata
{
    /// <summary>
    /// Frame capture timestamp (UTC)
    /// </summary>
    public DateTimeOffset Timestamp { get; set; }

    /// <summary>
    /// Panel temperature at capture time (Celsius)
    /// </summary>
    public float Temperature { get; set; }

    /// <summary>
    /// Idle time before this frame (seconds)
    /// </summary>
    public uint IdleTimeBeforeCapture { get; set; }

    /// <summary>
    /// Whether this is a warm-up frame (discard after correction)
    /// </summary>
    public bool IsWarmupFrame { get; set; }

    /// <summary>
    /// Idle mode before capture
    /// </summary>
    public IdleMode IdleModeBeforeCapture { get; set; }

    /// <summary>
    /// Current drift rate (DN/min)
    /// </summary>
    public float DriftRate { get; set; }

    /// <summary>
    /// Serializes to binary format for transmission
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

        return ms.ToArray();
    }
}
```

---

## 4. Temperature Monitoring

### 4.1 TemperatureMonitor Service

```csharp
namespace TftLeakage.Core.Services;

/// <summary>
/// Monitors panel temperature via ADC I2C sensor
/// </summary>
public class TemperatureMonitor : BackgroundService
{
    private readonly ILogger<TemperatureMonitor> _logger;
    private readonly ITemperatureSensor _sensor;
    private readonly SystemState _state;
    private readonly PeriodicTimer _timer;

    public event Action<float>? TemperatureUpdated;

    public TemperatureMonitor(
        ITemperatureSensor sensor,
        SystemState state,
        ILogger<TemperatureMonitor> logger)
    {
        _sensor = sensor;
        _state = state;
        _logger = logger;

        // 1 Hz polling
        _timer = new PeriodicTimer(TimeSpan.FromSeconds(1));
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Temperature monitor started");

        while (await _timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                float temperature = await _sensor.ReadTemperatureAsync();

                // Apply low-pass filter (exponential moving average)
                float filtered = FilterTemperature(temperature, _state.Temperature);
                _state.Temperature = filtered;

                TemperatureUpdated?.Invoke(filtered);

                _logger.LogTrace("Temperature: {Temperature}°C", filtered);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to read temperature");
            }
        }
    }

    private float FilterTemperature(float newValue, float oldValue)
    {
        const float alpha = 0.1f; // Smoothing factor
        return alpha * newValue + (1 - alpha) * oldValue;
    }
}
```

### 4.2 ITemperatureSensor Interface

```csharp
namespace TftLeakage.Core.Hardware;

/// <summary>
/// Interface for temperature sensor (e.g., TMP100, LM75 via I2C)
/// </summary>
public interface ITemperatureSensor
{
    /// <summary>
    /// Reads temperature in Celsius
    /// </summary>
    Task<float> ReadTemperatureAsync(CancellationToken cancellationToken = default);
}

/// <summary>
/// I2C temperature sensor implementation
/// </summary>
public class I2cTemperatureSensor : ITemperatureSensor, IDisposable
{
    private readonly I2cDevice _device;
    private const byte TemperatureRegister = 0x00;

    public I2cTemperatureSensor(int busId, byte deviceAddress)
    {
        var settings = new I2cConnectionSettings(busId, deviceAddress)
        {
            BusSpeed = I2cBusSpeed.FastMode
        };
        _device = I2cDevice.Create(settings);
    }

    public async Task<float> ReadTemperatureAsync(CancellationToken cancellationToken = default)
    {
        _device.WriteByte(TemperatureRegister);

        byte[] data = new byte[2];
        _device.Read(data);

        // Parse temperature (TMP100 format)
        short rawValue = (short)((data[0] << 8) | data[1]);
        return rawValue / 256.0f;
    }

    public void Dispose() => _device?.Dispose();
}
```

---

## 5. Arrhenius Drift Model

### 5.1 ArrheniusCalculator Service

```csharp
namespace TftLeakage.Core.Services;

/// <summary>
/// Calculates drift rate and t_max using Arrhenius model
/// </summary>
public class ArrheniusCalculator
{
    // Physical constants
    private const float ActivationEnergy_Ev = 0.45f;  // E_A for a-Si:H
    private const float BoltzmannConstant_EvPerK = 8.617e-5f;

    // Reference values
    private const float ReferenceTemperature_K = 298.15f;  // 25°C in Kelvin
    private const float ReferenceDriftRate_DnPerMin = 1.0f;  // k_ref @ 25°C

    // Maximum allowable dark increase
    private const float Delta_D_Max_DN = 50.0f;

    /// <summary>
    /// Calculates drift rate at given temperature using Arrhenius equation
    /// k(T) = k_ref * exp((E_A / k_B) * (1/T - 1/T_ref))
    /// </summary>
    public float CalculateDriftRate(float temperatureCelsius)
    {
        float temperatureK = temperatureCelsius + 273.15f;

        float exponent = (ActivationEnergy_Ev / BoltzmannConstant_EvPerK) *
                        (1.0f / temperatureK - 1.0f / ReferenceTemperature_K);

        float driftRate = ReferenceDriftRate_DnPerMin * MathF.Exp(exponent);

        return driftRate;
    }

    /// <summary>
    /// Calculates maximum allowable idle time at given temperature
    /// t_max = Delta_D_max / k(T)
    /// </summary>
    public uint CalculateTMax(float temperatureCelsius)
    {
        float driftRate = CalculateDriftRate(temperatureCelsius);
        float tMaxMinutes = Delta_D_Max_DN / driftRate;
        uint tMaxSeconds = (uint)(tMaxMinutes * 60.0f);

        return tMaxSeconds;
    }

    /// <summary>
    /// Pre-calculated drift rate LUT for 15-40°C range
    /// </summary>
    public static readonly Dictionary<int, float> DriftRateLUT = new()
    {
        { 15, 0.50f },  // DN/min
        { 20, 0.75f },
        { 25, 1.00f },
        { 30, 1.80f },
        { 35, 3.20f },
        { 40, 5.80f }
    };
}
```

---

## 6. Idle State Machine

### 6.1 IdleStateMachine Service

```csharp
namespace TftLeakage.Core.Services;

/// <summary>
/// Manages L1/L2/L3 idle mode transitions
/// </summary>
public class IdleStateMachine : BackgroundService
{
    private readonly ILogger<IdleStateMachine> _logger;
    private readonly SystemState _state;
    private readonly IFpgaClient _fpgaClient;
    private readonly ArrheniusCalculator _calculator;
    private readonly PeriodicTimer _timer;
    private DateTimeOffset _lastResetTime;

    public event Action<IdleModeTransition>? ModeTransition;

    public IdleStateMachine(
        SystemState state,
        IFpgaClient fpgaClient,
        ArrheniusCalculator calculator,
        ILogger<IdleStateMachine> logger)
    {
        _state = state;
        _fpgaClient = fpgaClient;
        _calculator = calculator;
        _logger = logger;

        _timer = new PeriodicTimer(TimeSpan.FromSeconds(1));
        _lastResetTime = DateTimeOffset.UtcNow;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Idle state machine started");

        while (await _timer.WaitForNextTickAsync(stoppingToken))
        {
            _state.IdleTimeSeconds = (uint)(DateTimeOffset.UtcNow - _lastResetTime).TotalSeconds;

            // Update t_max calculation based on current temperature
            _state.CalculatedTMax = _calculator.CalculateTMax(_state.Temperature);
            _state.CurrentDriftRate = _calculator.CalculateDriftRate(_state.Temperature);

            // Check if mode transition is needed
            IdleMode newMode = DetermineNextMode();

            if (newMode != _state.CurrentIdleMode)
            {
                await TransitionToMode(newMode);
            }
        }
    }

    private IdleMode DetermineNextMode()
    {
        uint tIdle = _state.IdleTimeSeconds;
        uint tMax = _state.CalculatedTMax;

        // L1 → L2 transition
        if (_state.CurrentIdleMode == IdleMode.L1_Normal)
        {
            if (tIdle > TimeSpan.FromMinutes(10).TotalSeconds() ||
                tIdle > tMax * 0.8)  // 80% of t_max as safety margin
            {
                return IdleMode.L2_LowBias;
            }
        }

        // L2 → L3 transition
        if (_state.CurrentIdleMode == IdleMode.L2_LowBias)
        {
            if (tIdle > TimeSpan.FromMinutes(100).TotalSeconds())
            {
                return IdleMode.L3_DeepSleep;
            }
        }

        return _state.CurrentIdleMode;
    }

    private async Task TransitionToMode(IdleMode newMode)
    {
        _logger.LogInformation(
            "Transitioning from {OldMode} to {NewMode} (Idle: {Idle}s, T_max: {TMax}s)",
            _state.CurrentIdleMode,
            newMode,
            _state.IdleTimeSeconds,
            _state.CalculatedTMax);

        // Send bias mode change command to FPGA
        await _fpgaClient.SetBiasModeAsync(MapBiasMode(newMode));

        // Configure dummy scan for L2
        if (newMode == IdleMode.L2_LowBias)
        {
            await _fpgaClient.EnableDummyScanAsync(periodSeconds: 60);
        }
        else
        {
            await _fpgaClient.DisableDummyScanAsync();
        }

        var transition = new IdleModeTransition
        {
            From = _state.CurrentIdleMode,
            To = newMode,
            Timestamp = DateTimeOffset.UtcNow,
            IdleTime = _state.IdleTimeSeconds,
            Temperature = _state.Temperature
        };

        _state.CurrentIdleMode = newMode;
        _state.LastTransitionTime = DateTimeOffset.UtcNow;

        ModeTransition?.Invoke(transition);
    }

    private BiasMode MapBiasMode(IdleMode idleMode)
    {
        return idleMode switch
        {
            IdleMode.L1_Normal => BiasMode.NormalBias,
            IdleMode.L2_LowBias => BiasMode.IdleLowBias,
            IdleMode.L3_DeepSleep => BiasMode.SleepBias,
            _ => BiasMode.NormalBias
        };
    }

    /// <summary>
    /// Call when frame capture is requested (resets idle timer)
    /// </summary>
    public void ResetIdleTimer()
    {
        _lastResetTime = DateTimeOffset.UtcNow;
        _state.IdleTimeSeconds = 0;

        // Check if warm-up is needed
        _state.WarmupRequired = (_state.CurrentIdleMode == IdleMode.L2_LowBias &&
                                 _state.IdleTimeSeconds > TimeSpan.FromMinutes(30).TotalSeconds()) ||
                                (_state.CurrentIdleMode == IdleMode.L3_DeepSleep);

        // Transition back to L1 on activity
        if (_state.CurrentIdleMode != IdleMode.L1_Normal)
        {
            _ = TransitionToMode(IdleMode.L1_Normal);
        }
    }
}

public record IdleModeTransition
{
    public IdleMode From { get; init; }
    public IdleMode To { get; init; }
    public DateTimeOffset Timestamp { get; init; }
    public uint IdleTime { get; init; }
    public float Temperature { get; init; }
}

public enum BiasMode : byte
{
    NormalBias = 0b00,
    IdleLowBias = 0b01,
    SleepBias = 0b10
}
```

---

## 7. FPGA Communication

### 7.1 IFpgaClient Interface

```csharp
namespace TftLeakage.Core.Hardware;

/// <summary>
/// Interface for FPGA communication via SPI
/// </summary>
public interface IFpgaClient
{
    /// <summary>
    /// Reads a register from FPGA
    /// </summary>
    Task<byte> ReadRegisterAsync(byte address, CancellationToken cancellationToken = default);

    /// <summary>
    /// Writes to a register in FPGA
    /// </summary>
    Task WriteRegisterAsync(byte address, byte value, CancellationToken cancellationToken = default);

    /// <summary>
    /// Sets bias mode
    /// </summary>
    Task SetBiasModeAsync(BiasMode mode, CancellationToken cancellationToken = default);

    /// <summary>
    /// Enables periodic dummy scan
    /// </summary>
    Task EnableDummyScanAsync(uint periodSeconds, CancellationToken cancellationToken = default);

    /// <summary>
    /// Disables dummy scan
    /// </summary>
    Task DisableDummyScanAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Starts frame capture
    /// </summary>
    Task StartFrameCaptureAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Gets FPGA status
    /// </summary>
    Task<FpgaStatus> GetStatusAsync(CancellationToken cancellationToken = default);
}

public record FpgaStatus
{
    public bool FrameBusy { get; init; }
    public bool DummyActive { get; init; }
    public bool BiasModeReady { get; init; }
    public bool FifoEmpty { get; init; }
    public bool Idle { get; init; }
    public bool PowerGood { get; init; }
}
```

### 7.2 SpiFpgaClient Implementation

```csharp
namespace TftLeakage.Core.Hardware;

/// <summary>
/// SPI-based FPGA client implementation
/// </summary>
public class SpiFpgaClient : IFpgaClient, IDisposable
{
    private readonly SpiDevice _spiDevice;
    private readonly ILogger<SpiFpgaClient> _logger;

    // Register addresses (from FPGA spec)
    private const byte REG_CTRL = 0x00;
    private const byte REG_STATUS = 0x01;
    private const byte REG_BIAS_SELECT = 0x02;
    private const byte REG_DUMMY_PERIOD = 0x03;
    private const byte REG_DUMMY_CONTROL = 0x04;

    // Control register bits
    private const byte CTRL_FRAME_START = 0x80;
    private const byte CTRL_FRAME_RESET = 0x40;
    private const byte CTRL_DUMMY_ENABLE = 0x20;

    public SpiFpgaClient(SpiConnectionSettings settings, ILogger<SpiFpgaClient> logger)
    {
        _spiDevice = SpiDevice.Create(settings);
        _logger = logger;
    }

    public async Task<byte> ReadRegisterAsync(byte address, CancellationToken cancellationToken = default)
    {
        byte[] writeBuffer = { 0x02, address, 0x00 };  // READ_CMD, ADDR, DUMMY
        byte[] readBuffer = new byte[3];

        _spiDevice.TransferFullDuplex(writeBuffer, readBuffer);
        await Task.Delay(1, cancellationToken);  // SPI propagation delay

        return readBuffer[2];
    }

    public async Task WriteRegisterAsync(byte address, byte value, CancellationToken cancellationToken = default)
    {
        byte[] writeBuffer = { 0x01, address, value };  // WRITE_CMD, ADDR, DATA

        _spiDevice.Write(writeBuffer);
        await Task.Delay(1, cancellationToken);

        _logger.LogTrace("Wrote register {Address} = {Value:X2}", address, value);
    }

    public async Task SetBiasModeAsync(BiasMode mode, CancellationToken cancellationToken = default)
    {
        await WriteRegisterAsync(REG_BIAS_SELECT, (byte)mode, cancellationToken);

        // Wait for bias to settle (< 10 µs, but add margin)
        await Task.Delay(1, cancellationToken);

        // Poll status until BIAS_MODE_READY
        while (!cancellationToken.IsCancellationRequested)
        {
            FpgaStatus status = await GetStatusAsync(cancellationToken);
            if (status.BiasModeReady)
                break;
        }
    }

    public async Task EnableDummyScanAsync(uint periodSeconds, CancellationToken cancellationToken = default)
    {
        // Set period (16-bit register, low byte first)
        await WriteRegisterAsync(REG_DUMMY_PERIOD, (byte)(periodSeconds & 0xFF), cancellationToken);
        await WriteRegisterAsync(REG_DUMMY_PERIOD + 1, (byte)((periodSeconds >> 8) & 0xFF), cancellationToken);

        // Enable dummy scan
        await WriteRegisterAsync(REG_DUMMY_CONTROL, CTRL_DUMMY_ENABLE, cancellationToken);
    }

    public async Task DisableDummyScanAsync(CancellationToken cancellationToken = default)
    {
        await WriteRegisterAsync(REG_DUMMY_CONTROL, 0x00, cancellationToken);
    }

    public async Task StartFrameCaptureAsync(CancellationToken cancellationToken = default)
    {
        await WriteRegisterAsync(REG_CTRL, CTRL_FRAME_START, cancellationToken);
    }

    public async Task<FpgaStatus> GetStatusAsync(CancellationToken cancellationToken = default)
    {
        byte status = await ReadRegisterAsync(REG_STATUS, cancellationToken);

        return new FpgaStatus
        {
            FrameBusy = (status & 0x80) != 0,
            DummyActive = (status & 0x40) != 0,
            BiasModeReady = (status & 0x20) != 0,
            FifoEmpty = (status & 0x10) != 0,
            Idle = (status & 0x02) != 0,
            PowerGood = (status & 0x01) != 0
        };
    }

    public void Dispose() => _spiDevice?.Dispose();
}
```

---

## 8. Control Commands

### 8.1 Command Protocol Definition

```csharp
namespace TftLeakage.Core.Protocols;

/// <summary>
/// Command protocol for Host → MCU communication
/// </summary>
public enum HostCommand : byte
{
    // System commands
    Ping = 0x00,
    GetStatus = 0x01,
    Reset = 0x02,

    // Frame operations
    StartFrameCapture = 0x10,
    AbortFrameCapture = 0x11,

    // Idle mode control
    ForceIdleMode = 0x20,
    GetIdleState = 0x21,

    // Configuration
    SetTemperatureThreshold = 0x30,
    SetDummyScanPeriod = 0x31,

    // Diagnostics
    GetTemperatureHistory = 0x40,
    GetDriftRateTable = 0x41
}

/// <summary>
/// Response codes
/// </summary>
public enum ResponseCode : byte
{
    Success = 0x00,
    InvalidCommand = 0x01,
    InvalidParameter = 0x02,
    DeviceBusy = 0x03,
    HardwareError = 0x04,
    Timeout = 0x05
}
```

### 8.2 HostCommandProcessor

```csharp
namespace TftLeakage.Core.Services;

/// <summary>
/// Processes commands from Host via USB/Ethernet
/// </summary>
public class HostCommandProcessor
{
    private readonly IdleStateMachine _idleStateMachine;
    private readonly IFpgaClient _fpgaClient;
    private readonly SystemState _state;
    private readonly MetadataService _metadataService;

    public async Task<byte[]> ProcessCommandAsync(byte[] command)
    {
        if (command.Length == 0)
            return new byte[] { (byte)ResponseCode.InvalidCommand };

        HostCommand cmd = (HostCommand)command[0];
        byte[] payload = command.Length > 1 ? command[1..] : Array.Empty<byte>();

        try
        {
            return cmd switch
            {
                HostCommand.Ping => HandlePing(),
                HostCommand.GetStatus => await HandleGetStatusAsync(),
                HostCommand.StartFrameCapture => await HandleStartFrameCaptureAsync(),
                HostCommand.ForceIdleMode => await HandleForceIdleModeAsync(payload),
                HostCommand.GetIdleState => HandleGetIdleState(),
                _ => new byte[] { (byte)ResponseCode.InvalidCommand }
            };
        }
        catch (Exception ex)
        {
            return new byte[] { (byte)ResponseCode.HardwareError };
        }
    }

    private byte[] HandlePing()
    {
        return new byte[] { (byte)ResponseCode.Success, 0xA5 };  // 0xA5 = pong
    }

    private async Task<byte[]> HandleGetStatusAsync()
    {
        var status = await _fpgaClient.GetStatusAsync();

        using var ms = new MemoryStream();
        using var writer = new BinaryWriter(ms);

        writer.Write((byte)ResponseCode.Success);
        writer.Write((byte)_state.CurrentIdleMode);
        writer.Write(_state.Temperature);
        writer.Write(_state.IdleTimeSeconds);
        writer.Write(_state.CalculatedTMax);
        writer.Write(_state.CurrentDriftRate);
        writer.Write(_state.WarmupRequired);

        return ms.ToArray();
    }

    private async Task<byte[]> HandleStartFrameCaptureAsync()
    {
        // Reset idle timer
        _idleStateMachine.ResetIdleTimer();

        // Check if warm-up is needed
        if (_state.WarmupRequired)
        {
            // Capture warm-up frame first (dark only)
            await _fpgaClient.StartFrameCaptureAsync();
            await Task.Delay(150);  // Wait for frame completion

            // Mark warm-up frame in metadata
            var warmupMetadata = _metadataService.CreateMetadata(isWarmup: true);

            // Now capture clinical frame
            await _fpgaClient.StartFrameCaptureAsync();
            var clinicalMetadata = _metadataService.CreateMetadata(isWarmup: false);

            // Return both metadata
            using var ms = new MemoryStream();
            using var writer = new BinaryWriter(ms);
            writer.Write((byte)ResponseCode.Success);
            writer.Write((byte)0x02);  // Two frames
            writer.Write(warmupMetadata.ToBytes());
            writer.Write(clinicalMetadata.ToBytes());

            return ms.ToArray();
        }
        else
        {
            // Normal single frame capture
            await _fpgaClient.StartFrameCaptureAsync();
            var metadata = _metadataService.CreateMetadata(isWarmup: false);

            using var ms = new MemoryStream();
            using var writer = new BinaryWriter(ms);
            writer.Write((byte)ResponseCode.Success);
            writer.Write((byte)0x01);  // One frame
            writer.Write(metadata.ToBytes());

            return ms.ToArray();
        }
    }

    private async Task<byte[]> HandleForceIdleModeAsync(byte[] payload)
    {
        if (payload.Length == 0)
            return new byte[] { (byte)ResponseCode.InvalidParameter };

        IdleMode mode = (IdleMode)payload[0];

        // Force mode transition (bypassing normal logic)
        // Use with caution - mainly for diagnostics
        return new byte[] { (byte)ResponseCode.Success };
    }

    private byte[] HandleGetIdleState()
    {
        using var ms = new MemoryStream();
        using var writer = new BinaryWriter(ms);

        writer.Write((byte)ResponseCode.Success);
        writer.Write((byte)_state.CurrentIdleMode);
        writer.Write(_state.IdleTimeSeconds);
        writer.Write(_state.CalculatedTMax);

        return ms.ToArray();
    }
}
```

---

## 9. Configuration

### 9.1 appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "TftLeakage.Core.Services": "Debug"
    }
  },
  "TemperatureSensor": {
    "BusId": 1,
    "DeviceAddress": 72,
    "PollingIntervalHz": 1,
    "FilterAlpha": 0.1
  },
  "FpgaSpi": {
    "BusId": 0,
    "ChipSelectLine": 0,
    "Mode": 0,
    "ClockFrequency": 1000000
  },
  "IdleMode": {
    "L1_MaxTimeSeconds": 600,
    "L2_MaxTimeSeconds": 6000,
    "L2_DummyScanPeriodSeconds": 60,
    "DeltaDMaxDN": 50
  },
  "DriftModel": {
    "ActivationEnergyEv": 0.45,
    "ReferenceTemperatureC": 25,
    "ReferenceDriftRateDnPerMin": 1.0
  },
  "Warmup": {
    "RequiredIdleTimeMinutes": 30
  }
}
```

---

## 10. Deployment

### 10.1 Yocto Recipe

```bitbake
SUMMARY = "TFT Leakage Controller for i.MX8 Plus"
LICENSE = "CLOSED"
SRC_URI = "file://TftLeakageController-${PV}.tar.gz"

DEPENDS = "dotnet-native libgpiod"

RDEPENDS_${PN} = "dotnet-runtime bash"

FILES_${PN} += "\
    ${bindir}/TftLeakageController \
    ${sysconfdir}/tft-leakage/* \
    ${systemd_system_unitdir}/* \
"

do_install() {
    # Install binary
    install -d ${D}${bindir}
    install -m 0755 ${S}/TftLeakageController ${D}${bindir}/

    # Install configuration
    install -d ${D}${sysconfdir}/tft-leakage
    install -m 0644 ${S}/appsettings.json ${D}${sysconfdir}/tft-leakage/

    # Install systemd service
    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${S}/tft-leakage.service ${D}${systemd_system_unitdir}/
}

SYSTEMD_SERVICE_${PN} = "tft-leakage.service"
```

### 10.2 Systemd Service

```ini
[Unit]
Description=TFT Leakage Controller
After=network.target

[Service]
Type=notify
ExecStart=/usr/bin/dotnet /usr/bin/TftLeakageController.dll
Restart=always
RestartSec=10

# Environment
Environment=DOTNET_ENVIRONMENT=Production
Environment=ASPNETCORE_URLS=http://0.0.0.0:5000

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/tft-leakage /var/log

[Install]
WantedBy=multi-user.target
```

---

## 11. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial specification |

---

**End of Specification**
