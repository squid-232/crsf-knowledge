# 🧭 01 协议总览与系统架构

**中文** | [English](en/01-overview.md) · [知识体系首页](../README.md)

> **本章摘要**：CRSF 是 TBS 为 Crossfire 设计、现被 ExpressLRS 等系统广泛采用的串口协议。它只定义“有线”部分：遥控器 ⇄ TX 模块、接收机 ⇄ 飞控。本章建立全局模型：谁和谁通信、用什么物理层、设备如何寻址、帧分哪几类。
## 1. CRSF 要解决什么问题
TBS 规范在 Features 一节列出了五个设计目标：
1. **低延迟、高更新率**地传输遥控信号（RC ⇄ TX、RX ⇄ FC 两段）
2. **双向通信**：同一条链路既下发控制，也回传数据
3. **遥测**：把飞行平台的数据（电池、GPS、姿态、链路质量…）送回遥控器
4. **远程配置**：既能配置直连设备，也能配置远端设备，例如遥控器通过 CRSF 配置飞控或 OSD
5. **共享接收机序列号**给 TX，以便与遥控器中的模型存储匹配（Model Match）
规范的定位是由 TBS 维护的公开“单一可信来源（single source of truth）”。ExpressLRS 的实现以 crsf-wg 社区文档（CapnBry 整理）为依据，与 TBS 规范兼容，但在若干细节上有自己的约定（见模块 05）。
## 2. CRSF 在系统中的位置
```text
  Handset (RC)          TX module               Receiver (RX)          Flight Controller
  EdgeTX  [0xEA]        [0xEE]                  [0xEC]                 [0xC8]
 +-------------+  CRSF  +-------------+  RF     +-------------+  CRSF  +-------------+
 |             |------->|             |~~~~~~~~>|             |------->|             |
 |             |<-------|             |<~~~~~~~~|             |<-------|             |
 +-------------+        +-------------+         +-------------+        +-------------+
   half-duplex 1-wire                  NOT CRSF   full-duplex UART
```
| 链路 | 物理层 | 收发关系 | 主要内容 |
| --- | --- | --- | --- |
| 遥控器 ⇄ TX 模块 | 单线半双工 UART | 遥控器为主机 | 下行：RC 通道帧；上行：链路统计、遥测、参数应答 |
| TX 模块 ⇄ 接收机 | 无线空口（**不是 CRSF**） | 双向、时分 | Crossfire / ELRS 自有空口协议，负责“搬运”CRSF 数据 |
| 接收机 ⇄ 飞控 | 双线全双工 UART | 双方都可发送 | RX→FC：RC 通道、链路统计、参数/MSP 请求；FC→RX：遥测、应答 |
> **关键认识**：CRSF 帧在空中并不是原样传输的。ELRS 会把通道压缩成自己的空口格式（开关模式、Full Res 等），遥测按“遥测比”分时回传。因此“空口包率”和“串口上的 CRSF 帧率”是两个概念，调试时不要混淆。
## 3. 设备地址
每个设备有 1 字节地址：用于扩展帧的 Dest / Orig 字段，也常作为帧首的 Sync 字节。
| 地址 | 设备 |
| --- | --- |
| 0x00 | 广播（Broadcast），如 Ping 所有设备 |
| 0x0E | Cloud |
| 0x10 | USB 设备 |
| 0x12 | 蓝牙 / WiFi 模块 |
| 0x13 | WiFi 接收机（游戏 / 模拟器） |
| 0x14 | 视频接收机 VRX |
| 0x20–0x7F | 动态地址（NAT） |
| 0x80 | OSD / TBS PNP PRO |
| 0x90–0x97 | ESC 1–8 |
| 0xC0 | 电压 / 电流传感器 |
| 0xC2 | GPS |
| 0xC4 | TBS Blackbox |
| **0xC8** | **飞控 FC**（同时是最常用的 Sync 字节） |
| 0xCC | Race Tag |
| 0xCE | VTX 图传 |
| **0xEA** | **遥控器（Remote Control）** |
| 0xEB | 中继接收机（Repeater RX） |
| **0xEC** | **接收机 RX** |
| 0xED | 中继发射机（Repeater TX） |
| **0xEE** | **TX 模块（高频头）** |
## 4. 网络拓扑与路由
- CRSF 设备构成**星形网络**。一个设备可以有多个 CRSF 端口，收到不是发给自己的帧时，需要**转发到其他端口**；规范禁止形成环路，否则同一帧会被无限转发。
- 例：遥控器（0xEA）读取飞控（0xC8）参数的请求，路径为 遥控器 → TX 模块 → 空口 → 接收机 → 飞控，应答沿原路返回。正因为有转发，遥控器才能“隔空”配置飞控、OSD、VTX。
- 广播帧没有地址字段，由关心它的设备按 Type 消费，例如飞控只处理 0x16 通道帧和 0x14 链路统计。
## 5. 帧家族一览
| 类别 | Type 范围 | 结构 | 典型用途 |
| --- | --- | --- | --- |
| 广播帧（短帧头） | 0x27 以下 | \[Sync\]\[Len\]\[Type\]\[Payload\]\[CRC\] | RC 通道、链路统计、各类遥测 |
| 扩展帧（扩展帧头） | 0x28 及以上 | \[Sync\]\[Len\]\[Type\]\[Dest\]\[Orig\]\[Payload\]\[CRC\] | 设备发现、参数配置、命令、MSP |
个别厂商自定义类型（如 MAVLink 封装 0xAA）的帧头形式以规范对应小节为准。细节见模块 02 / 03 / 04。
## 6. 生态与相关协议
- **TBS Crossfire / Tracer**：CRSF 的发明者，空口为 TBS 私有协议。
- **ExpressLRS（ELRS）**：开源远控系统。TX 与遥控器、RX 与飞控之间都用 CRSF；ELRS 接收机也能输出 SBUS、SUMD、HoTT 等，但官方建议“硬件支持 CRSF 就用 CRSF”，因为 SBUS 不能回传飞控遥测。
- **EdgeTX / OpenTX**：遥控器固件，内置 CRSF 驱动与 Lua 配置脚本运行环境。
- **Betaflight / INAV / ArduPilot / PX4**：飞控固件，原生支持 CRSF 输入与遥测输出。
> **本章要点**
> - CRSF = 两段“有线”协议：RC ⇄ TX（单线半双工）、RX ⇄ FC（全双工 UART）
> - 无线空口不是 CRSF；包率、遥测比属于空口概念
> - 关键地址：0xEA 遥控器、0xEE TX 模块、0xEC 接收机、0xC8 飞控、0x00 广播
> - 帧分广播帧（短帧头）与扩展帧（Type ≥ 0x28，带 Dest / Orig）
> - 多端口设备负责转发，使遥控器可以远程配置飞控、OSD、VTX
📖 出处：[TBS CRSF 规范（Features / Hardware / Routing / Device Addresses）](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [ExpressLRS 接收机串口协议](https://www.expresslrs.org/software/serial-protocols) · [crsf-wg CRSF Wiki](https://github.com/crsf-wg/crsf/wiki)
