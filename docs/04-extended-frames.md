# 🛠️ 04 扩展帧：设备发现、参数配置与命令

> **本章摘要**：扩展帧在 Type 后多了 Dest / Orig 两个地址字节，可点对点寻址并经多端口设备转发。本章覆盖：设备发现（0x28 / 0x29）、参数系统（0x2B / 0x2C / 0x2D，含分块与数据类型）、直接命令 0x32（双 CRC）、RC 同步 0x3A、MSP over CRSF（0x7A / 0x7B）以及厂商扩展帧。ExpressLRS Lua 菜单、遥控器上的“绑定”按钮、远程调 VTX，底层都是这些帧。
## 1. 扩展帧回顾
- 结构：`[Sync][Len][Type][Dest][Orig][Payload][CRC]`，Type ≥ 0x28；Len = Payload + 4（Type、Dest、Orig、CRC），Payload 最多 58 字节；
- **Host**（发起方，通常是遥控器 0xEA）向 **Device**（TX 模块 0xEE、接收机 0xEC、飞控 0xC8……）发请求，应答时**交换 Dest / Orig**；
- 例外：0x34 Logging、0x80 ArduPilot、0x81 / 0x82 mLRS、0xAA MAVLink 封装虽然 Type ≥ 0x28，却使用**短帧头**——判断帧头形式时要查规范对应小节。
## 2. 设备发现：0x28 Ping → 0x29 Device Information
1. Host 发送 **0x28**（无 Payload），Dest = 指定设备地址，或 0x00 广播询问所有设备；
2. 每个收到的设备回复 **0x29**：
| 字段 | 类型 | 说明 |
| --- | --- | --- |
| Device_name | 字符串（`\0` 结尾） | 如 "ELRS RX" |
| Serial_number | uint32 | ExpressLRS 固定为 0x454C5253（ASCII "ELRS"），ELRS Lua 据此识别 ELRS 设备 |
| Hardware_ID | uint32 | 硬件标识 |
| Firmware_ID | uint32 | 固件版本标识 |
| Parameters_total | uint8 | 参数总数 |
| Parameter_version_number | uint8 | 参数表版本号 |
测试向量（已校验 CRC）：
| 帧 | 字节 |
| --- | --- |
| 遥控器广播 Ping（Sync 0xEE 指向 TX 模块，Dest 0x00，Orig 0xEA） | `EE 04 28 00 EA 54` |
| 接收机回复 Device Info（Dest 0xEA，Orig 0xEC，"ELRS RX"，12 个参数） | `EA 1A 29 EA EC 45 4C 52 53 20 52 58 00 45 4C 52 53 00 00 00 00 00 04 01 00 0C 00 9C` |
## 3. 参数系统（Lua 菜单的底层协议）
### 3.1 一次完整的交互
```text
Host（遥控器 0xEA）                          Device（TX 模块 0xEE）
  0x28 Ping, Dest=0x00          ------------>
                                <------------  0x29 Device Info（名称、参数总数 N）
  0x2C Read [param 0][chunk 0]  ------------>
                                <------------  0x2B [0][剩余 0] FOLDER "ROOT" + 子项列表
  0x2C Read [param 1][chunk 0]  ------------>
                                <------------  0x2B [1][剩余 1] 第 1 块
  0x2C Read [param 1][chunk 1]  ------------>
                                <------------  0x2B [1][剩余 0] 最后一块，拼接完成
  ... 依次读完 1..N ...
  0x2D Write [param k][新值]    ------------>
                                <------------  0x2D 回显新值（FLOAT / TEXT_SELECTION / STRING）
                                               或 0x2B 完整条目（COMMAND）
```
### 3.2 0x2C 读请求与分块规则
- Payload：`[Parameter_number][Chunk_number]`，块号从 0 开始；
- Host **总是先读 chunk 0**；设备在 0x2B 里给出“剩余块数”，大于 0 就把块号 +1 继续读**同一个参数**，等于 0 才能开始读下一个参数；
- 每块最多 **56 字节**条目数据（58 − 参数号 − 剩余块数）；0x2C 也用于重传在链路上丢失的块；
- 测试向量：`EE 06 2C EE EA 01 00 86`（遥控器请求 TX 模块参数 1 的第 0 块）。
### 3.3 0x2B 参数条目
每帧 Payload = `[Parameter_number][Chunks_remaining][数据块]`。把所有块按顺序拼起来得到完整的“条目数据”：
`[Parent_folder][Data_type][Name\0][类型相关字段……]`
- Data_type **bit 7 = 隐藏标志**（1 = 隐藏，可根据其他参数动态显示 / 隐藏），bit 0–6 = 类型；
- **参数 0 是根目录**：类型必须是 FOLDER，建议名称 "ROOT"。若设备不回应参数 0，说明它不支持根目录结构，Host 应从 1 读到 Parameters_total。
| 值 | 类型 | 类型相关字段 | 0x2D 写入的数据 |
| --- | --- | --- | --- |
| 0–5 | UINT8 … INT32 | 建议废弃，统一用 FLOAT 表达 | — |
| 8 | FLOAT | int32 Value / Min / Max / Default、uint8 小数位数、int32 步长、Unit `\0` | int32（4 字节） |
| 9 | TEXT_SELECTION | Options `\0`（分号分隔的选项）、uint8 Value / Min / Max / Default、Unit `\0` | uint8 选项索引（1 字节） |
| 10 | STRING | Value `\0`、uint8 最大长度 | 新字符串 |
| 11 | FOLDER | 子参数编号列表，以 0xFF 结束 | 不可写 |
| 12 | INFO | Info `\0`（只读，没有最大长度） | 不可写 |
| 13 | COMMAND | uint8 Status、uint8 Timeout（规范注释 “ms \* 100”）、Info `\0` | uint8 Status |
| 127 | OUT_OF_RANGE | 请求的编号超出范围 / 参数列表结束 | — |
FLOAT 用“整数 + 小数位数”表达：Value = 1234、Decimal_point = 2 表示 12.34。
### 3.4 0x2D 写入与确认
- Payload：`[Parameter_number][Data]`；
- 设备**必须**确认：FLOAT / TEXT_SELECTION / STRING → 回 **0x2D**（相同参数号 + 已接受的新值）；COMMAND → 回完整的 **0x2B**（带新的 Status 与 Info，供 Host 显示进度）。
### 3.5 COMMAND 状态机（以“绑定”为例）
| 值 | 状态 | 方向 | 含义 |
| --- | --- | --- | --- |
| 0 | READY | Device → Host | 空闲 / 完成 |
| 1 | START | Host → Device | 开始执行 |
| 2 | PROGRESS | Device → Host | 执行中，Info 显示进度 |
| 3 | CONFIRMATION_NEEDED | Device → Host | Host 弹出“确认 / 取消”对话框 |
| 4 | CONFIRM | Host → Device | 用户确认 |
| 5 | CANCEL | Host → Device | 用户取消 |
| 6 | POLL | Host → Device | 强制设备重发最新的 0x2B 状态 |
```text
Host: 0x2C 读参数 "Bind"            Device: 0x2B COMMAND  Status=READY     Info=""
Host: 0x2D Status=START             Device: 0x2B COMMAND  Status=PROGRESS  Info="Binding"
Host: 0x2D Status=POLL（可多次）     Device: 0x2B 当前状态
                                    Device: 0x2B COMMAND  Status=READY     Info="OK"（完成）
```
规范特别指出：读取 TX 模块（0xEE）的 Bind 这类命令时 Host 必须发 POLL，否则拿不到最新状态。
### 3.6 做一个“能被 Lua 配置”的设备：最小功能集
- 响应 0x28（Dest 为自己或 0x00）→ 回 0x29，Parameters_total 填参数个数；
- 维护参数表：参数 0 = ROOT 文件夹并列出子项，其余参数按类型序列化；
- 响应 0x2C：把条目数据按 56 字节切块，正确填写剩余块数；
- 响应 0x2D：检查范围 → 保存（建议写 Flash 前做防抖）→ 回确认；
- 遥控器 Lua 经 TX 模块和无线链路转发访问远端设备，你的设备只需挂在 CRSF 总线上、对正确的 Dest 作出回应（在 ELRS Lua 的 “Other Devices” 中出现）。
## 4. 0x32 Direct Commands（直接命令）
- 结构：`[Sync][Len][0x32][Dest][Orig][Command_ID][子命令 / 数据][Cmd_CRC8][CRC8]`；
- **命令 CRC**：多项式 **0xBA**，覆盖 0x32、Dest、Orig、Command_ID 和 Payload；
- **外层 CRC**：仍是 0xD5，覆盖 Type 到命令 CRC——两个 CRC 都要算。
Betaflight 让接收机进入绑定模式时发送的帧：
```text
C8 07 32 EC C8 10 01 9E E8
|  |  |  |  |  |  |  |  '-- 外层 CRC8（poly 0xD5，覆盖 32 ... 9E）
|  |  |  |  |  |  |  '----- 命令 CRC8（poly 0xBA，覆盖 32 EC C8 10 01）
|  |  |  |  |  |  '-------- 子命令 0x01 = 进入绑定
|  |  |  |  |  '----------- Command_ID 0x10 = Crossfire（接收机命令）
|  |  |  |  '-------------- Orig 0xC8 = 飞控
|  |  |  '----------------- Dest 0xEC = 接收机
|  |  '-------------------- Type 0x32 = Direct Commands
|  '----------------------- Len = 7
'-------------------------- Sync 0xC8
```
规范自带示例：`C8 07 32 EE 13 13 01 66 06`（向 TX 模块发送“开启 RC over WiFi”）。
| Command_ID | 类别 | 主要子命令 |
| --- | --- | --- |
| 0xFF | Command ACK | 回执：Command_ID、SubCommand_ID、Action（1 = 已执行，0 = 不支持）、Info |
| 0x01 | FC | 0x01 强制上锁（Force Disarm）、0x02 Scale Channel |
| 0x03 | Bluetooth | 0x01 复位、0x02 启用 / 禁用、0x64 Echo |
| 0x05 | OSD | 0x01 发送按键（bit7 Enter、bit6 上、bit5 下、bit4 左、bit3 右） |
| 0x08 | VTX | 0x02 设频率（MHz）、0x04 上电进入 PitMode、0x05 退出 PitMode、0x06 动态功率（需 1 Hz 持续发送，3 s 未收到即回退）、0x08 设功率（dBm） |
| 0x09 | LED | 恢复默认、覆盖颜色 / 呼吸 / 闪烁 / 渐变（HSV 打包） |
| 0x0A | General | 0x70 波特率提议（uint8 port_id、uint32 波特率）、0x71 提议应答（port_id、1 接受 / 0 拒绝） |
| 0x10 | Crossfire | 0x01 进入绑定、0x02 取消绑定、0x03 设置绑定 ID、0x05 选择模型、0x06 / 0x07 查询 / 回复当前模型、0x0A / 0x0B 开 / 关 RX 遥测、0x0C 恢复出厂设置、0x0D 恢复出厂固件 |
| 0x13 | RC over WiFi | 0x01 开启、0x02 关闭 |
| 0x20 | Flow Control | 0x01 订阅（帧类型 + 最大间隔 ms）、0x02 取消订阅 |
| 0x22 | Screen | 0x01 弹窗消息、0x02 选择结果回传 |
### 4.1 波特率协商（0x32 / 0x0A / 0x70、0x71）
- Host 提议更高波特率 → Device 回应接受 / 拒绝 → 双方切换；
- Betaflight 的实现：切换后若**连续 3 次**帧错误就退回默认波特率；
- ExpressLRS **不使用**这套协商：RX ⇄ FC 是固定波特率（默认 420000，可在 WebUI 修改）；遥控器 ⇄ TX 的波特率由 EdgeTX 设置决定，ELRS TX 模块自动识别。
## 5. 0x3A.0x10 RC 同步（Timing Correction，“CRSF Shot”）
- 方向：TX 模块 → 遥控器（半双工链路）；布局：`EA 0D 3A EA EE 10 [update_interval] [offset] CRC`；
- update_interval：uint32，LSB = 100 ns，即 TX 模块空口发包周期（250 Hz → 4 ms → 40000）；
- offset：int32，LSB = 100 ns，**正值 = 遥控器的数据来得太早**，负值 = 太晚；
- 作用：遥控器据此微调混控 / 发送周期，让 RC 帧恰好在空口发包前到达，降低延迟和抖动（ExpressLRS 用它把 EdgeTX 混控器同步到包率）；EdgeTX 收到后按 1 µs 精度处理。
## 6. MSP over CRSF（0x7A 请求 / 0x7B 应答）
- 用途：经无线链路透传 MSP，遥控器上的 Betaflight Lua 脚本、VTX 设置等靠它远程读写飞控；
- 去掉 MSP 帧头（`$` + M / X + 方向符）和 MSP 校验，只传 MSP 主体；
- 每帧 Payload 前加 1 字节 **Status**：bit 0–3 循环序号；bit 4 = 新 MSP 包的第一块；bit 5–6 MSP 版本（1 或 2）；bit 7 错误（仅应答）；
- 每帧 MSP 块最多 **57 字节**，长包分块传输；不传 MSP CRC（已有 CRSF CRC 保护）；
- 应答必须**交换 Dest / Orig**（如 Betaflight 应答 Dest 0xEA、Orig 0xC8）；
- 依赖遥测下行：ExpressLRS 遥测比设为 Off 时 MSP 无法工作，且 MSP 与遥测共享下行带宽（见模块 05）。
## 7. 其他扩展 / 厂商帧（了解即可）
| Type | 名称 | 说明 |
| --- | --- | --- |
| 0x34 | Logging | 调试日志，短帧头 |
| 0x3C | Game | 加分、指令码 |
| 0x80 | ArduPilot Passthrough | 短帧头；子类型 0xF0 单包、0xF2 多包（最多 9 个）、0xF1 状态文本 |
| 0x81 / 0x82 | mLRS | 封装 mLRS 的 mBridge 协议，用于其 Lua 配置脚本 |
| 0x88 | Rotorflight 遥测封装 | 格式随 Rotorflight 版本变化 |
| 0xAA | MAVLink 封装 | total_chunks:4 / current_chunk:4（都从 0 开始计）、data_size 最多 58 |
| 0xAC | MAVLink 系统状态 | 详见规范 |
> **本章要点**
> - 扩展帧 = 广播帧 + Dest / Orig；应答交换地址；少数 ≥ 0x28 的厂商帧仍是短帧头
> - 设备发现：0x28 Ping → 0x29 Device Info（名称、序列号、参数总数；ELRS 序列号为 "ELRS"）
> - 参数：0x2C 读（参数号 + 块号）→ 0x2B 条目（剩余块数，每块 ≤ 56 B）→ 0x2D 写并必须确认
> - 参数 0 为 ROOT 文件夹；类型常用 FLOAT / TEXT_SELECTION / STRING / FOLDER / INFO / COMMAND
> - 0x32 命令有两层 CRC：内层 0xBA、外层 0xD5
> - 0x3A.0x10 用于遥控器与 TX 同步；MSP over CRSF 依赖遥测下行
📖 出处：[TBS CRSF 规范（Extended Frame Types）](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [Betaflight rx/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/rx/crsf.c) · [ExpressLRS Lua 说明](https://www.expresslrs.org/quick-start/transmitters/lua-howto/) · [ExpressLRS CRSFHandset.cpp](https://github.com/ExpressLRS/ExpressLRS/blob/master/src/lib/Handset/CRSFHandset.cpp)
