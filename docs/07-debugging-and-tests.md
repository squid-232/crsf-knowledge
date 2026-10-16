# 🐞 07 调试、测试与常见坑

> **本章摘要**：CRSF 的问题大多出在"线"和"边界条件"上：接线、极性、波特率误差、Len 语义、CRC 覆盖范围、位序、单位。本章给出调试工具箱、首次上电检查顺序、可直接运行的 Python 抓包脚本、一组已校验 CRC 的测试向量，以及 22 条常见坑及解决办法。
## 1. 调试工具箱
| 工具 | 用途 | 使用要点 |
| --- | --- | --- |
| 逻辑分析仪 + PulseView（sigrok） | 看波形、按 UART 解码 | UART 解码器设 420000（或 416666 / 400000）、8N1、LSB 优先、不反相；采样率 ≥ 4 MHz，常见 24 MHz 逻辑分析仪足够 |
| USB-TTL 串口 | PC 抓包、注入测试帧 | 选支持非标准波特率的芯片（如 FT232R、CP2102N）；老款 CP2102 只支持固定波特率表，用前确认；只监听时只接 RX 和 GND |
| Betaflight Configurator → Receiver 页 | 看通道值与通道映射 | 拨动摇杆核对 AETR 顺序和 988–2012 µs 的范围 |
| Betaflight CLI | 核对串口接收配置 | `get serialrx`；CRSF 需 `serialrx_inverted = off`、`serialrx_halfduplex = off` |
| Betaflight 调试模式 | 黑匣子记录链路数据 | `debug_mode` 设为 `CRSF_LINK_STATISTICS_UPLINK` / `_PWR` / `_DOWN` |
| EdgeTX 遥测页 → 发现传感器 | 验证遥测帧 | 能看到 1RSS、RQly、RSNR、RFMD、TPWR 等链路传感器，以及你回传的 RxBt、FM 等 |
| ELRS Lua 顶部 Bad / Good | 检查遥控器 ⇄ TX 段 | 正常应为 0 / 包率（如 0/500）；Good 偏低说明波特率或同步有问题 |
| ELRS 接收机 WebUI | 改接收机串口参数 | 可修改 RX 串口波特率与输出协议 |
| 解析器统计计数 | MCU 侧自检 | `frames_ok` 稳定增长、`crc_errors` 基本不增长，才算链路干净 |
## 2. 首次上电检查顺序
1. **先确认链路已连接**：ELRS 接收机未与 TX 连接时**不输出 RC 帧**，LED 常亮才代表已连接；
2. **接线**：接收机 TX → MCU RX（交叉），共地；按接收机要求供电（通常 5 V）；
3. **看波形**：UART 空闲为**高电平**（正常极性）；若空闲为低电平，说明信号被反相了；
4. **看时序**：420k 下 1 字节约 23.8 µs，26 字节 RC 帧约 0.62 ms；帧间隔等于包率周期（500 Hz 为 2 ms）；
5. **PC 抓包**：用下面的脚本确认能解出 C8 开头的帧，`CRC errors` 不再增长；
6. **MCU 统计**：`frames_ok` 每秒增量 ≈ RC 帧率 + 链路统计帧率，`crc_errors` ≈ 0；
7. **核对通道**：拨动每个摇杆和开关，确认顺序与范围；关掉遥控器，确认约 1 s 内进入失控；
8. **遥测**：在 EdgeTX 里删除旧传感器后重新"发现传感器"，逐项核对数值与单位。
## 3. PC 抓包脚本 crsf_sniff.py
- 依赖：Python 3.8+、`pip install pyserial`；源码包 `tools/` 目录中也有这个文件；
- 用法：`python crsf_sniff.py COM5`（Windows）或 `python crsf_sniff.py /dev/ttyUSB0 420000`（Linux）；没有硬件时可运行 `python crsf_sniff.py --selftest`；
- 原理与 MCU 解析器相同：Sync 集合 \{0xC8, 0xEA, 0xEE, 0xEC, 0x00\} → Len 2–62 → CRC8（0xD5）→ 解码；出错丢 1 字节重扫；串口空闲 50 ms 时抢救被假 Sync 挡住的帧；
- 注入测试帧：`ser.write(bytes.fromhex("C8 04 28 00 EA 54"))`。
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
自测输出（噪声 + 假 Sync 中混入 8 帧，全部找回）：
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
## 4. 测试向量（均已校验 CRC）
| 帧 | 内容 | 字节 |
| --- | --- | --- |
| 0x16 RC 通道 | 16 路全部 992（1500 µs） | `C8 18 16 E0 03 1F F8 C0 07 3E F0 81 0F 7C E0 03 1F F8 C0 07 3E F0 81 0F 7C AD` |
| 0x16 RC 通道 | CH3 = CH5 = 172（油门最低、未解锁），其余 992 | `C8 18 16 E0 03 1F 2B C0 C7 0A F0 81 0F 7C E0 03 1F F8 C0 07 3E F0 81 0F 7C 45` |
| 0x14 链路统计 | RSSI −60 / −62 dBm、LQ 100、SNR 9、天线 0、RFMD 7、功率索引 3、下行 −55 dBm / 100 / 10 | `C8 0C 14 3C 3E 64 09 00 07 03 37 64 0A DF` |
| 0x08 电池 | 16.8 V、12.3 A、1234 mAh、56 % | `C8 0A 08 00 A8 00 7B 00 04 D2 38 09` |
| 0x21 飞行模式 | "ACRO" | `C8 07 21 41 43 52 4F 00 80` |
| 0x02 GPS | 31.1234567°、121.4567890°、52.3 km/h、90.00°、15 m、12 颗 | `C8 11 02 12 8D 10 07 48 64 D5 D2 02 0B 23 28 03 F7 0C BA` |
| 0x1E 姿态 | 俯仰 10°、横滚 −5°、偏航 180° | `C8 08 1E 06 D1 FC 97 7A B8 EE` |
| 0x07 垂直速度 | −150 cm/s | `C8 04 07 FF 6A 34` |
| 0x28 Ping | 遥控器广播（发往 TX 模块） | `EE 04 28 00 EA 54` |
| 0x29 Device Info | 接收机回复 "ELRS RX"、序列号 "ELRS"、12 个参数 | `EA 1A 29 EA EC 45 4C 52 53 20 52 58 00 45 4C 52 53 00 00 00 00 00 04 01 00 0C 00 9C` |
| 0x2C 读参数 | 遥控器读 TX 模块参数 1、块 0 | `EE 06 2C EE EA 01 00 86` |
| 0x32 命令 | 绑定：Dest 0xEC、Orig 0xC8、0x10.0x01（含内层 CRC） | `C8 07 32 EC C8 10 01 9E E8` |
| 0x32 命令 | 规范示例：Dest 0xEE、Orig 0x13、0x13.0x01 | `C8 07 32 EE 13 13 01 66 06` |
用法：PC 单元测试把每条向量喂给解析器，检查回调次数与解码字段；再把多条向量拼接、插入随机字节和截断帧，确认帧一条不丢、也不多出假帧。
## 5. 常见坑清单
| # | 现象 | 原因 | 解决 |
| --- | --- | --- | --- |
| 1 | 完全收不到数据 | TX / RX 没交叉，或没共地 | 接收机 TX 接 MCU RX，必须共地 |
| 2 | 全是乱码、CRC 全错 | 打开了反相（照搬 SBUS 配置） | RX ⇄ FC 段不反相；Betaflight `serialrx_inverted = off` |
| 3 | 偶发 CRC 错误 | 波特率误差大，或线长、干扰 | 核对实际波特率误差；缩短连线、远离电调与电源线 |
| 4 | 帧长度算错 | 把 Len 当成整帧长或负载长 | Len = Type + Payload + CRC；整帧 = Len + 2 |
| 5 | CRC 永远对不上 | 把 Sync / Len 算进了 CRC | CRC8 只覆盖 Type 到 Payload 末尾 |
| 6 | 通道值乱跳 | 按 MSB 优先拆 11 bit | 通道是 LSB 优先的小端位流 |
| 7 | 电池电压差 10 倍以上 | 按规范注释的 10 µV 单位处理 | 实践中按 0.1 V / 0.1 A |
| 8 | 多字节字段数值离谱 | 按小端读取 | 普通多字节字段是大端（通道位流除外） |
| 9 | 升级 ELRS 4.x 后通道不更新 | 要求 Len 严格等于 0x18 | 只要求 Len ≥ 0x18，多余字节忽略 |
| 10 | 偶发 HardFault / 越界 | 负载不足仍按固定偏移读取 | 解码前检查负载长度 |
| 11 | 一次干扰丢好几帧 | 出错时清空整个缓冲区 | 只丢 1 字节，从下一字节重新找 Sync |
| 12 | 偶尔"卡住"不出帧 | 没有帧内超时，Len 读错后一直等 | 超过约 1.75 ms 未收齐就丢弃残帧 |
| 13 | STM32 收几秒后不再收 | HAL 在 UART 错误时中止 DMA 接收 | 在 `HAL_UART_ErrorCallback()` 里重启接收 |
| 14 | 某些帧始终收不到 | Sync 过滤过严或地址判断错误 | 按角色接受 0xC8 / 0x00 / 自身地址；扩展帧看 Dest |
| 15 | 解锁失败或解锁不稳 | 解锁开关不在 AUX1（CH5） | ELRS 约定 CH5 为解锁通道，约 1000 / 2000 µs |
| 16 | 遥控器上没有遥测 | ELRS 遥测比为 Off，或 Race 模式已解锁 | 改为 Std 或固定比例；Race 只在未解锁时发遥测 |
| 17 | Betaflight Lua / VTX 调参无响应 | MSP over CRSF 依赖下行遥测带宽 | 打开遥测，适当提高遥测比 |
| 18 | 飞控 RSSI 显示不对 | RSSI 源设成了 ADC 或某个通道 | 使用 CRSF 链路统计作为 RSSI / LQ 来源 |
| 19 | RFMD 数值和包率对不上 | RFMD 是厂商、版本相关的索引 | 按当前 ELRS 版本文档查表 |
| 20 | 400K 下高包率无法工作 | 半双工带宽不够 | 333 Hz 及以上包率把波特率调高 |
| 21 | 主循环抖动大、偶发丢帧 | 回调里做重活（printf、阻塞发送） | 回调只解码、拷贝、置标志，发送放到主循环 |
| 22 | 厂商帧解析错位 | 把 Type ≥ 0x28 一律当扩展帧 | 0x34、0x80–0x82、0xAA 等使用短帧头 |
> **本章要点**
> - 先连接、再接线、再看波形、再抓包、最后看 MCU 统计——按层次排查最快
> - 逻辑分析仪 + UART 解码（420000 8N1、不反相）是最直接的工具；空闲高电平 = 正常极性
> - PC 抓包脚本与 MCU 解析器逻辑一致：Sync → Len → CRC → 解码，出错丢 1 字节
> - 测试向量既能验证解析器，也能验证编码器；拼接 + 噪声注入测试鲁棒性
> - 最常见的坑：TX / RX 交叉、反相、Len 语义、CRC 覆盖范围、位序、电池单位、Len 严格相等、HAL 错误后不重启接收
📖 出处：[TBS CRSF 规范](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [ELRS 配置飞控](https://www.expresslrs.org/quick-start/receivers/configuring-fc) · [ELRS 信号健康](https://www.expresslrs.org/info/signal-health/) · [ELRS 遥测带宽](https://www.expresslrs.org/info/telem-bandwidth/) · [ELRS Lua 说明](https://www.expresslrs.org/quick-start/transmitters/lua-howto/) · [Betaflight rx/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/rx/crsf.c)
