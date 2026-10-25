# SGW V1 — Architecture Baseline

**Status:** WORKING BASELINE — architecture under validation
**Date:** 2026-10-25

## 1. Purpose

SGW is a specialized Modbus RTU gateway for a high-latency satellite IP link. The existing system uses Moxa NPort serial device servers. The project objective is to preserve the field-side RS-232/Modbus RTU interface while decoupling ordinary Modbus polling from the ~2200 ms satellite RTT.

The architecture is deliberately split into a compatibility path and an optimized data-concentration path.

## 2. Reference topology

Current system:

```text
SCADA / Modbus Master
        |
     RS-232
        |
  NPort 5030 (8 ports)
        |
   IP / satellite
        |
  NPort 5010
        |
     RS-232
        |
  Remote Modbus device
```

The eight channels are logically independent. The exact distribution of the approximately 80 polled devices across those channels is still TBD and must be confirmed from the real system.

Target laboratory system:

```text
             Ethernet / simulated satellite
                       |
             +---------+---------+
             |                   |
          SGW-A               SGW-B
       central side          remote side
       2 x RS-232            2 x RS-232
             |                   |
          Master              Modbus RTU
                              device/emulator
```

Both prototype units use the same hardware. Role and channel configuration are software-defined.

## 3. Architectural principle

The key optimization is NOT to make Ethernet faster. It is to reduce the number of satellite RTTs required by the Modbus application.

A transparent bridge has this timing dependency:

```text
SCADA request -> satellite -> RTU device -> satellite -> SCADA response
                           one RTT per transaction
```

The preferred SGW architecture instead makes the remote gateway an active Modbus participant:

```text
Remote side

Modbus RTU device
       ^
       | local polling
       v
Remote SGW cache
       ^
       | aggregated update
       v
Satellite transport
       ^
       v
Central SGW
       ^
       | local virtual Modbus access
       v
SCADA
```

This is consistent with established industrial gateway patterns: Moxa documents agent mode in which a gateway actively polls serial devices and serves cached data to SCADA, while SCADALink documents local polling/data concentration specifically for satellite links. These are architectural references, not implementation dependencies.

## 4. Three major software layers

### 4.1 Compatibility Layer

Purpose: replace the basic NPort serial-device-server function and provide a safe migration path.

Responsibilities:

- RS-232 interface;
- configurable UART framing;
- Modbus RTU byte/frame handling;
- transparent request/response transport;
- TCP Client/Server compatibility where required for bench testing and migration;
- per-port configuration;
- serial diagnostics.

The compatibility layer must allow the field Modbus device to remain unchanged.

### 4.2 Gateway Core

Purpose: understand Modbus transactions and maintain the application-side model.

Responsibilities:

- Modbus RTU Master on the remote side;
- Modbus RTU Server/virtual device on the central side where required;
- transaction manager;
- polling scheduler;
- register/value model;
- cache;
- freshness/quality state;
- command manager;
- independent state per logical channel.

### 4.3 SGW Transport

Purpose: carry SGW transactions efficiently and reliably over the high-latency IP path.

Responsibilities:

- framing;
- channel identification;
- transaction IDs / sequence numbers;
- batching;
- acknowledgements;
- retransmission;
- duplicate detection;
- ordering;
- connection/reconnection state;
- priority handling;
- link diagnostics.

TCP vs UDP is intentionally not frozen yet. The transport decision must be based on measured satellite behaviour and the SGW reliability requirements.

## 5. Two logical planes

The gateway core shall keep ordinary data acquisition and control commands logically separate.

### Data plane

```text
Remote RTU
   |
Local Modbus Master
   |
Poll Scheduler
   |
Cache / Register Model
   |
SGW Transport
   |
Central SGW
   |
SCADA
```

Telemetry can be delayed, aggregated and cached according to configured freshness requirements.

### Control plane

```text
SCADA
  |
Central SGW
  |
Priority Command Queue
  |
SGW Transport
  |
Remote SGW
  |
Modbus RTU WRITE
  |
Field device
  |
Execution result
  |
Central SGW
  |
SCADA
```

Control traffic must be able to bypass ordinary telemetry queues.

## 6. Per-channel isolation

Every serial/satellite channel is an independent logical context.

Each context contains at minimum:

```text
channel_id
serial configuration
connection state
RX/TX buffers
poll scheduler state
telemetry queue
command queue
sequence state
pending transactions
cache state
error counters
last RTT
```

A failure in one channel must not block or corrupt another channel.

The future eight-port central gateway is therefore a set of eight independent channel state machines sharing common Ethernet and system resources.

## 7. Remote polling model

The optimized mode requires a configured polling table. A first conceptual entry is:

```text
channel
unit_id
function
start_address
quantity
period
priority
timeout
retry
deadband (optional)
cache_object
```

The remote SGW executes the RTU polling locally. Results enter the cache with:

```text
value
source
local timestamp
quality
age/freshness
communication status
```

The central side receives changes or scheduled snapshots in aggregated SGW messages.

The exact register model is TBD until the real SCADA polling pattern is observed.

## 8. Central virtual Modbus model

The preferred optimized mode presents the central SCADA with a local logical data model instead of forcing every ordinary read to cross the satellite link.

Conceptually:

```text
SCADA Modbus request
        |
        v
Central SGW virtual register
        |
        +--> fresh cache -> immediate response
        |
        +--> stale/missing -> policy-dependent action
```

The response must carry enough status internally to distinguish fresh, stale, unavailable and communication-failed data. The exact external representation is TBD because it depends on the existing SCADA expectations and cannot be invented safely.

## 9. Control command model

A control command is not equivalent to telemetry.

Every command receives a unique immutable `COMMAND_ID`.

Conceptual lifecycle:

```text
CREATED
  |
QUEUED
  |
SENT
  |
DELIVERY_ACK
  |
EXECUTING
  |
+--------+---------+
|                  |
EXECUTED          FAILED
```

A timeout after delivery must not cause blind re-execution.

If the response is lost, the central gateway must query the remote command state/history before deciding what to report. The remote side must be able to recognize a duplicate `COMMAND_ID` and return the previous result without executing the physical command again.

`DELIVERY_ACK` and `EXECUTED` are separate states.

## 10. Compatibility / optimization modes

SGW V1 should support these development modes:

### Mode 0 — Transparent

```text
RS-232 -> SGW-A -> IP -> SGW-B -> RS-232
```

No Modbus optimization. This is the reference mode for proving electrical, UART, Ethernet and NPort interoperability.

### Mode 1 — Transaction / packetized

SGW detects complete Modbus RTU transactions, places several transactions into one SGW transport exchange where timing permits, and executes them remotely.

This is a transitional optimization mode.

### Mode 2 — Agent / local polling

Remote SGW polls the field device locally, maintains cache/register state, and central SGW serves the existing application from the synchronized model.

This is the principal performance target for cyclic telemetry.

### Mode 3 — Priority control

Control commands use a separate high-priority path and explicit execution state. This mode is mandatory for safe control operation before field deployment.

## 11. Transport architecture

The transport interface shall be abstracted from the gateway core:

```text
Gateway Core
     |
Transport API
     |
+----+----------------+
|                     |
TCP implementation   UDP/custom implementation
```

The application must not depend directly on socket details.

This allows the transport to be changed after measurements without rewriting the Modbus/cache/command logic.

Candidate transports:

- TCP;
- UDP with SGW reliability layer;
- another IP transport if justified by the satellite environment.

Decision gate: implement at least a common abstraction and benchmark the candidates under RTT ~2200 ms, packet loss, reconnect and command scenarios before freezing the protocol.

## 12. Packet architecture

The SGW packet is an internal transport object, not a Modbus frame.

Conceptual header:

```text
+---------+---------+---------+---------+---------+---------+
| MAGIC   | VERSION | TYPE    | CHANNEL | SEQ/ID  | LENGTH  |
+---------+---------+---------+---------+---------+---------+
| FLAGS   | ...     | PAYLOAD                         |
+---------+---------+---------------------------------+
| INTEGRITY / CRC / authentication (as required)      |
+------------------------------------------------------+
```

Payload may contain one or more typed records.

The exact binary format is deliberately deferred until transaction semantics and transport selection are defined.

## 13. Resource and hardware abstraction

Prototype hardware:

- ESP32-S3 development board ×2;
- W5500 Ethernet controller/module ×2;
- external RS-232 transceiver per serial port;
- two RS-232 ports per prototype;
- common Ethernet interface.

The firmware shall isolate hardware-specific code behind interfaces so that the proven gateway core can later migrate to a different MCU or custom PCB.

Conceptually:

```text
Application / Gateway Core
          |
       HAL/API
          |
+---------+---------+
|                   |
UART/DMA          Ethernet
|                   |
ESP32-S3          W5500
```

The ESP32-S3 is the prototype platform, not yet the irrevocable production MCU decision.

## 14. Configuration model

Configuration must be data-driven rather than compiled into firmware.

At minimum the model must eventually contain:

```text
Device identity
Channel configuration
Serial parameters
Network parameters
Remote endpoint
Polling table
Cache policy
Command policy
Diagnostics
```

For the first bench firmware, configuration may be intentionally minimal and compile-time or serial-console based. A persistent structured configuration model is a later requirement.

## 15. Failure model

The architecture explicitly distinguishes:

```text
LOCAL RTU FAILURE
SATELLITE/IP FAILURE
SGW-A FAILURE
SGW-B FAILURE
STALE CACHE
COMMAND UNKNOWN
COMMAND FAILED
```

Each channel has its own state machine.

On communication loss:

- remote polling may continue locally;
- cache values acquire increasing age;
- telemetry may be buffered/aggregated according to policy;
- commands are not automatically replayed;
- command state remains queryable after reconnection if persistence requirements are satisfied.

The exact safe state for each actuator remains a system-level requirement to be obtained from the real САУ.

## 16. Development sequence derived from the architecture

```text
A0  Freeze functional requirements
 |
A1  Hardware bring-up
 |
A2  UART + RS-232
 |
A3  Ethernet + W5500
 |
A4  Transparent SGW transport
 |
A5  NPort <-> SGW compatibility
 |
A6  SGW <-> SGW transaction transport
 |
A7  Artificial RTT / loss / reconnect
 |
A8  Packetized transactions
 |
A9  Remote local polling + cache
 |
A10 Central virtual Modbus model
 |
A11 Priority command state machine
 |
A12 Safety / duplicate / lost-response tests
 |
A13 Real satellite single-channel test
 |
A14 Incremental migration
 |
A15 Eight-port production architecture
```

A later phase must not be considered complete merely because the code runs. Each phase requires an observable acceptance test.

## 17. Architecture decisions still OPEN

The following are deliberately not frozen:

1. TCP vs UDP/custom reliability layer.
2. Exact SGW binary packet format.
3. Exact virtual register/address mapping.
4. Cache freshness policy.
5. Poll scheduling algorithm.
6. Command persistence requirements.
7. Exact RS-232 signalling set (TX/RX only vs hardware flow control).
8. Production MCU.
9. Production Ethernet implementation.
10. Production power/protection design.
11. Exact distribution of the ~80 devices across the eight logical satellite channels.
12. Whether optimized mode can be introduced without any SCADA configuration change, or requires a controlled mapping/configuration layer.

## 18. Architecture acceptance criteria

The architecture is considered sufficiently defined for firmware implementation when:

- the end-to-end data-plane transaction is specified;
- the end-to-end control-plane transaction is specified;
- per-channel isolation is specified;
- transparent compatibility behaviour is specified;
- packetization semantics are specified;
- failure/retry/duplicate semantics are specified;
- cache freshness semantics are specified;
- the transport abstraction is defined;
- the remaining TBD items are explicitly identified as requiring real-system data or measurement.

## 19. Current next decision gate

Before writing the optimized protocol, determine the actual information flow of the existing SCADA system.

Required observations from the real system:

1. What Modbus requests are generated during a normal scan?
2. How many requests per device?
3. Which function codes?
4. What addresses and quantities are read?
5. Are reads contiguous and therefore groupable?
6. What writes/commands are generated?
7. Does SCADA require a response to every original request?
8. What response timeout does SCADA use?
9. What happens when a device does not respond?
10. How are the eight channels represented to SCADA?

Once this is known, we can calculate the actual RTT savings and choose the minimum viable gateway model rather than over-engineering it.
