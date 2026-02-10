# Communication Protocol

## .NET Project Team Delivery Package

**Document**: 04_communication_protocol.md
**Version**: 1.0
**Date**: 2026-02-10

---

## 1. Protocol Overview

| Parameter | Value |
|-----------|-------|
| **Transport** | TCP (control), UDP (data) |
| **Host** | 192.168.1.100 (i.MX8) |
| **Port (Control)** | 5000 |
| **Port (Data)** | 5001 |
| **Byte Order** | Little Endian |

---

## 2. Message Format

```
Header (24 bytes):
┌─────────────────────────────────────────────────────────┐
│ Magic (4) │ Length (4) │ Type (4) │ Seq (4) │ Time (8) │
├─────────────────────────────────────────────────────────┤
│ 0xA5A5A5A5 │ Payload Len │ Msg Type  │ Sequence  │ Unix ms │
└─────────────────────────────────────────────────────────┘

Payload (variable):
┌─────────────────────────────────────────────────────────┐
│ Message-specific data                                   │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Message Types

| Type | Value | Direction |
|------|-------|-----------|
| Ping | 0x0001 | Host → i.MX8 |
| Pong | 0x0002 | i.MX8 → Host |
| GetStatus | 0x0010 | Host → i.MX8 |
| StatusResponse | 0x0011 | i.MX8 → Host |
| StartFrame | 0x0020 | Host → i.MX8 |
| FrameData | 0x0021 | i.MX8 → Host (UDP) |
| FrameComplete | 0x0022 | i.MX8 → Host |

---

## 4. Status Response Payload

```
┌─────────────────────────────────────────────────────────┐
│ Temp (4) │ IdleTime (4) │ Mode (1) │ TMax (4) │ Drift (4) │
├─────────────────────────────────────────────────────────┤
│ float°C  │ seconds       │ L1/L2/L3  │ seconds   │ DN/min   │
└─────────────────────────────────────────────────────────┘
```

---

## 5. C# Implementation

```csharp
public class TftLeakageClient : IDisposable
{
    private TcpClient _tcp;
    private UdpClient _udp;

    public async Task ConnectAsync(string host, int port)
    {
        _tcp = new TcpClient();
        await _tcp.ConnectAsync(host, port);
        _udp = new UdpClient();
    }

    public async Task<SystemStatus> GetStatusAsync()
    {
        var header = new MessageHeader
        {
            MessageType = (uint)MessageType.GetStatus,
            Timestamp = (ulong)DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
        };

        await SendMessageAsync(header);
        var response = await ReceiveMessageAsync();
        return ParseStatusResponse(response);
    }
}
```

---

## 6. UDP Frame Data

```
UDP Packet for Frame Data:
┌─────────────────────────────────────────────────────────┐
│ Magic (4) │ PacketIdx (4) │ Total (4) │ Count (4) │ Data... │
├─────────────────────────────────────────────────────────┤
│ 0xD0D0D0D0 │ 0-based       │ Total pkts │ Pixel cnt │ 16-bit  │
└─────────────────────────────────────────────────────────┘
```

---

## 7. Error Handling

| Error Code | Value | Description |
|------------|-------|-------------|
| Success | 0x00 | OK |
| InvalidCommand | 0x01 | Unknown message type |
| DeviceBusy | 0x03 | Frame capture in progress |
| HardwareError | 0x04 | FPGA communication error |

---

## 8. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | MoAI | Initial protocol |

---

**End of Document**
