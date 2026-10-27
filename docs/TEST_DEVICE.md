# SGW — Experimental NPort-Compatible Test Device

**Status:** PLANNED / FIRST HARDWARE PROTOTYPE
**Purpose:** быстро проверить базовую архитектуру Serial ↔ IP на доступном и недорогом MCU до реализации полной SGW-архитектуры.

## Hardware baseline

- MCU: **STM32F103C8T6**
- Ethernet controller: **W5500** via SPI
- RS-232 transceiver: **MAX3232**
- Initial serial ports: **1 × RS-232**
- Ethernet: **1 × W5500**

```text
PC / LAN
   │
Ethernet
   │
 W5500
   │ SPI
   ▼
STM32F103C8T6
   │ USART1
   ▼
MAX3232
   │
RS-232
   │
Test equipment / NPort / САУ
```

## Firmware objective

First implementation is a minimal **NPort-compatible transparent TCP ↔ RS-232 bridge**.

The payload is opaque. No Modbus parsing, CRC processing, packetisation or satellite-link optimisation is performed in this test device.

### Required first functions

1. W5500 initialisation and Ethernet connectivity.
2. TCP Server mode.
3. Bidirectional TCP ↔ USART1 byte forwarding.
4. Bounded RX/TX buffers.
5. RX/TX and overflow counters.
6. Connection/disconnection state indication.
7. UART parameters configurable for the connected equipment.

## Test sequence

**T0 — Hardware bring-up**
- STM32F103C8T6 boots and is programmable via SWD.
- W5500 responds over SPI.
- MAX3232 electrical interface is verified before connection to equipment.

**T1 — Ethernet**
- Device obtains/configures an IP address.
- Device is reachable from the test PC.

**T2 — TCP bridge**
- PC connects to the configured TCP port.
- Bytes sent by PC appear unchanged on RS-232 TX.
- Bytes received on RS-232 appear unchanged at the TCP peer.

**T3 — NPort interoperability**
- Test against the actual NPort using its verified TCP Client/Server configuration.
- Confirm bidirectional byte-for-byte transfer.

**T4 — Modbus equipment**
- Only after transparent bridging passes, connect the real Modbus RTU equipment over RS-232.
- Verify that Modbus RTU frames pass unchanged through the bridge.

## PASS criterion

The test device passes when a sustained bidirectional serial byte stream is transferred through Ethernet with **identical byte content and order**, without buffer overflow or unexplained connection loss.

Successful T4 demonstrates that the transparent bridge can carry the target Modbus RTU traffic; it does **not** yet demonstrate SGW's future satellite optimisation.

## Relationship to SGW

This device is an **experimental compatibility/transport prototype**, not the final SGW architecture.

The SGW project remains protocol-agnostic at V0. After the transparent bridge is proven, the same transport concepts can be used as the baseline for later SGW milestones: two-node operation, controlled high-latency testing, and only then protocol-aware optimisation.

The final MCU and port count remain **TBD**. STM32F103C8T6 is selected here only for the first low-cost laboratory prototype.
