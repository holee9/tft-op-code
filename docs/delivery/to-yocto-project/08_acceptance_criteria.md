# Acceptance Criteria

## Yocto Project Team Delivery Package

**Document**: 08_acceptance_criteria.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Overview

This document defines the acceptance criteria for the Yocto BSP implementation. All criteria must be met before the BSP is considered complete.

---

## 2. Functional Requirements Validation

### FR-1: SPI Driver

| ID | Test Case | Description | Pass Criteria |
|----|-----------|-------------|---------------|
| FR-1.1 | TB_SPI_001 | Device node creation | /dev/spidev1.0 exists |
| FR-1.2 | TB_SPI_002 | Mode 0 operation | Correct CPOL=0, CPHA=0 behavior |
| FR-1.3 | TB_SPI_003 | 10 MHz operation | Clean operation at 10 MHz |
| FR-1.4 | TB_SPI_004 | Register write | FPGA registers write correctly |
| FR-1.5 | TB_SPI_005 | Register read | FPGA registers read correctly |
| FR-1.6 | TB_SPI_006 | Burst read | Sequential read works |

### FR-2: I2C Temperature Sensor

| ID | Test Case | Description | Pass Criteria |
|----|-----------|-------------|---------------|
| FR-2.1 | TB_I2C_001 | Device detection | Sensor detected at 0x48 |
| FR-2.2 | TB_I2C_002 | Temperature read | Valid temperature returned |
| FR-2.3 | TB_I2C_003 | Accuracy | ±2°C at 25°C |
| FR-2.4 | TB_I2C_004 | Polling rate | 1 Hz ±5% |

### FR-3: GPIO Control

| ID | Test Case | Description | Pass Criteria |
|----|-----------|-------------|---------------|
| FR-3.1 | TB_GPIO_001 | Output mode | GPIO can set output |
| FR-3.2 | TB_GPIO_002 | Value setting | High/Low sets correctly |
| FR-3.3 | TB_GPIO_003 | Bias control | FPGA bias requests work |

### FR-4: Ethernet Communication

| ID | Test Case | Description | Pass Criteria |
|----|-----------|-------------|---------------|
| FR-4.1 | TB_ETH_001 | Link up | eth0 shows carrier |
| FR-4.2 | TB_ETH_002 | IP configuration | Static/DHCP works |
| FR-4.3 | TB_ETH_003 | TCP socket | Port 5000 accepts connections |
| FR-4.4 | TB_ETH_004 | UDP data | Port 5001 receives data |
| FR-4.5 | TB_ETH_005 | Bandwidth | >500 Mbps achievable |

### FR-5: .NET Runtime

| ID | Test Case | Description | Pass Criteria |
|----|-----------|-------------|---------------|
| FR-5.1 | TB_DOTNET_001 | Runtime install | .NET 8 packages installed |
| FR-5.2 | TB_DOTNET_002 | Application startup | Main app starts without errors |
| FR-5.3 | TB_DOTNET_003 | SPI access | System.Device.Spi works |
| FR-5.4 | TB_DOTNET_004 | I2C access | System.Device.I2c works |
| FR-5.5 | TB_DOTNET_005 | GPIO access | System.Device.Gpio works |

### FR-6: System Services

| ID | Test Case | Description | Pass Criteria |
|----|-----------|-------------|---------------|
| FR-6.1 | TB_SVC_001 | systemd service | Service file installed |
| FR-6.2 | TB_SVC_002 | Auto-start | Service starts at boot |
| FR-6.3 | TB_SVC_003 | Restart on failure | Auto-restart works |

---

## 3. Performance Requirements

### PR-1: Boot Time

| Metric | Target | Measurement |
|--------|--------|-------------|
| Time to systemd | < 10 seconds | systemd-analyze |
| Time to network | < 15 seconds | ip link show |
| Time to service ready | < 20 seconds | systemctl status |

### PR-2: SPI Performance

| Metric | Target | Measurement |
|--------|--------|-------------|
| Transfer rate | ≥ 5 Mbit/s | Loopback test |
| Register access latency | < 100 µs | oscilloscope/logic analyzer |
| Error rate | < 0.01% | Extended test |

### PR-3: Temperature Monitoring

| Metric | Target | Measurement |
|--------|--------|-------------|
| Polling rate | 1 Hz ±5% | Log timestamps |
| Filter settling | < 10 samples | Step response |
| Arrhenius accuracy | ±5% | Compare to LUT |

### PR-4: Network Performance

| Metric | Target | Measurement |
|--------|--------|-------------|
| TCP throughput | > 500 Mbps | iperf3 |
| UDP throughput | > 600 Mbps | iperf3 |
| Latency | < 1 ms | ping |
| Jitter | < 100 µs | iperf3 |

---

## 4. Test Procedures

### 4.1 Automated Tests

```bash
#!/bin/bash
# acceptance_test.sh

echo "=== Yocto BSP Acceptance Tests ==="

# Test 1: SPI Device
if [ -e /dev/spidev1.0 ]; then
    echo "[PASS] SPI device exists"
else
    echo "[FAIL] SPI device missing"
fi

# Test 2: I2C Sensor
if i2cdetect -y 1 | grep -q "48"; then
    echo "[PASS] Temperature sensor detected"
else
    echo "[FAIL] Temperature sensor not detected"
fi

# Test 3: Network
if ip link show eth0 | grep -q "state UP"; then
    echo "[PASS] Ethernet link up"
else
    echo "[FAIL] Ethernet link down"
fi

# Test 4: .NET Runtime
if dotnet --info | grep -q "8.0"; then
    echo "[PASS] .NET 8 installed"
else
    echo "[FAIL] .NET 8 not found"
fi
```

### 4.2 Manual Tests

| Test | Steps | Expected Result |
|------|-------|-----------------|
| SPI Write | Use spidev_test | Data echoed back |
| FPGA Register | Write CTRL_REG | Register changes |
| Temperature | Read I2C register | Valid temperature |
| Network | Ping host | < 1ms latency |
| Service | systemctl status | Active (running) |

---

## 5. Deliverables Checklist

### 5.1 Yocto Layer

- [ ] `meta-tft-leakage` layer directory
- [ ] `conf/layer.conf`
- [ ] `recipes-core/images/tft-leakage-image.bb`
- [ ] `recipes-graphics/dotnet/dotnet-runtime_8.0.bb`
- [ ] `recipes-apps/tft-leakage/tft-leakage_1.0.bb`

### 5.2 Device Tree Files

- [ ] `imx8mp-tft-leakage.dts`
- [ ] Compiled `imx8mp-tft-leakage.dtb`

### 5.3 Kernel Files

- [ ] Kernel configuration fragment `tft-leakage.cfg`
- [ | Kernel `.config` file
- [ ] `Image` kernel image
- [ ] Kernel modules archive

### 5.4 Service Files

- [ ] `tft-leakage.service`
- [ ] `tft-leakage.socket` (if using socket activation)

### 5.5 Configuration Files

- [ ] `appsettings.json`
- [ ] `10-eth0.network` (systemd-networkd)

### 5.6 Documentation

- [ ] Build instructions
- [ ] Flashing guide
- [ ] Troubleshooting guide

---

## 6. Validation Matrix

### 6.1 Build Validation

| Check | Command | Expected |
|-------|---------|----------|
| BitBake version | `bitbake --version` | ≥ 2.0 |
| Layer path | `bitbake-layers show` | meta-tft-leakage shown |
| Build success | `bitbake tft-leakage-image` | Exit code 0 |

### 6.2 Boot Validation

| Check | Command | Expected |
|-------|---------|----------|
| Kernel boots | - | Login prompt |
| Device nodes | `ls -l /dev/spidev*` | spidev1.0 present |
| Network up | `ip addr` | eth0 has IP |
| Service running | `systemctl status` | tft-leakage active |

### 6.3 Functional Validation

| Check | Test | Expected |
|-------|------|----------|
| SPI comm | `spidev_test` | Clean output |
| I2C comm | `i2cdetect -y 1` | Device at 0x48 |
| Temp read | Custom app | Valid temperature |
| Network | `ping 192.168.1.1` | < 1ms latency |
| .NET app | `systemctl start tft-leakage` | Service starts |

---

## 7. Sign-Off Criteria

The Yocto BSP is considered complete when:

1. [ ] All functional requirements (FR) pass tests
2. [ ] All performance requirements (PR) met
3. [ ] Automated test suite passes 100%
4. [ ] Manual tests pass
5. [ ] All deliverables submitted
6. [ ] Build documentation complete
7. [ ] Can flash to target hardware
8. [ ] System boots and runs application

---

## 8. Test Report Template

```markdown
## Yocto BSP Test Report

**Image**: tft-leakage-image
**Version**: 1.0
**Date**: YYYY-MM-DD
**Tester**: Name

### Test Summary

| Category | Run | Passed | Failed | Blocked |
|----------|-----|--------|--------|---------|
| SPI Tests | 6 | 6 | 0 | 0 |
| I2C Tests | 4 | 4 | 0 | 0 |
| GPIO Tests | 3 | 3 | 0 | 0 |
| Network Tests | 5 | 5 | 0 | 0 |
| .NET Tests | 5 | 5 | 0 | 0 |
| Service Tests | 3 | 3 | 0 | 0 |
| **Total** | **26** | **26** | **0** | **0** |

### Performance Results

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Boot time | < 20 s | 18.5 s | ✓ |
| SPI rate | ≥ 5 Mbit/s | 6.2 Mbit/s | ✓ |
| Network throughput | > 500 Mbps | 850 Mbps | ✓ |

### Deliverables

- [x] Yocto layer
- [x] Device tree files
- [x] Kernel configuration
- [x] Service files
- [x] Documentation

### Sign-Off

[ ] Approved for integration
[ ] Requires rework
[ ] Failed - see issues

**Signature**: _________________
**Date**: _________________
```

---

## 9. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial acceptance criteria |

---

**End of Document**
