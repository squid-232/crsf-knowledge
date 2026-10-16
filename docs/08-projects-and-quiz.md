# 🏆 08 实践项目与自测题

**中文** | [English](en/08-projects-and-quiz.md) · [知识体系首页](../README.md)

> **本章摘要**：用 5 个由浅入深的项目把知识变成能力——从 PC 上的纯软件解码器，到抓包可视化、CRSF 转 PWM、遥测传感器节点，最后做一个能被遥控器 Lua 菜单配置的 CRSF 设备。每个项目给出目标、步骤、验收标准和对应章节。最后是 15 道自测题（点开看答案），用来检验是否真正掌握。
## 1. 项目总览
| # | 项目 | 难度 | 用时 | 硬件 | 主要涉及 |
| --- | --- | --- | --- | --- | --- |
| 1 | PC 端 CRSF 解码器 | ⭐ | 半天 | 无 | 02、03、06 第 4 节 |
| 2 | 串口抓包与可视化 | ⭐⭐ | 1 天 | USB-TTL + ELRS 接收机 | 05、07 |
| 3 | CRSF 转 PWM 舵机输出 | ⭐⭐⭐ | 2–3 天 | STM32 / ESP32 + 舵机 | 06 第 2–5 节 |
| 4 | 遥测传感器节点 | ⭐⭐⭐ | 2–3 天 | MCU + 分压电阻 + IMU | 03、05、06 第 6 节 |
| 5 | 可用 Lua 配置的 CRSF 设备 | ⭐⭐⭐⭐ | 1 周 | MCU + 完整 ELRS 系统 | 04 全章 |
## 2. 项目 1 · PC 端 CRSF 解码器
- **目标**：不碰硬件，先把帧层和协议层写对；
- **步骤**：
	1. 在 PC 上编译 crsf-mini，运行 `make test`（或参照它从零实现自己的版本）；
	2. 写一个命令行工具：从标准输入读十六进制字符串，打印每帧的类型和解码结果；
	3. 把模块 07 的全部测试向量与随机字节、截断帧、假 Sync 拼成一个流，按随机大小分块喂给解析器；
- **验收**：所有向量解码正确；噪声流中真帧一条不丢、也不多出假帧；`crc_errors` / `len_errors` 计数符合预期；
- **加分**：用 `-fsanitize=address,undefined` 编译并跑模糊测试（随机字节流跑 1 亿字节不崩溃）。
## 3. 项目 2 · 串口抓包与可视化
- **目标**：亲眼看到真实的 ELRS 数据流；
- **步骤**：
	1. USB-TTL 的 RX 接接收机 TX，共地，运行 `crsf_sniff.py`；
	2. 扩展脚本：用 matplotlib 实时画出 16 路通道条形图和 LQ / RSSI 曲线；
	3. 统计每秒各类帧的数量（RC 帧率、链路统计帧率）；
- **验收**：拨动摇杆时曲线实时变化；切换包率后 RC 帧率随之变化；遮挡天线或拉远距离时 LQ / RSSI 下降；
- **思考**：RC 帧率是否严格等于包率？链路统计多久发一次？丢包时 RC 帧会不会停？
## 4. 项目 3 · CRSF 转 PWM 舵机输出
- **目标**：做一个"CRSF → PWM"的小转换板；
- **步骤**：
	1. 按模块 06 用 UART DMA + IDLE 接收，解析 RC 帧；
	2. 用定时器输出 4 路 50 Hz 舵机 PWM，脉宽 = `CRSF_TICKS_TO_US(ch)`，限制在 1000–2000 µs；
	3. 实现失控：超过 1 s 无有效 RC 帧时输出预设位置；恢复后自动跟随；
	4. 收到 RC 帧后立即更新比较寄存器，不要等主循环；
- **验收**：舵机平滑跟随摇杆；关掉遥控器约 1 s 后舵机回到失控位置；用逻辑分析仪测"RC 帧结束 → PWM 更新"的延迟；
- **加分**：输出 DShot / OneShot；用 CH15（LQ）驱动一个 LED 指示信号质量。
## 5. 项目 4 · 遥测传感器节点
- **目标**：让遥控器屏幕显示你的数据；
- **步骤**：
	1. MCU 扮演"飞控"（地址 0xC8）接在接收机上，双向接线；
	2. ADC 经分压电阻测电池电压 → 0x08（0.1 V 单位）；IMU（如 MPU6050）解算姿态 → 0x1E（100 µrad 单位）；状态字符串 → 0x21；
	3. 按模块 06 第 6 节轮转调度，每类约 10 Hz；
- **验收**：EdgeTX "发现传感器"后出现 RxBt、Ptch、Roll、Yaw、FM，数值与万用表 / 实际姿态一致；把遥测比从 1:2 改到 1:128，观察刷新变慢；
- **实验**：把一个每 100 ms 加 1 的计数器作为 0x21 字符串发送，在低遥测比下观察遥控器上的数字是"跳着走"还是"逐个排队"，从而判断接收机对超出带宽的遥测是丢弃旧值还是排队
## 6. 项目 5 · 可用 Lua 配置的 CRSF 设备
- **目标**：实现完整的设备发现与参数协议；
- **步骤**：
	1. 响应 0x28 → 0x29，`param_count = 3`；
	2. 参数树：参数 0 为 FOLDER "ROOT"；参数 1 为 FLOAT "LED Brightness"（0–100 %，步长 1）；参数 2 为 TEXT_SELECTION "Mode"（"Off;Blink;On"）；参数 3 为 COMMAND "Save"；
	3. 支持 0x2C 分块读取（每块最多 56 字节）、0x2D 写入、COMMAND 状态机（START → PROGRESS → READY）；参数存 Flash；
- **验收**：
	1. 先用 PC 脚本扮演遥控器（发送 0x28 / 0x2C / 0x2D 并打印应答），离线跑通全部流程；
	2. 接入真实系统，在 ELRS Lua 的 Other Devices 中看到设备、修改参数，设备即时响应；
	3. 断电重启后参数保留；
- **加分**：让条目超过 56 字节以测试多块读取；故意丢弃某一块，验证 Host 重读同一块。
## 7. 拓展方向
- CRSF → SBUS / PPM 转换器（注意 SBUS 为 100000 baud、8E2、反相）；
- CRSF → USB HID 游戏手柄：把接收机输出接到电脑，用于飞行模拟器；
- MSP over CRSF（0x7A / 0x7B）：从遥控器端读取飞控版本、调整 VTX；
- 遥控器侧：用 MCU 驱动 ELRS TX 模块（单线半双工、主机节拍、0x3A.0x10 同步）；
- 双接收机冗余：两路 CRSF 输入，按 LQ 和帧到达情况自动切换。
## 8. 自测题（点开看答案）
<details>
<summary>Q1 一帧以 `C8 18 16` 开头，整帧多少字节？负载多少字节？</summary>

答：Len = 0x18 = 24，整帧 = Len + 2 = **26 字节**；负载 = Len − 2（去掉 Type 和 CRC）= **22 字节**，正好是 16 × 11 bit。

</details>
<details>
<summary>Q2 CRC8 的多项式、初值和覆盖范围是什么？0x32 命令帧有什么特别之处？</summary>

答：多项式 **0xD5**（CRC-8/DVB-S2），初值 0，覆盖 **Type 到 Payload 末尾**，不含 Sync、Len 和 CRC 本身。0x32 命令帧在外层 CRC 之前还有一个**内层 CRC8**，多项式 **0xBA**，覆盖 0x32、Dest、Orig、Command ID 和命令负载。

</details>
<details>
<summary>Q3 为什么 RC 通道负载是 22 字节？</summary>

答：16 路 × 11 bit = 176 bit = 22 字节，按 LSB 优先连续打包，没有填充位。

</details>
<details>
<summary>Q4 通道值 172 / 992 / 1811 分别对应多少 µs？换算公式是什么？</summary>

答：约 **988 / 1500 / 2012 µs**。µs = (ticks − 992) × 5 / 8 + 1500；反过来 ticks = (µs − 1500) × 8 / 5 + 992。1811 精确换算为 2011.875 µs，整数运算截断得 2011。

</details>
<details>
<summary>Q5 广播帧和扩展帧有什么区别？</summary>

答：Type 小于 0x28 的是广播帧，结构为 `[Sync][Len][Type][Payload][CRC]`，没有地址，Len = Payload + 2；Type ≥ 0x28 的是扩展帧，在 Type 后多了 Dest、Orig 两个地址字节，Len = Payload + 4，应答时交换 Dest / Orig。少数 Type ≥ 0x28 的帧（0x34、0x80–0x82、0xAA 等）例外，使用短帧头。

</details>
<details>
<summary>Q6 400000、416666、420000 三个波特率分别用在哪里？</summary>

答：**400000** 是 TBS 规定的遥控器 ⇄ TX 模块默认波特率；**416666** 是 TBS 规范中接收机 ⇄ 飞控的波特率；**420000** 是 ExpressLRS 接收机 ⇄ 飞控的默认波特率。416666 与 420000 相差约 0.8%，可以互通。

</details>
<details>
<summary>Q7 为什么 400K 波特率下 ELRS 外置模块最高只能用 250 Hz？</summary>

答：遥控器 ⇄ TX 是单线半双工，一个包周期内要容纳遥控器和模块各一个最大 64 字节帧：2 × 64 × 10 bit = 1280 bit，400 kbaud 下需要 3.2 ms，对应约 312 Hz。250 Hz（4 ms）放得下，333 Hz（3 ms）就放不下了，所以更高包率要提高波特率。

</details>
<details>
<summary>Q8 0x14 链路统计中 Uplink RSSI Ant.1 = 0x50，表示多少？</summary>

答：0x50 = 80，RSSI 字段存的是 dBm 的相反数，所以是 **−80 dBm**。

</details>
<details>
<summary>Q9 ELRS 4.x 下收到 Len = 0x19 的 0x16 帧，解析器应该怎么做？</summary>

答：正常接受。只要求 Len ≥ 0x18，按前 22 字节解出 16 路通道；多出来的 1 字节（可选的 armStatus）忽略或按需解析。要求 Len 严格等于 0x18 的实现会在升级后"收不到通道"。

</details>
<details>
<summary>Q10 如何从 0x29 Device Info 判断对方是 ExpressLRS 设备？</summary>

答：看 Serial_number 字段：ExpressLRS 固定为 **0x454C5253**（ASCII "ELRS"），ELRS Lua 也据此识别。

</details>
<details>
<summary>Q11 一个参数条目太长、一帧装不下怎么办？</summary>

答：分块传输。设备的 0x2B 中有"剩余块数"，每块最多 56 字节（58 − 参数号 − 剩余块数）；Host 总是先读 chunk 0，剩余块数大于 0 就用 0x2C 继续读同一参数的下一块，直到剩余块数为 0，再按顺序拼接；丢块时重读该块。

</details>
<details>
<summary>Q12 COMMAND 类型参数（如 Bind）的执行流程是怎样的？</summary>

答：Host 用 0x2D 写入 **START** → 设备回复 0x2B，状态为 **PROGRESS**（需要确认时为 CONFIRMATION_NEEDED，Host 再写 CONFIRM 或 CANCEL）→ 执行期间 Host 发 **POLL** 获取最新状态 → 完成后设备回到 **READY**。读取 TX 模块的命令时必须 POLL，否则拿不到最新状态。

</details>
<details>
<summary>Q13 接收机停止输出 RC 帧后，飞控应该怎么做？</summary>

答：把"有效 RC 帧停止到达"作为失控判据。规范说明接收机失控（cut）时会停发 0x16，建议飞控等待约 **1 s** 再启动自己的 failsafe 流程；不要依赖 LQ = 0 或某个通道值。恢复后应要求重新解锁。

</details>
<details>
<summary>Q14 为什么 ELRS 要求把解锁开关放在 AUX1（CH5）？</summary>

答：ELRS 在每个包里都发送 CH5 并把它当作解锁状态，用来启用安全与性能相关的行为（例如 Race 遥测模式在解锁后停发遥测）。约 1000 µs 为未解锁、约 2000 µs 为已解锁；3 段开关要在 EdgeTX 混控里重映射成 2 段。

</details>
<details>
<summary>Q15 50 Hz 包率、遥测比 1:128 时，下行遥测带宽大约多少？说明什么？</summary>

答：大约 **8 bps**，只够偶尔刷新少量传感器；MSP 与飞控遥测共享这一点带宽，基本无法调参。需要更多遥测时提高包率或遥测比（如 250 Hz、1:2 约 2500 bps）。

</details>
评分参考：答对 12 题以上，说明已掌握协议核心，可以挑战项目 4、5；答对 8 题以下，建议重读 02、03、05。
> **本章要点**
> - 先在 PC 上把解析与编解码做对（项目 1），再上硬件，调试成本最低
> - 每个项目都有明确的验收标准：能测量、能复现，才算真正完成
> - 项目 3 练接收与失控，项目 4 练遥测上行，项目 5 练扩展帧与参数协议，覆盖 CRSF 的全部核心能力
> - 自测题覆盖帧结构、CRC、通道换算、波特率、链路统计、ELRS 约定与参数协议；答错的题回到对应章节复习
📖 出处：[TBS CRSF 规范](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [ELRS 配置飞控](https://www.expresslrs.org/quick-start/receivers/configuring-fc) · [ELRS 开关模式](https://www.expresslrs.org/software/switch-config/) · [ELRS 遥测带宽](https://www.expresslrs.org/info/telem-bandwidth/) · [ELRS Lua 说明](https://www.expresslrs.org/quick-start/transmitters/lua-howto/)
