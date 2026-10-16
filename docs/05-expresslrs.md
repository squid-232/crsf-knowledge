# 🚀 05 ExpressLRS 中的 CRSF 实践

**中文** | [English](en/05-expresslrs.md) · [知识体系首页](../README.md)

> **本章摘要**：把协议知识落到 ExpressLRS（ELRS）的真实系统里：遥控器 ⇄ TX 模块的波特率与半双工、RX ⇄ 飞控的接线与飞控配置；以及 ELRS 在 CRSF 之上的约定——开关模式决定通道分辨率、CH15 / CH16 携带 LQ / RSSI、遥测比决定下行带宽、RFMD 编号、Lua 配置菜单。读完能把每个协议概念对应到遥控器 / 飞控里的具体设置项。
## 1. 遥控器 ⇄ TX 模块
| 项目 | ExpressLRS 实践 |
| --- | --- |
| 物理层 | 外置模块：单线半双工（ELRS 在 TX、RX 引脚为同一个时自动工作在半双工）；部分内置模块使用独立的 TX / RX 两根线 |
| 波特率设置 | EdgeTX 的模型设置（外置 RF）或 系统菜单 → 硬件 中的 Baudrate；没有该设置项的固件固定为 400000 |
| 包率与波特率 | 400K 下外置模块最高 250 Hz；333 Hz 及更快的包率（500 Hz、F1000 等）需要高于 400K 的波特率 |
| 波特率上限 | F4 主控的 EdgeTX 遥控器外置模块最高 3.75M；较新的 H7 遥控器可设 5.25M（不确定主控时建议 3.75M） |
| 自动识别 | ELRS TX 失去与遥控器的连接后约每 1 s 轮换尝试：400000、115200、5250000、3750000、1870000、921600、2250000 |
| 极性 | 部分遥控器外置口是反相信号（ELRS 构建选项 UART_INVERTED，面向 QX7、Tango 2、TX16S 等）；ESP32 平台可自动检测 |
| 混控同步 | TX 模块用 0x3A.0x10 把 EdgeTX 的发送周期同步到包率（见模块 04） |
| 诊断 | ELRS Lua 顶部的 Bad / Good 计数 = TX 模块每秒从遥控器收到的坏 / 好 CRSF 帧数，正常应为 0 / 包率（如 0/500） |
为什么 400K 最多 250 Hz？半双工下一个周期要容纳遥控器和 TX 模块各一个 64 字节最大帧：400000 ÷ (2 × 64 × 10) ≈ 312 Hz，所以 333 Hz 及以上放不下（推导见模块 02）。
## 2. 接收机 ⇄ 飞控
### 2.1 接线
- **交叉连接**：RX 的 TX → 飞控 UART 的 RX，RX 的 RX → 同一 UART 的 TX，再接 GND 和供电（电压按接收机规格）；
- CRSF 需要**完整的 UART 对**、**不反相**、**全双工**：只接一根线就没有遥测，也无法从飞控侧配置；不要接带硬件反相器的 SBUS 焊盘；
- 接收机串口默认 **420000** baud（可在接收机 WebUI 修改），Betaflight 的 CRSF 默认同为 420000。
### 2.2 飞控固件配置
| 固件 | 必做设置 | 补充 |
| --- | --- | --- |
| Betaflight | Ports：接收机所在 UART 勾选 Serial RX；Receiver：Receiver Mode = Serial，Provider = CRSF；Telemetry 打开 | RSSI Channel 设为 Disabled，RSSI ADC 关闭（RSSI / LQ 已经由 CRSF 协议提供） |
| INAV | 同上；Telemetry 开关在 Configuration 页 | OSD 想显示协议 LQ：CLI `set rssi_source = PROTOCOL` |
| ArduPilot | `SERIALx_PROTOCOL = 23`（RCIN），协议自动识别 | `RSSI_TYPE = 3`（接收机协议）；RC_OPTIONS 可启用 420 kbaud、bit 9 抑制 CRSF 模式 / 速率提示 |
Betaflight / INAV CLI 检查（ELRS 文档要求两项都为 OFF）：
```text
get serialrx
set serialrx_inverted = off
set serialrx_halfduplex = off
save
```
### 2.3 接收机输出协议
ELRS 接收机的 Protocol 选项包括 CRSF（及反相版）、SBUS（及反相版）、SUMD、DJI RS2 Pro、HoTT、Tramp、SmartAudio，3.5 起还有 MAVLink 和第二串口（Serial2，CRSF / MAVLink 需同时接 TX / RX 引脚）。官方建议：**硬件支持 CRSF 就用 CRSF**——SBUS 等单向协议无法把飞控遥测回传给遥控器。
## 3. 开关模式：16 个通道怎样“挤”进空口
空口包很小，ELRS 用“开关模式”决定各通道的分辨率和更新频率。注意：**飞控收到的仍是 16 路 11 bit 的 0x16 帧**，只是某些通道的有效分辨率较低、更新较慢。
| 通道 | 飞控功能 | Hybrid | Wide | Full Res 8ch | Full Res 16ch Rate/2 | Full Res 12ch Mixed |
| --- | --- | --- | --- | --- | --- | --- |
| 1–4 | 摇杆 | Normal Range（每包） | Normal Range（每包） | Ext Limits（每包） | Ext Limits（半速） | Ext Limits（每包） |
| 5 | AUX1 | 2 档解锁（每包） | 2 档解锁（每包） | Ext Limits 解锁\* | Ext Limits 半速 解锁\* | Ext Limits 半速 解锁\* |
| 6–11 | AUX2–7 | 6 档轮询 | 64 档轮询 | 6–8 为 Ext Limits，9–11 无 | Ext Limits 半速 | Ext Limits 半速 |
| 12 | AUX8 | 16 档轮询 | 64 档轮询 | — | Ext Limits 半速 | Ext Limits 半速 |
| 13 | AUX9 | — | — | — | Ext Limits 半速 | — |
| 14 | AUX10 | 2 档解锁状态\* | 2 档解锁状态\* | 2 档解锁状态\* | Ext Limits 半速 | 2 档解锁状态\* |
| 15 | AUX11 | LQ % | LQ % | LQ % | Ext Limits 半速 | LQ % |
| 16 | AUX12 | RSSI % | RSSI % | RSSI % | Ext Limits 半速 | RSSI % |
| 可用包率 | — | 50 / 150 / 250 / 500 / D250 / D500 / F500 / F1000 | 同 Hybrid | 仅 100Hz Full、333Hz Full | 同左 | 同左 |
带 \* 的项以 ELRS 文档对应版本的注释为准。
- **Normal Range**：10 bit，映射到 988–2012 µs（1 bit = 1 µs）；**CRSF Ext Limits**：10 bit，映射到 885–2115 µs（1 bit ≈ 1.23 µs）；
- 4.x 的 Full Res 模式下 CH5 也是全比例通道；3.x 中 CH5 仍为 1 bit 解锁开关；
- **解锁开关务必放在 AUX1（CH5），约 1000 µs = 未解锁、约 2000 µs = 已解锁**：ELRS 用它启用安全与性能相关的行为（如 Race 遥测模式在解锁后停发遥测）。3 段开关当解锁开关时，要在 EdgeTX 混控里重映射成 2 段。
## 4. 链路信息与信号健康
- 0x14 链路统计由 RX 发给飞控，同时 **CH15 = LQ、CH16 = RSSI**（都映射到 988–2012 µs）；
- **RSSI**（dBm，越接近 0 越强，−130 dBm 为硬下限）表示“声音多大”；**LQ**（%）表示“听懂了多少”——LQ 才是能否飞行的关键，RSSI 告诉你离灵敏度悬崖还有多远；
- 告警阈值建议设在当前包率灵敏度极限之上 5–10 dB（例：2.4 GHz 250 Hz 为 −108 dBm，则告警设在 −103 至 −98 dBm）；
- OSD 的 LQ 字段常带 RF 模式前缀，如 `7:100` 表示 RFMD 7、LQ 100 %。
RF 模式索引 RFMD（ExpressLRS 4.x 文档节选，**编号随版本变化，以对应版本文档为准**）：
| RFMD | 频段 | Lua 显示 | 灵敏度极限 |
| --- | --- | --- | --- |
| 0 / 1 / 2 | 900 MHz | 25Hz / 50Hz / 100Hz | −123 / −120 / −117 dBm |
| 3 | 900 MHz | 100Hz Full | −112 dBm |
| 5 / 6 / 7 | 900 MHz | 200Hz / 200Hz Full / 250Hz | −112 / −111 / −111 dBm |
| 10 / 11 | 900 MHz | D50 / K1000 Full | −112 / −101 dBm |
| 21 / 23 / 24 | 2.4 GHz | 50Hz / 100Hz Full / 150Hz | −115 / −112 / −112 dBm |
| 27 / 28 / 29 | 2.4 GHz | 250Hz / 333Hz Full / 500Hz | −108 / −105 / −105 dBm |
| 30 / 31 | 2.4 GHz | D250 / D500 | −104 / −104 dBm |
对 CRSF 实现者的启示：0x14 的 rf_profile 字段在 ELRS 里就是这个 RFMD，**不要按 TBS 的 4 / 50 / 150 Hz 去解读**。
## 5. 遥测比与下行带宽
- 遥测比（Telem Ratio）可选 Off、1:128、1:64、1:32、1:16、1:8、1:4、1:2，以及 3.0 起的 **Std**（随包率自动选择）和 **Race**（同 Std，但**解锁后停止遥测和同步**）；
- 下行包分两类：**LINK**（链路统计，约每 512 ms 一次，始终存在）和 **DATA**（飞控遥测，与 MSP 共享带宽）；
- 带宽（官方遥测带宽表节选）：
| 包率 | 遥测比 | 遥测包 / 秒 | 带宽（无 burst） | 带宽（burst） |
| --- | --- | --- | --- | --- |
| 500 Hz | 1:2 | 250 | 5000 bps | 9922 bps |
| 500 Hz | 1:128 | 3.9 | 78 bps | 78 bps |
| 250 Hz | 1:64 | 3.9 | 78 bps | 78 bps |
| 250 Hz | 1:2 | 125 | 2500 bps | 4922 bps |
| 100 Hz | 1:8 | 12.5 | 250 bps | 417 bps |
| 50 Hz | 1:128 | 0.4 | 8 bps | 8 bps |
- 推论：飞控（Betaflight 每种传感器约 10 Hz）产生的遥测远多于低遥测比能传的量，遥控器上的数值会“慢半拍”；GPS、电池等关键数据优先；不需要的传感器可在 Betaflight 中用 `telemetry_disabled_*` 系列参数关闭。
## 6. Lua 配置脚本 = CRSF 参数协议的“前端”
- ELRS Lua 运行在 EdgeTX 上，用 0x28 / 0x29 发现设备，用 0x2C / 0x2B 读参数、0x2D 写参数（模块 04）；
- 常见菜单项：Packet Rate、Telem Ratio、Switch Mode、Antenna Mode、Link Mode、Model Match、TX Power、VTX Administrator、WiFi Connectivity、Backpack、Bind；
- **Other Devices**：列出总线上其他 CRSF 设备（如接收机、飞控、你自己做的设备），可进入它们的参数菜单；
- Link Mode（3.5.0 起）：Normal（默认）或 MAVLink（TX 模块与遥控器成为地面站和支持 MAVLink 的飞行器之间的中介）。
## 7. 场景推荐（ELRS 官方建议）
| 场景 | 包率 | 遥测 | 开关模式 |
| --- | --- | --- | --- |
| 竞速 | 500 Hz 或更快 | Race | Hybrid 也不错 |
| 花飞 / 日常 | 250 Hz 或更快 | Std | 按飞行模式需求选择 |
| 固定翼 | 100Hz Full | Std 或自选 | 取决于需要多少全分辨率通道 |
> **飞行中不要修改包率**：会强制 TX 与 RX 断开重连。
> **本章要点**
> - 遥控器 ⇄ TX：半双工；400K 最高 250 Hz，更高包率需更高波特率；ELRS TX 自动识别波特率
> - RX ⇄ FC：完整 UART 对、不反相、全双工、默认 420000；Betaflight 选 Serial + CRSF 并打开 Telemetry
> - 开关模式决定通道分辨率；解锁放在 AUX1（CH5，约 1000 / 2000 µs）；CH15 = LQ、CH16 = RSSI
> - LQ 决定能否飞，RSSI 提示离灵敏度悬崖多远；RFMD 编号以 ELRS 文档为准
> - 遥测带宽很有限（50Hz 1:128 只有约 8 bps），MSP 与飞控遥测共享 DATA 带宽
📖 出处：[ELRS 配置飞控](https://www.expresslrs.org/quick-start/receivers/configuring-fc) · [ELRS 开关模式](https://www.expresslrs.org/software/switch-config/) · [ELRS 信号健康](https://www.expresslrs.org/info/signal-health/) · [ELRS 遥测带宽](https://www.expresslrs.org/info/telem-bandwidth/) · [ELRS Lua 说明](https://www.expresslrs.org/quick-start/transmitters/lua-howto/) · [ELRS 遥控器准备](https://www.expresslrs.org/quick-start/transmitters/tx-prep) · [ELRS 接收机串口协议](https://www.expresslrs.org/software/serial-protocols) · [ELRS ArduPilot 设置](https://www.expresslrs.org/quick-start/ardupilot-setup)
