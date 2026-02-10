# SPI Driver Requirements

## Yocto Project Team Delivery Package

**Document**: 02_spi_driver_requirements.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Overview

The i.MX8 Plus acts as SPI master, communicating with the FPGA which implements a SPI slave interface. This document specifies the driver requirements.

---

## 2. SPI Interface Requirements

### 2.1 Electrical Specifications

| Parameter | Value | Notes |
|-----------|-------|-------|
| **SPI Mode** | Mode 0 (CPOL=0, CPHA=0) | Clock idle low, sample on rising edge |
| **Clock Frequency** | 10 MHz | Maximum |
| **Data Width** | 8 bits | Per transfer |
| **Byte Order** | MSB first | Most significant bit first |
| **Chip Select** | Active low | GPIO controlled |

### 2.2 Signal Mapping

| i.MX8 Signal | FPGA Signal | Direction |
|--------------|-------------|-----------|
| ECSPI2_MOSI | spi_mosi | Output |
| ECSPI2_MISO | spi_miso | Input |
| ECSPI2_SCLK | spi_sclk | Output |
| GPIO5_IO13 | spi_cs_n | Output |

---

## 3. Driver Requirements

### 3.1 Functional Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| DR-1 | Use mainline Linux SPI driver | Critical |
| DR-2 | Support spidev user interface | Critical |
| DR-3 | Implement Mode 0 timing | Critical |
| DR-4 | Support up to 10 MHz clock | High |
| DR-5 | Provide read/write ioctl interface | High |
| DR-6 | Support burst transfers | Medium |

### 3.2 Driver Options

```
Kernel module: spi_imx
Device class: spidev
Device nodes: /dev/spidevX.Y
```

---

## 4. Device Tree Configuration

### 4.1 ECSPI2 Node

```dts
&ecspi2 {
    #address-cells = <1>;
    #size-cells = <0>;
    compatible = "fsl,imx8mp-spi", "fsl,imx51-spi";
    reg = <0x5e004000 0x4000>;
    interrupts = <GIC_SPI 12 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clk IMX8MP_CLK_ECSPI2_ROOT>;
    clock-names = "per";
    dmas = <&sdma1 0 7 1>, <&sdma1 1 7 2>;
    dma-names = "rx", "tx";
    status = "okay";

    fpga_spi: fpga-spi@0 {
        compatible = "spidev";
        reg = <0>;
        spi-max-frequency = <10000000>;
        spi-cpol;
        spi-cpha;
        spi-cs-high = <0>; /* CS active low */
        num-chipselects = <1>;
        cs-gpios = <&gpio5 13 GPIO_ACTIVE_LOW>;
    };
};
```

### 4.2 Pin Control

```dts
pinctrl_ecspi2: ecspi2grp {
    fsl,pins = <
        MX8MP_PAD_ECSPI2__ECSPI2_SCLK      0x82
        MX8MP_PAD_ECSPI2_MOSI__ECSPI2_MOSI  0x82
        MX8MP_PAD_ECSPI2_MISO__ECSPI2_MISO  0x82
    >;
};

pinctrl_ecspi2_cs: ecspi2cs {
    fsl,pins = <
        MX8MP_PAD_ECSPI2_SS0__GPIO5_IO13   0x40000
    >;
};
```

---

## 5. Userspace Interface

### 5.1 Device Node

| Path | Description |
|------|-------------|
| `/dev/spidev1.0` | ECSPI2, chip select 0 |

### 5.2 IOCTL Commands

| Command | Value | Description |
|---------|-------|-------------|
| `SPI_IOC_WR_MODE` | 0x40016b01 | Set SPI mode |
| `SPI_IOC_RD_MODE` | 0x80016b01 | Get SPI mode |
| `SPI_IOC_WR_BITS_PER_WORD` | 0x40016b03 | Set bits per word |
| `SPI_IOC_RD_BITS_PER_WORD` | 0x80016b03 | Get bits per word |
| `SPI_IOC_WR_MAX_SPEED_HZ` | 0x40046b04 | Set max speed |
| `SPI_IOC_RD_MAX_SPEED_HZ` | 0x80046b04 | Get max speed |
| `SPI_IOC_MESSAGE(N)` | 0x4000+0x6b00*+N | Transfer message |

### 5.3 C Code Example

```c
#include <fcntl.h>
#include <linux/spi/spidev.h>
#include <sys/ioctl.h>

int spi_fd;
uint8_t mode = SPI_MODE_0;
uint8_t bits = 8;
uint32_t speed = 10000000;

/* Open SPI device */
spi_fd = open("/dev/spidev1.0", O_RDWR);

/* Configure SPI */
ioctl(spi_fd, SPI_IOC_WR_MODE, &mode);
ioctl(spi_fd, SPI_IOC_WR_BITS_PER_WORD, &bits);
ioctl(spi_fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);

/* Write transaction */
uint8_t tx_buf[3] = {0x01, 0x00, 0xFF}; /* CMD, ADDR, DATA */
struct spi_ioc_transfer tr = {
    .tx_buf = (unsigned long)tx_buf,
    .rx_buf = (unsigned long)rx_buf,
    .len = 3,
    .speed_hz = speed,
    .bits_per_word = bits,
};
ioctl(spi_fd, SPI_IOC_MESSAGE(1), &tr);
```

### 5.4 C# Code Example (.NET)

```csharp
using System.Device.Spi;

// SPI configuration
var spiSettings = new SpiConnectionSettings(busId: 1, chipSelectLine: 0)
{
    Mode = SpiMode.Mode0,
    ClockFrequency = 10000000,
    DataBitLength = 8
};

using var spi = SpiDevice.Create(spiSettings);

// Write transaction
byte[] writeBuffer = new byte[] { 0x01, 0x00, 0xFF };
spi.Write(writeBuffer);

// Read transaction
byte[] readBuffer = new byte[3];
spi.TransferFullDuplex(writeBuffer, readBuffer);
```

---

## 6. Transfer Protocol

### 6.1 Write Transaction Format

```
Byte 0: Command Code (0x01 = WRITE)
Byte 1: Register Address (0x00-0x3F)
Byte 2: Data Byte

Timing:
CS_N LOW → Send 3 bytes → CS_N HIGH
```

### 6.2 Read Transaction Format

```
Byte 0: Command Code (0x02 = READ)
Byte 1: Register Address (0x00-0x3F)
Byte 2: Dummy byte (0x00)

Timing:
CS_N LOW → Send 3 bytes, receive data → CS_N HIGH
Response: Data in byte 2 (during dummy byte)
```

### 6.3 Burst Read Format

```
Byte 0: Command Code (0x03 = BURST_READ)
Byte 1: Start Address (0x00-0x3F)
Byte N: Dummy bytes (one per register to read)

Timing:
CS_N LOW → Send 2 bytes → Send N dummy bytes → CS_N HIGH
Response: Sequential data starting at byte 2
```

---

## 7. Timing Requirements

### 7.1 SPI Timing Parameters

| Parameter | Min | Typical | Max | Unit |
|-----------|-----|---------|-----|------|
| Clock frequency | - | 10 | 10 | MHz |
| CS setup time | 10 | 20 | - | ns |
| CS hold time | 10 | 20 | - | ns |
| SCK to MOSI valid | - | 50 | 100 | ns |
| MOSI hold time | 10 | 20 | - | ns |
| MISO valid delay | - | 50 | 100 | ns |

### 7.2 Transfer Timing

```
Byte transfer time at 10 MHz:
8 bits × (1/10 MHz) = 0.8 µs per byte

Typical transaction:
3 bytes × 0.8 µs = 2.4 µs
```

---

## 8. Error Handling

### 8.1 Error Conditions

| Error | Detection | Recovery |
|-------|-----------|----------|
| **Timeout** | No response | Retry up to 3 times |
| **CRC Error** | (if implemented) | Request resend |
| **Invalid Address** | Known addresses | Return error |
| **Communication Error** | IOCTL return | Log and retry |

### 8.2 Error Handling Pattern

```c
int spi_write_register(int fd, uint8_t addr, uint8_t data) {
    uint8_t tx[3] = {0x01, addr, data};
    uint8_t rx[3] = {0};
    int retry = 0;
    int ret;

    while (retry < 3) {
        ret = spi_transfer(fd, tx, rx, 3);
        if (ret == 0)
            return 0; /* Success */
        retry++;
        usleep(1000); /* 1 ms delay */
    }
    return -1; /* Failed after retries */
}
```

---

## 9. Testing Requirements

### 9.1 Unit Tests

| Test | Description | Pass Criteria |
|------|-------------|---------------|
| SPI_LOOPBACK | Loopback test | Data echoed correctly |
| SPI_MODE | Mode verification | Mode 0 active |
| SPI_SPEED | Speed test | 10 MHz achievable |
| SPI_CS | Chip select | CS asserts correctly |

### 9.2 Integration Tests

| Test | Description | Pass Criteria |
|------|-------------|---------------|
| FPGA_COMM | FPGA communication | Registers read/write |
| BURST_READ | Sequential read | Multiple registers read |
| STRESS | Continuous operation | 1000+ transfers |

### 9.3 Test Utility

```bash
# SPI test tool
spidev_test -D /dev/spidev1.0 -v -s 10000000

# Manual test
echo "01 00 FF" > /sys/kernel/debug/spi/spi1.0/push_block
```

---

## 10. Kernel Configuration

### 10.1 Required Config Options

```kconfig
CONFIG_SPI=y
CONFIG_SPI_MASTER=y
CONFIG_SPI_IMX=y
CONFIG_SPI_SPIDEV=y
CONFIG_SPI_BITBANG=n
```

### 10.2 Optional Config Options

```kconfig
CONFIG_SPI_SLAVE=y
CONFIG_SPI_LOOPBACK=y
CONFIG_SPI_GPIO=y
```

---

## 11. Troubleshooting

### 11.1 Common Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| Wrong mode | Garbage data | Verify CPOL=0, CPHA=0 |
| Clock too fast | Data corruption | Reduce to ≤10 MHz |
| CS issues | No communication | Check GPIO mux |
| Permissions | Access denied | Add user to spi group |

### 11.2 Debug Commands

```bash
# Check SPI device
ls -l /dev/spidev*

# Check SPI controller
cat /sys/class/spi_master/spi1//*

# Check pin mux
cat /sys/kernel/debug/pinctrl/*/pins

# spi debug
cat /sys/kernel/debug/spi/spi1.0/stats
```

---

## 12. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial SPI driver requirements |

---

**End of Document**
