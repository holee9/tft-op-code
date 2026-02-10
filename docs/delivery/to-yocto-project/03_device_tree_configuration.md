# Device Tree Configuration

## Yocto Project Team Delivery Package

**Document**: 03_device_tree_configuration.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Overview

This document provides the complete Device Tree configuration for the i.MX8 Plus TFT Leakage Control system.

---

## 2. Complete Device Tree Source

### 2.1 Main DTS File

```dts
// SPDX-License-Identifier: (GPL-2.0 OR MIT)
/*
 * Copyright 2026 TFT Leakage Control Project
 *
 * i.MX8 Plus TFT Leakage Control Board Device Tree
 */

/dts-v1/;

#include "imx8mp.dtsi"
#include "imx8mp-pinfunc.h"

/ {
    model = "TFT Leakage Control i.MX8 Plus Board";
    compatible = "fsl,imx8mp-tft-leakage", "fsl,imx8mp";

    chosen {
        stdout-path = &lpuart1;
    };

    /* 1.8 V supply for peripherals */
    reg_1p8v: regulator-1p8v {
        compatible = "regulator-fixed";
        regulator-name = "1P8V";
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
    };

    /* 3.3 V supply for FPGA I/O */
    reg_3p3v: regulator-3p3v {
        compatible = "regulator-fixed";
        regulator-name = "3P3V";
        regulator-min-microvolt = <3300000>;
        regulator-max-microvolt = <3300000>;
        regulator-always-on;
    };
};

/* ==========================================================================
 * ECSPI2 - FPGA SPI Interface
 * ==========================================================================
 */
&ecspi2 {
    #address-cells = <1>;
    #size-cells = <0>;
    compatible = "fsl,imx8mp-spi", "fsl,imx51-spi";
    reg = <0x5e004000 0x4000>;
    interrupts = <GIC_SPI 12 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clk IMX8MP_CLK_ECSPI2_ROOT>,
             <&clk IMX8MP_CLK_ECSPI2>;
    clock-names = "ipg", "per";
    assigned-clocks = <&clk IMX8MP_CLK_ECSPI2>;
    assigned-clock-parents = <&clk IMX8MP_CLK_SYS_PLL1_80M>;
    assigned-clock-rates = <10000000>; /* 10 MHz for FPGA */
    dmas = <&sdma1 0 7 1>, <&sdma1 1 7 2>;
    dma-names = "rx", "tx";
    status = "okay";

    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_ecspi2>, <&pinctrl_ecspi2_cs>;

    /* FPGA SPI slave device */
    fpga_spi: fpga@0 {
        compatible = "spidev";
        reg = <0>;
        spi-max-frequency = <10000000>;
        spi-cpol;       /* CPOL = 0 (but inverted by fsl,imx51-spi) */
        spi-cpha;       /* CPHA = 0 */
        spi-cs-high = <0>;  /* CS active low */
        num-chipselects = <1>;
        cs-gpios = <&gpio5 13 GPIO_ACTIVE_LOW>;
    };
};

/* ==========================================================================
 * I2C1 - Temperature Sensor
 * ==========================================================================
 */
&i2c1 {
    clock-frequency = <100000>; /* 100 kHz standard mode */
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_i2c1>;
    status = "okay";

    /* Temperature sensor (TMP100 or compatible) */
    temp_sensor: temp-sensor@48 {
        compatible = "ti,tmp100";
        reg = <0x48>;
        /* Optional: shutdown mode for power saving */
        #thermal-sensor-cells = <0>;
    };
};

/* ==========================================================================
 * EQOS Ethernet - Host Communication
 * ==========================================================================
 */
&eqos {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_eqos>;
    phy-mode = "rgmii-id";
    phy-handle = <&ethphy0>;
    mac-address = [00 04 25 1c a0 b0]; /* Default, can override */
    status = "okay";

    mdio {
        #address-cells = <1>;
        #size-cells = <0>;

        /* RGMII PHY */
        ethphy0: ethernet-phy@0 {
            compatible = "ethernet-phy-ieee802.3-c22";
            reg = <0>;
            max-speed = <1000>;
            reset-gpios = <&gpio4 22 GPIO_ACTIVE_LOW>;
            reset-assert-us = <10000>;
            reset-deassert-us = <1000>;
            interrupt-parent = <&gpio4>;
            interrupts = <23 IRQ_TYPE_LEVEL_LOW>;
        };
    };
};

/* ==========================================================================
 * UART1 - Debug Console
 * ==========================================================================
 */
&lpuart1 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_lpuart1>;
    status = "okay";
};

/* ==========================================================================
 * GPIO - Additional Control Signals
 * ==========================================================================
 */
&gpio4 {
    /* Additional control signals */
    fpga_bias_req: bias-req-pin {
        gpio-hog;
        gpios = <24 GPIO_ACTIVE_HIGH>;
        output-high;
        line-name = "fpga-bias-req";
    };

    fpga_reset: reset-pin {
        gpio-hog;
        gpios = <25 GPIO_ACTIVE_LOW>;
        output-high;
        line-name = "fpga-reset";
    };
};

/* ==========================================================================
 * Thermal - Temperature Monitoring
 * ==========================================================================
 */
&cpu_thermal {
    trips {
        cpu_passive: trip-passive@0 {
            temperature = <85000>; /* 85°C */
            hysteresis = <2000>;
            type = "passive";
        };

        cpu_hot: trip-hot@1 {
            temperature = <90000>; /* 90°C */
            hysteresis = <2000>;
            type = "hot";
        };

        cpu_critical: trip-critical@2 {
            temperature = <95000>; /* 95°C */
            hysteresis = <2000>;
            type = "critical";
        };
    };

    cooling-maps {
        map0 {
            trip = <&cpu_passive>;
            cooling-device = <&A53_0 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>;
        };
    };
};

/* ==========================================================================
 * Pin Control Groups
 * ==========================================================================
 */
&iomuxc {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_hog>;

    /* ======================================================================
     * ECSPI2 Pin Configuration
     * ======================================================================
     */
    pinctrl_ecspi2: ecspi2grp {
        fsl,pins = <
            /* SCLK */
            MX8MP_PAD_ECSPI2_SCLK__ECSPI2_SCLK    0x82
            /* MOSI */
            MX8MP_PAD_ECSPI2_MOSI__ECSPI2_MOSI    0x82
            /* MISO */
            MX8MP_PAD_ECSPI2_MISO__ECSPI2_MISO    0x82
        >;
    };

    pinctrl_ecspi2_cs: ecspi2csgrp {
        fsl,pins = <
            /* CS0 as GPIO for manual control */
            MX8MP_PAD_ECSPI2_SS0__GPIO5_IO13      0x40000
        >;
    };

    /* ======================================================================
     * I2C1 Pin Configuration
     * ======================================================================
     */
    pinctrl_i2c1: i2c1grp {
        fsl,pins = <
            /* SCL */
            MX8MP_PAD_I2C1_SCL__I2C1_SCL          0x4000007f
            /* SDA */
            MX8MP_PAD_I2C1_SDA__I2C1_SDA          0x4000007f
        >;
    };

    /* ======================================================================
     * EQOS Ethernet Pin Configuration
     * ======================================================================
     */
    pinctrl_eqos: eqosgrp {
        fsl,pins = <
            /* RGMII TX */
            MX8MP_PAD_ENET1_MDC__ENET_QOS_MDC             0x3
            MX8MP_PAD_ENET1_MDIO__ENET_QOS_MDIO           0x3
            MX8MP_PAD_ENET1_TX_CTL__ENET_QOS_RGMII_TX_CTL 0x3
            MX8MP_PAD_ENET1_TXC__ENET_QOS_RGMII_TXC       0x1f
            MX8MP_PAD_ENET1_TD0__ENET_QOS_RGMII_TD0       0x3
            MX8MP_PAD_ENET1_TD1__ENET_QOS_RGMII_TD1       0x3
            MX8MP_PAD_ENET1_TD2__ENET_QOS_RGMII_TD2       0x3
            MX8MP_PAD_ENET1_TD3__ENET_QOS_RGMII_TD3       0x3
            /* RGMII RX */
            MX8MP_PAD_ENET1_RX_CTL__ENET_QOS_RGMII_RX_CTL 0x3
            MX8MP_PAD_ENET1_RXC__ENET_QOS_RGMII_RXC       0x83
            MX8MP_PAD_ENET1_RD0__ENET_QOS_RGMII_RD0       0x3
            MX8MP_PAD_ENET1_RD1__ENET_QOS_RGMII_RD1       0x3
            MX8MP_PAD_ENET1_RD2__ENET_QOS_RGMII_RD2       0x3
            MX8MP_PAD_ENET1_RD3__ENET_QOS_RGMII_RD3       0x3
            /* PHY control */
            MX8MP_PAD_ENET1_TX_CLK__GPIO4_IO22           0x1f
            MX8MP_PAD_ENET1_RX_ER__GPIO4_IO23            0x1f
        >;
    };

    /* ======================================================================
     * LPUART1 Debug Console
     * ======================================================================
     */
    pinctrl_lpuart1: lpuart1grp {
        fsl,pins = <
            MX8MP_PAD_UART1_RXD__LPUART1_RX      0x140
            MX8MP_PAD_UART1_TXD__LPUART1_TX      0x140
        >;
    };

    /* ======================================================================
     * Default Pin Hog
     * ======================================================================
     */
    pinctrl_hog: hoggrp {
        fsl,pins = <
            /* FPGA bias request */
            MX8MP_PAD_SAI2_TXFS__GPIO4_IO24      0x41
            /* FPGA reset */
            MX8MP_PAD_SAI2_TXC__GPIO4_IO25      0x41
        >;
    };
};
```

---

## 3. Pin Configuration Values

### 3.1 Pad Configuration Bit Fields

```
Bit 31: SION - Software Input On
Bit 30-27: Reserved
Bit 26-24: DSE - Drive Strength
Bit 23-21: Reserved
Bit 20-19: ODE - Open Drain Enable
Bit 18: PUE - Pull/Keep Select
Bit 17: PKE - Pull/Keeper Enable
Bit 16: HYS - Hysteresis Enable
Bit 15-0: Mux Mode Input
```

### 3.2 Common Pad Values

| Value | Hex | Description |
|-------|-----|-------------|
| 0x00000082 | 0x82 | SPI output: 100 ohm ODE, PUE, PKE, HYS, MUX3 |
| 0x0000007f | 0x7f | I2C: SION, ODE, PUE, PKE, HYS, MUX1 |
| 0x00000003 | 0x03 | RGMII: HYS, MUX3 |
| 0x00000140 | 0x140 | UART: HYS, MUX0 |
| 0x00040000 | 0x40000 | GPIO input: PUE, PKE, HYS |

---

## 4. Device Tree Compilation

### 4.1 Build Commands

```bash
# Compile DTS to DTB
dtc -I dts -O dtb -o imx8mp-tft-leakage.dtb imx8mp-tft-leakage.dts

# Or use Yocto build
bitbake linux-imx -c compile

# Check DTB
dtc -I dtb -O dts -o test.dts imx8mp-tft-leakage.dtb
```

### 4.2 DTB Installation

```bash
# Copy to boot partition
cp imx8mp-tft-leakage.dtb /boot/

# Or update in U-Boot
tftpboot imx8mp-tft-leakage.dtb
```

---

## 5. Device Tree Overlays

### 5.1 Overlay for Optional Features

```dts
/dts-v1/;
/plugin/;

/* Overlay for additional FPGA control signals */

&{
    fragment@0 {
        target = <&gpio4>;
        __overlay__ {
            fpga_irq: irq-pin {
                gpio-hog;
                gpios = <26 IRQ_TYPE_EDGE_RISING>;
                line-name = "fpga-irq";
            };
        };
    };

    fragment@1 {
        target-path = "/";
        __overlay__ {
            fpga_irq_key {
                compatible = "gpio-keys";
                pinctrl-names = "default";
                pinctrl-0 = <&pinctrl_fpga_irq>;

                fpga-interrupt {
                    label = "FPGA Interrupt";
                    gpios = <&gpio4 26 GPIO_ACTIVE_HIGH>;
                    linux,code = <256>; /* Custom keycode */
                    wakeup-source;
                };
            };
        };
    };
};
```

---

## 6. Device Tree Validation

### 6.1 Validation Commands

```bash
# Check compiled DTB
dtc -I dtb -O dts imx8mp-tft-leakage.dtb | less

# Check for syntax errors
dtc -I dts -O dtb imx8mp-tft-leakage.dts

# Check at runtime (Linux)
cat /sys/firmware/devicetree/base/model
cat /proc/device-tree/model

# Check SPI nodes
ls -l /sys/bus/spi/devices/

# Check I2C nodes
ls -l /sys/bus/i2c/devices/i2c-1/
```

### 6.2 Debug Output

```bash
# Enable DTB debug
echo 1 > /sys/kernel/debug/tracing/events/kprobes/enable

# Check pin mux
cat /sys/kernel/debug/pinctrl/5e004000.pinctrl-iomuxc/pins
```

---

## 7. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial device tree configuration |

---

**End of Document**
