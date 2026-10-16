# 🐞 07 Debugging, Testing, and Common Pitfalls

[中文](../07-debugging-and-tests.md) | **English** · [Guide home](../../README.en.md)

> **Chapter summary:** Most CRSF problems involve wiring or boundary conditions: polarity, baud error, length semantics, CRC coverage, bit order, and units. This chapter provides tools, a first-power-up checklist, a runnable Python capture script, test vectors, and 22 pitfalls with fixes.

## 1. Debugging toolbox

| Tool | Purpose | Usage notes |
| --- | --- | --- |
| Logic analyzer + PulseView/sigrok | Inspect waveforms and decode UART | Select 420000, or 416666/400000 as appropriate, 8N1, LSB-first, non-inverted. Sample at ≥4 MHz; common 24 MHz analyzers are sufficient. |
| USB-TTL adapter | PC capture and test-frame injection | Choose a chip supporting custom baud rates, such as FT232R or CP2102N. Older CP2102 chips use a fixed rate table; verify support. Passive monitoring needs RX and ground only. |
| Betaflight Configurator → Receiver | Inspect channel values and mapping | Move sticks to check AETR order and approximately 988–2012 µs travel. |
| Betaflight CLI | Verify serial receiver configuration | Use get serialrx; CRSF needs serialrx_inverted=off and serialrx_halfduplex=off. |
| Betaflight debug mode | Record link data in Blackbox | Set debug_mode to CRSF_LINK_STATISTICS_UPLINK, CRSF_LINK_STATISTICS_PWR, or CRSF_LINK_STATISTICS_DOWN. |
| EdgeTX telemetry → Discover sensors | Validate telemetry | Inspect link sensors 1RSS, RQly, RSNR, RFMD, TPWR, plus returned sensors such as RxBt and FM. |
| ELRS Lua Bad / Good | Check handset⇄TX | Normally 0 / packet rate, for example 0/500. Low Good counts suggest baud or synchronization issues. |
| ELRS receiver WebUI | Configure serial output | Change receiver baud and output protocol. |
| Parser counters | MCU-side self-check | frames_ok should grow steadily; crc_errors should stay nearly unchanged on a clean link. |

## 2. First-power-up checklist

1. **Verify the radio link first:** an ELRS receiver disconnected from its TX **does not output RC frames**. A steady receiver LED indicates a connected link.
2. **Check wiring:** receiver TX → MCU RX, common ground, and the receiver's required supply voltage, commonly 5 V.
3. **Inspect polarity:** normal UART idles **high**. An idle-low signal is inverted.
4. **Inspect timing:** at 420k, one byte takes about 23.8 µs and a 26-byte frame about 0.62 ms. A 500 Hz packet period is 2 ms.
5. **Capture on a PC:** use the script below to decode C8 frames and confirm CRC errors no longer increase.
6. **Check MCU counters:** frames_ok per second should approximately equal RC plus link-statistics frame rates, with crc_errors approximately zero.
7. **Check channels:** move every stick and switch to verify order and range. Turn the handset off and verify failsafe around the configured 1 s timeout.
8. **Check telemetry:** delete stale EdgeTX sensors, discover them again, and verify every value and unit.

## 3. PC capture script: crsf_sniff.py

- Requirements: Python 3.8+ and `pip install pyserial`. The source archive also includes the file under tools/.
- Windows: `python crsf_sniff.py COM5`. Linux: `python crsf_sniff.py /dev/ttyUSB0 420000`. Without hardware: `python crsf_sniff.py --selftest`.
- It follows the MCU parser: Sync in {0xC8, 0xEA, 0xEE, 0xEC, 0x00} → Len 2–62 → CRC8 0xD5 → decoding. Drop one byte and rescan on failure. After 50 ms of serial idle, recover complete frames blocked by a false Sync.
- To inject a test frame: `ser.write(bytes.fromhex("C8 04 28 00 EA 54"))`.

```python
#!/usr/bin/env python3
"""crsf_sniff.py - capture and decode CRSF frames with a USB-TTL adapter.

Usage:  python crsf_sniff.py COM5                  (Windows)
        python crsf_sniff.py /dev/ttyUSB0 420000   (Linux, baud optional)
        python crsf_sniff.py --selftest            (no hardware needed)
Wiring: receiver TX -> USB-TTL RX, common GND. Listen only, leave USB-TTL TX open.
Needs:  pip install pyserial   (Python 3.8+)
"""
import random
import sys
import time

SYNC = {0xC8, 0xEA, 0xEE, 0xEC, 0x00}


def crc8(data, poly=0xD5):
    crc = 0
    for b in data:
        crc ^= b
        for _ in range(8):
            crc = ((crc << 1) ^ poly) & 0xFF if crc & 0x80 else (crc << 1) & 0xFF
    return crc


class Parser:
    """Sync -> Len (2..62) -> CRC; on error drop one byte and rescan."""

    def __init__(self):
        self.buf = bytearray()
        self.ok = 0
        self.bad = 0

    def feed(self, data):
        """Append bytes, return the list of complete frames with a valid CRC."""
        self.buf += data
        frames = []
        while len(self.buf) >= 2:
            if self.buf[0] not in SYNC or not 2 <= self.buf[1] <= 62:
                del self.buf[0]                    # not a frame start
                continue
            size = self.buf[1] + 2
            if len(self.buf) < size:
                break                              # wait for more bytes
            frame = bytes(self.buf[:size])
            if crc8(frame[2:-1]) == frame[-1]:
                self.ok += 1
                del self.buf[:size]
                frames.append(frame)
            else:
                self.bad += 1                      # also counts false sync bytes
                del self.buf[0]                    # resync from the next byte
        return frames

    def flush(self):
        """Line idle: nothing can still be incomplete -> drop stale header, rescan.
        Recovers real frames hidden behind a false sync (like the MCU timeout)."""
        frames = []
        while self.buf:
            del self.buf[0]
            frames += self.feed(b"")
        return frames


def u(p, a, n):
    return int.from_bytes(p[a:a + n], "big")


def s(p, a, n):
    return int.from_bytes(p[a:a + n], "big", signed=True)


def describe(frame):
    t, p = frame[2], frame[3:-1]
    if t == 0x16 and len(p) >= 22:
        v = int.from_bytes(p[:22], "little")       # 11-bit channels, LSB first
        us = [round(1500 + (((v >> (11 * i)) & 0x7FF) - 992) * 0.625) for i in range(16)]
        return "RC   " + " ".join(f"{x:4d}" for x in us[:8]) + "  (CH1-8, us)"
    if t == 0x14 and len(p) >= 10:
        return (f"LINK RSSI -{p[0]}/-{p[1]} dBm  LQ {p[2]}%  SNR {s(p, 3, 1)} dB  ANT {p[4]}  "
                f"RFMD {p[5]}  PWR {p[6]}  | TLM RSSI -{p[7]} dBm  LQ {p[8]}%  SNR {s(p, 9, 1)} dB")
    if t == 0x08 and len(p) >= 8:
        return f"BATT {u(p, 0, 2) / 10:.1f} V  {u(p, 2, 2) / 10:.1f} A  {u(p, 4, 3)} mAh  {p[7]}%"
    if t == 0x02 and len(p) >= 15:
        return (f"GPS  {s(p, 0, 4) / 1e7:.7f}, {s(p, 4, 4) / 1e7:.7f}  {u(p, 8, 2) / 10:.1f} km/h  "
                f"{u(p, 10, 2) / 100:.2f} deg  {u(p, 12, 2) - 1000} m  {p[14]} sats")
    if t == 0x1E and len(p) >= 6:
        deg = [s(p, i, 2) / 174.533 for i in (0, 2, 4)]
        return "ATT  pitch {:.1f}  roll {:.1f}  yaw {:.1f} deg".format(*deg)
    if t == 0x07 and len(p) >= 2:
        return f"VARIO {s(p, 0, 2)} cm/s"
    if t == 0x21:
        return "MODE " + p.split(b"\0")[0].decode("ascii", "replace")
    if t >= 0x28 and len(p) >= 2:
        return f"EXT  type 0x{t:02X} dest 0x{p[0]:02X} orig 0x{p[1]:02X}  " + frame.hex(" ")
    return f"T{t:02X}  " + frame.hex(" ")


def selftest():
    vectors = [
        "C8 18 16 E0 03 1F F8 C0 07 3E F0 81 0F 7C E0 03 1F F8 C0 07 3E F0 81 0F 7C AD",
        "C8 0C 14 3C 3E 64 09 00 07 03 37 64 0A DF",
        "C8 0A 08 00 A8 00 7B 00 04 D2 38 09",
        "C8 11 02 12 8D 10 07 48 64 D5 D2 02 0B 23 28 03 F7 0C BA",
        "C8 08 1E 06 D1 FC 97 7A B8 EE",
        "C8 04 07 FF 6A 34",
        "C8 07 21 41 43 52 4F 00 80",
        "EE 04 28 00 EA 54",
    ]
    rnd = random.Random(1)
    stream = bytearray()
    for v in vectors:                              # noise + false sync before each frame
        stream += bytes(rnd.getrandbits(8) for _ in range(5)) + b"\xC8\x30" + bytes.fromhex(v)
    p = Parser()
    frames = []
    for i in range(0, len(stream), 7):             # feed in odd-sized chunks
        frames += p.feed(stream[i:i + 7])
    frames += p.flush()                            # end of stream = idle line
    for frame in frames:
        print(describe(frame))
    print(f"--- {p.ok} frames ok ({len(vectors)} expected), {p.bad} CRC errors")


def main():
    if len(sys.argv) < 2 or sys.argv[1] == "--selftest":
        return selftest()
    import serial
    baud = int(sys.argv[2]) if len(sys.argv) > 2 else 420000
    ser = serial.Serial(sys.argv[1], baud, timeout=0.02)
    p = Parser()
    try:
        capture(ser, p)
    except KeyboardInterrupt:                      # Ctrl+C
        print(f"\n=== frames ok {p.ok}, CRC errors {p.bad}")


def capture(ser, p):
    last_stat = last_rx = time.time()
    last_rc = 0.0
    while True:
        data = ser.read(256)
        if data:
            last_rx = time.time()
            frames = p.feed(data)
        elif p.buf and time.time() - last_rx > 0.05:
            frames = p.flush()                     # line idle for 50 ms
        else:
            frames = []
        for frame in frames:
            if frame[2] == 0x16:                   # RC frames arrive at 50..1000 Hz:
                if time.time() - last_rc < 0.2:    # print at most 5 per second
                    continue
                last_rc = time.time()
            print(describe(frame))
        if time.time() - last_stat >= 1.0:
            last_stat = time.time()
            print(f"--- frames ok {p.ok}, CRC errors {p.bad}")


if __name__ == "__main__":
    main()
```

Self-test output reported in the source notes: all eight frames recovered from a stream containing noise and false Sync bytes.

```text
RC   1500 1500 1500 1500 1500 1500 1500 1500  (CH1-8, us)
LINK RSSI -60/-62 dBm  LQ 100%  SNR 9 dB  ANT 0  RFMD 7  PWR 3  | TLM RSSI -55 dBm  LQ 100%  SNR 10 dB
BATT 16.8 V  12.3 A  1234 mAh  56%
GPS  31.1234567, 121.4567890  52.3 km/h  90.00 deg  15 m  12 sats
ATT  pitch 10.0  roll -5.0  yaw 180.0 deg
VARIO -150 cm/s
MODE ACRO
EXT  type 0x28 dest 0x00 orig 0xEA  ee 04 28 00 ea 54
--- 8 frames ok (8 expected), 5 CRC errors
```

## 4. Test vectors

The source notes report CRC validation for these vectors.

| Frame | Data | Bytes |
| --- | --- | --- |
| 0x16 RC channels | All 16 channels at 992, 1500 µs | `C8 18 16 E0 03 1F F8 C0 07 3E F0 81 0F 7C E0 03 1F F8 C0 07 3E F0 81 0F 7C AD` |
| 0x16 RC channels | CH3=CH5=172, minimum throttle/disarmed; others 992 | `C8 18 16 E0 03 1F 2B C0 C7 0A F0 81 0F 7C E0 03 1F F8 C0 07 3E F0 81 0F 7C 45` |
| 0x14 link statistics | RSSI −60/−62 dBm, LQ 100, SNR 9, antenna index 0, RFMD 7, power index 3; downlink −55 dBm / 100 / 10 | `C8 0C 14 3C 3E 64 09 00 07 03 37 64 0A DF` |
| 0x08 battery | 16.8 V, 12.3 A, 1234 mAh, 56% | `C8 0A 08 00 A8 00 7B 00 04 D2 38 09` |
| 0x21 flight mode | "ACRO" | `C8 07 21 41 43 52 4F 00 80` |
| 0x02 GPS | 31.1234567°, 121.4567890°, 52.3 km/h, 90.00°, 15 m, 12 satellites | `C8 11 02 12 8D 10 07 48 64 D5 D2 02 0B 23 28 03 F7 0C BA` |
| 0x1E attitude | Pitch 10°, roll −5°, yaw 180° | `C8 08 1E 06 D1 FC 97 7A B8 EE` |
| 0x07 vertical speed | −150 cm/s | `C8 04 07 FF 6A 34` |
| 0x28 Ping | Handset broadcast through TX module | `EE 04 28 00 EA 54` |
| 0x29 Device Info | Receiver "ELRS RX", serial "ELRS", 12 parameters | `EA 1A 29 EA EC 45 4C 52 53 20 52 58 00 45 4C 52 53 00 00 00 00 00 04 01 00 0C 00 9C` |
| 0x2C parameter read | Handset reads TX parameter 1, chunk 0 | `EE 06 2C EE EA 01 00 86` |
| 0x32 command | Bind: Dest 0xEC, Orig 0xC8, 0x10.0x01, including inner CRC | `C8 07 32 EC C8 10 01 9E E8` |
| 0x32 command | Specification example: Dest 0xEE, Orig 0x13, 0x13.0x01 | `C8 07 32 EE 13 13 01 66 06` |

Feed each vector to the parser in a PC unit test and check callback counts and decoded fields. Then concatenate vectors and insert random bytes and truncated frames; verify that no real frame is lost and no false frame is produced.

## 5. Common pitfalls

| # | Symptom | Cause | Fix |
| --- | --- | --- | --- |
| 1 | No data | TX/RX not crossed or no common ground | Receiver TX → MCU RX; connect ground |
| 2 | Garbage and all CRCs fail | Inversion copied from SBUS | RX⇄FC is non-inverted; serialrx_inverted=off |
| 3 | Occasional CRC errors | Baud error, long wires, or interference | Check actual baud; shorten wires and keep away from ESC/power wiring |
| 4 | Wrong frame size | Len interpreted as total size or payload size | Len counts Type through CRC; total = Len + 2 |
| 5 | CRC never matches | Sync/Len included in CRC | Cover Type through payload end only |
| 6 | Channels jump unpredictably | Unpacking 11-bit values MSB-first | Use LSB-first little-endian bit packing |
| 7 | Battery voltage off by 10× or more | Using the specification's 10 µV comment | Use practical 0.1 V / 0.1 A units |
| 8 | Nonsensical multi-byte fields | Little-endian decoding | Ordinary fields are big-endian; channel packing is the exception |
| 9 | Channels stop after ELRS 4.x upgrade | Requiring Len exactly 0x18 | Require Len ≥0x18; ignore extra data |
| 10 | HardFault or out-of-bounds access | Fixed-offset reads from a short payload | Validate length before decoding |
| 11 | One disturbance loses several frames | Clearing the entire buffer on error | Drop one byte and rescan |
| 12 | Parser occasionally stalls | No timeout after a false length | Discard incomplete frames after about 1.75 ms, with recovery scanning |
| 13 | STM32 reception stops after a few seconds | HAL aborts DMA on UART errors | Restart in HAL_UART_ErrorCallback() |
| 14 | Some frame types never arrive | Overly strict Sync or wrong addressing | Accept applicable 0xC8/0x00/local address; inspect Dest in extended frames |
| 15 | Arming fails or is unstable | Arm switch is not AUX1/CH5 | Use CH5, approximately 1000/2000 µs |
| 16 | No handset telemetry | Telem Ratio Off, or armed in Race mode | Use Std or a fixed ratio; Race sends telemetry only while disarmed |
| 17 | Betaflight Lua/VTX configuration does not respond | MSP requires downlink telemetry | Enable telemetry and increase its ratio as needed |
| 18 | FC RSSI display is wrong | RSSI source set to ADC or a channel | Use CRSF link statistics for RSSI/LQ |
| 19 | RFMD does not match expected rate | Vendor/version-specific mapping | Check the matching ELRS version's table |
| 20 | High packet rates fail at 400K | Insufficient half-duplex bandwidth | Increase baud for 333 Hz and above |
| 21 | Main-loop jitter and dropped frames | printf or blocking sends in callbacks | Decode/copy/set flags in callbacks; transmit from the main loop |
| 22 | Vendor frames decode at wrong offsets | Treating every Type ≥0x28 as extended | 0x34, 0x80–0x82, and 0xAA use short headers |

> **Key takeaways**
> - Troubleshoot by layer: radio connection → wiring → waveform → PC capture → MCU counters.
> - A logic analyzer with non-inverted UART decoding at 420000 8N1 is the most direct tool; idle high indicates normal polarity.
> - PC and MCU parsers follow Sync → Len → CRC → decode, dropping one byte on errors.
> - Vectors test encoders as well as parsers; concatenation and noise injection test robustness.
> - Frequent mistakes involve crossed wires, inversion, length/CRC semantics, bit order, battery units, exact-length checks, and failure to restart HAL reception.

📖 Sources: [TBS CRSF specification](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [ELRS FC configuration](https://www.expresslrs.org/quick-start/receivers/configuring-fc) · [Signal health](https://www.expresslrs.org/info/signal-health/) · [Telemetry bandwidth](https://www.expresslrs.org/info/telem-bandwidth/) · [Lua](https://www.expresslrs.org/quick-start/transmitters/lua-howto/) · [Betaflight rx/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/rx/crsf.c)

---

[← Previous](06-embedded-implementation.md) · [Contents](../../README.en.md#-knowledge-map) · [Next →](08-projects-and-quiz.md)
