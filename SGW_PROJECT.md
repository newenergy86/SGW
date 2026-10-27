# SGW — Satellite Modbus Gateway

**Project status:** Development / laboratory prototype  
**Last updated:** 2026-10-27  
**Primary source of truth:** this file

---

## 1. Project purpose

SGW is a protocol-optimized serial gateway intended eventually to replace Moxa NPort devices in a system communicating over a high-latency satellite IP channel.

The current estimated satellite round-trip time is approximately **2200 ms**. The broader system has approximately **80 polled devices**. The key architectural objective is to prevent the satellite RTT from forcing the supervisory system to wait for each individual remote Modbus transaction.

### First engineering objective

> Build two SGW nodes that make a remote Modbus RTU device appear to the supervisory server as a normal remotely accessible device, while the remote node polls the field device locally and the master node exchanges accumulated results with it in batches.

The first laboratory implementation will still use a transparent serial-byte bridge as a compatibility and hardware-validation stage. Modbus-aware operation is a later milestone, not a requirement for the first electrical/network bring-up.

---

## 2. Existing system and reference equipment

The known system uses Moxa NPort serial device servers. The central side has been described as an eight-port NPort and the remote side typically as a one-port NPort. Model numbers mentioned in project history include NPort 5250A/5030 on the central side and 5010 on the remote side; **the exact installed model and configuration for the intended bench test must be verified**.

The NPort is the behavioural reference for the compatibility stage, but the final SGW architecture is deliberately different from a simple Serial-over-TCP bridge because of the satellite latency.

Expected compatibility boundary for the initial stage:

- serial byte stream remains unchanged;
- serial parameters are configurable;
- network connection and recovery behaviour are observable;
- SGW can operate as a transparent serial device-server when required;
- mixed NPort ↔ SGW testing is performed only using a compatible, explicitly configured NPort operating mode.

NPort operating modes are configurable, not universal. Before a real interoperability test, record the exact model, port mode (e.g. TCP Client, TCP Server or Pair Connection), IP addresses/ports, and serial settings. Do not assume that any generic TCP socket automatically makes two devices interchangeable.

---

## 3. Target SGW topology

The final prototype target remains two SGW nodes. Each node is intended to have two RS-232 ports, although the first hardware prototype may use only one port.

```text
                         IP network / satellite
                    ┌────────────────────────────┐
                    │                            │
              ┌─────▼─────┐                ┌─────▼─────┐
              │   SGW-A   │◄──────────────►│   SGW-B   │
              │  MASTER   │   SGW transport │   SLAVE   │
              └─────┬─────┘                └─────┬─────┘
                    │                              │
              RS-232 ×2                       RS-232 ×2
                    │                              │
                    ▼                              ▼
              SCADA / server                  field SAU
```

### Roles

**SGW-A — MASTER**

- connected toward the supervisory server/SCADA side;
- presents the remote devices/data to the server;
- receives local requests from the server when operating in compatibility mode;
- in optimized SGW mode, receives the remote slave's accumulated data/results in batches rather than waiting for every remote Modbus transaction individually.

**SGW-B — SLAVE**

- connected to the remote SAU/field device side;
- performs the required Modbus RTU polling locally;
- maintains the latest values/results needed by the master;
- sends accumulated results to SGW-A as a batch over the inter-node transport.

The exact mapping between server-visible addresses and remote Modbus unit/register addresses remains TBD until the real SAU polling scheme is documented.

---

## 4. Core architectural principle: local polling + batched inter-node exchange

The central architectural idea of SGW is **not** to tunnel every Modbus request synchronously through the satellite link.

A simplified conventional sequence would be:

```text
SCADA
  │
  │ request #1
  ▼
SGW-A ───────────── satellite ─────────────► SGW-B
                                                │
                                             Modbus RTU
                                                │
                                                ▼
                                               SAU
                                                │
                                             response
                                                │
SGW-A ◄──────────── satellite ───────────── SGW-B
  │
  ▼
SCADA

then repeat for request #2, #3, #4 ...
```

With the intended SGW architecture, SGW-B performs the field polling locally:

```text
                         LOCAL, FAST
                    ┌─────────────────┐
                    │                 │
                    │  SGW-B SLAVE    │
                    │                 │
                    │ Modbus polling  │
                    └────────┬────────┘
                             │
                         RS-232 / RTU
                             │
                             ▼
                            SAU
```

The results are accumulated and transferred to SGW-A as one transport batch:

```text
SAU
 │
 │ local Modbus requests/responses
 ▼
SGW-B
 │
 │ collect results
 │
 │  ┌──────────────────────────────┐
 └─►│ batch:                       │
    │ value A                      │
    │ value B                      │
    │ value C                      │
    │ status / timestamp / seq.    │
    │ ...                          │
    └──────────────┬───────────────┘
                   │
             one SGW packet
                   │
             satellite / IP
                   │
                   ▼
                 SGW-A
                   │
              data model
                   │
                   ▼
                 SCADA
```

This architecture is intended to hide the satellite RTT from the normal field polling cycle. The remote device is polled locally; the satellite link carries aggregated state/results rather than requiring a round trip for every individual field request.

### Important distinction

The **transparent bridge** is the first engineering baseline and compatibility layer. The **master/slave batched architecture** is the target SGW functionality for the high-latency link.

We must not assume the optimized architecture works merely because a local prototype works. It requires its own measurable tests after the transparent transport has been validated.

---

## 5. Hardware state — distinguish available from ordered

### Available now

- **1 × ESP32-S3-DevKitC-1 N16R8** development board is available and may temporarily be borrowed from another project for SGW bench experiments.
- Board configuration: **16 MB Flash + 8 MB PSRAM**.
- Wireless interfaces available for the temporary bench: **Wi-Fi and BLE**; SGW V0 uses Wi-Fi only.

### Previously ordered for the intended prototype

- 2 × ESP32-S3 development boards;
- 2 × W5500 Ethernet modules.

These are marked **ORDERED / NOT YET VERIFIED AS DELIVERED** until the user confirms arrival.

### Alternative MCU prototype under consideration

A low-cost laboratory implementation is also being considered using:

- **STM32F103C8T6**;
- **W5500** for Ethernet;
- **MAX3232** for the RS-232 electrical interface.

This is a prototype implementation candidate, not yet the final MCU decision. The application architecture should remain portable enough that the MCU can be changed if memory, UART count, DMA or timing requirements justify it.

### Serial electrical interface

The planned RS-232 transceiver for the current ESP32-S3 bench prototype is **MAX238**. For the STM32F103C8T6 test device, **MAX3232** is the current low-cost alternative under consideration.

MCU UART signals are logic-level signals, not RS-232 line levels. A suitable RS-232 transceiver must be used between the MCU UART and the physical RS-232 connection. Do not connect MCU UART GPIO directly to an RS-232 port.

Before wiring, verify the exact transceiver variant, supply voltage, charge-pump capacitors, pinout and intended channel/wiring against its datasheet and the actual component/package being used.

### Temporary network path

For the first experiment, the available ESP32-S3 will use its built-in Wi-Fi instead of W5500. This is a temporary laboratory substitution, not a change to the target wired-Ethernet design.

The ESP32-S3 Wi-Fi path can validate IP/TCP/UDP application logic and basic interoperability. It does **not** validate W5500 hardware, wired Ethernet performance, nor the eventual satellite-link behaviour.

---

## 6. Development stages

The development is intentionally staged so that each layer can be tested independently.

### Stage A — Transparent compatibility baseline

Build a minimal Serial-over-IP bridge similar in behaviour to an NPort.

```text
RS-232 device
     │
RS-232 transceiver
     │
MCU UART
     │
opaque byte buffers
     │
IP transport
     │
NPort / SGW / test endpoint
```

This stage does not parse Modbus.

### Stage B — Two-node SGW

Build two identical nodes with configurable roles:

```text
SGW-A MASTER  ◄──── IP transport ────► SGW-B SLAVE
     │                                      │
 server/SCADA                            SAU
```

### Stage C — Local Modbus polling on SLAVE

SGW-B becomes Modbus-aware and polls the SAU locally. Polling is no longer dependent on the satellite RTT.

### Stage D — Batch transport

SGW-B packages multiple collected results into one SGW transport packet and sends them to SGW-A.

### Stage E — MASTER presentation

SGW-A maintains the server-facing representation of remote data so that the supervisory server can access it without performing a satellite round trip for every individual field value.

### Stage F — High-latency validation

Introduce controlled delay and loss, progressing toward RTT ≈ 2200 ms, and measure whether the batch architecture provides the intended improvement.

---

## 7. Temporary ESP32-S3 + Wi-Fi + Moxa experiment

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

This first test proves network-side interoperability only. It does not prove the complete serial bridge until a suitable RS-232 transceiver and UART-to-socket application are connected.

### Test stages

**W0 — Board identity and firmware toolchain**
- Record exact board/module marking and available USB/serial interfaces.
- Build and flash a minimal firmware.
- Acceptance: stable boot log, known firmware/toolchain version, and repeatable flashing.

**W1 — IP network**
- Connect ESP32-S3-DevKitC-1 N16R8 to a controlled local Wi-Fi access point.
- Log assigned IP, RSSI (as available), disconnect/reconnect events.
- Acceptance: stable IP connectivity and recovery after AP disconnection.

**W2 — TCP/UDP transport proof**
- Prove a basic IP socket exchange with a PC before involving the real SAU.
- UDP may be used as the first smoke test because it removes TCP connection-state complexity.
- Then prove TCP operation for the NPort compatibility layer.
- Acceptance: repeatable bidirectional payload exchange and visible connection/error state.

**W3 — RS-232 electrical connection**
- Verify the exact transceiver, supply and external charge-pump components before powering the interface.
- Confirm DB9 signals and whether the connected equipment expects DTE or DCE wiring, including any null-modem crossover requirement.
- Acceptance: error-free repeatable UART↔RS-232 loopback.

**W4 — Transparent byte-stream bridge**
- Couple UART and IP socket in both directions.
- Use a repeatable test pattern including all byte values where practical, long bursts and irregular pauses.
- Acceptance: exact payload bytes and order are preserved; counters match; queue overflow and disconnect behaviour are visible.

---

## 8. SGW master/slave protocol — planned

The optimized SGW transport must support batched exchange between the two nodes.

### Direction SLAVE → MASTER

SGW-B polls the SAU locally and accumulates results. A single transport packet may contain multiple logical results:

```text
┌──────────────────────────────────────────┐
│ SGW packet header                        │
│ protocol/version                         │
│ source/destination node                  │
│ sequence number                          │
│ batch ID                                 │
│ item count                               │
├──────────────────────────────────────────┤
│ result 1                                 │
│ result 2                                 │
│ result 3                                 │
│ ...                                      │
│ result N                                 │
├──────────────────────────────────────────┤
│ integrity check / CRC                    │
└──────────────────────────────────────────┘
```

The exact binary format is **TBD** and must be designed only after the transparent transport is working.

### Direction MASTER → SLAVE

The master may later send a batch of commands/requests in one transport packet. The slave executes them locally against the SAU and returns the results in a corresponding batch.

This is important for future command traffic because the design should not accidentally reintroduce one satellite RTT per Modbus request.

### Reliability requirements

Because the optimized transport is expected to operate over a high-latency and potentially lossy satellite path, the SGW protocol will need explicit mechanisms rather than relying on TCP semantics:

- sequence number;
- batch identifier;
- acknowledgement;
- retransmission policy;
- duplicate detection;
- timeout policy;
- integrity check;
- bounded queues;
- overflow reporting;
- link/session state.

The precise reliability mechanism is **TBD until latency/loss experiments provide measurements**.

### UDP vs TCP

UDP is a candidate transport for the optimized SGW node-to-node protocol because it exposes individual datagrams and allows SGW to control batching, acknowledgement and retransmission itself.

TCP remains a candidate/required mechanism for the NPort compatibility layer and ordinary Serial-over-IP operation.

**DECISION:** Do not equate the optimized SGW transport with TCP merely because the compatibility prototype uses TCP. The optimized transport may use UDP plus SGW-level reliability if experiments confirm this is appropriate.

---

## 9. Milestones

### M0 — Project baseline
**Status: COMPLETED**
- Repository established as persistent project state.
- Prototype strategy recorded.
- AI-assisted workflow defined.

### M1 — Single-board IP proof
**Status: NEXT / IN PROGRESS**
- Use the available ESP32-S3-DevKitC-1 N16R8 temporarily.
- Prove basic UDP exchange as the fastest socket smoke test.
- Prove TCP socket operation for the NPort compatibility layer.
- Identify the NPort model and exact configuration required for interoperability.

**Acceptance:** ESP32-S3 is reachable over the intended test LAN and can exchange test payloads bidirectionally with a PC; TCP interoperability is demonstrated in the role required by the configured NPort mode.

### M2 — RS-232 hardware bring-up
**Status: BLOCKED BY INTERFACE DETAILS**
- Verify MAX238 implementation for the ESP32 prototype or MAX3232 for the STM32 prototype.
- Confirm wiring and serial parameters.
- Verify UART loopback and RS-232 loopback.

**Acceptance:** error-free repeatable data exchange through the physical RS-232 interface.

### M3 — SGW V0 transparent byte-stream bridge
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
- Use two nodes and W5500 modules.
- Establish bidirectional transport between MASTER and SLAVE.
- Preserve the byte-stream API and bridge logic where possible.

**Acceptance:** two SGW nodes pass bidirectional byte-stream tests over wired Ethernet.

### M6 — Local Modbus polling
**Status: FUTURE**
- Add Modbus RTU handling to the SLAVE only.
- Implement local polling of the real SAU.
- Store the latest valid results in the SLAVE data model.

**Acceptance:** SLAVE can repeatedly poll the real SAU locally with no dependence on satellite RTT and produces verified values/statuses.

### M7 — Batched MASTER/SLAVE transport
**Status: FUTURE**
- Define and implement the first SGW packet format.
- Transfer multiple logical Modbus results in one packet.
- Add sequence, batch ID, integrity check and explicit reliability handling.
- Implement MASTER-side presentation of remote values to the supervisory server.

**Acceptance:** N independent remote values can be delivered from SLAVE to MASTER using a single inter-node batch exchange, and the values/statuses are correctly reconstructed at MASTER.

### M8 — High-latency and loss validation
**Status: FUTURE**
- Apply controlled latency, progressing toward RTT ≈ 2200 ms.
- Test packet loss, duplication, reordering and disconnect/reconnect.
- Compare sequential tunnelling against batched SGW transport.

**Acceptance:** measured latency, throughput, packet efficiency, recovery time and stale-data behaviour are documented; the batched architecture demonstrates a measurable advantage for the intended polling workload.

### M9 — Expansion
**Status: FUTURE**
- Add second RS-232 port per node.
- Evaluate scaling to more ports only after the two-port architecture is proven.
- Consider additional protocol/management functions based on measured requirements.

---

## 10. Requirements for the transparent byte bridge

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

## 11. Requirements for the optimized SGW mode

- SLAVE must be able to poll the SAU locally.
- Remote values must be represented by an explicit data model rather than an implicit TCP stream.
- Multiple logical values/results must be transferable in one inter-node packet.
- Every batch must be identifiable and verifiable.
- Duplicate batches must not corrupt the data model.
- Missing batches must be detectable.
- The MASTER must expose the latest known remote values to the supervisory side.
- Stale data must be distinguishable from fresh data.
- Commands requiring execution at the remote SAU must have an explicit lifecycle/state.
- Satellite RTT must not be assumed to be a reason to block all local polling.
- The batch size and transmission interval must be measurable and configurable.
- The protocol must remain independent of the physical Ethernet implementation so that Wi-Fi can be used during laboratory development and W5500/Ethernet later.

---

## 12. Open questions

- Exact Moxa model used for the first compatibility bench experiment.
- Current NPort operating mode and serial/network parameters.
- Exact MAX238 variant/package, supply voltage and charge-pump component values.
- Suitable RS-232 cable/DB9 wiring for the NPort.
- Whether the NPort and ESP32 can be placed on the same local IP network for the first test.
- Which firmware framework will be used for ESP32-S3. ESP-IDF is the current candidate.
- Exact W5500 module wiring, to be checked when delivered.
- Final MCU choice for the multi-port prototype remains open; STM32F103C8T6 is currently a low-cost prototype candidate.
- Exact SGW batch packet format.
- Exact acknowledgement/retransmission mechanism.
- How the MASTER will present remote values to the supervisory server.
- Exact Modbus polling map and required update rates for the real SAU.

---

## 13. Development rule: reuse before invention

Before implementing a substantial subsystem, check existing drivers, examples, protocol implementations and reference designs. Reuse is preferred when compatible with requirements and maintainability. Do not build an elaborate protocol before the basic bridge has been measured.

For the optimized SGW protocol, existing reliable transport mechanisms may be reused as implementation components, but the project must retain control over batching, transaction identity, acknowledgement, retransmission and duplicate handling where the satellite link requires it.

---

## 14. AI-assisted development protocol

The GitHub repository is the persistent source of truth; the conversation is the engineering workspace.

- Before a work session, inspect relevant repository state and identify the current milestone.
- Update repository files after significant decisions, implementation changes and verified tests; do not copy the whole conversation into the repository.
- Distinguish **FACT**, **DECISION**, **ASSUMPTION**, **TBD** and **REJECTED**.
- Never silently turn an assumption into a fact.
- Define one next task with an objective acceptance criterion.
- Prefer a measurement or small experiment when behaviour is unknown.
- Do not mark a milestone complete until its acceptance criteria are demonstrated.
- Avoid premature implementation of the optimized Modbus transport before the transparent baseline has been validated.
- The user should not need to maintain documentation manually.

---

## 15. Current state summary

**FACT**
- One **ESP32-S3-DevKitC-1 N16R8 (16 MB Flash + 8 MB PSRAM)** board is available now for temporary experiments.
- The planned RS-232 transceiver for the ESP32 bench prototype is **MAX238**.
- Two ESP32-S3 boards and two W5500 modules were previously ordered for the intended prototype.
- The intended SGW system consists of a **MASTER node and a SLAVE node**.
- The SLAVE is intended to poll the remote SAU locally.
- The MASTER is intended to serve the supervisory server.
- The optimized design is intended to transfer multiple remote values/results in batches rather than requiring one satellite exchange per individual Modbus transaction.

**DECISION**
- Use the available ESP32-S3-DevKitC-1 N16R8 and Wi-Fi for a temporary early network test.
- Keep wired Ethernet/W5500 as the intended prototype path.
- Use MAX238 as the planned ESP32 RS-232 transceiver, subject to electrical verification.
- Treat TCP as the compatibility transport and keep UDP as a candidate for the optimized SGW inter-node transport.
- Prototype hardware may use STM32F103C8T6 + W5500 + MAX3232 if this provides a faster/cheaper experimental path.
- Build and validate the transparent bridge before implementing the optimized Modbus batch protocol.

**ASSUMPTION**
- The available ESP32-S3 board can be connected to a local Wi-Fi network suitable for the laboratory experiment.

**TBD**
- Exact NPort configuration, MAX238 implementation/cabling and firmware environment.
- Final MCU and port count.
- SGW batch packet format and reliability mechanism.
- Exact remote SAU polling map and server-facing data representation.

**Immediate next action**
1. Use the available ESP32-S3 to prove basic UDP/IP exchange, then TCP compatibility behaviour.
2. Verify the RS-232 transceiver and physical wiring.
3. Build the first transparent serial-byte bridge.
4. After the bridge is proven, implement the two-node MASTER/SLAVE architecture with local polling and batched transport.

The optimized SGW mode is a planned architectural feature and is not claimed to be implemented until its dedicated milestones are experimentally demonstrated.
