# SGW — Satellite Modbus Gateway

**Project status:** Development / laboratory prototype
**Last updated:** 2026-10-25
**Primary source of truth:** this file

---

## 1. Project purpose

SGW (Satellite Modbus Gateway) is a custom gateway intended to replace Moxa NPort serial device servers in a system where Modbus RTU devices communicate over a high-latency satellite IP channel.

The key problem is the satellite link latency. The current estimated round-trip time (RTT) is approximately **2200 ms**. With approximately **80 polled devices**, a conventional request/response exchange can become extremely slow.

The project goal is therefore two-stage:

1. reproduce the required serial/Ethernet gateway behaviour so SGW can replace an NPort without changing the Modbus RTU field devices;
2. introduce an SGW-specific transport/gateway mechanism that reduces the effect of satellite latency while preserving correct and safe Modbus control semantics.

The system is used in a control/automation context, so command integrity and predictable failure behaviour have priority over maximum throughput.

---

## 2. Existing system

The known topology is approximately:

```text
                    Satellite / IP channel
                           │
             ┌─────────────┴─────────────┐
             │                           │
        Moxa NPort 5250A             Moxa NPort 5010
        (central side)               (remote side)
             │                           │
           RS-232                    RS-232
             │                           │
          Modbus RTU                Modbus RTU
             │                           │
           SAU / master              remote device
```

The central NPort 5250A has eight serial ports. The current system uses one serial port per remote channel, with each channel communicating with its corresponding remote object. A single eight-port NPort is preferred over eight one-port units partly to avoid unnecessary multiplication of IP addresses.

The exact current NPort operating parameters are not yet available and remain **TBD** until access to the operational equipment is available.

Known/expected compatibility target:

- RS-232;
- Modbus RTU byte stream;
- Ethernet/IP transport;
- NPort-like serial-device-server interoperability sufficient for staged replacement.

---

## 3. Target SGW topology

The first prototype consists of **two identical two-port SGW devices**.

```text
              Ethernet / IP
                   │
          ┌────────┴────────┐
          │                 │
        SGW-A             SGW-B
          │                 │
       RS-232 ×2         RS-232 ×2
          │                 │
      Modbus RTU        Modbus RTU
```

The prototype hardware is intentionally small. After the architecture and transport are proven, the design can be expanded to an eight-port implementation.

---

## 4. Prototype hardware

### Selected

- **MCU:** ESP32-S3
- **Ethernet controller:** W5500
- **Prototype quantity:** 2 MCU boards + 2 W5500 modules
- **Serial side:** RS-232 transceivers are required between ESP32-S3 UARTs and the physical RS-232 interface.

The ESP32-S3 is a development/proof-of-concept choice. It is not yet the final industrial MCU decision.

The W5500 is intended to provide a hardware Ethernet interface and reduce the amount of Ethernet/TCP/IP implementation that must be handled directly by the MCU application.

### Hardware principle

Keep the application/gateway architecture independent of the final PCB and MCU where practical. The laboratory firmware should therefore be structured so that a later industrial MCU/PCB can replace the ESP32-S3 prototype without redesigning the SGW protocol and gateway logic.

---

## 5. Architecture

The firmware should be separated into logical layers:

```text
+------------------------------------------------+
| SGW Application / Gateway Core                 |
| polling / cache / commands / channel state     |
+------------------------------------------------+
| SGW Transport                                  |
| packetization / transaction IDs / ACK / retry  |
+------------------------------------------------+
| Compatibility Layer                             |
| serial parameters / Modbus RTU / NPort mode    |
+------------------------------------------------+
| Hardware Abstraction                            |
| UART / RS-232 / W5500 / timers / GPIO          |
+------------------------------------------------+
| ESP32-S3 + W5500                                |
+------------------------------------------------+
```

The compatibility layer must preserve the Modbus RTU interface seen by field devices. The optimization layer must not require changes to those field devices.

---

## 6. NPort interoperability strategy

Moxa NPort remains the functional reference for the first stage. SGW does **not** need to reproduce every feature of the NPort product; it must first reproduce the subset required by this system.

The migration must support mixed operation during development where practical:

```text
NPort ↔ SGW
```

as well as the final configuration:

```text
SGW ↔ SGW
```

The first objective is transparent operation. Only after transparent operation is demonstrated should the SGW-specific optimization be enabled.

---

## 7. Development sequence

### M0 — Project baseline

Status: **IN PROGRESS / BASELINE CREATED**

- Define the problem and existing topology.
- Establish the repository as the persistent project source.
- Record the selected prototype hardware.
- Define the staged development strategy.

### M1 — Hardware bring-up

Status: **WAITING FOR HARDWARE**

- ESP32-S3 development environment.
- Basic firmware boot and diagnostics.
- UART operation.
- W5500 SPI interface.
- Ethernet link.
- TCP connectivity.
- RS-232 electrical interface.

Acceptance: each hardware subsystem operates independently and diagnostics are available.

### M2 — Basic two-node Ethernet gateway

Status: **PLANNED**

Build a minimal SGW-A ↔ SGW-B connection over ordinary LAN Ethernet.

Initial transport may be ordinary TCP. The transport choice for the optimized satellite mode remains open until experiments establish the requirements.

Acceptance: stable bidirectional byte transfer between the two nodes.

### M3 — Transparent Modbus RTU bridge

Status: **PLANNED**

Implement:

```text
RS-232 → SGW-A → Ethernet → SGW-B → RS-232
```

and reverse direction.

At this stage SGW should behave as a transparent serial gateway from the Modbus device's perspective.

Acceptance:

- Modbus request arrives unchanged;
- response returns correctly;
- serial timing/parameters are preserved;
- independent ports do not interfere;
- RX/TX/error counters are available.

### M4 — NPort interoperability

Status: **PLANNED**

Test SGW against an existing NPort in the laboratory.

Test both directions where applicable:

```text
NPort ↔ SGW
SGW ↔ NPort
```

The exact NPort mode and parameters will be determined from the operational configuration when access becomes available.

Acceptance: SGW can replace an NPort for the selected test channel without modification of the Modbus field device.

### M5 — SGW-to-SGW transport

Status: **PLANNED**

Introduce the first version of the SGW transport protocol.

Required concepts include:

- packet framing;
- protocol version;
- channel/port identification;
- transaction ID;
- payload length;
- integrity/error detection;
- transport acknowledgement;
- duplicate detection;
- retransmission/recovery;
- connection/session state.

The exact wire format is **not frozen yet**.

### M6 — Satellite simulation

Status: **PLANNED**

Test the system with artificial latency approximately matching the real link:

**RTT ≈ 2200 ms**.

Also test:

- packet loss;
- delayed packets;
- Ethernet/TCP disconnect;
- reconnection;
- duplicate delivery;
- restart of one gateway.

Acceptance: deterministic recovery and clearly defined channel state after failures.

### M7 — Latency optimization

Status: **PLANNED**

Compare conventional Modbus request/response transport with SGW optimized transport.

Potential mechanisms to evaluate:

- aggregation of transactions;
- pipelining;
- local polling on the remote side;
- telemetry cache;
- block transfer of results;
- priority queues;
- independent control-command path.

The project must measure actual improvement rather than assume that simply placing several Modbus frames into one TCP packet solves the RTT problem.

### M8 — Control-command safety

Status: **PLANNED**

Control commands such as changing a valve position require stronger semantics than ordinary telemetry.

The protocol must distinguish at least:

```text
RECEIVED / transport ACK
EXECUTED / execution confirmed
FAILED / device reported failure
UNKNOWN / execution state cannot be established
```

A transport acknowledgement must never be treated as proof that a control action was executed.

After reconnection, an old control command must not be silently replayed merely because it was not acknowledged.

### M9 — First field channel

Status: **PLANNED**

Replace one existing NPort channel under controlled conditions.

Recommended sequence:

1. establish known-good NPort baseline;
2. install SGW in compatibility mode;
3. observe/log operation;
4. validate failure recovery;
5. enable optimized transport only after compatibility is proven;
6. compare performance with baseline.

The migration must remain reversible.

### M10 — Multi-channel / final hardware

Status: **PLANNED**

After successful two-port laboratory and field validation:

- scale the design toward eight serial ports;
- select final MCU;
- design custom PCB;
- define enclosure/power requirements;
- perform long-duration testing;
- progressively replace remaining NPort channels.

---

## 8. Critical design requirements

### 8.1 Channel isolation

Each logical channel must remain independent. Traffic or failure on one channel must not corrupt another channel's state.

### 8.2 Control priority

Control commands have higher priority than ordinary telemetry where simultaneous traffic competes for the satellite link.

### 8.3 Duplicate protection

The protocol must make it possible to identify and suppress duplicate execution of a control transaction.

### 8.4 Failure semantics

The gateway must distinguish transport state from remote-device execution state.

### 8.5 Freshness

Cached telemetry must carry freshness/age information so stale values cannot be mistaken for current measurements.

### 8.6 Recovery

A link interruption must result in a deterministic state machine for reconnect, resynchronization and outstanding transactions.

---

## 9. What is intentionally NOT decided yet

The following must remain open until laboratory tests and requirements justify a decision:

- TCP vs UDP vs hybrid SGW transport;
- exact SGW packet format;
- exact aggregation/pipelining algorithm;
- local remote-side polling strategy;
- final MCU;
- final number of serial ports;
- final PCB;
- final power supply;
- exact NPort compatibility feature subset.

We should not prematurely freeze these items.

---

## 10. Development principle: reuse before invention

For each significant subsystem, first search for an existing proven implementation, library, protocol or hardware design that can reduce development time and risk.

This applies especially to:

- Modbus RTU framing;
- serial/Ethernet gateway logic;
- W5500 drivers;
- TCP/IP handling;
- packet framing and CRC;
- retransmission/session logic;
- embedded diagnostics;
- configuration mechanisms.

Custom code should be introduced where existing solutions cannot satisfy the satellite-latency/control requirements.

---

## 11. Current project state

**Hardware ordered:**

- 2 × ESP32-S3 development boards;
- 2 × W5500 modules.

**Current laboratory target:**

Two identical two-port SGW prototypes.

**Current architectural decision:**

First implement transparent/compatibility operation, including NPort interoperability, then introduce optimized SGW transport.

**Known satellite condition:**

Estimated RTT ≈ 2200 ms.

**Known scale:**

Approximately 80 polled devices in the broader system; the first prototype is intentionally limited to two ports.

**Immediate next task:**

Prepare the ESP32-S3/W5500 laboratory firmware and test plan so that hardware can be tested immediately when the ordered components arrive.

---

## 12. Working rule for this repository

This file is the persistent project state.

When a significant technical decision, implementation result, test result, hardware change or requirement is established during SGW development, it should be reflected here. The user should not be required to manually maintain project documentation; project-state updates are to be made through the GitHub repository as part of the development workflow.

Before beginning a new SGW work session, inspect the repository state and use this document to determine the current milestone and next action.
