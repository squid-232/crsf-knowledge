# 📡 CRSF Protocol Learning Guide (TBS Specification × ExpressLRS Practice)

[中文](README.md) | **English**

A learning and implementation handbook for embedded developers working with the **Crossfire Serial Protocol (CRSF)**.

- **Specification:** the official TBS CRSF Protocol Specification.
- **Practice:** ExpressLRS 4.x documentation, with Betaflight source code and the crsf-wg community wiki.
- **Resources:** a portable C99 reference implementation, Python capture script, and test vectors.
- **Compiled:** October 2026.

The introduction and all eight detailed chapters are available in Chinese and English. Use the language links at the top of each page to switch. The original notes report 63 passing unit tests for crsf-mini; the tests were not rerun during publication.

## 🎯 CRSF in one sentence

CRSF is the **wired serial protocol** developed by TBS for Crossfire and widely adopted in FPV systems. It carries **16 channels with 11-bit values, bidirectional telemetry, and remote configuration** on two links: handset ⇄ TX module, and receiver ⇄ flight controller. The wireless air link uses the radio system's own protocol.

## 🗺️ System overview

```text
  Handset (RC)          TX module               Receiver (RX)          Flight Controller
  EdgeTX  [0xEA]        [0xEE]                  [0xEC]                 [0xC8]
 +-------------+  CRSF  +-------------+  RF     +-------------+  CRSF  +-------------+
 |             |------->|             |~~~~~~~~>|             |------->|             |
 |             |<-------|             |<~~~~~~~~|             |<-------|             |
 +-------------+        +-------------+         +-------------+        +-------------+
  half-duplex, 1 wire                  NOT CRSF   full-duplex UART, 8N1, not inverted
  400 kbaud default (ELRS: up to 5.25M) (air link) 416666 (TBS) / 420000 (ExpressLRS)
  --> RC channels 0x16                            --> RC channels 0x16, link stats 0x14
  <-- link stats, telemetry, params               <-- telemetry 0x08 0x02 0x1E 0x21 ...
```

| Item | Handset ⇄ TX module | Receiver ⇄ flight controller |
| --- | --- | --- |
| Physical layer | Single-wire half-duplex UART | Two-wire full-duplex UART, TX/RX crossed |
| Baud rate | 400 kbaud default; ELRS options from 115200 to 5.25 Mbaud | 416666 in the specification; 420000 by default in ELRS |
| Signal | 3.3 V; inverted or non-inverted | 3.0–3.3 V; **non-inverted** |
| Timing | The handset is the master; the module replies between handset frames after synchronizing | Either side may transmit at any time |
| Typical traffic | RC channels downstream; link statistics, telemetry, and parameter responses upstream | RX→FC: channels and link statistics; FC→RX: telemetry and MSP responses |

## 🧭 Knowledge map

All chapter links below open the English documentation.

| # | Chapter | Main topics | Difficulty | Suggested time |
| --- | --- | --- | --- | --- |
| 01 | [Protocol overview and system architecture](docs/en/01-overview.md) | The two wired links, device addresses, star routing, frame families | ⭐ | 1 hour |
| 02 | [Physical layer and frame structure](docs/en/02-physical-layer-and-frames.md) | Baud rates, polarity, half-duplex timing, frame layout, length, CRC8, tolerant parsing | ⭐⭐ | 2 hours |
| 03 | [Core frame types](docs/en/03-frame-types.md) | 11-bit channel packing, link statistics, telemetry fields and units | ⭐⭐ | 3 hours |
| 04 | [Extended frames: discovery, parameters, and commands](docs/en/04-extended-frames.md) | Ping, Device Info, parameter trees and chunks, COMMAND state machine, 0x32 commands, MSP | ⭐⭐⭐ | 4 hours |
| 05 | [CRSF in ExpressLRS](docs/en/05-expresslrs.md) | Baud and packet rates, FC configuration, switch modes, telemetry ratio and bandwidth, LQ/RSSI | ⭐⭐ | 2 hours |
| 06 | [Embedded implementation with C reference code](docs/en/06-embedded-implementation.md) | UART/DMA/IDLE, parser state machine, timeout and resynchronization, failsafe, telemetry scheduling | ⭐⭐⭐ | 6 hours |
| 07 | [Debugging, testing, and common pitfalls](docs/en/07-debugging-and-tests.md) | Capture tools and Python script, test vectors, 22 common pitfalls | ⭐⭐ | 2 hours |
| 08 | [Hands-on projects and self-assessment](docs/en/08-projects-and-quiz.md) | Five progressive projects and 15 questions with answers | ⭐⭐⭐ | 1–2 weeks |

## 🛤️ Learning path

### Stage 1 · Build the big picture (day 1)

- [ ] Read chapter 01: distinguish RC⇄TX from RX⇄FC and learn addresses 0xEA, 0xEE, 0xEC, and 0xC8.
- [ ] Read chapter 02: explain frame structure, the length field, and CRC coverage.
- [ ] Verify a CRC8 calculation using the test vectors in chapter 07.

### Stage 2 · Understand the core frames (days 2–3)

- [ ] Read chapter 03: implement packing and unpacking for 16 × 11-bit channels and understand the 172 / 992 / 1811 values and pulse-width conversion.
- [ ] Decode 0x14 link statistics: RSSI, LQ, SNR, RF mode, and transmit-power enumeration.
- [ ] Read chapter 05: connect an ELRS receiver to Betaflight/INAV, configure CRSF, and inspect channel data and telemetry.

### Stage 3 · Implement on hardware (days 4–7)

- [ ] Read chapter 06: receive and parse CRSF on STM32/ESP32 using UART DMA and IDLE detection.
- [ ] Implement failsafe detection and channel output for servos or ESCs.
- [ ] Send battery, attitude, and flight-mode telemetry from the MCU and discover the sensors in EdgeTX.
- [ ] Use chapter 07's checklist and test vectors to verify your implementation.

### Stage 4 · Advanced work (week 2 onward)

- [ ] Read chapter 04: implement Ping, Device Info, and a parameter tree so the ELRS Lua script can configure your device.
- [ ] Explore MSP over CRSF, handset-side half-duplex operation, and RC synchronization (0x3A.0x10).
- [ ] Complete chapter 08's projects and self-assessment.

## ⚡ Quick reference

```text
Basic frame     [Sync][Len][Type][Payload...][CRC]
Extended frame  [Sync][Len][Type][Dest][Orig][Payload...][CRC]
Length          Len counts bytes from Type through CRC (2..62).
                Total frame size = Len + 2; maximum 64 bytes.
Sync            Commonly 0xC8; may also be 0x00 or a device address.
CRC8            Polynomial 0xD5, initial value 0.
                Covers Type through the end of Payload; excludes Sync and Len.
0x32 commands   Additional inner CRC8, polynomial 0xBA.
                Covers 0x32, Dest, Orig, command ID, and command payload.
Byte order      Multi-byte fields are big-endian; RC channel packing is LSB-first.
RX <-> FC       416666 (TBS) / 420000 (ELRS), 8N1, non-inverted, full-duplex.
RC <-> TX       400000 default, single-wire half-duplex.
                ELRS at 400K supports packet rates up to 250 Hz.
Channels        16 x 11 bits = 22 bytes.
                172 ≈ 988 us; 992 = 1500 us; 1811 ≈ 2012 us.
                us = (ticks - 992) * 5 / 8 + 1500
                ticks = (us - 1500) * 8 / 5 + 992
Addresses       0xEA handset; 0xEE TX module; 0xEC receiver;
                0xC8 flight controller; 0x00 broadcast.
Failsafe        In cut mode, the RX stops sending 0x16 frames.
                The specification suggests approximately 1 s before FC failsafe.
```

The extended-frame rule has exceptions; see chapters 04 and 07 for types that use a short header.

| Type | Name | Typical direction | Payload essentials |
| --- | --- | --- | --- |
| 0x16 | RC Channels Packed | RC→TX, RX→FC | 22 bytes; 16 × 11 bits |
| 0x14 | Link Statistics | TX→RC, RX→FC | 10 bytes: two uplink RSSI values, LQ, SNR, antenna, RF mode, power, and downlink RSSI/LQ/SNR |
| 0x08 | Battery Sensor | FC→RX→RC | Voltage/current in practical 0.1 V / 0.1 A units; uint24 consumed mAh; remaining percentage |
| 0x02 | GPS | FC→RX→RC | Latitude/longitude in degrees × 1e7, ground speed, heading in degrees × 100, altitude in meters + 1000, satellites |
| 0x1E | Attitude | FC→RX→RC | Pitch/roll/yaw as int16, in 100 µrad units |
| 0x21 | Flight Mode | FC→RX→RC | Null-terminated string, such as ACRO or !FS! |
| 0x07 | Variometer | FC→RX→RC | Vertical speed as int16, in cm/s |
| 0x28 / 0x29 | Ping / Device Info | Bidirectional | Discovery: name, serial number, HW/FW IDs, parameter count |
| 0x2B / 0x2C / 0x2D | Parameter entry / read / write | Bidirectional | Lua configuration protocol with chunking |
| 0x32 | Direct Commands | Bidirectional | Binding, VTX, baud negotiation, and other commands; includes an inner CRC |
| 0x7A / 0x7B | MSP request / response | Bidirectional | MSP tunneling over the RC link, including Betaflight Lua and VTX Admin |

## 📖 Using this guide

- Follow chapters 01–08 in order. Each chapter starts with a summary and ends with key takeaways for review.
- For implementation work, use chapter 06's C reference code and [download the original crsf-mini source archive](assets/crsf-mini.zip). Chapter 07 provides the Python capture script and test vectors.
- For troubleshooting, start with chapter 07's checklist. Refer back to the TBS specification for field definitions.

## 🔗 Sources

- [TBS CRSF Protocol Specification](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md)
- [crsf-wg CRSF Wiki](https://github.com/crsf-wg/crsf/wiki), maintained by CapnBry and used as a reference for ExpressLRS.
- ExpressLRS documentation: [FC configuration](https://www.expresslrs.org/quick-start/receivers/configuring-fc) · [Receiver serial protocols](https://www.expresslrs.org/software/serial-protocols) · [Switch modes](https://www.expresslrs.org/software/switch-config/) · [Signal health](https://www.expresslrs.org/info/signal-health/) · [Telemetry bandwidth](https://www.expresslrs.org/info/telem-bandwidth/) · [Lua script](https://www.expresslrs.org/quick-start/transmitters/lua-howto/) · [TX preparation](https://www.expresslrs.org/quick-start/transmitters/tx-prep) · [ArduPilot setup](https://www.expresslrs.org/quick-start/ardupilot-setup)
- Source code: [Betaflight rx/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/rx/crsf.c) · [Betaflight telemetry/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/telemetry/crsf.c) · [ExpressLRS CRSFHandset.cpp](https://github.com/ExpressLRS/ExpressLRS/blob/master/src/lib/Handset/CRSFHandset.cpp)
