# Project Overview

## Yocto Project Team Delivery Package

**Document**: 00_project_overview.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Project Goal

**Main Objective**: Build a Yocto Project Linux BSP for NXP i.MX8 Plus that hosts the .NET 8 application controlling a-Si TFT FPD panel leakage through FPGA communication.

### Yocto BSP Responsibilities

1. **SPI Master Driver** - Communicate with FPGA slave interface
2. **Temperature Monitoring** - I2C temperature sensor access
3. **.NET Runtime** - Host .NET 8 control application
4. **System Services** - systemd service management
5. **Network Stack** - Ethernet communication with host

---

## 2. System Architecture

### 2.1 Hardware Context

```
+----------------+     SPI      +----------------+     LVDS     +----------------+
|                | <---------> |                | <----------->|                |
|  i.MX8 Plus    |   Master    |   FPGA         |             |  aSi TFT Panel |
|  (Yocto BSP)   |             |   Artix-7 35T  |   Gate/Row/  |  R1717AS01.3    |
|                |             |                |   Col/Data   |                |
+----------------+             +--------+-------+             +----------------+
       |                              |
       | I2C                          |
       v                              |
+----------------+                    |
|  Temperature   |                    |
|  Sensor (NTC)  |                    |
+----------------+                    |
                                     |
       Ethernet                      |
             v                       |
+----------------+                   |
|  Host PC       |                   |
|  (.NET App)    |                   |
+----------------+                   |
```

### 2.2 Software Architecture

```
Yocto Image Layers:
├── BSP Layer (meta-imx8)
│   ├── U-Boot bootloader
│   ├── Linux kernel (custom defconfig)
│   └── Device tree (.dts)
├── Application Layer (meta-tft-leakage)
│   ├── .NET 8 runtime
│   ├── TFT Leakage Control App
│   └── systemd service files
└── Dependencies
    ├── SPI master driver
    ├── GPIO driver
    ├── I2C driver
    └── Ethernet driver
```

---

## 3. i.MX8 Plus Platform Details

### 3.1 SoC Specifications

| Parameter | Value |
|-----------|-------|
| **Part Number** | i.MX8 Plus (various SKUs) |
| **CPU Cores** | 4x Cortex-A53 @ 1.5 GHz |
| **MCU Core** | 1x Cortex-M4 @ 400 MHz |
| **GPU** | Vivante GC7000 |
| **RAM** | Up to 4 GB LPDDR4 |
| **Storage** | eMMC 5.1, SD 3.0 |

### 3.2 Relevant Peripherals

| Peripheral | Instance | Purpose |
|------------|----------|---------|
| **ECSPI** | ECSPI2 | SPI master to FPGA |
| **I2C** | I2C1 | Temperature sensor |
| **Ethernet** | EQOS | 1 Gbps to host |
| **GPIO** | Multiple | Control signals |
| **UART** | LPUART1 | Debug console |

### 3.3 Pin Assignment (Recommended)

| Signal | Pad | Mux | Function |
|--------|-----|-----|----------|
| ECSPI2_SCLK | GPIO5_IO10 | ALT3 | SPI clock |
| ECSPI2_MOSI | GPIO5_IO11 | ALT3 | SPI MOSI |
| ECSPI2_MISO | GPIO5_IO12 | ALT3 | SPI MISO |
| ECSPI2_SS0 | GPIO5_IO13 | ALT3 | SPI chip select |
| I2C1_SCL | GPIO5_IO14 | ALT1 | I2C clock |
| I2C1_SDA | GPIO5_IO15 | ALT1 | I2C data |

---

## 4. Communication Interfaces

### 4.1 SPI Interface (to FPGA)

| Parameter | Value |
|-----------|-------|
| **Mode** | Mode 0 (CPOL=0, CPHA=0) |
| **Clock Frequency** | Up to 10 MHz |
| **Data Width** | 8 bits |
| **Device** | /dev/spidevX.Y |

### 4.2 I2C Interface (Temperature Sensor)

| Parameter | Value |
|-----------|-------|
| **Bus** | I2C-1 |
| **Frequency** | 100 kHz (standard) |
| **Sensor Address** | 0x48-0x4F |
| **Device** | /dev/i2c-X |

### 4.3 Ethernet Interface (to Host)

| Parameter | Value |
|-----------|-------|
| **Controller** | EQOS (10/100/1000) |
| **Speed** | 1 Gbps |
| **Protocol** | TCP/IP (custom application protocol) |

---

## 5. Software Components

### 5.1 Linux Kernel

| Component | Version | Notes |
|-----------|---------|-------|
| **Kernel** | 5.15+ | LTS recommended |
| **SPI Driver** | Mainline | spidev compatible |
| **GPIO Driver** | Mainline | libgpiod support |
| **I2C Driver** | Mainline | i2c-dev support |

### 5.2 User Space

| Component | Version | Notes |
|-----------|---------|-------|
| **.NET Runtime** | 8.0 | Official packages |
| **systemd** | Latest | Service management |
| **gpiod** | 1.x | GPIO access library |

### 5.3 Application

| Component | Language | Framework |
|-----------|----------|-----------|
| **TFT Leakage Control** | C# 12 | .NET 8 |

---

## 6. System Services

### 6.1 Required systemd Services

| Service | Description | Auto-start |
|---------|-------------|------------|
| `tft-leakage.service` | Main control application | Yes |
| `network-online.target` | Network wait | Required |
| `time-sync.target` | Time synchronization | Recommended |

### 6.2 Service Template

```ini
[Unit]
Description=TFT Leakage Controller
After=network.target
Requires=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dotnet /usr/lib/TftLeakage/TftLeakage.dll
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

---

## 7. Yocto Recipe Structure

### 7.1 Layer Structure

```
meta-tft-leakage/
├── conf/
│   └── layer.conf
├── recipes-core/
│   └── images/
│       └── tft-leakage-image.bb
├── recipes-graphics/
│   └── dotnet/
│       └── dotnet-runtime_8.0.bb
├── recipes-apps/
│   └── tft-leakage/
│       ├── tft-leakage_1.0.bb
│       └── files/
│           ├── appsettings.json
│           └── tft-leakage.service
└── recipes-kernel/
    └── linux/
        └── linux-imx8%.bbappend
```

### 7.2 Image Recipe

```bitbake
SUMMARY = "TFT Leakage Control Image"
LICENSE = "MIT"

inherit core-image

IMAGE_INSTALL += " \
    dotnet-runtime-8.0 \
    tft-leakage \
    bash \
    coreutils \
    i2c-tools \
    spi-tools \
"
```

---

## 8. Development Workflow

### 8.1 Build Steps

```bash
# 1. Setup Yocto environment
source setup-env.sh

# 2. Bake image
bitbake tft-leakage-image

# 3. Flash to SD card
bmaptool copy tmp/deploy/images/tft-leakage-image.wic /dev/sdX

# 4. Boot and verify
```

### 8.2 Debug Tools

| Tool | Purpose |
|------|---------|
| `spidev_test` | SPI communication test |
| `i2cdetect` | I2C device scan |
| `gpioset` | GPIO control test |
| `journalctl` | System log viewing |

---

## 9. Configuration Requirements

### 9.1 Kernel Configuration

Required kernel options (see `04_kernel_config.md`):
- SPI master support
- spidev device driver
- I2C chardev
- GPIO libgpiod
- Ethernet drivers
- .NET prerequisites

### 9.2 Device Tree

Required Device Tree nodes (see `03_device_tree_configuration.md`):
- ECSPI2 node for FPGA SPI
- I2C1 node for temperature sensor
- Ethernet node
- GPIO pin configurations

---

## 10. Related Documents

| Document | Description |
|----------|-------------|
| Platform Details | `01_platform_specifications.md` |
| SPI Driver | `02_spi_driver_requirements.md` |
| Device Tree | `03_device_tree_configuration.md` |
| Kernel Config | `04_kernel_config.md` |
| Userspace API | `05_userspace_interface.md` |
| Temperature | `06_temperature_monitoring.md` |
| Ethernet | `07_ethernet_communication.md` |
| Acceptance | `08_acceptance_criteria.md` |

---

## 11. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial document |

---

**End of Document**
