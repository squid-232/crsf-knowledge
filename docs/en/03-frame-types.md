# 📦 03 Core Frame Types in Detail

[中文](../03-frame-types.md) | **English** · [Guide home](../../README.en.md)

> **Chapter summary:** Everyday CRSF traffic consists mainly of RC channels (0x16), link statistics (0x14), and sensor telemetry (0x08 / 0x02 / 0x1E / 0x21 and others). This chapter explains their byte layouts, units, and compatibility issues, with complete byte vectors for unit tests. The source notes report that their CRCs have been checked.

## 1. Frame types at a glance

| Type | Name | Typical direction | Payload | Priority |
| --- | --- | --- | --- | --- |
| 0x16 | RC Channels Packed | Handset→TX; RX→FC | 22 bytes | ★★★ |
| 0x14 | Link Statistics | RX→FC; TX→handset | 10 bytes | ★★★ |
| 0x08 | Battery Sensor | FC→RX→handset | 8 bytes | ★★★ |
| 0x21 | Flight Mode | FC→RX→handset | String | ★★ |
| 0x1E | Attitude | FC→RX→handset | 6 bytes | ★★ |
| 0x02 | GPS | FC→RX→handset | 15 bytes | ★★ |
| 0x07 / 0x09 | Vario / barometric altitude + vertical speed | FC→RX→handset | 2 / 3 bytes | ★ |
| 0x1C / 0x1D | Link Statistics RX / TX | TX→handset | 5–7 / 6–8 bytes | ★ |
| 0x17 / 0x18 | Subset RC / 11-bit RC | — | — | Discouraged / unused |

## 2. 0x16 RC Channels Packed

### 2.1 Byte layout

- The standard channel payload is **22 bytes = 16 channels × 11 bits**, giving a 26-byte frame: `[Sync] 18 16 [22 bytes of channels] [CRC]`.
- The same frame is used for handset→TX and RX→FC.
- Packing is **LSB-first**: CH1 occupies bits 0–10, CH2 bits 11–21, and so on. Treating the 22 bytes as a little-endian integer v, zero-based channel i is `(v >> (11 * i)) & 0x7FF`.
- C bitfields in the specification illustrate the layout; their memory representation depends on compiler and endianness. GCC on little-endian targets happens to match, as used in Betaflight. **Use shifts for portable code.**

### 2.2 Channel values and pulse widths

| CRSF value | Meaning | TBS macro, C integer arithmetic | Betaflight floating-point formula |
| --- | --- | --- | --- |
| 172 | Approximately −100% travel | 988 µs | 988.5 µs |
| 992 | Center | 1500 µs | 1500.8 µs |
| 1811 | Approximately +100% travel | 2011 µs, truncated | 2012.5 µs |

- TBS macros: `TICKS_TO_US(x) = (x - 992) * 5 / 8 + 1500`; `US_TO_TICKS(x) = (x - 1500) * 8 / 5 + 992`.
- Betaflight uses `0.62477120195241 * x + 881`, based on the 172→988 and 1811→2012 endpoints. Documentation commonly gives 172 ↔ 988 µs, 992 ↔ 1500 µs, and 1811 ↔ 2012 µs; roughly 1 µs differences arise from rounding.
- Eleven bits represent 0–2047; the crsf-wg wiki lists 0–1984 as the valid range. Travel beyond ±100% can produce values outside 172–1811. Accept the representable input range and clamp as needed for the application.

### 2.3 Portable unpacking and packing

```c
/* 22 bytes -> 16 x 11-bit channels (LSB-first), independent of CPU endianness and compiler bitfields */
void crsf_unpack_channels(const uint8_t in[22], uint16_t ch[16])
{
    uint32_t bits = 0;            /* Bit accumulator                   */
    uint8_t  nbits = 0, c = 0, i; /* nbits = valid bits in the accumulator */
    for (i = 0; i < 22; i++) {
        bits |= (uint32_t)in[i] << nbits;   /* Append the new byte above the accumulated bits */
        nbits += 8;
        while (nbits >= 11 && c < 16) {
            ch[c++] = (uint16_t)(bits & 0x7FF);
            bits >>= 11;
            nbits -= 11;
        }
    }
}

/* 16 channels -> 22 bytes: inverse operation (16 x 11 = 176 bits = 22 bytes) */
void crsf_pack_channels(const uint16_t ch[16], uint8_t out[22])
{
    uint32_t bits = 0;
    uint8_t  nbits = 0, n = 0, i;
    for (i = 0; i < 16; i++) {
        bits |= (uint32_t)(ch[i] & 0x7FF) << nbits;
        nbits += 11;
        while (nbits >= 8) {
            out[n++] = (uint8_t)bits;
            bits >>= 8;
            nbits -= 8;
        }
    }
}
```

### 2.4 Test vectors

| Scenario | Complete hexadecimal frame |
| --- | --- |
| All 16 channels at 992 (1500 µs) | `C8 18 16 E0 03 1F F8 C0 07 3E F0 81 0F 7C E0 03 1F F8 C0 07 3E F0 81 0F 7C AD` |
| CH3 = CH5 = 172; all others 992 | `C8 18 16 E0 03 1F 2B C0 C7 0A F0 81 0F 7C E0 03 1F F8 C0 07 3E F0 81 0F 7C 45` |

The second vector detects reversed bit packing: if channels 3 and 5 are not the ones decoded as 172, the packing direction is wrong.

### 2.5 Failsafe

- In failsafe **cut** mode, the specification says that **0x16 frames stop**. It suggests waiting **one second** before the FC starts its own failsafe procedure.
- ExpressLRS similarly stops channel frames on CRSF output when the control link is lost. The FC uses their absence to detect signal loss.
- Record the time of the last valid 0x16 frame and check the timeout in the main loop. Whether to hold the last values or switch to safe values depends on the application; servos and ESCs usually require defined safe values.
- Do not infer failsafe solely from LQ = 0 or a channel value. Check whether **valid channel frames continue to arrive**.

### 2.6 Compatibility

- **0x17 Subset RC:** the specification discourages implementation while a revision is in progress. Avoid it in new designs. Type 0x18 is marked unused.
- **Longer 0x16 frames:** the crsf-wg wiki notes that ExpressLRS 4.0 may append an optional one-byte armStatus after the 16 channels, making Len = 0x19. Require **Len ≥ 0x18**, and ignore or decode extra bytes as appropriate.
- Betaflight processes RC frames whose first address byte is **0xC8**.

## 3. 0x14 Link Statistics

The specification defines **uplink = ground→aircraft**, the control direction, and **downlink = aircraft→ground**, the telemetry direction. Both RX→FC and TX→handset use this frame.

| Offset | Field | Type | Meaning |
| --- | --- | --- | --- |
| 0 | up_rssi_ant1 | uint8 | Uplink antenna 1 RSSI, dBm × −1; 60 means −60 dBm |
| 1 | up_rssi_ant2 | uint8 | Uplink antenna 2 RSSI, dBm × −1 |
| 2 | up_link_quality | uint8 | Uplink packet success rate, 0–100% |
| 3 | up_snr | int8 | Uplink SNR in dB; may be negative |
| 4 | active_antenna | uint8 | Best antenna: 0 = antenna 1, 1 = antenna 2 |
| 5 | rf_profile | uint8 | RF mode; TBS enumerates 4fps=0 / 50fps / 150fps; **ELRS uses its own indices**, see chapter 05 |
| 6 | up_rf_power | uint8 | Transmit-power enumeration below |
| 7 | down_rssi | uint8 | Downlink RSSI, dBm × −1 |
| 8 | down_link_quality | uint8 | Downlink LQ, % |
| 9 | down_snr | int8 | Downlink SNR, dB |

The up_rf_power enumeration is **not monotonic**: entries 7 and 8 were added later, so indices cannot be compared as power values.

| Value | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Power | 0 mW | 10 mW | 25 mW | 100 mW | 500 mW | 1000 mW | 2000 mW | 250 mW | 50 mW |

**Decoding example:** `C8 0C 14 3C 3E 64 09 00 07 03 37 64 0A DF`

| Byte | 3C | 3E | 64 | 09 | 00 | 07 | 03 | 37 | 64 | 0A |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Meaning | −60 dBm | −62 dBm | LQ 100% | SNR 9 dB | Antenna 1 | Mode 7 | 100 mW | Downlink −55 dBm | Downlink LQ 100% | Downlink SNR 10 dB |

Implementation notes:

- Betaflight uses RSSI from the **active antenna**, selecting ant1 or ant2 via active_antenna. If no 0x14 frame arrives for **250 ms**, it sets RSSI to its minimum and LQ to zero.
- RSSI measures signal strength; LQ measures packet reception success. Read them together: rapidly falling LQ with strong RSSI often indicates interference rather than range.
- rf_profile is vendor-specific. Do not hard-code the TBS 4/50/150 Hz mapping for every device.

## 4. 0x1C / 0x1D Link Statistics RX / TX

These supplemental statistics are sent by the TX module to the handset, enabling displays such as RSSI percentage and RF power in dBm.

| Frame | Reports | Fields |
| --- | --- | --- |
| 0x1C RX | Ground-side TX module's **downlink reception** | rssi_db, rssi_percent, link_quality, snr, rf_power_db; optional rssi_ant2_db and active_antenna |
| 0x1D TX | Airborne receiver's **uplink reception** | The same fields plus fps, frame rate ÷ 10; optional rssi_ant2_db and active_antenna |

The two additional antenna fields extend payloads from 5→7 bytes for 0x1C and 6→8 bytes for 0x1D. Readers must also support the older short format, illustrating why every optional read needs a length check.

## 5. Sensor telemetry: FC → RX → handset

All multi-byte fields are **big-endian**. Specification units below reproduce the source notes' reading of TBS; implementation notes describe mainstream firmware behavior.

| Type | Name | Payload layout | Units / implementation notes |
| --- | --- | --- | --- |
| 0x02 | GPS | int32 latitude, int32 longitude, uint16 ground speed, uint16 heading, uint16 altitude, uint8 satellites; 15 bytes | Coordinates: degrees × 1e7; heading: degrees × 100; altitude: meters + 1000. Specification comment says “km/h / 100”; Betaflight sends ground speed in **0.1 km/h** units. |
| 0x03 | GPS Time | Year, month, day, hour, minute, second, milliseconds | Synchronization with u-blox time pulses |
| 0x06 | GPS Extended | Fix type, NED velocity, accuracy, DOP, etc. | Additional GPS information |
| 0x07 | Vario | int16 vertical speed | cm/s |
| 0x08 | Battery | int16 voltage, int16 current, uint24 used capacity, uint8 remaining | Specification comments give 10 µV / 10 µA per LSB; Betaflight, ELRS, and EdgeTX use **0.1 V / 0.1 A**. Capacity: mAh; remaining: %. |
| 0x09 | Barometric altitude + vertical speed | uint16 altitude_packed, int8 vspeed_packed | Altitude bit15=0: decimeters with −10000 dm offset; bit15=1: meters without offset. Logarithmic vertical speed: Kl=100, Kr=0.026, range ±25 m/s. |
| 0x0A | Airspeed | uint16 | 0.1 km/h |
| 0x0B | Heartbeat | int16 origin_address | Device heartbeat |
| 0x0C | RPM | uint8 source_id + 1–19 int24 values | Negative speed indicates reverse rotation |
| 0x0D | TEMP | uint8 source_id + up to 20 int16 values | 0.1 °C; 0xEE / 0xEC identify TX / RX device temperature |
| 0x0E | Voltages | uint8 source_id + up to 29 uint16 values | mV; source_id 0–127 identifies individual cells of battery N, up to 29S; 128–255 identifies general voltage groups, such as ESC measurements |
| 0x10 | VTX Telemetry | Origin, power in dBm, frequency in MHz, pitmode bitfield | VTX status |
| 0x11 | Barometer | int32 pressure, int32 temperature | Pa; 0.01 °C |
| 0x12 | Magnetometer | Three int16 values | Three axes in milligauss |
| 0x13 | Accel / Gyro | uint32 timestamp + seven int16 values | Gyroscope scaling: INT16_MAX / 2000 dps; accelerometer scaling: INT16_MAX / 16 g |
| 0x1E | Attitude | int16 pitch, roll, yaw | 100 µrad per LSB, radians × 10000; within ±180° |
| 0x21 | Flight Mode | Null-terminated string | Betaflight sends ACRO, !FS!, RTH, PASS, and other strings |

> **Remember these two unit discrepancies**
> - **0x08 battery:** int16 at 10 µV per LSB would represent only about 0.33 V at its positive maximum. Use **0.1 V / 0.1 A** to interoperate with EdgeTX and Betaflight.
> - **0x02 ground speed:** use **0.1 km/h**. Betaflight converts cm/s using `(cm/s * 36 + 50) / 100`. Verify actual behavior when integrating third-party devices.

### 5.1 Telemetry test vectors

| Data | Complete frame |
| --- | --- |
| Battery: 16.8 V, 12.3 A, 1234 mAh, 56% | `C8 0A 08 00 A8 00 7B 00 04 D2 38 09` |
| Flight mode: "ACRO" | `C8 07 21 41 43 52 4F 00 80` |
| GPS: 31.1234567° N, 121.4567890° E, 52.3 km/h, heading 90°, altitude 15 m, 12 satellites | `C8 11 02 12 8D 10 07 48 64 D5 D2 02 0B 23 28 03 F7 0C BA` |
| Attitude: pitch 10°, roll −5°, yaw 180°; raw 1745 / −873 / 31416 | `C8 08 1E 06 D1 FC 97 7A B8 EE` |
| Vertical speed: −150 cm/s | `C8 04 07 FF 6A 34` |

### 5.2 How telemetry reaches the handset

1. The FC writes telemetry frames to the receiver UART. Betaflight sends each sensor frame at most once every 100 ms, approximately 10 Hz.
2. The receiver buffers the data and sends it to the TX module in air-link telemetry slots. The telemetry ratio limits bandwidth; see chapter 05.
3. The TX module forwards CRSF frames to the handset. In EdgeTX, use Telemetry → Discover new sensors to find RxBt, Curr, Capa, Bat%, GPS, Ptch/Roll/Yaw, FM, and others.

> **Key takeaways**
> - 0x16: 22 bytes, 16 × 11 bits, LSB-first. Values 172 / 992 / 1811 correspond approximately to 988 / 1500 / 2012 µs. Use shifts instead of bitfields.
> - Missing RC frames indicate signal loss; the specification suggests about 1 s before FC failsafe.
> - 0x14 has ten fields. RSSI is stored as dBm × −1; power indices are non-monotonic; rf_profile is vendor-specific.
> - Telemetry fields are big-endian. Use 0.1 V / 0.1 A for battery and 0.1 km/h for GPS ground speed in mainstream implementations.
> - Support added fields, such as ELRS armStatus and extended antenna statistics, while bounds-checking short frames.

📖 Sources: [TBS CRSF specification: Broadcast Frame Types](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [crsf-wg: RC_CHANNELS_PACKED](https://github.com/crsf-wg/crsf/wiki/CRSF_FRAMETYPE_RC_CHANNELS_PACKED) · [Betaflight rx/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/rx/crsf.c) · [Betaflight telemetry/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/telemetry/crsf.c)

---

[← Previous](02-physical-layer-and-frames.md) · [Contents](../../README.en.md#-knowledge-map) · [Next →](04-extended-frames.md)
