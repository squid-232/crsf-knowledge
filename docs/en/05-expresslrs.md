# 🚀 05 CRSF in ExpressLRS Practice

[中文](../05-expresslrs.md) | **English** · [Guide home](../../README.en.md)

> **Chapter summary:** Apply CRSF knowledge to an ExpressLRS (ELRS) system: handset⇄TX baud rates and half-duplex timing, RX⇄FC wiring and configuration, and ELRS conventions layered on top of CRSF. These include channel resolution and switch modes, LQ/RSSI channels, telemetry ratio and bandwidth, RFMD indices, and Lua menus. Connect each protocol concept to its actual handset or FC setting.

## 1. Handset ⇄ TX module

| Item | ExpressLRS practice |
| --- | --- |
| Physical layer | External modules use single-wire half-duplex; ELRS automatically uses half-duplex when TX/RX pins are identical. Some internal modules have separate TX/RX wires. |
| Baud setting | EdgeTX model settings, External RF, or System → Hardware → Baudrate. Firmware without this setting uses 400000. |
| Packet rate versus baud rate | External modules at 400K support at most 250 Hz. Rates of 333 Hz and higher, including 500 Hz and F1000, require more than 400K. |
| Baud limits | F4-based EdgeTX handsets support up to 3.75M for external modules. Newer H7 handsets can use 5.25M. If the MCU is unknown, the source guide suggests 3.75M. |
| Automatic detection | After losing handset communication, ELRS TX cycles approximately once per second through 400000, 115200, 5250000, 3750000, 1870000, 921600, and 2250000. |
| Polarity | Some external module ports are inverted. The UART_INVERTED build option covers handsets such as QX7, Tango 2, and TX16S. ESP32 platforms can detect polarity automatically. |
| Mixer synchronization | The module uses 0x3A.0x10 to synchronize EdgeTX transmission to the packet rate; see chapter 04. |
| Diagnostics | Bad / Good at the top of ELRS Lua shows bad/good CRSF frames per second received from the handset. Normal is approximately 0 / packet rate, for example 0/500. |

Why does 400K stop at 250 Hz? A half-duplex cycle must fit one 64-byte maximum frame from each side: 400000 ÷ (2 × 64 × 10) ≈ 312 Hz. A 333 Hz cycle is too short; see chapter 02.

## 2. Receiver ⇄ flight controller

### 2.1 Wiring

- **Cross the signals:** receiver TX → FC UART RX; receiver RX → the same UART's TX. Connect ground and power according to the receiver's voltage rating.
- Use a **complete, non-inverted, full-duplex UART pair** for CRSF. A single signal wire provides no return telemetry or FC-side configuration. Avoid SBUS pads with hardware inversion.
- The receiver defaults to **420000 baud**, configurable in its WebUI. Betaflight CRSF also defaults to 420000.

### 2.2 FC firmware settings

| Firmware | Required settings | Additional settings |
| --- | --- | --- |
| Betaflight | Ports: enable Serial RX on the receiver UART. Receiver: Serial mode, CRSF provider. Enable Telemetry. | Disable RSSI Channel and RSSI ADC; CRSF already supplies RSSI/LQ. |
| INAV | Similar setup; Telemetry is enabled on the Configuration page. | For protocol LQ in OSD, use CLI `set rssi_source = PROTOCOL`. |
| ArduPilot | `SERIALx_PROTOCOL = 23` (RCIN), with protocol autodetection | `RSSI_TYPE = 3` for receiver-protocol RSSI. RC_OPTIONS can enable 420 kbaud; bit 9 suppresses CRSF mode/rate messages. |

Check these Betaflight/INAV CLI options; the ELRS documentation requires both OFF:

```text
get serialrx
set serialrx_inverted = off
set serialrx_halfduplex = off
save
```

### 2.3 Receiver output protocols

ELRS Protocol options include CRSF and its inverted variant, SBUS and its inverted variant, SUMD, DJI RS2 Pro, HoTT, Tramp, and SmartAudio. Version 3.5 adds MAVLink and a second serial port, Serial2; CRSF/MAVLink require both TX and RX pins. The official recommendation is to **use CRSF when the hardware supports it**: one-way protocols such as SBUS cannot return FC telemetry to the handset.

## 3. Switch modes: fitting 16 channels into air packets

Air packets are small. ELRS switch modes determine each channel's resolution and update frequency. The FC still receives **16-channel, 11-bit 0x16 frames**; some channels simply contain lower-resolution values or update less often.

| Channel | FC function | Hybrid | Wide | Full Res 8ch | Full Res 16ch Rate/2 | Full Res 12ch Mixed |
| --- | --- | --- | --- | --- | --- | --- |
| 1–4 | Sticks | Normal Range, every packet | Normal Range, every packet | Ext Limits, every packet | Ext Limits, half rate | Ext Limits, every packet |
| 5 | AUX1 | 2-position arm, every packet | 2-position arm, every packet | Ext Limits, arm* | Ext Limits, half rate, arm* | Ext Limits, half rate, arm* |
| 6–11 | AUX2–7 | 6-position, round-robin | 64-position, round-robin | 6–8 Ext Limits; 9–11 unavailable | Ext Limits, half rate | Ext Limits, half rate |
| 12 | AUX8 | 16-position, round-robin | 64-position, round-robin | — | Ext Limits, half rate | Ext Limits, half rate |
| 13 | AUX9 | — | — | — | Ext Limits, half rate | — |
| 14 | AUX10 | 2-position arm status* | 2-position arm status* | 2-position arm status* | Ext Limits, half rate | 2-position arm status* |
| 15 | AUX11 | LQ % | LQ % | LQ % | Ext Limits, half rate | LQ % |
| 16 | AUX12 | RSSI % | RSSI % | RSSI % | Ext Limits, half rate | RSSI % |
| Available packet rates | — | 50 / 150 / 250 / 500 / D250 / D500 / F500 / F1000 | Same as Hybrid | 100Hz Full and 333Hz Full only | Same as Full Res 8ch | Same as Full Res 8ch |

For entries marked *, consult the notes for the relevant ELRS documentation version.

- **Normal Range:** 10 bits mapped to 988–2012 µs, approximately 1 µs per step. **CRSF Ext Limits:** 10 bits mapped to 885–2115 µs, approximately 1.23 µs per step.
- In 4.x Full Res modes, CH5 is proportional. In 3.x, CH5 remains a one-bit arm switch.
- **Put the arm switch on AUX1 (CH5): approximately 1000 µs disarmed, 2000 µs armed.** ELRS uses this state for safety and performance behavior, such as stopping telemetry while armed in Race mode. Remap a three-position arm switch to two positions in the EdgeTX mixer.

## 4. Link information and signal health

- The receiver sends 0x14 statistics to the FC. Modes that reserve link-information channels also provide **CH15 = LQ and CH16 = RSSI**, mapped to 988–2012 µs; see the switch-mode table.
- **RSSI**, in dBm, is stronger nearer zero, with a hard lower bound of −130 dBm. It describes signal strength. **LQ**, in percent, describes successful packet reception. LQ is critical to control-link usability; RSSI indicates distance from the sensitivity limit.
- The source guide recommends a warning threshold 5–10 dB above the sensitivity limit for the selected packet rate. For example, 2.4 GHz 250 Hz has a listed −108 dBm limit, so a warning would be −103 to −98 dBm.
- OSD LQ often includes an RF-mode prefix: `7:100` means RFMD 7 and 100% LQ.

Selected RFMD indices from the ELRS 4.x documentation in the source notes. **Indices change across versions; consult the matching version's documentation.**

| RFMD | Band | Lua label | Sensitivity limit |
| --- | --- | --- | --- |
| 0 / 1 / 2 | 900 MHz | 25Hz / 50Hz / 100Hz | −123 / −120 / −117 dBm |
| 3 | 900 MHz | 100Hz Full | −112 dBm |
| 5 / 6 / 7 | 900 MHz | 200Hz / 200Hz Full / 250Hz | −112 / −111 / −111 dBm |
| 10 / 11 | 900 MHz | D50 / K1000 Full | −112 / −101 dBm |
| 21 / 23 / 24 | 2.4 GHz | 50Hz / 100Hz Full / 150Hz | −115 / −112 / −112 dBm |
| 27 / 28 / 29 | 2.4 GHz | 250Hz / 333Hz Full / 500Hz | −108 / −105 / −105 dBm |
| 30 / 31 | 2.4 GHz | D250 / D500 | −104 / −104 dBm |

For CRSF implementers, the 0x14 rf_profile field contains this RFMD value in ELRS. **Do not decode it with TBS's 4/50/150 Hz mapping.**

## 5. Telemetry ratio and downlink bandwidth

- Telem Ratio options include Off, 1:128, 1:64, 1:32, 1:16, 1:8, 1:4, and 1:2. Since 3.0, **Std** automatically selects a ratio for the packet rate; **Race** behaves like Std but **stops telemetry and synchronization while armed**.
- Downlink packets have two classes: **LINK**, link statistics approximately every 512 ms when telemetry is active, and **DATA**, FC telemetry sharing bandwidth with MSP.
- Selected values from the official telemetry bandwidth table:

| Packet rate | Telemetry ratio | Telemetry packets/s | Bandwidth without burst | Bandwidth with burst |
| --- | --- | --- | --- | --- |
| 500 Hz | 1:2 | 250 | 5000 bps | 9922 bps |
| 500 Hz | 1:128 | 3.9 | 78 bps | 78 bps |
| 250 Hz | 1:64 | 3.9 | 78 bps | 78 bps |
| 250 Hz | 1:2 | 125 | 2500 bps | 4922 bps |
| 100 Hz | 1:8 | 12.5 | 250 bps | 417 bps |
| 50 Hz | 1:128 | 0.4 | 8 bps | 8 bps |

The FC can generate much more telemetry, with Betaflight updating each sensor at about 10 Hz, than low ratios can carry. Handset readings then refresh slowly. Prioritize important data such as GPS and battery; disable unneeded sensors using Betaflight's `telemetry_disabled_*` settings.

## 6. Lua configuration is the frontend of the CRSF parameter protocol

- ELRS Lua runs in EdgeTX. It discovers devices with 0x28/0x29, reads parameters with 0x2C/0x2B, and writes with 0x2D; see chapter 04.
- Common menu items: Packet Rate, Telem Ratio, Switch Mode, Antenna Mode, Link Mode, Model Match, TX Power, VTX Administrator, WiFi Connectivity, Backpack, and Bind.
- **Other Devices** lists additional CRSF devices, including receivers, FCs, and custom nodes, and opens their parameter menus.
- Since 3.5.0, Link Mode can be Normal, the default, or MAVLink, where the module/handset bridge a ground station to a MAVLink-capable aircraft.

## 7. Suggested settings from the ELRS source documentation

| Use case | Packet rate | Telemetry | Switch mode |
| --- | --- | --- | --- |
| Racing | 500 Hz or faster | Race | Hybrid is a suitable option |
| Freestyle / everyday flying | 250 Hz or faster | Std | Choose according to flight-mode needs |
| Fixed wing | 100Hz Full | Std or manual | Choose according to the number of proportional channels needed |

> **Do not change packet rate in flight:** it forces the TX/RX link to disconnect and reconnect.

> **Key takeaways**
> - Handset⇄TX is generally half-duplex. 400K supports up to 250 Hz; higher packet rates require more baud. ELRS TX detects baud automatically.
> - RX⇄FC uses a full UART pair, non-inverted, full-duplex, default 420000. In Betaflight select Serial + CRSF and enable Telemetry.
> - Switch modes set effective resolution and update rate. Arm on AUX1/CH5 at approximately 1000/2000 µs. Modes that reserve link channels use CH15=LQ and CH16=RSSI.
> - Evaluate LQ and RSSI together. Interpret RFMD using the corresponding ELRS documentation.
> - Telemetry bandwidth is limited: approximately 8 bps at 50 Hz, 1:128. MSP and FC telemetry share DATA bandwidth.

📖 Sources: [FC configuration](https://www.expresslrs.org/quick-start/receivers/configuring-fc) · [Switch modes](https://www.expresslrs.org/software/switch-config/) · [Signal health](https://www.expresslrs.org/info/signal-health/) · [Telemetry bandwidth](https://www.expresslrs.org/info/telem-bandwidth/) · [Lua](https://www.expresslrs.org/quick-start/transmitters/lua-howto/) · [Handset preparation](https://www.expresslrs.org/quick-start/transmitters/tx-prep) · [Receiver serial protocols](https://www.expresslrs.org/software/serial-protocols) · [ArduPilot setup](https://www.expresslrs.org/quick-start/ardupilot-setup)

---

[← Previous](04-extended-frames.md) · [Contents](../../README.en.md#-knowledge-map) · [Next →](06-embedded-implementation.md)
