# Userspace Interface

## Yocto Project Team Delivery Package

**Document**: 05_userspace_interface.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Overview

This document defines the userspace APIs for accessing hardware peripherals from the .NET application.

---

## 2. SPI Interface (/dev/spidevX.Y)

### 2.1 Device Node

| Device | Description |
|--------|-------------|
| `/dev/spidev1.0` | ECSPI2, chip select 0 (FPGA interface) |

### 2.2 IOCTL Commands

```c
#define SPI_IOC_MAGIC 'k'
#define SPI_IOC_WR_MODE _IOW(SPI_IOC_MAGIC, 1, __u8)
#define SPI_IOC_RD_MODE _IOR(SPI_IOC_MAGIC, 1, __u8)
#define SPI_IOC_WR_BITS_PER_WORD _IOW(SPI_IOC_MAGIC, 3, __u8)
#define SPI_IOC_RD_BITS_PER_WORD _IOR(SPI_IOC_MAGIC, 3, __u8)
#define SPI_IOC_WR_MAX_SPEED_HZ _IOW(SPI_IOC_MAGIC, 4, __u32)
#define SPI_IOC_RD_MAX_SPEED_HZ _IOR(SPI_IOC_MAGIC, 4, __u32)
#define SPI_IOC_MESSAGE(N) _IOW(SPI_IOC_MAGIC, 0, struct spi_ioc_transfer) + N
```

### 2.3 C# Wrapper (System.Device.Spi)

```csharp
using System.Device.Spi;

// SPI configuration
var settings = new SpiConnectionSettings(busId: 1, chipSelectLine: 0)
{
    Mode = SpiMode.Mode0,           // CPOL=0, CPHA=0
    ClockFrequency = 10_000_000,    // 10 MHz
    DataBitLength = 8,
    ChipSelectLineActiveState = PinValue.Low
};

using var spi = SpiDevice.Create(settings);

// Write register
spi.Write(new byte[] { 0x01, address, data }); // CMD, ADDR, DATA

// Read register
byte[] writeBuffer = new byte[] { 0x02, address, 0x00 };
byte[] readBuffer = new byte[3];
spi.TransferFullDuplex(writeBuffer, readBuffer);
byte value = readBuffer[2];
```

### 2.4 Direct IOCTL Example (C)

```c
#include <fcntl.h>
#include <linux/spi/spidev.h>
#include <sys/ioctl.h>

int fd = open("/dev/spidev1.0", O_RDWR);

// Set mode
uint8_t mode = SPI_MODE_0;
ioctl(fd, SPI_IOC_WR_MODE, &mode);

// Set speed
uint32_t speed = 10000000;
ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);

// Transfer
struct spi_ioc_transfer tr = {
    .tx_buf = (unsigned long)tx_buf,
    .rx_buf = (unsigned long)rx_buf,
    .len = 3,
    .speed_hz = speed,
    .bits_per_word = 8,
};
ioctl(fd, SPI_IOC_MESSAGE(1), &tr);
```

---

## 3. I2C Interface (/dev/i2c-X)

### 3.1 Device Node

| Device | Description |
|--------|-------------|
| `/dev/i2c-1` | I2C adapter 1 (temperature sensor) |

### 3.2 I2C Commands

```c
#include <linux/i2c-dev.h>
#include <fcntl.h>
#include <sys/ioctl.h>

int fd = open("/dev/i2c-1", O_RDWR);

// Select device
ioctl(fd, I2C_SLAVE, 0x48); // TMP100 address

// Write register
uint8_t reg = 0x00; // Temperature register
write(fd, &reg, 1);

// Read data
uint8_t data[2];
read(fd, data, 2);
```

### 3.3 C# Wrapper (I2cDevice)

```csharp
using System.Device.I2c;

// I2C configuration
var settings = new I2cConnectionSettings(busId: 1, deviceAddress: 0x48)
{
    BusSpeed = I2cBusSpeed.FastMode
};

using var i2c = I2cDevice.Create(settings);

// Read temperature (2 bytes, big-endian)
i2c.WriteByte(0x00); // Select temperature register
byte[] data = new byte[2];
i2c.Read(data);

// Convert to temperature (TMP100 format)
short rawValue = (short)((data[0] << 8) | data[1]);
float temperature = rawValue / 256.0f;
```

### 3.4 i2c-tools Usage

```bash
# Scan I2C bus
i2cdetect -y 1

# Read byte
i2cget -y 1 0x48 0x00

# Write byte
i2cset -y 1 0x48 0x01 0x60
```

---

## 4. GPIO Interface (/dev/gpiochipX)

### 4.1 Device Nodes

| Device | Description |
|--------|-------------|
| `/dev/gpiochip0` | GPIO Bank 1 (32 pins) |
| `/dev/gpiochip1` | GPIO Bank 2 (32 pins) |
| `/dev/gpiochip2` | GPIO Bank 3 (32 pins) |
| `/dev/gpiochip3` | GPIO Bank 4 (32 pins) |
| `/dev/gpiochip4` | GPIO Bank 5 (32 pins) |

### 4.2 libgpiod C API

```c
#include <gpiod.h>

struct gpiod_chip *chip;
struct gpiod_line *line;
int req;

// Open chip
chip = gpiod_chip_open("/dev/gpiochip4");

// Get line
line = gpiod_chip_get_line(chip, 24); // GPIO4_IO24

// Request output
req = gpiod_line_request_output(line, "fpga-bias-req", 0);

// Set value
gpiod_line_set_value(line, 1);

// Release
gpiod_line_release(line);
gpiod_chip_close(chip);
```

### 4.3 C# Wrapper (GpioController)

```csharp
using System.Device.Gpio;

var controller = new GpioController();

// Open pin (Linux pin number)
int biasReqPin = 24 + 128; // GPIO4_IO24 = 4*32 + 24
controller.OpenPin(biasReqPin, PinMode.Output);

// Set value
controller.Write(biasReqPin, PinValue.High);

// Cleanup
controller.ClosePin(biasReqPin);
```

### 4.4 GPIO Sysfs Interface (Legacy)

```bash
# Export GPIO
echo 152 > /sys/class/gpio/export  # GPIO4_IO24 = 4*32 + 24 = 152

# Set direction
echo out > /sys/class/gpio/gpio152/direction

# Set value
echo 1 > /sys/class/gpio/gpio152/value

# Read value
cat /sys/class/gpio/gpio152/value

# Unexport
echo 152 > /sys/class/gpio/unexport
```

---

## 5. Network Interface (Ethernet)

### 5.1 Network Configuration

| Interface | Purpose |
|-----------|---------|
| `eth0` | EQOS Ethernet (host communication) |

### 5.2 IP Configuration

```bash
# Static IP
ip addr add 192.168.1.100/24 dev eth0
ip link set eth0 up

# Or use DHCP
udhcpc -i eth0

# Default route
ip route add default via 192.168.1.1
```

### 5.3 systemd-networkd Configuration

Create `/etc/systemd/network/10-eth0.network`:

```ini
[Match]
Name=eth0

[Network]
DHCP=yes
# Or static:
# Address=192.168.1.100/24
# Gateway=192.168.1.1
# DNS=8.8.8.8
```

### 5.4 C# Socket Example

```csharp
using System.Net;
using System.Net.Sockets;

// UDP client for FPGA data
using var udp = new UdpClient();
udp.Connect(new IPEndPoint(IPAddress.Parse("192.168.1.1"), 5000));

// Send command
byte[] command = Encoding.UTF8.GetBytes("START_FRAME");
udp.Send(command, command.Length);

// Receive response
IPEndPoint endPoint = new IPEndPoint(IPAddress.Any, 0);
byte[] response = udp.Receive(ref endPoint);
```

---

## 6. UART Interface (Debug)

### 6.1 Device Node

| Device | Description |
|--------|-------------|
| `/dev/ttymxc0` | LPUART1 (debug console) |

### 6.2 Serial Configuration

```bash
# Configure serial port
stty -F /dev/ttymxc0 115200 cs8 -cstopb -parenb

# Write to serial
echo "Debug message" > /dev/ttymxc0

# Read from serial
cat /dev/ttymxc0
```

### 6.3 C# Serial Example

```csharp
using System.IO.Ports;

var serial = new SerialPort("/dev/ttymxc0")
{
    BaudRate = 115200,
    DataBits = 8,
    Parity = Parity.None,
    StopBits = StopBits.One
};

serial.Open();
serial.WriteLine("Debug message\n");
string response = serial.ReadLine();
serial.Close();
```

---

## 7. Thermal Interface

### 7.1 Sysfs Thermal Nodes

| Path | Description |
|------|-------------|
| `/sys/class/thermal/` | Thermal subsystem |
| `/sys/class/thermal/thermal_zone0/temp` | CPU temperature (millidegrees) |

### 7.2 Read Temperature

```bash
# Read CPU temperature
cat /sys/class/thermal/thermal_zone0/temp
# Output: 45000 = 45.000°C

# Convert to human-readable
awk '{print $1/1000"°C"}' /sys/class/thermal/thermal_zone0/temp
```

### 7.3 C# Temperature Reader

```csharp
float ReadCpuTemperature()
{
    string tempStr = File.ReadAllText("/sys/class/thermal/thermal_zone0/temp");
    int millidegrees = int.Parse(tempStr.Trim());
    return millidegrees / 1000.0f;
}
```

---

## 8. File System Access

### 8.1 Application Data Paths

| Path | Purpose |
|------|---------|
| `/var/lib/tft-leakage/` | Application data |
| `/var/log/tft-leakage/` | Log files |
| `/etc/tft-leakage/` | Configuration |
| `/run/tft-leakage/` | Runtime data |

### 8.2 Configuration File

```json
// /etc/tft-leakage/appsettings.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "Spi": {
    "BusId": 1,
    "ChipSelectLine": 0,
    "ClockFrequency": 10000000
  },
  "I2c": {
    "BusId": 1,
    "SensorAddress": 72
  },
  "Network": {
    "ListenPort": 5000
  }
}
```

---

## 9. System Services

### 9.1 systemd Service File

```ini
[Unit]
Description=TFT Leakage Controller
Documentation=https://github.com/holee9/tft-op-code
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dotnet /usr/lib/TftLeakage/TftLeakage.dll
WorkingDirectory=/var/lib/tft-leakage/
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
ReadWritePaths=/var/lib/tft-leakage /var/log/tft-leakage

# Resource limits
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### 9.2 Service Management

```bash
# Enable service
systemctl enable tft-leakage.service

# Start service
systemctl start tft-leakage.service

# Check status
systemctl status tft-leakage.service

# View logs
journalctl -u tft-leakage.service -f
```

---

## 10. Troubleshooting

### 10.1 Permission Issues

```bash
# Add user to required groups
usermod -a -G spi,i2c,gpio tft-leakage

# Create udev rules for automatic permissions
# /etc/udev/rules.d/99-tft-leakage.rules
KERNEL=="spidev*", MODE="0660", GROUP="spi"
KERNEL=="i2c-*", MODE="0660", GROUP="i2c"
```

### 10.2 Device Access Tests

```bash
# Test SPI
spidev_test -D /dev/spidev1.0

# Test I2C
i2cdetect -y 1

# Test GPIO
gpioset gpio4 24=1

# Test network
ping 192.168.1.1
```

---

## 11. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial userspace interface document |

---

**End of Document**
