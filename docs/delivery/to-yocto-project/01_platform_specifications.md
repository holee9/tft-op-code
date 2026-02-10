# Platform Specifications

## Yocto Project Team Delivery Package

**Document**: 01_platform_specifications.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. i.MX8 Plus Overview

### 1.1 SoC Identification

| Parameter | Value |
|-----------|-------|
| **Family** | i.MX8 Plus (i.MX8X) |
| **Part Number Range** | MV21... to MV27... |
| **Package** | BGA 19x19 mm, 0.8 mm pitch |
| **Process** | 28 nm |

### 1.2 Core Configuration

| Core | Count | Frequency | Notes |
|------|-------|-----------|-------|
| **Cortex-A53** | 4 | 1.0-1.5 GHz | Linux kernel |
| **Cortex-M4** | 1 | 400 MHz | Real-time tasks |

---

## 2. Memory Map

### 2.1 Memory Types

| Memory | Size | Range | Notes |
|--------|------|-------|-------|
| **LPDDR4** | 1-4 GB | 0x8000_0000+ | Main memory |
| **TCM** | 64 KB | 0x1FFE_0000+ | M4 tightly coupled |
| **OCRAM** | 64 KB | 0x9000_0000+ | On-chip RAM |
| **Boot ROM** | 128 KB | 0x0000_0000+ | Boot code |

### 2.2 Peripheral Address Map

| Peripheral | Base Address | Size |
|------------|--------------|------|
| **AIPS1** | 0x5B00_0000 | 1 MB |
| **AIPS2** | 0x5C00_0000 | 1 MB |
| **AIPS3** | 0x5D00_0000 | 1 MB |
| **AIPS4** | 0x5E00_0000 | 1 MB |

---

## 3. ECSPI2 (SPI) Interface

### 3.1 ECSPI2 Specifications

| Parameter | Value |
|-----------|-------|
| **Instance** | ECSPI2 |
| **Base Address** | 0x5E00_4000 |
| **Clock Source** | PLL3 (main clock) |
| **Max Frequency** | 60 MHz (use 10 MHz for FPGA) |
| **FIFO Depth** | 64 x 32 bits |

### 3.2 ECSPI2 Register Map

| Register | Offset | Description |
|----------|--------|-------------|
| **RXDATA** | 0x00 | Receive data |
| **TXDATA** | 0x04 | Transmit data |
| **CONREG** | 0x08 | Control register |
| **CONFIGREG** | 0x0C | Configuration register |
| **INTREG** | 0x10 | Interrupt register |
| **DMAREG** | 0x14 | DMA control |
| **STATREG** | 0x18 | Status register |
| **PERIODREG** | 0x1C | Sample period |

### 3.3 CONREG (Control Register)

```
Bit 31: ENABLE - Enable module
Bit 28: XCH - Exchange data
Bit 27: SMC - Start mode (manual/auto)
Bit 24: HT - High speed toggle
Bits 23-20: CHANNEL_MASK - Channel select
Bits 19-16: BURST_LENGTH - Transfer length
Bits 15-12: PRE_DIVIDER - Prescaler
Bits 3-0: CHANNEL_SELECT - Current channel
```

### 3.4 Clock Configuration

```dts
/* ECSPI2 clock configuration */
clock-frequency = <10000000>; /* 10 MHz */
cs-gpios = <&gpio5 13 GPIO_ACTIVE_LOW>;
pinctrl-0 = <&pinctrl_ecspi2>;
pinctrl-1 = <&pinctrl_ecspi2_cs>;
```

---

## 4. I2C1 Interface

### 4.1 I2C1 Specifications

| Parameter | Value |
|-----------|-------|
| **Instance** | I2C1 |
| **Base Address** | 0x5A80_0000 |
| **Max Frequency** | 400 kHz (fast mode) |
| **Bus Voltage** | 1.8 V |

### 4.2 Temperature Sensor (Typical)

| Parameter | Value |
|-----------|-------|
| **Part** | TMP100 or similar |
| **Address** | 0x48 (default) |
| **Resolution** | 0.0625°C |
| **Accuracy** | ±2°C |

### 4.3 I2C1 Device Tree Node

```dts
&i2c1 {
    clock-frequency = <100000>;
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_i2c1>;
    status = "okay";

    temp_sensor: tmp100@48 {
        compatible = "ti,tmp100";
        reg = <0x48>;
    };
};
```

---

## 5. Ethernet Interface

### 5.1 EQOS Specifications

| Parameter | Value |
|-----------|-------|
| **Controller** | EQOS (Synopsys) |
| **Speed** | 10/100/1000 Mbps |
| **Interface** | RGMII |
| **PHY Address** | 0x00 (RGMII) |
| **MDIO** | Internal |

### 5.2 EQOS Device Tree Node

```dts
&eqos {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_eqos>;
    phy-mode = "rgmii-id";
    phy-handle = <&ethphy0>;
    status = "okay";

    mdio {
        ethphy0: ethernet-phy@0 {
            compatible = "ethernet-phy-ieee802.3-c22";
            reg = <0>;
        };
    };
};
```

---

## 6. GPIO Configuration

### 6.1 GPIO Banks

| Bank | Pins | Base Address |
|------|------|--------------|
| **GPIO1** | 32 | 0x5E00_4000 |
| **GPIO2** | 32 | 0x5E00_5000 |
| **GPIO3** | 32 | 0x5E00_6000 |
| **GPIO4** | 32 | 0x5E00_7000 |
| **GPIO5** | 32 | 0x5E00_8000 |

### 6.2 Recommended Pin Mux

| Function | Pad | Mux Mode | Pin |
|----------|-----|----------|-----|
| **ECSPI2_SCLK** | GPIO5_IO10 | ALT3 | 138 |
| **ECSPI2_MOSI** | GPIO5_IO11 | ALT3 | 139 |
| **ECSPI2_MISO** | GPIO5_IO12 | ALT3 | 140 |
| **ECSPI2_SS0** | GPIO5_IO13 | ALT3 | 141 |
| **I2C1_SCL** | GPIO5_IO14 | ALT1 | 142 |
| **I2C1_SDA** | GPIO5_IO15 | ALT1 | 143 |

### 6.3 Pin Control Group

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

## 7. Power Management

### 7.1 Power Domains

| Domain | Voltage | Typical Current |
|--------|---------|-----------------|
| **VDD_SOC** | 1.0 V | 500 mA |
| **VDD_ARM** | 1.0 V | 800 mA |
| **VDD_GPU** | 1.0 V | 300 mA |
| **VDD_DRAM** | 1.1 V | 600 mA |
| **VDD_1P8** | 1.8 V | 100 mA |
| **VDD_3P3** | 3.3 V | 200 mA |

### 7.2 Power States

| State | A53 | M4 | peripherals |
|-------|-----|----|-------------|
| **RUN** | On | On | On |
| **WAIT** | On | On | Selective |
| **STOP** | On | On | Clock gated |
| **SUSPEND** | Off | Optional | Low power |

---

## 8. Boot Configuration

### 8.1 Boot Sequence

```
Power On → ROM Code → SPL → U-Boot → Linux Kernel → systemd
```

### 8.2 Boot Devices

| Device | Priority | Notes |
|--------|----------|-------|
| **eMMC** | 1 | Primary |
| **SD Card** | 2 | Development |
| **USB** | 3 | Recovery |

### 8.3 Bootloader Configuration

```
U-Boot Environment:

setenv bootargs 'console=ttyLP0,115200 earlycon root=/dev/mmcblk0p2 rootwait rw'
setenv fdt_file 'imx8mp-tft-leakage.dtb'
setenv initrd_addr '0x43800000'
setenv initrd_size '0x2000000'
```

---

## 9. Thermal Management

### 9.1 Thermal Specifications

| Parameter | Value |
|-----------|-------|
| **Operating Temperature** | 0°C to 70°C |
| **Junction Temperature** | -40°C to 125°C |
| **Thermal Trip Point** | 95°C (CPU) |
| **Passive Cooling** | Copper plane + heatsink |

### 9.2 Thermal Zones

| Zone | Trip Point | Action |
|------|------------|--------|
| **cpu-thermal** | 85°C | Throttling |
| **soc-thermal** | 95°C | Shutdown |

---

## 10. Reference Documents

| Document | Source |
|----------|--------|
| i.MX8 Plus Reference Manual | NXP |
| i.MX8 Plus Datasheet | NXP |
| Yocto BSP Guide | NXP |
| Device Tree Bindings | Linux kernel |

---

## 11. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial platform specifications |

---

**End of Document**
