# 📦 03 核心帧类型详解

> **本章摘要**：CRSF 的“日常流量”主要是三类帧：RC 通道（0x16）、链路统计（0x14）、传感器遥测（0x08 / 0x02 / 0x1E / 0x21 等）。本章逐个拆解它们的字节布局、单位换算和兼容性坑，并给出可直接用于单元测试的真实字节向量（全部已用 CRC 校验）。
## 1. 帧类型速览
| Type | 名称 | 典型方向 | Payload | 重要度 |
| --- | --- | --- | --- | --- |
| 0x16 | RC Channels Packed | 遥控器→TX；RX→FC | 22 B | ★★★ |
| 0x14 | Link Statistics | RX→FC；TX→遥控器 | 10 B | ★★★ |
| 0x08 | Battery Sensor | FC→RX→遥控器 | 8 B | ★★★ |
| 0x21 | Flight Mode | FC→RX→遥控器 | 字符串 | ★★ |
| 0x1E | Attitude | FC→RX→遥控器 | 6 B | ★★ |
| 0x02 | GPS | FC→RX→遥控器 | 15 B | ★★ |
| 0x07 / 0x09 | Vario / 气压高度 + 垂速 | FC→RX→遥控器 | 2 B / 3 B | ★ |
| 0x1C / 0x1D | Link Statistics RX / TX | TX→遥控器 | 5–7 B / 6–8 B | ★ |
| 0x17 / 0x18 | Subset RC / 11-bit RC | — | — | 不推荐 / 未使用 |
## 2. 0x16 RC Channels Packed（核心中的核心）
### 2.1 字节布局
- Payload 固定 **22 字节 = 16 路 × 11 bit**，整帧 26 字节：`[Sync] 18 16 [22 B 通道数据] [CRC]`；
- 同一种帧既用于“遥控器 → TX 模块”，也用于“RX → 飞控”；
- 打包顺序 **LSB 优先**：CH1 占位流的 bit 0–10，CH2 占 bit 11–21……等价于把 22 字节当成一个**小端大整数** v，第 i 路 = (v ≫ 11·i) & 0x7FF；
- 规范用 C 位域描述结构体，只能当“示意”：位域布局由编译器和字节序决定（GCC + 小端恰好吻合，Betaflight 就这么用），**可移植代码请用移位**。
### 2.2 数值与脉宽（µs）换算
| CRSF 值 | 含义 | TBS 宏（C 整数运算） | Betaflight 浮点公式 |
| --- | --- | --- | --- |
| 172 | 约 −100% 行程 | 988 µs | 988.5 µs |
| 992 | 中位 | 1500 µs | 1500.8 µs |
| 1811 | 约 +100% 行程 | 2011 µs（整数截断） | 2012.5 µs |
- TBS 规范宏：`TICKS_TO_US(x) = (x − 992) × 5 / 8 + 1500`，`US_TO_TICKS(x) = (x − 1500) × 8 / 5 + 992`；
- Betaflight：`0.62477120195241 × x + 881`（由 172→988、1811→2012 两点拟合）；各文档常写 172 ↔ 988 µs、992 ↔ 1500 µs、1811 ↔ 2012 µs，差 1 µs 只是取整差异；
- 11 bit 可表示 0–2047（crsf-wg Wiki 给出的有效范围是 0–1984）。遥控器行程超过 ±100% 时数值会落到 172–1811 之外，接收端应接受全范围后自行限幅。
### 2.3 可移植的解包 / 打包代码
```c
/* 22 字节 -> 16 路 11 bit（LSB 优先），与 CPU 字节序、编译器位域实现无关 */
void crsf_unpack_channels(const uint8_t in[22], uint16_t ch[16])
{
    uint32_t bits = 0;            /* 位累加器                   */
    uint8_t  nbits = 0, c = 0, i; /* nbits = 累加器里的有效位数 */
    for (i = 0; i < 22; i++) {
        bits |= (uint32_t)in[i] << nbits;   /* 新字节放在已有位的上方 */
        nbits += 8;
        while (nbits >= 11 && c < 16) {
            ch[c++] = (uint16_t)(bits & 0x7FF);
            bits >>= 11;
            nbits -= 11;
        }
    }
}

/* 16 路 -> 22 字节：逆过程（16 x 11 = 176 bit，正好 22 字节） */
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
### 2.4 测试向量（已校验 CRC）
| 场景 | 完整帧（十六进制） |
| --- | --- |
| 16 路全部 992（1500 µs） | `C8 18 16 E0 03 1F F8 C0 07 3E F0 81 0F 7C E0 03 1F F8 C0 07 3E F0 81 0F 7C AD` |
| CH3 = CH5 = 172，其余 992 | `C8 18 16 E0 03 1F 2B C0 C7 0A F0 81 0F 7C E0 03 1F F8 C0 07 3E F0 81 0F 7C 45` |
第二个向量专门用来抓“位序写反”的 bug：解出来如果不是第 3、5 路变成 172，说明打包方向错了。
### 2.5 失效保护（Failsafe）
- 规范要点：进入失控保护（failsafe 类型为 "cut"）后，**0x16 帧将不再发送**；建议飞控等待 **1 秒**后再启动自己的 failsafe 流程；
- ExpressLRS 接收机在 CRSF 输出下同样是“失联即停发通道帧”，飞控据此判定信号丢失；
- 实现建议：记录“最后一次收到有效 0x16 的时刻”，在主循环判断超时；超时后是**保持最后值**还是**切到安全值**由应用决定（驱动舵机 / 电调时通常切到安全值）；
- 不要用 LQ = 0 或某个通道值来判断失控——以“有效帧是否持续到达”为准。
### 2.6 兼容性
- **0x17 Subset RC**：规范明确警告 “discouraged for implementation. Revision is in progress”，新设计不要实现；0x18 标注为 Unused；
- **更长的 0x16**：crsf-wg Wiki 标注 ExpressLRS 4.0 可能在 16 路之后追加 1 字节可选的 armStatus，此时 Len = 0x19。解析器应只要求 **Len ≥ 0x18**，多出的字节忽略或按需解析；
- Betaflight 只处理首字节（地址）为 **0xC8** 的 RC 帧。
## 3. 0x14 Link Statistics（链路统计）
规范定义：**Uplink = 地面 → 飞机**（遥控方向），**Downlink = 飞机 → 地面**（遥测方向）。RX 发给飞控、TX 模块发给遥控器的都是这一帧。
| 偏移 | 字段 | 类型 | 含义 |
| --- | --- | --- | --- |
| 0 | up_rssi_ant1 | uint8 | 上行 RSSI 天线 1，dBm × −1（60 表示 −60 dBm） |
| 1 | up_rssi_ant2 | uint8 | 上行 RSSI 天线 2，dBm × −1 |
| 2 | up_link_quality | uint8 | 上行 LQ（收包成功率），0–100 % |
| 3 | up_snr | int8 | 上行 SNR，dB（可为负） |
| 4 | active_antenna | uint8 | 当前最佳天线：0 = 天线 1，1 = 天线 2 |
| 5 | rf_profile | uint8 | 射频模式：规范枚举 4fps=0 / 50fps / 150fps；**ELRS 有自己的编号**（见模块 05） |
| 6 | up_rf_power | uint8 | 发射功率枚举（见下表） |
| 7 | down_rssi | uint8 | 下行 RSSI，dBm × −1 |
| 8 | down_link_quality | uint8 | 下行 LQ，% |
| 9 | down_snr | int8 | 下行 SNR，dB |
up_rf_power 枚举（注意 7、8 是后来追加的，**不单调**，不能直接比较大小）：
| 值 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 功率 | 0 mW | 10 mW | 25 mW | 100 mW | 500 mW | 1000 mW | 2000 mW | 250 mW | 50 mW |
**解码示例**：`C8 0C 14 3C 3E 64 09 00 07 03 37 64 0A DF`
| 字节 | 3C | 3E | 64 | 09 | 00 | 07 | 03 | 37 | 64 | 0A |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 含义 | −60 dBm | −62 dBm | LQ 100 % | SNR 9 dB | 天线 1 | 模式 7 | 100 mW | 下行 −55 dBm | 下行 LQ 100 % | 下行 SNR 10 dB |
实现要点：
- Betaflight 取**当前活动天线**的 RSSI（active_antenna 选 ant1 或 ant2）作为 RSSI 来源；超过 **250 ms** 没收到 0x14 就把 RSSI 置为最小值、LQ 置 0；
- RSSI 是“绝对信号强度”，LQ 是“丢包率”，两者要一起看：LQ 掉得快而 RSSI 还高，往往是干扰而不是距离问题；
- rf_profile 的含义由厂商决定，不要硬编码 TBS 的 4/50/150 Hz 枚举。
## 4. 0x1C / 0x1D Link Statistics RX / TX
TX 模块发给遥控器的补充统计（遥控器侧显示 RSSI%、射频功率 dBm 等）：
| 帧 | 报告内容 | 字段 |
| --- | --- | --- |
| 0x1C RX | 地面端（TX 模块）**下行接收**情况 | rssi_db、rssi_percent、link_quality、snr、rf_power_db，可选 rssi_ant2_db、active_antenna |
| 0x1D TX | 机载端（RX）**上行接收**情况 | 同上，另有 fps（帧率 ÷ 10），可选 rssi_ant2_db、active_antenna |
规范新增的两个天线字段把 Payload 从 5→7 字节（0x1C）、6→8 字节（0x1D），并要求**读取方必须兼容旧的短格式**——这是“短帧不越界”原则的典型案例。
## 5. 遥测传感器帧（FC → RX → 遥控器）
所有多字节字段均为**大端**。“规范单位”一栏照抄 TBS 规范；“实现提示”是主流固件的实际做法。
| Type | 名称 | Payload 布局 | 规范单位 / 实现提示 |
| --- | --- | --- | --- |
| 0x02 | GPS | int32 lat、int32 lon、uint16 地速、uint16 航向、uint16 高度、uint8 卫星数（15 B） | 经纬度 ° × 1e7；航向 ° × 100；高度 = 米 + 1000；地速规范注释为 “km/h / 100”，但 Betaflight 实际按 **0.1 km/h** 发送 |
| 0x03 | GPS Time | 年月日时分秒 + 毫秒 | 用于与 u-blox 授时脉冲同步 |
| 0x06 | GPS Extended | fix 类型、NED 速度、精度、DOP 等 | 扩展 GPS 信息 |
| 0x07 | Vario | int16 垂直速度 | cm/s |
| 0x08 | Battery | int16 电压、int16 电流、uint24 已用容量、uint8 剩余 | 规范注释写 LSB = 10 µV / 10 µA，但主流实现（Betaflight、ELRS、EdgeTX）都按 **0.1 V / 0.1 A**；容量 mAh；剩余 % |
| 0x09 | 气压高度 + 垂速 | uint16 altitude_packed、int8 vspeed_packed | 高度 bit15 = 0：分米且带 −10000 dm 偏移；bit15 = 1：米、无偏移；垂速为对数编码（Kl = 100，Kr = 0.026，量程 ±25 m/s） |
| 0x0A | Airspeed | uint16 | 0.1 km/h |
| 0x0B | Heartbeat | int16 origin_address | 设备心跳 |
| 0x0C | RPM | uint8 source_id + int24 × 1–19 | 转速，负值表示反转 |
| 0x0D | TEMP | uint8 source_id + int16 × ≤20 | 0.1 °C；0xEE / 0xEC 分别为 TX / RX 设备温度 |
| 0x0E | Voltages | uint8 source_id + uint16 × ≤29 | 毫伏；source_id 0–127 为第 N 块电池的单片电压（≤29S），128–255 为一般电压组（如电调） |
| 0x10 | VTX Telemetry | origin、功率 dBm、频率 MHz、pitmode 位域 | 图传状态 |
| 0x11 | Barometer | int32 气压、int32 温度 | Pa；0.01 °C |
| 0x12 | Magnetometer | int16 × 3 | 毫高斯 × 3 |
| 0x13 | Accel / Gyro | uint32 时间戳 + int16 × 7 | 陀螺 LSB = INT16_MAX / 2000 dps；加速度 LSB = INT16_MAX / 16 g |
| 0x1E | Attitude | int16 pitch、roll、yaw | LSB = 100 µrad（弧度 × 10000），必须在 ±180° 内 |
| 0x21 | Flight Mode | 以 `\0` 结尾的字符串 | Betaflight 发送 ACRO、!FS!、RTH、PASS 等 |
> **两个单位勘误要记牢**
> - **0x08 电池**：int16 按 10 µV 计最大只能表示 0.33 V，显然不合理——按 **0.1 V / 0.1 A** 实现才能和 EdgeTX、Betaflight 对上；
> - **0x02 地速**：按 **0.1 km/h** 实现（Betaflight 用 `(cm/s × 36 + 50) / 100` 换算）；与第三方设备对接时实测确认。
### 5.1 遥测帧测试向量（已校验 CRC）
| 内容 | 完整帧 |
| --- | --- |
| 电池 16.8 V、12.3 A、1234 mAh、56 % | `C8 0A 08 00 A8 00 7B 00 04 D2 38 09` |
| 飞行模式 "ACRO" | `C8 07 21 41 43 52 4F 00 80` |
| GPS 31.1234567° N、121.4567890° E、52.3 km/h、航向 90°、高度 15 m、12 颗星 | `C8 11 02 12 8D 10 07 48 64 D5 D2 02 0B 23 28 03 F7 0C BA` |
| 姿态 pitch 10°、roll −5°、yaw 180°（1745 / −873 / 31416） | `C8 08 1E 06 D1 FC 97 7A B8 EE` |
| 垂直速度 −150 cm/s | `C8 04 07 FF 6A 34` |
### 5.2 遥测是怎么到达遥控器的
1. 飞控把遥测帧写到 RX 的 UART（Betaflight 每种传感器帧最多每 100 ms 发一次，约 10 Hz）；
2. RX 缓存后通过无线空口的遥测时隙发给 TX 模块（带宽受“遥测比”限制，见模块 05）；
3. TX 模块再以 CRSF 帧转给遥控器；EdgeTX 在“遥测 → 发现新传感器”中自动出现 RxBt、Curr、Capa、Bat%、GPS、Ptch / Roll / Yaw、FM 等传感器。
> **本章要点**
> - 0x16：22 字节、16 × 11 bit、LSB 优先；172 / 992 / 1811 ≈ 988 / 1500 / 2012 µs；用移位而不是位域
> - RC 帧停发 = 失控；规范建议飞控等 1 s 再进入 failsafe
> - 0x14：10 字段，RSSI 以 dBm × −1 存储，功率枚举不单调，rf_profile 由厂商定义
> - 遥测字段全是大端；电池按 0.1 V / 0.1 A、GPS 地速按 0.1 km/h 实现
> - 帧变长要兼容（ELRS 4.0 的 armStatus、0x1C / 0x1D 新字段），变短要防越界
📖 出处：[TBS CRSF 规范（Broadcast Frame Types）](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [crsf-wg：RC_CHANNELS_PACKED](https://github.com/crsf-wg/crsf/wiki/CRSF_FRAMETYPE_RC_CHANNELS_PACKED) · [Betaflight rx/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/rx/crsf.c) · [Betaflight telemetry/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/telemetry/crsf.c)
