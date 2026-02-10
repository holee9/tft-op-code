# Kernel Configuration

## Yocto Project Team Delivery Package

**Document**: 04_kernel_config.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Overview

This document defines the Linux kernel configuration required for the TFT Leakage Control system on i.MX8 Plus.

---

## 2. Base Kernel

| Parameter | Value |
|-----------|-------|
| **Version** | 5.15.y LTS or 6.1.y LTS |
| **Source** | NXP mainline or FSL community |
| **Architecture** | arm64 |
| **Defconfig** | imx_v8_defconfig + fragment |

---

## 3. Configuration Fragment

### 3.1 Complete Fragment File

```kconfig
# SPDX-License-Identifier: GPL-2.0
#
# TFT Leakage Control Kernel Configuration Fragment
#

# ==========================================================================
# SPI Support (Required for FPGA communication)
# ==========================================================================
CONFIG_SPI=y
CONFIG_SPI_DEBUG=y
CONFIG_SPI_MASTER=y
CONFIG_SPI_MEM=y

# i.MX SPI controller
CONFIG_SPI_IMX=y
CONFIG_SPI_IMX_DEBUG=y

# SPI device driver (userspace interface)
CONFIG_SPI_SPIDEV=y

# ==========================================================================
# I2C Support (Required for temperature sensor)
# ==========================================================================
CONFIG_I2C=y
CONFIG_I2C_CHARDEV=y
CONFIG_I2C_IMX=y

# ==========================================================================
# GPIO Support (Required for control signals)
# ==========================================================================
CONFIG_GPIOLIB=y
CONFIG_GPIOLIB_IRQCHIP=y
CONFIG_GPIO_SYSFS=y
CONFIG_GPIO_GENERIC=y
CONFIG_GPIO_IMX=y

# libgpiod support
CONFIG_GPIO_CDEV=y
CONFIG_GPIO_CDEV_V1=y

# ==========================================================================
# Network Support (Required for host communication)
# ==========================================================================
CONFIG_NET=y
CONFIG_INET=y
CONFIG_NETDEVICES=y

# EQOS Ethernet driver
CONFIG_STMMAC_ETH=y
CONFIG_DWMAC_GENERIC=y
CONFIG_DWMAC_IMX8=y

# ==========================================================================
# Serial Driver (Required for debug console)
# ==========================================================================
CONFIG_SERIAL_CORE=y
CONFIG_SERIAL_CORE_CONSOLE=y
CONFIG_SERIAL_FSL_LPUART=y
CONFIG_SERIAL_FSL_LPUART_CONSOLE=y

# ==========================================================================
# Thermal Management (Required for temperature monitoring)
# ==========================================================================
CONFIG_THERMAL=y
CONFIG_THERMAL_OF=y
CONFIG_THERMAL_WRITABLE_TRIPS=y
CONFIG_THERMAL_GOV_POWER_ALLOCATOR=y
CONFIG_CPU_THERMAL=y
CONFIG_IMX_SC_THERMAL=y

# ==========================================================================
# Watchdog (Optional, for system recovery)
# ==========================================================================
CONFIG_IMX2_WDT=y
CONFIG_WATCHDOG_NOWAYOUT=y

# ==========================================================================
# Device Tree Support
# ==========================================================================
CONFIG_OF=y
CONFIG_OF_GPIO=y
CONFIG_OF_SPI=y
CONFIG_OF_I2C=y
CONFIG_OF_NET=y
CONFIG_OF_MDIO=y

# ==========================================================================
# DMA Support (For SPI performance)
# ==========================================================================
CONFIG_DMADEVICES=y
CONFIG_FSL_EDMA=y
CONFIG_DMATEST=m

# ==========================================================================
# MMC/SD (For boot storage)
# ==========================================================================
CONFIG_MMC=y
CONFIG_MMC_SDHCI=y
CONFIG_MMC_SDHCI_ESDHC_IMX=y

# ==========================================================================
# USB (Optional, for development/debug)
# ==========================================================================
CONFIG_USB_SUPPORT=y
CONFIG_USB=y
CONFIG_USB_EHCI_HCD=y
CONFIG_USB_STORAGE=y
CONFIG_USB_SERIAL=y

# ==========================================================================
# File Systems
# ==========================================================================
CONFIG_EXT4_FS=y
CONFIG_EXT4_FS_POSIX_ACL=y
CONFIG_VFAT_FS=y
CONFIG_PROC_FS=y
CONFIG_SYSFS=y
CONFIG_TMPFS=y

# ==========================================================================
# .NET Runtime Dependencies
# ==========================================================================
CONFIG_64BIT=y
CONFIG_MMU=y
CONFIG_VMAP_STACK=y

# ==========================================================================
# Debug Options (For development)
# ==========================================================================
CONFIG_DEBUG_KERNEL=y
CONFIG_DEBUG_INFO=y
CONFIG_DEBUG_FS=y
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y

# ==========================================================================
# Security Options
# ==========================================================================
CONFIG_SECURITY=y
CONFIG_SECURITY_NETWORK=y
CONFIG_SECCOMP=y
CONFIG_SECCOMP_FILTER=y

# ==========================================================================
# Power Management
# ==========================================================================
CONFIG_PM=y
CONFIG_PM_SLEEP=y
CONFIG_PM_DEBUG=y
CONFIG_PM_WAKELOCKS=y
CONFIG_PM_WAKELOCKS_GC=y
CONFIG_PM_WAKELOCKS_LIMIT=100

# CPU Idle
CONFIG_CPU_IDLE=y
CONFIG_CPU_FREQ=y
CONFIG_CPU_FREQ_DEFAULT_GOV_ONDEMAND=y
CONFIG_CPU_FREQ_GOV_POWERSAVE=y
CONFIG_CPU_FREQ_GOV_USERSPACE=y

# ==========================================================================
# RTC (For system time)
# ==========================================================================
CONFIG_RTC_CLASS=y
CONFIG_RTC_DRV_IMX_SC=y
CONFIG_RTC_HCTOSYS=y

# ==========================================================================
# NVMEM (For calibration data)
# ==========================================================================
CONFIG_NVMEM=y
CONFIG_NVMEM_IMX_OCOTP=y

# ==========================================================================
# PHYLIB (For Ethernet PHY)
# ==========================================================================
CONFIG_PHYLIB=y
CONFIG_PHYLINK=y
CONFIG_GENERIC_PHY=y
CONFIG_MICREL_PHY=y
CONFIG_AT803X_PHY=y
CONFIG_MARVELL_PHY=y

# ==========================================================================
# pinctrl (Required for pin configuration)
# ==========================================================================
CONFIG_PINCTRL=y
CONFIG_PINCTRL_IMX8M=y
CONFIG_PINCTRL_IMX8MP=y
```

### 3.2 Configuration as BitBake Fragment

Save as `linux-imx/tft-leakage.cfg`:

```bitbake
# Kernel configuration fragment for TFT Leakage Control

# SPI Support
CONFIG_SPI=y
CONFIG_SPI_MASTER=y
CONFIG_SPI_IMX=y
CONFIG_SPI_SPIDEV=y

# I2C Support
CONFIG_I2C=y
CONFIG_I2C_CHARDEV=y
CONFIG_I2C_IMX=y

# GPIO Support
CONFIG_GPIOLIB=y
CONFIG_GPIO_SYSFS=y
CONFIG_GPIO_IMX=y
CONFIG_GPIO_CDEV=y

# Network
CONFIG_NETDEVICES=y
CONFIG_STMMAC_ETH=y
CONFIG_DWMAC_IMX8=y

# Serial
CONFIG_SERIAL_CORE=y
CONFIG_SERIAL_FSL_LPUART=y
CONFIG_SERIAL_FSL_LPUART_CONSOLE=y

# Thermal
CONFIG_THERMAL=y
CONFIG_CPU_THERMAL=y

# Device Tree
CONFIG_OF=y

# DMA
CONFIG_DMADEVICES=y
CONFIG_FSL_EDMA=y
```

---

## 4. Recipe Integration

### 4.1 Kernel Recipe Append

Create `recipes-kernel/linux/linux-imx_%.bbappend`:

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

SRC_URI += "file://tft-leakage.cfg"
```

### 4.2 Directory Structure

```
meta-tft-leakage/
└── recipes-kernel/
    └── linux/
        ├── linux-imx_%.bbappend
        └── linux-imx/
            └── tft-leakage.cfg
```

---

## 5. Configuration Verification

### 5.1 Build-Time Verification

```bash
# After kernel build
make imx_v8_defconfig
./scripts/kconfig/merge_config.sh imx_v8_defcorecfg tft-leakage.cfg

# Verify options
grep -E "CONFIG_SPI|CONFIG_I2C|CONFIG_GPIO|CONFIG_THERMAL" .config
```

### 5.2 Run-Time Verification

```bash
# Check SPI
ls -l /dev/spidev*

# Check I2C
ls -l /dev/i2c-*

# Check GPIO
ls -l /dev/gpiochip*

# Check thermal
ls -l /sys/class/thermal/

# Check network
ip link show
```

### 5.3 Module Verification

```bash
# Check loaded modules
lsmod | grep -E "spi|i2c|gpio|thermal|stmmac"

# Check module parameters
modinfo spi_imx
modinfo i2c_imx
```

---

## 6. Optional Features

### 6.1 Development Options

```kconfig
# Additional debugging
CONFIG_DYNAMIC_DEBUG=y
CONFIG_FTRACE=y
CONFIG_FUNCTION_TRACER=y

# Performance analysis
CONFIG_PERF_EVENTS=y
CONFIG_HW_PERF_EVENTS=y
```

### 6.2 Production Options

```kconfig
# Optimize for size
CONFIG_CC_OPTIMIZE_FOR_SIZE=y

# Reduce debug info
CONFIG_DEBUG_INFO_REDUCED=y

# Disable development features
# CONFIG_DEBUG_KERNEL is not set
```

---

## 7. Kernel Boot Args

### 7.1 Recommended Boot Arguments

```
console=ttymxc0,115200 earlycon=ttymxc0,115200
root=/dev/mmcblk0p2 rootwait rw
rootfstype=ext4
quiet
# Debug options (optional)
ignore_loglevel
initcall_debug
```

### 7.2 U-Boot Environment

```
setenv bootargs 'console=ttymxc0,115200 earlycon=ttymxc0,115200 root=/dev/mmcblk0p2 rootwait rw'
saveenv
```

---

## 8. Build and Deploy

### 8.1 Yocto Build Commands

```bash
# Initialize bitbake environment
source setup-env.sh

# Build kernel
bitbake linux-imx

# Build kernel with menuconfig (for development)
bitbake -c menuconfig linux-imx

# Clean build
bitbake -c cleansstate linux-imx
bitbake linux-imx
```

### 8.2 Output Files

| File | Location | Description |
|------|----------|-------------|
| `Image` | `tmp/deploy/images/` | Kernel image |
| `imx8mp-tft-leakage.dtb` | `tmp/deploy/images/` | Device tree blob |
| `modules-*.*.tgz` | `tmp/deploy/images/` | Kernel modules |

---

## 9. Troubleshooting

### 9.1 Common Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| SPI not working | /dev/spidevX missing | Enable CONFIG_SPI_SPIDEV |
| I2C not working | /dev/i2c-X missing | Enable CONFIG_I2C_CHARDEV |
| No GPIO access | Permission denied | Add user to gpio group |
| Network down | eth0 not present | Enable CONFIG_DWMAC_IMX8 |

### 9.2 Debug Commands

```bash
# Check kernel config
zcat /proc/config.gz | grep -E CONFIG_SPI

# Check device tree
ls -l /sys/firmware/devicetree/base/

# Check modules
lsmod

# Check kernel log
dmesg | tail -100
journalctl -k -b
```

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial kernel configuration |

---

**End of Document**
