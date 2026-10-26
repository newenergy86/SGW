# SGW checkpoint — W1 Wi-Fi bring-up

**Date:** 2026-10-26
**Milestone:** M1 — Single-board Wi-Fi/TCP proof
**Stage:** W1 — Wi-Fi network

## FACT

- Hardware: ESP32-S3-DevKitC-1 N16R8.
- Arduino IDE is used for the current bench firmware.
- Arduino-ESP32 Core 3.2.0 is the currently installed working version. Attempts to install newer versions were unsuccessful because the package download from GitHub terminated with `EOF`; no conclusion about a core bug was made.
- Correct board configuration was restored after an incorrect configuration was observed:
  - Board: ESP32S3 Dev Module
  - Flash Size: 16MB
  - PSRAM: OPI PSRAM
  - CPU: 240MHz (WiFi)
  - USB CDC On Boot: Enabled for the current serial-monitor setup
- Diagnostic firmware passed:
  - `BOOT TEST: PASS`
  - PSRAM detected as 8388608 bytes (8MB)
  - free heap approximately 348KB in the diagnostic run
- Wi-Fi scan passed. The ESP32 detected 6 networks, including the OpenWrt AP at approximately -25 dBm.
- A non-blocking Wi-Fi connection test was run with no `delay()` in the main loop.
- A different access point successfully accepted the ESP32 connection:
  - connection completed after approximately 6 seconds;
  - `WiFi status = 3 (WL_CONNECTED)`;
  - assigned IP: `192.168.100.3`;
  - RSSI was approximately -39 to -49 dBm during the observed run;
  - the loop continued running normally after connection.

## Observed issue

The original OpenWrt access point and a phone hotspot did not establish a connection during the observed tests. The phone hotspot showed `WiFi status = 6` followed by persistent `WiFi status = 1` (`WL_NO_SSID_AVAIL`). The OpenWrt network was visible to scanning and was reported to connect only after a long wait in an earlier test.

The successful connection to another AP demonstrates that the ESP32-S3 Wi-Fi hardware, current Arduino core and basic configuration are functional. The remaining OpenWrt/phone issue is therefore treated as an AP compatibility/configuration issue until experimentally proven otherwise.

## DECISION

- Do not spend further time on phone hotspot compatibility at this stage.
- Keep the successful AP as the controlled network for the next TCP experiment.
- Use non-blocking network logic in SGW firmware; do not wait indefinitely in `while` loops or use delays for external network events.
- Do not change Arduino core version solely because of the unsuccessful OpenWrt/phone connection.

## PASS/FAIL

**W1 Wi-Fi basic connectivity: PASS** for the controlled AP used in the successful test.

**W1 AP compatibility: PARTIAL** — OpenWrt and phone hotspot behaviour remains unresolved and is not currently a blocker for TCP development.

## Next single task

W2 — Implement a minimal TCP endpoint on the ESP32-S3 and exchange a known arbitrary byte pattern with a TCP peer over the successfully working Wi-Fi network.
