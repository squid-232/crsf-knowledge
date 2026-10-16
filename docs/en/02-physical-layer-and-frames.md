# 🔌 02 Physical Layer and Frame Structure

[中文](../02-physical-layer-and-frames.md) | **English** · [Guide home](../../README.en.md)

> **Chapter summary:** Electrical parameters and timing for single-wire half-duplex and two-wire full-duplex links; common frame layout, length semantics, byte order, CRC8; and tolerant parsing rules. Understand these before implementing a parser.

## 1. Physical layer

### 1.1 Handset ⇄ TX module: single-wire half-duplex UART

- **One signal wire** is shared in time. The **handset is the master**: it sends a frame, usually an RC channel frame, each cycle, then makes its pin high-impedance to release the bus.
- The TX module replies in the gaps with link statistics, telemetry, or parameter responses only after **synchronizing** with the handset's transmission cadence.
- The default is **400 kbaud, 8N1**, at 3.3 V, with either inverted or non-inverted signaling. The specification also lists 115.2 kbaud and higher rates such as 1 and 2 Mbaud.
- Each cycle must accommodate **one maximum-size 64-byte frame from each side**.

The theoretical maximum rate is therefore approximately baud rate ÷ (2 × 64 bytes × 10 bits).

| Baud rate | Time per byte | 26-byte channel frame | 64-byte maximum frame | Theoretical maximum rate |
| --- | --- | --- | --- | --- |
| 400 000 | 25 µs | 650 µs | 1.6 ms | ≈ 312 Hz |
| 921 600 | 10.9 µs | 282 µs | 694 µs | ≈ 720 Hz |
| 1 870 000 | 5.3 µs | 139 µs | 342 µs | ≈ 1461 Hz |

> This agrees with ExpressLRS practice: an external TX module at 400K supports at most 250 Hz. Rates of 333 Hz and above require a higher handset-to-module baud rate; see chapter 05.

### 1.2 Receiver ⇄ flight controller: two-wire full-duplex UART

- **Cross the connections:** receiver TX → FC UART RX; receiver RX → FC UART TX; connect ground.
- The specification gives **416666 baud, 8N1, non-inverted**, at 3.0–3.3 V. The devices can negotiate a higher rate through a 0x32 baud-rate proposal; see chapter 04.
- ExpressLRS receivers default to **420000 baud**, about 0.8% different from 416666 and within typical UART tolerance.
- Either side can transmit at any time; there is no master/slave timing. FC telemetry throughput is still constrained by the air-link telemetry bandwidth; see chapter 05.

### 1.3 BST: legacy I2C interface

Early TBS devices carried CRSF over BST, a multi-master I2C bus at 100 kHz and 3.3 V. The guidance was to monitor heartbeats and reset the interface after a 1.5 s timeout. BST is end-of-life and need not be considered for new designs.

## 2. Frame structure

```text
Broadcast [Sync][Len][Type][Payload ...][CRC]
Extended  [Sync][Len][Type][Dest][Orig][Payload ...][CRC]
          1     1    1     1     1    0..58         1    (bytes)
```

| Field | Size | Meaning |
| --- | --- | --- |
| Sync | 1 byte | Frame start: 0xC8 serial sync, 0x00 broadcast, or a device address. Receivers should accept the applicable forms. |
| Len | 1 byte | Bytes excluding Sync and Len: Type + Payload + CRC, or total size − 2. Valid range 2–62; discard invalid values. |
| Type | 1 byte | Frame type; generally 0x28 and above uses an extended header. |
| Dest / Orig | 1 byte each | Destination and origin, in extended frames only |
| Payload | ≤60 bytes broadcast / ≤58 bytes extended | Data; multi-byte fields are **big-endian** |
| CRC | 1 byte | CRC8 over Type through the last payload byte |

- **Maximum total size: 64 bytes**, including Sync and CRC.
- Total size = Len + 2. A channel frame has Len = 0x18 = 24 and is 26 bytes long.

### RC channel frame, byte by byte

```text
C8                                   Sync (to FC)
18                                   Len = 24 = Type(1) + Payload(22) + CRC(1)
16                                   Type = 0x16 RC Channels Packed
E0 03 1F F8 C0 07 3E F0 81 0F 7C
E0 03 1F F8 C0 07 3E F0 81 0F 7C     Payload: 16 x 11 bits, all 992 (1500 us)
AD                                   CRC8(Type + Payload)
```

## 3. CRC8

- Polynomial **0xD5**, representing x⁸ + x⁷ + x⁶ + x⁴ + x² + 1 (CRC-8/DVB-S2); initial value 0, no reflection, no final XOR.
- Coverage is **Type through the end of Payload**, excluding Sync and Len.
- The specification provides a 256-byte lookup table. The following bitwise algorithm can generate it.
- Command frames of type 0x32 also carry a command CRC using polynomial 0xBA; see chapter 04.

```c
uint8_t crsf_crc8(const uint8_t *p, size_t n)
{
    uint8_t crc = 0;
    while (n--) {
        crc ^= *p++;
        for (int i = 0; i < 8; i++)
            crc = (crc & 0x80) ? (uint8_t)((crc << 1) ^ 0xD5) : (uint8_t)(crc << 1);
    }
    return crc;
}
/* Validate a frame: crsf_crc8(&f[2], f[1] - 1) == f[f[1] + 1] */
```

```python
def crc8(data, poly=0xD5):
    crc = 0
    for b in data:
        crc ^= b
        for _ in range(8):
            crc = ((crc << 1) ^ poly) & 0xFF if crc & 0x80 else (crc << 1) & 0xFF
    return crc

frame = bytes.fromhex("C8 0A 08 00 A8 00 7B 00 04 D2 38 09")   # Battery frame
assert crc8(frame[2:-1]) == frame[-1]
```

## 4. Byte order and data types

- Multi-byte fields are **big-endian**, most significant byte first. For example, battery voltage bytes `00 A8` represent 168.
- RC channel **11-bit packing is LSB-first**, equivalent to treating the 22 bytes as a little-endian integer; see chapter 03.
- Common types are int8/uint8, int16/uint16, uint24 (such as consumed battery capacity), int32/uint32, and null-terminated strings.

## 5. Tolerant parsing rules

| Condition | Specification / practice | Implementation guidance |
| --- | --- | --- |
| Len outside 2–62 | Invalid | Discard the current Sync and keep searching |
| CRC failure | Discard | Drop only one byte, then rescan buffered data to avoid losing the following real frame |
| Frame **longer** than expected | Valid: later versions may append fields | Read known fields and ignore trailing data |
| Frame **shorter** than expected | Optional fields may be absent | Check lengths before decoding; never read beyond the payload |
| Unknown Type | Valid | Ignore locally; multi-port devices may need to forward it |
| Gap within a frame | Frame bytes should arrive continuously | Treat it as incomplete once time since the header exceeds maximum frame duration plus margin, about 1.75 ms at 420k |

> **Key takeaways**
> - RC⇄TX: single-wire half-duplex, handset master, 400k default. RX⇄FC: full-duplex, non-inverted, 416666 in the specification / 420000 in ELRS.
> - Frame = Sync + Len + Type + optional Dest/Orig + Payload + CRC; maximum 64 bytes.
> - Len counts Type through CRC; total size = Len + 2.
> - CRC8 uses 0xD5 and initial value 0, covering Type and the following payload/header data.
> - Fields are big-endian; channel packing is LSB-first. Accept longer frames and bounds-check shorter ones.

📖 Sources: [TBS CRSF specification: Hardware, Frame Construction, CRC](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [Betaflight rx/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/rx/crsf.c) · [ExpressLRS Lua: packet and baud rates](https://www.expresslrs.org/quick-start/transmitters/lua-howto/)

---

[← Previous](01-overview.md) · [Contents](../../README.en.md#-knowledge-map) · [Next →](03-frame-types.md)
