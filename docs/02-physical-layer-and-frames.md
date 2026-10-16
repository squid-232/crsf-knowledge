# 🔌 02 物理层与帧结构

**中文** | [English](en/02-physical-layer-and-frames.md) · [知识体系首页](../README.md)

> **本章摘要**：两种物理层（单线半双工 / 双线全双工）的电气参数与时序约束；帧的通用结构、Len 语义、字节序、CRC8 计算；以及规范要求的“容错读取”规则。这是写解析器之前必须吃透的一章。
## 1. 物理层
### 1.1 遥控器 ⇄ TX 模块：单线半双工 UART
- **一根信号线**分时收发，**遥控器是主机**：每个周期先发送一帧（通常是 RC 通道帧），随后把引脚置为高阻、释放总线；
- TX 模块只有在与遥控器的发送节奏**同步之后**，才在间隙里回复（链路统计、遥测、参数应答）；
- 默认 **400 kbaud、8N1**，3.3 V 电平，可正相也可反相；规范还列出 115.2 kbaud 以及 1 M / 2 Mbaud 等更高速率；
- 帧率上限约束：一个周期内必须能让遥控器和 TX 模块**各发送一个 64 字节的最大帧**。
由此可推出：单线半双工理论最大帧率 ≈ 波特率 ÷ (2 × 64 字节 × 10 bit)。
| 波特率 | 1 字节耗时 | 26 B 通道帧 | 64 B 最大帧 | 理论最大帧率 |
| --- | --- | --- | --- | --- |
| 400 000 | 25 µs | 650 µs | 1.6 ms | ≈ 312 Hz |
| 921 600 | 10.9 µs | 282 µs | 694 µs | ≈ 720 Hz |
| 1 870 000 | 5.3 µs | 139 µs | 342 µs | ≈ 1461 Hz |
> 这与 ExpressLRS 的经验一致：外置高频头在 400K 下最高只能跑 250 Hz 包率，333 Hz 及以上需要提高遥控器与模块之间的波特率（见模块 05）。
### 1.2 接收机 ⇄ 飞控：双线全双工 UART
- 接线**交叉**：RX 的 TX → 飞控 UART 的 RX，RX 的 RX → 飞控 UART 的 TX，共地；
- 规范：**416666 baud、8N1、不反相**，3.0–3.3 V；双方可协商更高速率（0x32 命令中的波特率提议，见模块 04）；
- ExpressLRS 接收机默认 **420000 baud**（与 416666 相差约 0.8%，UART 能容忍）；
- 双方可以随时发送，不存在主从关系；但飞控发得再多，也受空口遥测带宽限制（见模块 05）。
### 1.3 BST（I2C，已停产）
早期 TBS 设备用 BST（多主 I2C，100 kHz、3.3 V）承载 CRSF 帧，建议监控心跳、1.5 s 超时后复位接口。该接口已 EOL，新设计无需关注。
## 2. 帧结构
```text
广播帧  [Sync][Len][Type][Payload ...][CRC]
扩展帧  [Sync][Len][Type][Dest][Orig][Payload ...][CRC]
          1     1    1     1     1    0..58         1    (字节)
```
| 字段 | 长度 | 说明 |
| --- | --- | --- |
| Sync | 1 B | 帧起始。可能是 0xC8（串口同步字节）、0x00（广播地址）或**设备地址**；接收方应能接受其中任意一种 |
| Len | 1 B | 除 Sync 和 Len 之外的字节数 = Type + Payload + CRC（即整帧长度 − 2）；有效范围 2–62，否则丢弃 |
| Type | 1 B | 帧类型；0x28 及以上为扩展帧 |
| Dest / Orig | 各 1 B | 仅扩展帧：目的地址、源地址 |
| Payload | 广播帧 ≤ 60 B / 扩展帧 ≤ 58 B | 负载；多字节字段为**大端**（Big Endian） |
| CRC | 1 B | CRC8，覆盖 Type 到 Payload 末尾 |
- **整帧最大 64 字节**（含 Sync 与 CRC）。
- 总长 = Len + 2。例如 RC 通道帧 Len = 0x18 = 24，整帧 26 字节。
### 逐字节拆解一个 RC 通道帧
```text
C8                                   Sync（发往飞控）
18                                   Len = 24 = Type(1) + Payload(22) + CRC(1)
16                                   Type = 0x16 RC Channels Packed
E0 03 1F F8 C0 07 3E F0 81 0F 7C
E0 03 1F F8 C0 07 3E F0 81 0F 7C     Payload：16 路 x 11 bit，全部为 992（1500 us）
AD                                   CRC8(Type + Payload)
```
## 3. CRC8
- 多项式 **0xD5**（x⁸ + x⁷ + x⁶ + x⁴ + x² + 1，即 CRC-8/DVB-S2），初值 0，不反射、无最终异或；
- 覆盖范围：**从 Type 开始到 Payload 结束**，不含 Sync 与 Len；
- 规范给出了 256 字节查表实现，表可以用下面的逐位算法生成；
- 0x32 命令帧额外带一个“命令 CRC”（多项式 0xBA），见模块 04。
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
/* 校验一帧：crsf_crc8(&f[2], f[1] - 1) == f[f[1] + 1] */
```
```python
def crc8(data, poly=0xD5):
    crc = 0
    for b in data:
        crc ^= b
        for _ in range(8):
            crc = ((crc << 1) ^ poly) & 0xFF if crc & 0x80 else (crc << 1) & 0xFF
    return crc

frame = bytes.fromhex("C8 0A 08 00 A8 00 7B 00 04 D2 38 09")   # 电池帧
assert crc8(frame[2:-1]) == frame[-1]
```
## 4. 字节序与数据类型
- 帧内多字节字段一律**大端**（高字节在前），例如电池电压 `00 A8` = 168；
- 唯一的例外是 RC 通道的 **11-bit 位打包：LSB 优先**（相当于把 22 字节当成一个小端大整数），详见模块 03；
- 常见类型：int8 / uint8、int16 / uint16、uint24（如电池已用容量）、int32 / uint32，以及以 `\0` 结尾的字符串。
## 5. 容错读取规则（解析器必须遵守）
| 情况 | 规范 / 实践要求 | 实现建议 |
| --- | --- | --- |
| Len 不在 2–62 | 视为无效 | 丢弃当前 Sync，继续寻找下一个 |
| CRC 错误 | 丢弃 | 只丢 1 字节后在已收数据中重扫，避免漏掉紧随其后的真帧 |
| 帧比预期**长** | 合法：新版本可能追加字段 | 只读已知字段，忽略多余部分 |
| 帧比预期**短** | 可选字段可能缺失 | 解码前检查长度，绝不读越过 Payload 末尾 |
| 未知 Type | 合法 | 忽略；多端口设备需要转发 |
| 帧内停顿 | 一帧的字节应连续到达 | 自帧头起超过“最大帧时长 + 余量”（420k 下约 1.75 ms）判为残帧 |
> **本章要点**
> - RC⇄TX：单线半双工，遥控器为主机，默认 400k；RX⇄FC：全双工、不反相，416666（规范）/ 420000（ELRS）
> - 帧 = Sync + Len + Type +（Dest + Orig）+ Payload + CRC，整帧最多 64 字节
> - Len 计 Type 到 CRC 的字节数，总长 = Len + 2
> - CRC8：多项式 0xD5、初值 0，只算 Type + Payload
> - 字段大端，通道位打包 LSB 优先；长帧要兼容，短帧不越界
📖 出处：[TBS CRSF 规范（Hardware / Frame Construction / CRC）](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [Betaflight rx/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/rx/crsf.c) · [ExpressLRS Lua 说明（包率与波特率）](https://www.expresslrs.org/quick-start/transmitters/lua-howto/)
