# SGW — Satellite Modbus Gateway

**Project status:** Development / laboratory prototype  
**Last updated:** 2026-10-25  
**Primary source of truth:** this file

---

## 1. Project purpose

SGW is a protocol-agnostic serial device-server prototype intended eventually to replace Moxa NPort devices in a system communicating over a high-latency satellite IP channel.

The current estimated satellite round-trip time is approximately **2200 ms**. The broader system has approximately **80 polled devices**, but those higher-level Modbus concerns are explicitly outside the first prototype.

### First engineering objective

> Demonstrate a reliable, bidirectional, protocol-independent serial-byte bridge over IP, using Moxa NPort as a behavioural reference.

For SGW V0, every serial payload is opaque. The firmware must not parse Modbus, apply CRC rules to the payload, identify command meaning, or optimise transactions. Those are potential later milestones, after the transparent transport works.

---

## 2. Existing system and reference equipment

The known system uses Moxa NPort serial device servers. The central side has been described as an eight-port NPort and the remote side typically as a one-port NPort. Model numbers mentioned in project history include NPort 5250A/5030 on the central side and 5010 on the remote side; **the exact installed model and configuration for the intended bench test must be verified**.

Expected compatibility boundary:

- serial byte stream remains unchanged;
- serial parameters are configurable;
- network connection and recovery behaviour are observable;
- SGW does not need to know the payload protocol;
- mixed NPort ↔ SGW testing is performed only using a compatible, explicitly configured NPort operating mode.

NPort operating modes are configurable, not universal. Before a real interoperability test, record the exact model, port mode (e.g. TCP Client, TCP Server or Pair Connection), IP addresses/ports, and serial settings. Do not assume that any generic TCP socket automatically makes two devices interchangeable.

---

## 3. Target topology

The final prototype target remains two identical two-port nodes:

```text
                  IP network / later satellite
                    ┌───────────────┐
                    │               │
                  SGW-A  ◄──────►  SGW-B
                    │               │
                RS-232 ×2       RS-232 ×2
```

The nodes should use a common firmware base with roles and network parameters defined by configuration. A later revision may scale to eight serial ports on the central side.

---

## 4. Hardware state — distinguish available from ordered

### Available now

- **1 × ESP32-S3 development board** is available and may temporarily be borrowed from another project for SGW bench experiments.
- The exact ESP32-S3 board SKU, flash size and PSRAM configuration have not yet been recorded.

### Previously ordered for the intended prototype

- 2 × ESP32-S3 development boards;
- 2 × W5500 Ethernet modules.

These are marked **ORDERED / NOT YET VERIFIED AS DELIVERED** until the user confirms arrival.

### Serial electrical interface

ESP32-S3 UART signals are logic-level signals, not RS-232 line levels. A suitable RS-232 transceiver is required between the MCU UART and a physical NPort RS-232 connection. Do not connect UART GPIO directly to an RS-232 port.

### Temporary network path

For the first experiment, the available ESP32-S3 may use its built-in Wi-Fi instead of W5500. This is a temporary laboratory substitution, not a change to the target wired-Ethernet design.

The ESP32-S3 Wi-Fi path can validate IP/TCP application logic and basic interoperability. It does **not** validate W5500 hardware, wired Ethernet performance, nor the eventual satellite-link behaviour.

---

## 5. Architecture principle

Develop the smallest working layer first:

```text
RS-232 device
     │
RS-232 transceiver
     │
ESP32-S3 UART
     │
opaque byte buffers
     │
TCP socket over Wi-Fi (temporary)
     │
Moxa NPort / test endpoint

Later: replace the temporary Wi-Fi link with the intended Ethernet path.
```

For the actual two-node byte bridge, the target flow is:

```text
Serial A → SGW-A → IP transport → SGW-B → Serial B
Serial B → SGW-B → IP transport → SGW-A → Serial A
```

The transport must preserve the stream's byte order and content. TCP does not preserve application write boundaries as message boundaries; the bridge must forward the stream as bytes and must not rely on one socket read corresponding to one serial frame or one socket send.

### Logical software layers

- **Hardware interface:** UART, Wi-Fi initially, W5500 later, GPIO and timers.
- **Byte-stream bridge:** bounded buffers, forward progress, backpressure/overflow accounting.
- **Connection manager:** socket state, reconnect handling, diagnostics.
- **Configuration/diagnostics:** network settings, serial parameters, counters and reset reason.

Modbus parsing, local polling, cache, data concentration and command semantics are explicitly postponed.

---

## 6. Temporary ESP32-S3 + Wi-Fi + Moxa experiment

The existing single ESP32-S3 can be used to start the network-side experiment before the remaining boards and W5500 arrive.

### Possible test arrangement

One possible first setup is:

```text
Moxa NPort (configured TCP Client or Server)
              │
          Wi-Fi/LAN
              │
      ESP32-S3 TCP endpoint
```

Which endpoint must act as TCP client/server depends on the actual NPort mode and test topology. Determine this from the NPort configuration; do not guess.

This **first test only proves that the ESP32 can communicate with the configured NPort over IP**. It does not yet prove a serial bridge because the ESP32 needs a suitable RS-232 transceiver and application that couples UART bytes to the socket.

### Test stages

**W0 — Board identity and firmware toolchain**
- Record exact board/module marking and available USB/serial interfaces.
- Build and flash a minimal firmware.
- Acceptance: stable boot log, known firmware/toolchain version, and repeatable flashing.

**W1 — Wi-Fi network**
- Connect ESP32-S3 to a controlled local Wi-Fi access point.
- Log assigned IP, RSSI (as available), disconnect/reconnect events.
- Acceptance: stable IP connectivity and recovery after AP disconnection.

**W2 — TCP socket interoperability with NPort**
- Record exact NPort model, operation mode, local port, destination IP/port and serial settings.
- Implement the required TCP role on ESP32-S3.
- Exchange a known test pattern with a TCP peer/NPort where the configured mode permits.
- Acceptance: the test confirms bidirectional TCP payload delivery; save logs and configuration summary.

**W3 — RS-232 electrical connection**
- Identify the available RS-232 transceiver/interface module.
- Confirm correct DB9 signals and whether the NPort/device expects DTE or DCE wiring, including any null-modem crossover requirement.
- Acceptance: UART↔RS-232 loopback works without directly connecting MCU GPIO to RS-232 voltage levels.

**W4 — Byte-stream bridge**
- Couple UART RX/TX and the TCP socket in both directions.
- Use a repeatable test pattern including all byte values where practical, long bursts and irregular pauses.
- Acceptance: exact payload bytes and order are preserved; counters match; queue overflow and disconnect behaviour are visible.

This sequence deliberately separates IP/TCP checks from serial electrical testing so failures can be located rather than debugged simultaneously.

---

## 7. Milestones

### M0 — Project baseline
**Status: COMPLETED**
- Repository established as persistent project state.
- Prototype strategy recorded.
- AI-assisted workflow defined.

### M1 — Single-board Wi-Fi/TCP proof
**Status: NEXT / IN PROGRESS**
- Use the available ESP32-S3 board temporarily.
- Establish reliable Wi-Fi connectivity and TCP socket operation.
- Identify the NPort model and exact configuration required for interoperability.
- Do not require W5500 or a second ESP32-S3 for this preliminary network test.

**Acceptance:** ESP32-S3 is reachable over the intended test LAN and can exchange test payloads with a TCP peer in the role required by the configured NPort mode. Save build details and logs.

### M2 — RS-232 hardware bring-up
**Status: BLOCKED BY INTERFACE DETAILS**
- Identify or obtain a suitable RS-232 transceiver/module.
- Confirm wiring and serial parameters.
- Verify UART loopback and RS-232 loopback.

**Acceptance:** error-free repeatable data exchange through the physical RS-232 interface.

### M3 — SGW V0 byte-stream bridge
**Status: PLANNED**
- Connect UART and TCP path both ways.
- Add bounded buffering and overflow counters.
- Implement visible disconnect/reconnect state.
- Keep the payload protocol-agnostic.

**Acceptance:** exact byte and order preservation in both directions over sustained traffic; predictable behaviour after connection loss.

### M4 — NPort interoperability
**Status: PLANNED**
- Run NPort ↔ SGW tests using the actual device configuration.
- Test in whichever TCP Client/Server topology is supported and practical.
- Record any serial data-packing, inactivity or framing settings that affect behaviour.

**Acceptance:** successful repeatable byte transfer via NPort and SGW with no assumption of compatibility beyond what has been tested.

### M5 — Two-node SGW over wired Ethernet
**Status: PLANNED / WAITING FOR ORDERED HARDWARE**
- Use the two ESP32-S3 boards and W5500 modules.
- Replace temporary Wi-Fi network with W5500-based wired Ethernet where appropriate.
- Preserve the byte-stream API and bridge logic where possible.

**Acceptance:** two SGW nodes pass bidirectional byte-stream tests over wired Ethernet.

### M6 — High-latency and recovery tests
**Status: PLANNED**
- Apply controlled latency, initially progressing toward RTT ≈ 2200 ms.
- Test disconnects, reconnection and sustained transfer.
- Measure throughput, latency and buffering limits.

**Acceptance:** metrics and failure/recovery behaviour are documented. This stage tests the transparent baseline; it does not yet claim satellite optimisation.

### M7 — SGW-specific transport optimisation
**Status: FUTURE**
Only after M3–M6 results, investigate improvements beyond a transparent bridge. Any optimisation must be justified by measurements. Modbus-specific features remain a later decision.

---

## 8. Requirements for the transparent byte bridge

- Payload bytes must not be interpreted or modified.
- Byte order must be preserved.
- Both directions must work.
- Each serial channel must be isolated from other channels.
- Serial RX/TX buffers must be bounded and monitored.
- Overflow must be counted and visible; silent data loss is unacceptable.
- TCP state and disconnects must be observable.
- Reconnection must not silently splice unrelated old-session bytes into a new session without a documented policy.
- Diagnostics should include firmware version, reset reason, IP/link state, socket state, RX/TX byte counters, overflow counters and connection-loss count.
- The bridge must not assume TCP packet boundaries correspond to serial messages.
- Do not introduce payload-level CRC or Modbus-aware behaviour into V0.

---

## 9. Open questions

- Exact ESP32-S3 board/module SKU and memory configuration.
- Exact Moxa model used for the first bench experiment.
- Current NPort operating mode and serial/network parameters.
- Availability of an RS-232 transceiver/module and suitable cable.
- Whether the NPort and ESP32 can be placed on the same local IP network for the first test.
- Which firmware framework will be used. ESP-IDF is the current candidate; the choice should be made based on the user's installed environment and simplest repeatable workflow.
- Exact W5500 model/module wiring, to be checked when delivered.
- Final transport design for a satellite link remains open.

---

## 10. Development rule: reuse before invention

Before implementing a substantial subsystem, check existing drivers, examples, protocol implementations and reference designs. Reuse is preferred when compatible with requirements and maintainability. Do not build an elaborate protocol before the basic bridge has been measured.

---

## 11. AI-assisted development protocol

The GitHub repository is the persistent source of truth; the conversation is the engineering workspace.

- Before a work session, inspect relevant repository state and identify the current milestone.
- Update repository files after significant decisions, implementation changes and verified tests; do not copy the whole conversation into the repository.
- Distinguish **FACT**, **DECISION**, **ASSUMPTION**, **TBD** and **REJECTED**.
- Never silently turn an assumption into a fact.
- Define one next task with an objective acceptance criterion.
- Prefer a measurement or small experiment when behaviour is unknown.
- Do not mark a milestone complete until its acceptance criteria are demonstrated.
- Avoid premature implementation of Modbus, satellite optimisation or production hardware features.
- The user should not need to maintain documentation manually.

---

## 12. Current state summary

**FACT**
- One ESP32-S3 board is available now for temporary experiments.
- Two ESP32-S3 boards and two W5500 modules were previously ordered for the intended prototype.
- The target first product is a two-node, two-port-per-node transparent serial device server.
- The current project goal is protocol-independent byte transport, not Modbus processing.

**DECISION**
- Use the available ESP32-S3 and Wi-Fi for a temporary early network test.
- Keep wired Ethernet/W5500 as the intended prototype path.
- Connect RS-232 only through an appropriate RS-232 transceiver.
- Determine the exact NPort mode/configuration before attempting compatibility testing.

**ASSUMPTION**
- The available board can be connected to a local Wi-Fi network suitable for the laboratory experiment.

**TBD**
- Exact board SKU, NPort configuration, RS-232 hardware/cabling and firmware environment.

**Immediate next action**
1. Identify the exact ESP32-S3 board.
2. Identify the exact Moxa NPort model and how its serial port is configured for network communication.
3. Confirm whether an RS-232 transceiver is available.

Then start with W0/W1 and use the existing board for the Wi-Fi/TCP proof before moving to the physical serial bridge.
