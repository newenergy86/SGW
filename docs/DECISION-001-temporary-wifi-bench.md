# SGW — Architecture decision: temporary Wi-Fi bench

**Date:** 2026-10-25  
**Status:** APPROVED FOR LAB EXPERIMENT  
**Scope:** SGW V0 only

## Decision

Use the one currently available ESP32-S3 board as a temporary development node. Use its built-in Wi-Fi for the first network/TCP experiment instead of waiting for the ordered W5500 modules.

This is a temporary bench adaptation only. The target prototype remains two nodes based on ESP32-S3 with wired Ethernet via W5500.

## Why

- Starts useful development before the remaining hardware arrives.
- Validates firmware build/flash, Wi-Fi connectivity, TCP sockets and basic connection recovery.
- Separates network/debugging problems from later RS-232 electrical and UART problems.

## Limits

- A one-board Wi-Fi/TCP test does not prove the full serial device-server function.
- It does not validate the W5500, wired Ethernet or satellite-link performance.
- Full NPort interoperation requires the actual NPort model and operation mode, network parameters, and an application endpoint compatible with that mode.
- A physical RS-232 connection requires a suitable RS-232 transceiver; ESP32-S3 UART GPIO must not be connected directly to RS-232 signal levels.

## Test steps

1. Identify the exact ESP32-S3 board/module and selected firmware environment.
2. Build/flash a minimal firmware and verify repeatable boot logging.
3. Join a controlled local Wi-Fi network; confirm IP connectivity and recovery after an access-point disconnect.
4. Inspect the NPort configuration: model, TCP Client/Server or Pair Connection mode, destination/listen IP and port, and serial port settings.
5. Run a TCP payload test in the role required by the NPort configuration.
6. Identify the RS-232 transceiver/cable, then validate UART↔RS-232 loopback.
7. Connect socket and UART into a bidirectional byte-stream bridge; test byte content/order, sustained traffic, queue overflow counters and disconnect/reconnect.
8. Repeat over W5500 wired Ethernet when the ordered modules arrive.

## Acceptance criteria for the current first step

The ESP32-S3 joins the local Wi-Fi network, has a known IP address, logs connection state, and completes repeatable bidirectional payload exchange with a TCP peer compatible with the configured NPort mode. Save firmware version, settings summary and serial logs.

## Open facts to collect

- Board/module marking and memory variant.
- Exact Moxa model used for the bench experiment.
- NPort operation mode and IP/port configuration.
- Whether an RS-232 transceiver/module and appropriate cable are available.
- Existing development environment and its version.

## Source note

Espressif documents Wi-Fi station (STA) mode for connecting ESP32-S3 to an access point and provides socket/TCP examples for ESP32-S3. Moxa NPort 5000-series modes include TCP Client, TCP Server and Pair Connection; Pair Connection is configured as a master/slave pair. The selected test topology must match the actual NPort configuration, not assume that any generic TCP endpoint is automatically compatible.
