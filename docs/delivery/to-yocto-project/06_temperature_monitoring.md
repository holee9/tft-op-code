# Temperature Monitoring

## Yocto Project Team Delivery Package

**Document**: 06_temperature_monitoring.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Overview

The i.MX8 Plus is responsible for monitoring panel temperature via I2C sensor and calculating maximum allowable idle time using the Arrhenius model.

---

## 2. Temperature Sensor Requirements

### 2.1 Sensor Specifications

| Parameter | Value | Notes |
|-----------|-------|-------|
| **Type** | I2C Digital Temperature Sensor | - |
| **Recommended Part** | TMP100, TMP101, LM75A | Compatible alternatives |
| **Bus** | I2C-1 | 100 kHz |
| **Address** | 0x48-0x4F | Configurable |
| **Resolution** | 0.0625°C | TMP100 |
| **Accuracy** | ±2°C | -25°C to +85°C |
| **Range** | -55°C to +125°C | Full range |

### 2.2 Sensor Location

The temperature sensor should be placed:
- **Near the panel** to measure actual panel temperature
- **Away from heat sources** (FPGA, CPU)
- **In good airflow** for accurate readings

---

## 3. I2C Sensor Interface

### 3.1 TMP100 Register Map

| Register | Address | Description |
|----------|---------|-------------|
| **Temperature** | 0x00 | Temperature data (2 bytes) |
| **Configuration** | 0x01 | Sensor configuration |
| **T_LOW** | 0x02 | Low threshold |
| **T_HIGH** | 0x03 | High threshold |

### 3.2 Temperature Data Format

```
Temperature Register (0x00):

Byte 0 (MSB):   D13 D12 D11 D10 D09 D08 D07 D06
Byte 1 (LSB):   D05 D04 D03 D02 D01 D00 x x

Temperature (°C) = (D13..D0) / 256

Example:
  Byte 0 = 0x0C = 0000 1100
  Byte 1 = 0x80 = 1000 0000
  Raw = 0x0C80 = 3200
  Temperature = 3200 / 256 = 12.5°C
```

### 3.3 C# Implementation

```csharp
using System.Device.I2c;

public class TemperatureSensor : IDisposable
{
    private readonly I2cDevice _sensor;
    private const byte TEMP_REGISTER = 0x00;

    public TemperatureSensor(int busId = 1, byte address = 0x48)
    {
        var settings = new I2cConnectionSettings(busId, address)
        {
            BusSpeed = I2cBusSpeed.StandardMode // 100 kHz
        };
        _sensor = I2cDevice.Create(settings);
    }

    public float ReadTemperature()
    {
        // Point to temperature register
        _sensor.WriteByte(TEMP_REGISTER);

        // Read 2 bytes
        byte[] data = new byte[2];
        _sensor.Read(data);

        // Convert to temperature
        short rawValue = (short)((data[0] << 8) | data[1]);
        return rawValue / 256.0f;
    }

    public void Dispose()
    {
        _sensor?.Dispose();
    }
}
```

---

## 4. Temperature Monitoring Service

### 4.1 Service Requirements

| Requirement | Value |
|-------------|-------|
| **Polling Rate** | 1 Hz (1 sample per second) |
| **Filtering** | Exponential Moving Average (α=0.1) |
| **Update Rate** | 1 Hz to Arrhenius calculator |
| **Logging** | Every 60 seconds or on change >1°C |

### 4.2 C# Service Implementation

```csharp
using System.Device.Gpio;

public class TemperatureMonitor : BackgroundService
{
    private readonly TemperatureSensor _sensor;
    private readonly ILogger<TemperatureMonitor> _logger;
    private float _filteredTemperature = 25.0f;
    private const float Alpha = 0.1f; // Smoothing factor

    public event Action<float>? TemperatureUpdated;

    public TemperatureMonitor(ILogger<TemperatureMonitor> logger)
    {
        _logger = logger;
        _sensor = new TemperatureSensor();
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Temperature monitor started");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // Read raw temperature
                float rawTemp = _sensor.ReadTemperature();

                // Apply low-pass filter
                _filteredTemperature = Alpha * rawTemp + (1 - Alpha) * _filteredTemperature;

                // Notify subscribers
                TemperatureUpdated?.Invoke(_filteredTemperature);

                await Task.Delay(1000, stoppingToken); // 1 Hz
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to read temperature");
            }
        }
    }
}
```

---

## 5. Arrhenius Model Implementation

### 5.1 Arrhenius Equation

```
Drift Rate Calculation:

    k(T) = k_ref × exp[(E_A/k_B) × (1/T - 1/T_ref)]

Where:
    k(T)   = Drift rate at temperature T (DN/min)
    k_ref  = Reference drift rate (1.0 DN/min at 25°C)
    E_A    = Activation energy (0.45 eV for a-Si:H)
    k_B    = Boltzmann constant (8.617×10⁻⁵ eV/K)
    T      = Current temperature in Kelvin
    T_ref  = Reference temperature (298.15 K = 25°C)
```

### 5.2 Maximum Idle Time Calculation

```
t_max = ΔD_max / k(T)

Where:
    t_max   = Maximum allowable idle time (seconds)
    ΔD_max  = Maximum allowable dark increase (50 DN)
    k(T)    = Drift rate at current temperature
```

### 5.3 C# Implementation

```csharp
public class ArrheniusCalculator
{
    private const float ActivationEnergy_Ev = 0.45f;  // E_A for a-Si:H
    private const float BoltzmannConstant_EvPerK = 8.617e-5f;
    private const float ReferenceTemperature_K = 298.15f;  // 25°C
    private const float ReferenceDriftRate_DnPerMin = 1.0f;  // k_ref @ 25°C
    private const float Delta_D_Max_DN = 50.0f;

    public float CalculateDriftRate(float temperatureCelsius)
    {
        float temperatureK = temperatureCelsius + 273.15f;
        float exponent = (ActivationEnergy_Ev / BoltzmannConstant_EvPerK) *
                        (1.0f / temperatureK - 1.0f / ReferenceTemperature_K);
        return ReferenceDriftRate_DnPerMin * MathF.Exp(exponent);
    }

    public uint CalculateTMax(float temperatureCelsius)
    {
        float driftRate = CalculateDriftRate(temperatureCelsius);
        float tMaxMinutes = Delta_D_Max_DN / driftRate;
        return (uint)(tMaxMinutes * 60.0f); // Convert to seconds
    }
}
```

---

## 6. Look-Up Table

### 6.1 Pre-Calculated Values

| Temperature (°C) | Drift Rate (DN/min) | t_max (sec) | t_max (min) |
|------------------|---------------------|-------------|-------------|
| 15 | 0.50 | 6000 | 100 |
| 16 | 0.56 | 5357 | 89 |
| 17 | 0.63 | 4762 | 79 |
| 18 | 0.71 | 4225 | 70 |
| 19 | 0.80 | 3750 | 63 |
| 20 | 0.75 | 4000 | 67 |
| 21 | 0.85 | 3529 | 59 |
| 22 | 0.95 | 3158 | 53 |
| 23 | 1.07 | 2804 | 47 |
| 24 | 1.20 | 2500 | 42 |
| 25 | 1.00 | 3000 | 50 |
| 26 | 1.12 | 2679 | 45 |
| 27 | 1.26 | 2381 | 40 |
| 28 | 1.41 | 2128 | 35 |
| 29 | 1.58 | 1899 | 32 |
| 30 | 1.80 | 1667 | 28 |
| 31 | 2.02 | 1485 | 25 |
| 32 | 2.26 | 1327 | 22 |
| 33 | 2.54 | 1181 | 20 |
| 34 | 2.84 | 1056 | 18 |
| 35 | 3.20 | 938 | 16 |
| 36 | 3.58 | 838 | 14 |
| 37 | 4.02 | 746 | 12 |
| 38 | 4.51 | 666 | 11 |
| 39 | 5.06 | 593 | 10 |
| 40 | 5.80 | 517 | 9 |

### 6.2 LUT Implementation

```csharp
public static class DriftRateLUT
{
    private static readonly (int Temp, float Rate, uint TMax)[] Table = new[]
    {
        (15, 0.50f, 6000), (16, 0.56f, 5357), (17, 0.63f, 4762),
        (18, 0.71f, 4225), (19, 0.80f, 3750), (20, 0.75f, 4000),
        (21, 0.85f, 3529), (22, 0.95f, 3158), (23, 1.07f, 2804),
        (24, 1.20f, 2500), (25, 1.00f, 3000), (26, 1.12f, 2679),
        (27, 1.26f, 2381), (28, 1.41f, 2128), (29, 1.58f, 1899),
        (30, 1.80f, 1667), (31, 2.02f, 1485), (32, 2.26f, 1327),
        (33, 2.54f, 1181), (34, 2.84f, 1056), (35, 3.20f, 938),
        (36, 3.58f, 838), (37, 4.02f, 746), (38, 4.51f, 666),
        (39, 5.06f, 593), (40, 5.80f, 517)
    };

    public static (float Rate, uint TMax) GetValues(float temperature)
    {
        int temp = (int)MathF.Round(temperature);
        temp = Math.Clamp(temp, 15, 40);
        var entry = Table[temp - 15];
        return (entry.Rate, entry.TMax);
    }
}
```

---

## 7. Integration with Idle State Machine

### 7.1 State Decision Logic

```
Temperature Update (1 Hz):
    1. Read temperature from sensor
    2. Apply low-pass filter
    3. Calculate t_max using Arrhenius
    4. Update current idle time
    5. Decide state transition:

    IF idle_time > t_max × 0.8:
        L1 → L2 (Safety margin)
    ELSE IF idle_time > 100 minutes:
        L2 → L3
    ELSE IF frame_capture_requested:
        Any → L1
```

### 7.2 State Transition Table

| Current | Condition | Next | Action |
|---------|-----------|------|--------|
| L1 | t_idle > 10 min OR t_idle > t_max | L2 | Set IDLE_LOW_BIAS |
| L2 | t_idle > 100 min | L3 | Set SLEEP_BIAS |
| L2 | Capture request | L1 | Set NORMAL_BIAS |
| L3 | Capture request | L1 | Set NORMAL_BIAS + Warm-up |

---

## 8. Device Tree Configuration

### 8.1 Temperature Sensor Node

```dts
&i2c1 {
    clock-frequency = <100000>;
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_i2c1>;
    status = "okay";

    temp_sensor: tmp100@48 {
        compatible = "ti,tmp100";
        reg = <0x48>;
        #thermal-sensor-cells = <0>;
    };
};
```

### 8.2 Thermal Zone Integration

```dts
&cpu_thermal {
    polling-delay = <1000>; /* 1 second */
    polling-delay-passive = <1000>;

    sensors {
        sensor0 {
            compatible = "iio-sensor";
            io-channels = <&adc 0>;
        };
    };
};
```

---

## 9. Testing and Validation

### 9.1 Test Procedure

1. **Sensor Communication Test**
   ```bash
   # Detect I2C sensor
   i2cdetect -y 1
   # Should show device at 0x48
   ```

2. **Temperature Read Test**
   ```bash
   # Read temperature
   i2cget -y 1 0x48 0x00 w
   i2cget -y 1 0x48 0x01
   ```

3. **Drift Rate Verification**
   - Use thermal chamber
   - Measure at 15°C, 25°C, 35°C
   - Verify ±10% of calculated values

### 9.2 Acceptance Criteria

| Test | Criteria |
|------|----------|
| Sensor detectable | i2cdetect shows address |
| Read accuracy | ±2°C at 25°C |
| Update rate | 1 Hz ±5% |
| Filter stability | Settles within 10 samples |
| Arrhenius calculation | Matches LUT within 5% |

---

## 10. Troubleshooting

### 10.1 Common Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| Sensor not detected | i2cdetect shows -- | Check wiring, address, pull-ups |
| Wrong temperature | Unrealistic values | Verify data format, byte order |
| No updates | Temperature never changes | Check polling loop, event notification |
| Inaccurate values | Large errors | Calibrate sensor, check placement |

### 10.2 Debug Commands

```bash
# Check I2C bus
i2cdetect -y 1

# Dump registers
i2cdump -y 1 0x48

# Monitor temperature
watch -n 1 "cat /sys/class/i2c-adapter/i2c-1/1-0048/temp"

# Check thermal zone
cat /sys/class/thermal/thermal_zone0/temp
```

---

## 11. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial temperature monitoring document |

---

**End of Document**
