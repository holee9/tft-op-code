# Ethernet Communication

## Yocto Project Team Delivery Package

**Document**: 07_ethernet_communication.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Overview

The i.MX8 Plus communicates with the Windows Host PC via 1 Gbps Ethernet. This document specifies the network configuration and protocol.

---

## 2. Network Configuration

### 2.1 Interface Specifications

| Parameter | Value |
|-----------|-------|
| **Controller** | EQOS (Enhanced Quality of Service) |
| **Speed** | 1 Gbps (auto-negotiate) |
| **Interface** | eth0 |
| **PHY** | RGMII |
| **MAC Address** | Configurable (default in Device Tree) |

### 2.2 IP Configuration

#### Static IP (Recommended for Production)

```
i.MX8 Side:
  IP Address: 192.168.1.100/24
  Gateway: 192.168.1.1
  DNS: 192.168.1.1

Host PC Side:
  IP Address: 192.168.1.1/24
  Gateway: (none)
```

#### DHCP (Development Mode)

```
i.MX8:
  DHCP client enabled
  Address assigned by DHCP server
```

### 2.3 Network Manager Configuration

```bash
# Using systemd-networkd

# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
DHCP=no
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=192.168.1.1

[Link]
MTUBytes=1500
```

---

## 3. Communication Protocol

### 3.1 Protocol Overview

| Parameter | Value |
|-----------|-------|
| **Transport** | TCP (for control), UDP (for data) |
| **Port (Control)** | 5000 |
| **Port (Data)** | 5001 |
| **Byte Order** | Little Endian |
| **Timeout** | 5 seconds |

### 3.2 Message Format

#### Control Message Header

```csharp
struct MessageHeader {
    uint32_t Magic;      // 0xA5A5A5A5 (sync)
    uint32_t Length;     // Payload length
    uint32_t MessageType;// See below
    uint32_t Sequence;   // Sequence number
    uint64_t Timestamp;  // Unix timestamp (ms)
};
```

#### Message Types

| Type | Value | Description |
|------|-------|-------------|
| `PING` | 0x0001 | Keep-alive ping |
| `PONG` | 0x0002 | Ping response |
| `GET_STATUS` | 0x0010 | Request system status |
| `STATUS_RESPONSE` | 0x0011 | Status data |
| `START_FRAME` | 0x0020 | Start frame capture |
| `FRAME_DATA` | 0x0021 | Frame pixel data |
| `FRAME_COMPLETE` | 0x0022 | Frame done notification |
| `SET_BIAS` | 0x0030 | Set bias mode |
| `GET_TEMP` | 0x0040 | Get temperature |
| `TEMP_RESPONSE` | 0x0041 | Temperature data |

---

## 4. System Status Message

### 4.1 Status Response Format

```csharp
struct SystemStatus {
    MessageHeader Header;
    float Temperature;       // Panel temperature (°C)
    uint32_t IdleTimeSeconds; // Idle time (s)
    uint8_t IdleMode;         // L1=0, L2=1, L3=2
    uint32_t CalculatedTMax;   // t_max at current T (s)
    float DriftRate;          // Current drift rate (DN/min)
    uint8_t WarmupRequired;    // 0/1
    uint8_t Reserved[3];       // Padding
};
```

### 4.2 C# Implementation

```csharp
using System.Net.Sockets;
using System.Buffers.Binary;

public class HostCommunicationServer : BackgroundService
{
    private readonly TcpListener _listener;
    private readonly ILogger<HostCommunicationServer> _logger;

    public HostCommunicationServer(ILogger<HostCommunicationServer> logger)
    {
        _logger = logger;
        _listener = new TcpListener(IPAddress.Any, 5000);
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _listener.Start();
        _logger.LogInformation("TCP server listening on port 5000");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var client = await _listener.AcceptTcpClientAsync(stoppingToken);
                _ = ProcessClientAsync(client, stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Accept failed");
            }
        }
    }

    private async Task ProcessClientAsync(TcpClient client, CancellationToken token)
    {
        using var stream = client.GetStream();
        using var reader = new BinaryReader(stream);
        using var writer = new BinaryWriter(stream);

        while (!token.IsCancellationRequested)
        {
            // Read header
            MessageHeader header = ReadHeader(reader);

            // Process message
            switch (header.MessageType)
            {
                case 0x0010: // GET_STATUS
                    await WriteStatusAsync(writer);
                    break;
                // ... other message types
            }
        }
    }
}
```

---

## 5. Frame Data Transfer

### 5.1 Frame Data Format

For high-speed frame data transfer, use UDP:

```csharp
struct FrameDataPacket {
    uint32_t Magic;          // 0xD0D0D0D0
    uint32_t PacketIndex;    // Packet number
    uint32_t TotalPackets;   // Total in frame
    uint32_t PixelCount;     // Pixels in this packet
    uint16_t PixelData[];    // 14-bit pixel data
};
```

### 5.2 UDP Server Configuration

```csharp
using System.Net;
using System.Net.Sockets;

public class FrameDataServer
{
    private readonly UdpClient _udp;

    public FrameDataServer()
    {
        _udp = new UdpClient(5001);
    }

    public async Task SendFrameAsync(ushort[] frameData, IPEndPoint host)
    {
        const int maxPixels = 1024;
        int totalPackets = (frameData.Length + maxPixels - 1) / maxPixels;

        for (int i = 0; i < totalPackets; i++)
        {
            int offset = i * maxPixels;
            int count = Math.Min(maxPixels, frameData.Length - offset);

            var packet = new byte[16 + count * 2];
            // ... pack packet ...

            await _udp.SendAsync(packet, packet.Length, host);
        }
    }
}
```

---

## 6. Device Tree Configuration

### 6.1 EQOS Ethernet Node

```dts
&eqos {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_eqos>;
    phy-mode = "rgmii-id";
    phy-handle = <&ethphy0>;
    mac-address = [00 04 25 1c a0 b0];
    status = "okay";

    mdio {
        #address-cells = <1>;
        #size-cells = <0>;

        ethphy0: ethernet-phy@0 {
            compatible = "ethernet-phy-ieee802.3-c22";
            reg = <0>;
            max-speed = <1000>;
            reset-gpios = <&gpio4 22 GPIO_ACTIVE_LOW>;
            reset-assert-us = <10000>;
            reset-deassert-us = <1000>;
        };
    };
};
```

### 6.2 Pin Configuration

```dts
pinctrl_eqos: eqosgrp {
    fsl,pins = <
        /* TX */
        MX8MP_PAD_ENET1_MDC__ENET_QOS_MDC              0x3
        MX8MP_PAD_ENET1_MDIO__ENET_QOS_MDIO            0x3
        MX8MP_PAD_ENET1_TX_CTL__ENET_QOS_RGMII_TX_CTL   0x3
        MX8MP_PAD_ENET1_TXC__ENET_QOS_RGMII_TXC        0x1f
        MX8MP_PAD_ENET1_TD0__ENET_QOS_RGMII_TD0        0x3
        MX8MP_PAD_ENET1_TD1__ENET_QOS_RGMII_TD1        0x3
        MX8MP_PAD_ENET1_TD2__ENET_QOS_RGMII_TD2        0x3
        MX8MP_PAD_ENET1_TD3__ENET_QOS_RGMII_TD3        0x3
        /* RX */
        MX8MP_PAD_ENET1_RX_CTL__ENET_QOS_RGMII_RX_CTL   0x3
        MX8MP_PAD_ENET1_RXC__ENET_QOS_RGMII_RXC        0x83
        MX8MP_PAD_ENET1_RD0__ENET_QOS_RGMII_RD0        0x3
        MX8MP_PAD_ENET1_RD1__ENET_QOS_RGMII_RD1        0x3
        MX8MP_PAD_ENET1_RD2__ENET_QOS_RGMII_RD2        0x3
        MX8MP_PAD_ENET1_RD3__ENET_QOS_RGMII_RD3        0x3
        /* Control */
        MX8MP_PAD_ENET1_TX_CLK__GPIO4_IO22             0x1f
        MX8MP_PAD_ENET1_RX_ER__GPIO4_IO23              0x1f
    >;
};
```

---

## 7. Kernel Configuration

### 7.1 Required Config Options

```kconfig
CONFIG_NETDEVICES=y
CONFIG_STMMAC_ETH=y
CONFIG_STMMAC_PCI=y
CONFIG_DWMAC_GENERIC=y
CONFIG_DWMAC_IMX8=y
CONFIG_PHYLIB=y
CONFIG_MICREL_PHY=y
CONFIG_AT803X_PHY=y
```

---

## 8. Network Testing

### 8.1 Basic Connectivity Tests

```bash
# Check interface
ip link show eth0

# Bring up interface
ip link set eth0 up

# Configure IP
ip addr add 192.168.1.100/24 dev eth0

# Test connectivity
ping 192.168.1.1

# Measure bandwidth
iperf3 -s
# On host: iperf3 -c 192.168.1.100
```

### 8.2 Port Testing

```bash
# Check if port is listening
netstat -tuln | grep 5000

# Test TCP port
nc -zv 192.168.1.100 5000

# Test UDP port
nc -uzv 192.168.1.100 5001
```

---

## 9. Firewall Configuration

### 9.1 iptables Rules

```bash
# Allow TCP port 5000 (control)
iptables -A INPUT -p tcp --dport 5000 -j ACCEPT

# Allow UDP port 5001 (data)
iptables -A INPUT -p udp --dport 5001 -j ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

### 9.2 firewalld Configuration

```bash
# Add TCP port
firewall-cmd --permanent --add-port=5000/tcp

# Add UDP port
firewall-cmd --permanent --add-port=5001/udp

# Reload
firewall-cmd --reload
```

---

## 10. Troubleshooting

### 10.1 Common Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| Link down | NO CARRIER | Check PHY, cable, pin mux |
| Can't ping | Request timed out | Check IP addresses, firewall |
| Slow transfer | Low bandwidth | Check duplex, negotiation |
| Port blocked | Connection refused | Check firewall, service status |

### 10.2 Debug Commands

```bash
# Check driver
dmesg | grep -i eqos

# Check PHY registers
cat /sys/class/mdio_bus/*/phy_id/0/0/*/speed

# Check statistics
ethtool -S eth0

# Check link
ethtool eth0

# Packet capture
tcpdump -i eth0 -w capture.pcap port 5000
```

---

## 11. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial Ethernet communication document |

---

**End of Document**
