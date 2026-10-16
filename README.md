# 📡 CRSF 协议学习知识体系（TBS 规范 × ExpressLRS 实践）

**中文** | [English](README.en.md)

> 首页介绍支持中英切换；8 个详细知识模块目前为中文。

> 面向嵌入式开发者的 CRSF（Crossfire Serial Protocol）学习与实现手册
> - **主线**：TBS 官方《CRSF Protocol Specification》
> - **实践**：ExpressLRS 官方文档（4.x），辅以 Betaflight 源码与 crsf-wg 社区文档
> - **配套**：可直接移植的 C99 参考实现 crsf-mini（63 项单元测试通过）+ Python 抓包脚本 + 测试向量
> - 整理时间：2026-10
## 🎯 一句话理解 CRSF
CRSF 是 TBS 为 Crossfire 设计、现已成为 FPV 事实标准的**有线串口协议**：遥控器 ⇄ 高频头（TX 模块）、接收机（RX）⇄ 飞控（FC）这两段都用它传输 **16 路 11-bit 通道 + 双向遥测 + 远程参数配置**。中间的无线空口（Crossfire / ExpressLRS 自有空口）**不是** CRSF——CRSF 只负责两端的“线”。
## 🗺️ 系统全景
```text
  Handset (RC)          TX module               Receiver (RX)          Flight Controller
  EdgeTX  [0xEA]        [0xEE]                  [0xEC]                 [0xC8]
 +-------------+  CRSF  +-------------+  RF     +-------------+  CRSF  +-------------+
 |             |------->|             |~~~~~~~~>|             |------->|             |
 |             |<-------|             |<~~~~~~~~|             |<-------|             |
 +-------------+        +-------------+         +-------------+        +-------------+
  half-duplex, 1 wire                  NOT CRSF   full-duplex UART, 8N1, not inverted
  400 kbaud default (ELRS: up to 5.25M) (air link) 416666 (TBS) / 420000 (ExpressLRS)
  --> RC channels 0x16                            --> RC channels 0x16, link stats 0x14
  <-- link stats, telemetry, params               <-- telemetry 0x08 0x02 0x1E 0x21 ...
```
| 对比项 | 遥控器 ⇄ TX 模块 | 接收机 ⇄ 飞控 |
| --- | --- | --- |
| 物理层 | 单线半双工 UART | 双线全双工 UART（TX/RX 交叉） |
| 波特率 | 默认 400 kbaud；ELRS 可选 115200 \~ 5.25 Mbaud | 规范 416666；ELRS 默认 420000 |
| 极性 | 3.3 V，可正相也可反相 | 3.0–3.3 V，**不反相** |
| 收发关系 | 遥控器为主机：每周期发一帧，模块同步后在间隙回复 | 双方都可随时发送 |
| 典型数据 | 下行 RC 通道帧；上行链路统计、遥测、参数应答 | RX→FC：通道、链路统计；FC→RX：遥测、MSP 应答 |
## 🧭 知识地图
| # | 模块 | 核心知识点 | 难度 | 建议用时 |
| --- | --- | --- | --- | --- |
| 01 | [01 协议总览与系统架构](docs/01-overview.md) | 两段链路、设备地址、星形路由、帧家族 | ⭐ | 1 h |
| 02 | [02 物理层与帧结构](docs/02-physical-layer-and-frames.md) | 波特率/极性/半双工时序、帧格式、Len 语义、CRC8、容错规则 | ⭐⭐ | 2 h |
| 03 | [03 核心帧类型详解](docs/03-frame-types.md) | 0x16 通道 11-bit 打包、0x14 链路统计、遥测帧字段与单位 | ⭐⭐ | 3 h |
| 04 | [04 扩展帧：设备发现、参数配置与命令](docs/04-extended-frames.md) | Ping/Device Info、参数树与分块、COMMAND 状态机、0x32 命令、MSP | ⭐⭐⭐ | 4 h |
| 05 | [05 ExpressLRS 中的 CRSF 实践](docs/05-expresslrs.md) | 波特率与包率、飞控配置、开关模式、遥测比与带宽、LQ/RSSI | ⭐⭐ | 2 h |
| 06 | [06 嵌入式实现指南（含 C 参考代码）](docs/06-embedded-implementation.md) | UART+DMA+IDLE、解析状态机、超时与重同步、失控、遥测调度 | ⭐⭐⭐ | 6 h |
| 07 | [07 调试、测试与常见坑](docs/07-debugging-and-tests.md) | 工具与抓包脚本、测试向量、22 条避坑清单 | ⭐⭐ | 2 h |
| 08 | [08 实践项目与自测题](docs/08-projects-and-quiz.md) | 5 个递进项目、15 道自测题 | ⭐⭐⭐ | 1–2 周 |
## 🛤️ 学习路线
### 阶段 1 · 建立全局认识（第 1 天）
- [ ] 读 01：分清 RC⇄TX、RX⇄FC 两段链路，记住 0xEA / 0xEE / 0xEC / 0xC8 四个地址
- [ ] 读 02：能默写帧结构，说清 Len 的含义与 CRC 覆盖范围
- [ ] 用 07 的测试向量手动（或用脚本）验证一次 CRC8
### 阶段 2 · 吃透核心帧（第 2–3 天）
- [ ] 读 03：实现 16×11-bit 通道打包/解包，掌握 172 / 992 / 1811 与微秒换算
- [ ] 解析 0x14 链路统计，理解 RSSI、LQ、SNR、RF 模式与功率枚举
- [ ] 读 05：把 ELRS 接收机接到 Betaflight/INAV，配置 CRSF 并观察通道与遥测
### 阶段 3 · 动手实现（第 4–7 天）
- [ ] 读 06：在 STM32/ESP32 上用 UART DMA + IDLE 接收并解析 CRSF
- [ ] 实现失控检测与通道输出（PWM 舵机 / 电调）
- [ ] 从 MCU 回传电池、姿态、飞行模式遥测，在 EdgeTX 上“发现传感器”
- [ ] 读 07：对照避坑清单自查，并用测试向量写单元测试
### 阶段 4 · 进阶（第 2 周起）
- [ ] 读 04：实现 Ping / Device Info 应答与参数树，让 ELRS Lua 能配置你的设备
- [ ] 尝试 MSP over CRSF、遥控器侧半双工驱动、RC 同步（0x3A.0x10）
- [ ] 完成 08 的项目与自测题
## ⚡ 速查卡
```text
帧格式   [Sync][Len][Type][Payload...][CRC]            Len = Type+Payload+CRC 字节数 (2..62)
扩展帧   [Sync][Len][Type][Dest][Orig][Payload][CRC]    Type >= 0x28，带目的/源地址
长度     整帧最多 64 字节；总长 = Len + 2
Sync     0xC8 最常用；也可能是 0x00 或目标设备地址（接收方都应接受）
CRC8     多项式 0xD5，初值 0，只覆盖 Type..Payload（不含 Sync、Len）
0x32     额外内层 CRC8：多项式 0xBA，覆盖 0x32,Dest,Orig,CmdID,Payload
字节序   多字节字段大端；RC 通道 11-bit 打包为 LSB 优先
波特率   RX<->FC：416666（规范）/ 420000（ELRS），8N1，不反相，全双工
         RC<->TX：默认 400000，单线半双工；ELRS 400K 下最高 250 Hz
通道     16 x 11 bit = 22 字节；172=988us  992=1500us  1811=2012us
         us = (ticks-992)*5/8+1500      ticks = (us-1500)*8/5+992
地址     0xEA 遥控器  0xEE TX 模块  0xEC 接收机  0xC8 飞控  0x00 广播
失控     RX 失控时停发 0x16（cut）；规范建议飞控等待约 1 s 再进入 failsafe 流程
```
| Type | 名称 | 典型方向 | 负载要点 |
| --- | --- | --- | --- |
| 0x16 | RC Channels Packed | RC→TX、RX→FC | 22 B，16 × 11 bit |
| 0x14 | Link Statistics | TX→RC、RX→FC | 10 B：RSSI×2、LQ、SNR、天线、RF 模式、功率、下行 RSSI/LQ/SNR |
| 0x08 | Battery Sensor | FC→RX→RC | 电压、电流（实践 0.1 V / 0.1 A）、uint24 已用 mAh、剩余 % |
| 0x02 | GPS | FC→RX→RC | 经纬度 °×1e7、地速、航向 °×100、高度 m+1000、卫星数 |
| 0x1E | Attitude | FC→RX→RC | 俯仰/横滚/偏航 int16，单位 100 µrad |
| 0x21 | Flight Mode | FC→RX→RC | 以 `\0` 结尾的字符串，如 ACRO、!FS! |
| 0x07 | Variometer | FC→RX→RC | 垂直速度 int16，cm/s |
| 0x28 / 0x29 | Ping / Device Info | 双向 | 设备发现：名称、序列号、HW/FW ID、参数个数 |
| 0x2B / 0x2C / 0x2D | 参数条目 / 读 / 写 | 双向 | Lua 配置菜单的底层协议，支持分块 |
| 0x32 | Direct Commands | 双向 | 绑定、VTX、波特率协商等，带内层 CRC |
| 0x7A / 0x7B | MSP 请求 / 响应 | 双向 | 经遥控链路隧道传输 MSP（Betaflight Lua、VTX Admin） |
## 📖 如何使用这套知识体系
- 按学习路线顺序阅读 01 → 08；每个模块开头是**本章摘要**，结尾是**本章要点**，复习时只看这两处即可。
- 动手时直接使用 06 的 C 参考代码（附源码包），以及 07 的 Python 抓包脚本和测试向量。
- 遇到问题先查 07 的避坑清单；需要确认字段细节时回到 TBS 规范原文。
## 🔗 资料来源
- [TBS CRSF Protocol Specification（官方规范）](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md)
- [crsf-wg CRSF Wiki（CapnBry 整理，ExpressLRS 实现依据）](https://github.com/crsf-wg/crsf/wiki)
- ExpressLRS 文档：[飞控配置](https://www.expresslrs.org/quick-start/receivers/configuring-fc) · [接收机串口协议](https://www.expresslrs.org/software/serial-protocols) · [开关模式](https://www.expresslrs.org/software/switch-config/) · [信号健康](https://www.expresslrs.org/info/signal-health/) · [遥测带宽](https://www.expresslrs.org/info/telem-bandwidth/) · [Lua 脚本](https://www.expresslrs.org/quick-start/transmitters/lua-howto/) · [TX 准备](https://www.expresslrs.org/quick-start/transmitters/tx-prep) · [ArduPilot 设置](https://www.expresslrs.org/quick-start/ardupilot-setup)
- 源码：[Betaflight rx/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/rx/crsf.c) · [Betaflight telemetry/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/telemetry/crsf.c) · [ExpressLRS CRSFHandset.cpp](https://github.com/ExpressLRS/ExpressLRS/blob/master/src/lib/Handset/CRSFHandset.cpp)
## 📚 知识模块
[01 协议总览与系统架构](docs/01-overview.md)
[02 物理层与帧结构](docs/02-physical-layer-and-frames.md)
[03 核心帧类型详解](docs/03-frame-types.md)
[04 扩展帧：设备发现、参数配置与命令](docs/04-extended-frames.md)
[05 ExpressLRS 中的 CRSF 实践](docs/05-expresslrs.md)
[06 嵌入式实现指南（含 C 参考代码）](docs/06-embedded-implementation.md)
[07 调试、测试与常见坑](docs/07-debugging-and-tests.md)
[08 实践项目与自测题](docs/08-projects-and-quiz.md)
