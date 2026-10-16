# 🧭 01 Protocol Overview and System Architecture

[中文](../01-overview.md) | **English** · [Guide home](../../README.en.md)

> **Chapter summary:** CRSF is the serial protocol developed by TBS for Crossfire and widely adopted by systems such as ExpressLRS. It defines the wired links between handset ⇄ TX module and receiver ⇄ flight controller. This chapter establishes who communicates with whom, the physical layers, device addressing, and frame families.

## 1. What problems does CRSF solve?

The Features section of the TBS specification lists five design goals:

1. Transfer RC signals with **low latency and high update rates** across the RC⇄TX and RX⇄FC links.
2. Provide **bidirectional communication**, carrying control data downstream and responses upstream.
3. Deliver **telemetry** from the aircraft to the handset: battery, GPS, attitude, link quality, and more.
4. Support **remote configuration** of both directly connected and remote devices, such as configuring a flight controller or OSD from the handset.
5. **Share the receiver serial number** with the TX for matching a receiver to a stored model (Model Match).

The public specification maintained by TBS is intended to be the single source of truth. ExpressLRS uses the crsf-wg community documentation compiled by CapnBry as an implementation reference. It is compatible with the TBS specification but has its own conventions in several areas; see chapter 05.

## 2. Where CRSF fits in the system

```text
  Handset (RC)          TX module               Receiver (RX)          Flight Controller
  EdgeTX  [0xEA]        [0xEE]                  [0xEC]                 [0xC8]
 +-------------+  CRSF  +-------------+  RF     +-------------+  CRSF  +-------------+
 |             |------->|             |~~~~~~~~>|             |------->|             |
 |             |<-------|             |<~~~~~~~~|             |<-------|             |
 +-------------+        +-------------+         +-------------+        +-------------+
   half-duplex 1-wire                  NOT CRSF   full-duplex UART
```

| Link | Physical layer | Communication | Main traffic |
| --- | --- | --- | --- |
| Handset ⇄ TX module | Single-wire half-duplex UART | Handset acts as master | Downstream RC channels; upstream link statistics, telemetry, and parameter responses |
| TX module ⇄ receiver | Wireless air link (**not CRSF**) | Bidirectional, time-division | Crossfire/ELRS air protocols transport the corresponding data |
| Receiver ⇄ flight controller | Two-wire full-duplex UART | Either side may transmit | RX→FC: channels, link statistics, parameter/MSP requests; FC→RX: telemetry and responses |

> **Key distinction:** CRSF frames are not transmitted unchanged over the air. ELRS compresses channel data into its own air format, including switch modes and Full Res, and allocates return slots using the telemetry ratio. The air packet rate and serial CRSF frame rate are separate quantities.

## 3. Device addresses

Each device has a one-byte address, used in the Dest/Orig fields of extended frames and often as the initial Sync byte.

| Address | Device |
| --- | --- |
| 0x00 | Broadcast, for example a Ping to all devices |
| 0x0E | Cloud |
| 0x10 | USB device |
| 0x12 | Bluetooth/WiFi module |
| 0x13 | WiFi receiver for gaming/simulation |
| 0x14 | Video receiver (VRX) |
| 0x20–0x7F | Dynamic addresses (NAT) |
| 0x80 | OSD / TBS PNP PRO |
| 0x90–0x97 | ESC 1–8 |
| 0xC0 | Voltage/current sensor |
| 0xC2 | GPS |
| 0xC4 | TBS Blackbox |
| **0xC8** | **Flight controller (FC)**; also the most common Sync byte |
| 0xCC | Race Tag |
| 0xCE | Video transmitter (VTX) |
| **0xEA** | **Handset / remote control** |
| 0xEB | Repeater receiver |
| **0xEC** | **Receiver (RX)** |
| 0xED | Repeater transmitter |
| **0xEE** | **TX module** |

## 4. Network topology and routing

- CRSF devices form a **star network**. A device can have multiple CRSF ports and must forward frames not addressed to itself to other ports. The specification prohibits loops, which would cause frames to circulate indefinitely.
- For example, a handset (0xEA) reading FC (0xC8) parameters sends its request through handset → TX module → air link → receiver → FC. The response travels back along the same path. This forwarding makes remote configuration of the FC, OSD, and VTX possible.
- Broadcast frames have no Dest/Orig fields. Devices consume the frame types they need; for example, an FC can process 0x16 channels and 0x14 link statistics.

## 5. Frame families

| Family | Type range | Structure | Typical uses |
| --- | --- | --- | --- |
| Broadcast frames, short header | Below 0x28 | `[Sync][Len][Type][Payload][CRC]` | Channels, link statistics, sensor telemetry |
| Extended frames, extended header | 0x28 and above | `[Sync][Len][Type][Dest][Orig][Payload][CRC]` | Discovery, parameters, commands, MSP |

For vendor-specific types such as 0xAA MAVLink encapsulation, consult the corresponding specification section for the header format. See chapters 02–04 for details.

## 6. Ecosystem and related protocols

- **TBS Crossfire / Tracer:** TBS originated CRSF; its air protocol is proprietary.
- **ExpressLRS (ELRS):** an open-source RC system using CRSF between TX and handset, and between RX and FC. ELRS receivers also support outputs such as SBUS, SUMD, and HoTT. The official documentation recommends CRSF where hardware supports it, since SBUS cannot return FC telemetry.
- **EdgeTX / OpenTX:** handset firmware with built-in CRSF drivers and a Lua configuration environment.
- **Betaflight / INAV / ArduPilot / PX4:** flight-controller firmware with native CRSF input and telemetry support.

> **Key takeaways**
> - CRSF covers two wired links: RC⇄TX, single-wire half-duplex; RX⇄FC, full-duplex UART.
> - The wireless air protocol is separate; packet rate and telemetry ratio describe the air link.
> - Core addresses: 0xEA handset, 0xEE TX module, 0xEC receiver, 0xC8 FC, 0x00 broadcast.
> - Frames use either a short header or an extended header with Dest/Orig, usually for Type ≥ 0x28.
> - Multi-port forwarding enables remote configuration of the FC, OSD, and VTX.

📖 Sources: [TBS CRSF specification: Features, Hardware, Routing, Device Addresses](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [ExpressLRS receiver serial protocols](https://www.expresslrs.org/software/serial-protocols) · [crsf-wg CRSF Wiki](https://github.com/crsf-wg/crsf/wiki)

---

[Contents](../../README.en.md#-knowledge-map) · [Next →](02-physical-layer-and-frames.md)
