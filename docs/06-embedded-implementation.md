# 🧩 06 嵌入式实现指南（含 C 参考代码）

> **本章摘要**：把前几章的协议知识落到 MCU 上：分层架构 → UART 配置 → DMA 循环 + IDLE 接收 → 流式解析器（Sync 过滤、Len 检查、CRC、超时与重同步）→ 失控处理 → 遥测调度 → 设备侧 Ping 应答 → 遥控器侧半双工要点。配套的 C99 参考实现 **crsf-mini**（无动态内存、无平台依赖、63 项单元测试）可直接移植到 STM32 / ESP32 / RP2040 等平台，完整源码见第 9 节。
## 1. 分层架构
```text
应用层   通道映射、失控逻辑、解锁判断、遥测数据来源、参数菜单
   ^ v
协议层   crsf_decode_*() / crsf_encode_*()：字段解析与单位换算
   ^ v
帧层     crsf_parser_feed()：Sync 过滤 -> Len 检查 -> CRC -> 回调
   ^ v
驱动层   UART + DMA 循环缓冲 + IDLE 事件 + 微秒时间戳 + DMA 发送
```
| 层 | 职责 | 与硬件相关 | 能否在 PC 上单测 |
| --- | --- | --- | --- |
| 驱动层 | 收发字节、打时间戳 | 是 | 否（用录制数据或测试向量代替） |
| 帧层 | 组帧、校验、超时、重同步 | 否 | 能 |
| 协议层 | 各帧字段编解码、单位换算 | 否 | 能 |
| 应用层 | 失控、解锁、输出、遥测来源 | 部分 | 部分 |
设计原则：**只有驱动层碰硬件**。帧层和协议层写成纯 C，先在 PC 上用模块 07 的测试向量跑通，再移植到 MCU——绝大多数 CRSF 问题都能在 PC 上提前发现。
## 2. UART 配置
| 角色 | 波特率 | 格式 | 电平 / 极性 | 双工与接线 |
| --- | --- | --- | --- | --- |
| MCU 作为飞控，接接收机 | 420000（ELRS 默认）或 416666（TBS 规范） | 8N1 | 3.3 V，**不反相** | 全双工：RX 的 TX → MCU 的 RX，RX 的 RX ← MCU 的 TX，共地 |
| MCU 作为遥控器，接 TX 模块 | 400000 起，ELRS 支持更高 | 8N1 | 3.3 V；部分外置模块口为反相 | 单线半双工，MCU 是主机 |
- 416666 与 420000 相差约 0.8%，在 UART 容差内可以互通（两端误差之和一般要控制在约 2% 以内）；
- 核对 MCU 实际能生成的波特率：例如 STM32 的 USART 时钟为 84 MHz 时 84 MHz ÷ 420000 = 200，误差为 0；72 MHz 时实际约 421053 baud（+0.25%）。CubeMX 的参数页会直接显示误差；
- 必须用**硬件 UART**：420k 下每字节只有约 23.8 µs，软件串口不可靠；
- CRSF 的 RX ⇄ FC 段是**正常极性**，不要照搬 SBUS 的反相配置；
- 只接收通道时可以只接 RX 的 TX → MCU 的 RX 一根线；要回传遥测或 MSP 就必须双向接线。
## 3. 接收：DMA 循环模式 + IDLE 事件
为什么不用逐字节中断：420k 下 26 字节的 RC 帧约 619 µs；ELRS 在 F1000 包率下每 1 ms 一帧，逐字节中断约 2.6 万次/秒。DMA 循环接收 + 空闲线（IDLE）事件则**每帧只进一次回调**，回调稍有延迟也不丢数据。帧长可变，所以也不能用"固定长度的普通 DMA"。
```text
UART RX --DMA(circular)--> rx_dma[128] --IDLE / HT / TC event--> HAL_UARTEx_RxEventCallback(pos)
                                                                   |
                         crsf_parser_feed(new bytes old..pos) <----+
                                       |
                                       v
                                  on_frame()  (valid CRC only)
```
1. CubeMX：USART 异步 420000 8N1，DMA RX 设为 **Circular**，打开 USART 全局中断；调用一次 `HAL_UARTEx_ReceiveToIdle_DMA(&huart1, rx_dma, 128)` 即可持续接收；
2. HAL 在 IDLE、DMA 半满（HT）、全满（TC）时都会调用 `HAL_UARTEx_RxEventCallback(huart, pos)`，`pos` 是 DMA 当前写到的位置；
3. 回调里只处理上次位置 `rx_old` 到 `pos` 之间的新数据；`pos` 小于 `rx_old` 说明绕回，分两段喂给解析器；
4. 缓冲区至少放得下 2 个最大帧（128 B）；
5. **HAL 的坑**：开启 DMA 接收后，任何 UART 错误（噪声、帧错误、溢出，常见于热插拔接收机时）都会让 HAL 中止接收。必须在 `HAL_UART_ErrorCallback()` 里重新启动，否则 CRSF 输入会永久停止；
6. 时间戳：TIM2 配成 32 bit、1 MHz 自由计数（预分频 = 定时器时钟 ÷ 1 MHz − 1），或用 `DWT->CYCCNT`；比较时间时用无符号减法，自然处理溢出。
```c
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t pos)
{
    uint32_t now = __HAL_TIM_GET_COUNTER(&htim2);   /* 1 MHz timestamp */
    if (huart != &huart1 || pos == rx_old) return;
    if (pos > rx_old) {
        crsf_parser_feed(&parser, &rx_dma[rx_old], (size_t)(pos - rx_old), now);
    } else {                                        /* wrapped around */
        crsf_parser_feed(&parser, &rx_dma[rx_old], (size_t)(RX_DMA_SIZE - rx_old), now);
        crsf_parser_feed(&parser, &rx_dma[0], pos, now);
    }
    rx_old = (pos == RX_DMA_SIZE) ? 0u : pos;
}

void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart)   /* restart after errors */
{
    if (huart != &huart1 || huart->RxState != HAL_UART_STATE_READY) return;
    rx_old = 0;
    HAL_UARTEx_ReceiveToIdle_DMA(&huart1, rx_dma, RX_DMA_SIZE);
}
```
其他平台：
| 平台 | 推荐做法 |
| --- | --- |
| ESP32（Arduino） | `Serial1.begin(420000, SERIAL_8N1, rx, tx)`，在 `loop()` 中读取；驱动自带缓冲，到达时间不精确，可设 `parser.timeout_us = 0`，依靠 CRC + 重同步；Arduino-ESP32 2.x 起也可用 `onReceive()` / `setRxTimeout()` 实现类似 IDLE 的回调 |
| ESP32（ESP-IDF） | `uart_driver_install()` 配合事件队列，用 `uart_set_rx_timeout()` 把接收超时设为 1–2 个字符时间 |
| RP2040 | PL011 UART 的接收超时中断（FIFO 非空且约 32 bit 时间无新数据触发）作用与 IDLE 相同，也可配合 DMA |
| 无 DMA 的小 MCU | 逐字节中断写入环形缓冲区，主循环取出喂给解析器；时间戳在中断里记录 |
## 4. 解析器：四道关卡 + 重同步
```text
WAIT_SYNC ---> READ_LEN ---> READ_BODY ---> CHECK_CRC ---> on_frame()
    ^             | bad Len      | timeout      | bad CRC
    +-------------+--------------+--------------+
        drop 1 byte and rescan from the next byte (timeout: salvage first)
```
1. **Sync 过滤**：只把"可接受的 Sync"当作帧头。默认 0xC8；规范要求接收方接受 0xC8、0x00 以及自身地址，用 `crsf_parser_accept()` 添加（遥控器侧加 0xEA）；
2. **Len 检查**：Len 必须在 2–62 之间，否则 `len_errors++`；
3. **收齐整帧**：总长 = Len + 2；
4. **CRC 校验**：CRC8（0xD5）覆盖 Type 到 Payload 末尾；不对则 `crc_errors++`；
5. **重同步**：出错时**只丢 1 字节**，从下一个字节重新找 Sync——被"假 Sync"（负载里恰好出现 0xC8）吞掉的真帧还在缓冲区里，可以找回；
6. **帧内超时**：从帧头起超过"最大帧时长 + 15%"（64 字节 × 10 bit ÷ 420000 × 1.15 ≈ 1752 µs，Betaflight 取约 1.75 ms）仍未收齐，`timeouts++`，先抢救缓冲区里可能存在的完整帧，再丢弃残帧。`timeout_us = 0` 可关闭；
7. **统计**：`frames_ok`、`crc_errors`、`len_errors`、`timeouts`，调试时每秒打印一次。
回调规则：
- `frame` 指针**只在回调期间有效**，要保留的数据必须拷贝出来；
- 回调可能运行在**中断上下文**（在 DMA / IDLE 回调里调用 feed 时），只做解码、拷贝、置标志；不要 printf、不要阻塞发送；
- RC 帧只接受 `frame[0] == 0xC8` 的（与 Betaflight 一致）；
- 未知 Type 直接忽略；帧比预期长时只解析已知字段（例如 ELRS 4.x 的 0x16 可能多 1 字节，只要求 Payload ≥ 22）；帧比预期短时拒绝解码。
最小用法：
```c
#include "crsf.h"

static crsf_parser_t rx;
static volatile uint16_t ch_raw[CRSF_NUM_CHANNELS];
static volatile uint32_t rc_last_ms;
static crsf_link_stats_t link;

static void on_frame(void *user, const uint8_t *f, uint8_t len)
{
    uint16_t ch[CRSF_NUM_CHANNELS];
    (void)user; (void)len;
    switch (crsf_frame_type(f)) {
    case CRSF_TYPE_RC_CHANNELS:
        if (f[0] == CRSF_ADDR_FC && crsf_decode_rc_channels(f, ch)) {
            for (int i = 0; i < CRSF_NUM_CHANNELS; i++) ch_raw[i] = ch[i];
            rc_last_ms = millis_now();
        }
        break;
    case CRSF_TYPE_LINK_STATS:
        crsf_decode_link_stats(f, &link);     /* LQ, RSSI, SNR, RF mode, power */
        break;
    case CRSF_TYPE_PING:                      /* extended frame, see section 7 */
        on_ping(crsf_ext_dest(f), crsf_ext_orig(f));
        break;
    default:
        break;                                /* unknown type: ignore */
    }
}

void crsf_setup(void)
{
    crsf_parser_init(&rx, CRSF_BAUD_ELRS_RX, on_frame, NULL); /* accepts sync 0xC8 */
    crsf_parser_accept(&rx, CRSF_ADDR_BROADCAST);             /* spec: also accept 0x00 */
}

/* called by the driver layer for every chunk of received bytes */
void crsf_on_bytes(const uint8_t *d, size_t n, uint32_t now_us)
{
    crsf_parser_feed(&rx, d, n, now_us);
}

/* once per second while debugging: growing crc_errors -> check baud, wiring, noise */
void crsf_print_stats(void)
{
    printf("ok=%lu crc=%lu len=%lu tmo=%lu\n",
           (unsigned long)rx.frames_ok, (unsigned long)rx.crc_errors,
           (unsigned long)rx.len_errors, (unsigned long)rx.timeouts);
}
```
## 5. 失控处理（Failsafe）
- 判据只有一个：**最后一帧有效 RC 帧的时间**。接收机失控时会停发 0x16，规范建议飞控等待约 1 s 再启动自己的 failsafe 流程；
- 不要用 LQ = 0、某个通道值或 RSSI 判断失控——它们只适合用来告警；
- 从失控恢复时，如果解锁开关仍在解锁位，应要求先拨回再重新解锁，防止信号恢复瞬间电机突然转起来；
- 中断里写、主循环里读的多字节数据（16 路通道）要用"关中断拷贝"或双缓冲，避免读到半新半旧的数据。
```c
#define RC_TIMEOUT_MS  1000u   /* spec suggests ~1 s; small devices may use 250-500 ms */

void app_control_1khz(void)
{
    static bool was_lost = true;
    uint16_t us[CRSF_NUM_CHANNELS];

    if (crsf_rc_lost(RC_TIMEOUT_MS)) {     /* no valid RC frame for too long */
        was_lost = true;
        output_failsafe();                 /* stop motors / center servos / land */
        return;
    }
    rc_snapshot(us);                       /* copy with IRQs disabled, ticks -> us */
    bool arm_sw = us[4] > 1700;            /* CH5 = AUX1, the ELRS arm channel */
    if (was_lost && arm_sw) {              /* link is back but still armed:      */
        output_failsafe();                 /* require the switch to be cycled    */
        return;
    }
    was_lost = false;
    output_apply(us, arm_sw);
}
```
## 6. 遥测调度（MCU → 接收机）
- RX ⇄ FC 是全双工，MCU 随时可以发送，不需要等 RC 帧；回传帧的 Sync 用 0xC8（Betaflight 同样如此）；
- 发送缓冲区在 DMA 发送完成前必须保持有效 → 用 `static`，并在发送前检查 UART 是否空闲；
- 下行带宽由 ELRS 的包率和遥测比决定（50 Hz、1:128 时只有约 8 bps），接收机按下行带宽转发，发得再快也只是浪费串口时间；参考 Betaflight 以 100 ms 为周期轮流发送各类遥测，每类约 10 Hz 即可；
- 在 EdgeTX 中"发现传感器"后可以看到 RxBt / Curr / Capa / Bat%（电池）、Ptch / Roll / Yaw（姿态）、FM（飞行模式）、VSpd（垂直速度）、GPS / GSpd / Hdg / Alt / Sats 等。
```c
/* call every 20 ms: 5 slots in rotation -> each telemetry type at about 10 Hz */
void crsf_telemetry_task(void)
{
    static uint8_t tx[CRSF_FRAME_SIZE_MAX];      /* must stay valid during DMA */
    static uint8_t slot;
    uint8_t n = 0;

    if (uart_tx_busy()) return;                  /* previous frame still sending */

    switch (slot) {
    case 0: {
        crsf_battery_t b = { 168, 123, 1234, 56 };   /* 16.8 V, 12.3 A, 1234 mAh, 56 % */
        n = crsf_encode_battery(tx, &b);
        break;
    }
    case 1:   /* unit 100 urad: degree * 174.53, yaw converted to -180..180 first */
        n = crsf_encode_attitude(tx, (int16_t)(pitch_deg * 174.53f),
                                     (int16_t)(roll_deg * 174.53f),
                                     (int16_t)(yaw_deg * 174.53f));
        break;
    case 2:   /* Betaflight style: "!FS!" in failsafe, '*' appended while disarmed */
        n = crsf_encode_flight_mode(tx, failsafe ? "!FS!" : (armed ? "ACRO" : "ACRO*"));
        break;
    case 3:
        n = crsf_encode_vario(tx, vspeed_cms);       /* vertical speed, cm/s */
        break;
    default: {
        /* 31.1234567 N, 121.456789 E, 52.3 km/h, heading 90.00 deg, 15 m, 12 sats */
        crsf_gps_t g = { 311234567, 1214567890, 523, 9000, 15, 12 };
        n = crsf_encode_gps(tx, &g);
        break;
    }
    }
    slot = (uint8_t)((slot + 1u) % 5u);
    if (n) uart_send_dma(tx, n);
}
```
上面电池帧的编码结果应为 `C8 0A 08 00 A8 00 7B 00 04 D2 38 09`（单位按 0.1 V / 0.1 A），可用来自测。
## 7. 设备侧：响应 Ping、上报 Device Info
遥控器端的配置脚本（ELRS Lua 的 Other Devices 等）用广播 Ping 发现设备。你的 MCU 想被"看见"，就要对 Dest 为自身地址或 0x00 的 0x28 回复 0x29，并且**交换 Dest / Orig**：
```c
static volatile bool    ping_pending;
static volatile uint8_t ping_from;

/* from on_frame() (maybe ISR context): only remember the request */
void on_ping(uint8_t dest, uint8_t orig)
{
    if (dest == CRSF_ADDR_FC || dest == CRSF_ADDR_BROADCAST) {
        ping_from = orig;                 /* usually the handset, 0xEA */
        ping_pending = true;
    }
}

/* main loop: answer with 0x29, Dest = requester, Orig = ourselves */
void crsf_device_task(void)
{
    static uint8_t tx[CRSF_FRAME_SIZE_MAX];
    crsf_device_info_t info;
    uint8_t n;

    if (!ping_pending || uart_tx_busy()) return;
    memset(&info, 0, sizeof info);
    strncpy(info.name, "MY CRSF NODE", sizeof info.name - 1u);
    info.serial        = 0x4D594E44u;    /* own serial; do not reuse ELRS 0x454C5253 */
    info.hw_id         = 0x00000001u;
    info.fw_id         = 0x00010000u;
    info.param_count   = 0;              /* no configurable parameters (yet) */
    info.param_version = 0;
    n = crsf_encode_device_info(tx, CRSF_SYNC_BYTE, ping_from, CRSF_ADDR_FC, &info);
    if (n) uart_send_dma(tx, n);
    ping_pending = false;
}
```
自测：用 USB-TTL 向 MCU 发送广播 Ping `C8 04 28 00 EA 54`，应收到 `C8 1F 29 EA C8 4D 59 20 43 52 53 46 20 4E 4F 44 45 00 4D 59 4E 44 00 00 00 01 00 01 00 00 00 00 11`。`param_count` 大于 0 时，还要实现 0x2C 读参数 → 0x2B 分块应答、0x2D 写参数（见模块 04）。
## 8. 遥控器侧：单线半双工要点
如果 MCU 扮演"遥控器"驱动 TX 模块（自制遥控器、地面站、模拟器转接）：
- **半双工 UART**：STM32 用 `HAL_HalfDuplex_Init()`，发送前 `HAL_HalfDuplex_EnableTransmitter()`，发送完成（TC 标志，而不是 TXE）后立刻 `HAL_HalfDuplex_EnableReceiver()`；信号线需要上拉；
- **回声**：部分接法（TX、RX 用电阻并到一根线上）会收到自己发出的字节，要丢弃；
- **主机节拍**：MCU 定时发送 0x16（周期与模块包率一致，例如 250 Hz 每 4 ms 一帧），模块在帧后的间隙回复链路统计、遥测、参数应答；
- **地址**：解析器加 `crsf_parser_accept(&p, CRSF_ADDR_RADIO)`，模块发给遥控器的帧以 0xEA 开头；
- **同步（可选）**：模块可用 0x3A.0x10 告诉遥控器包周期和相位偏差，遥控器据此微调发送时刻以降低抖动（见模块 04）；
- **极性**：外置模块口常为反相。STM32F3 / F7 / G4 / H7 等的 USART 支持引脚电平反转（TXINV / RXINV），F1 / F4 不支持，需要外部反相电路；
- **波特率**：从 400000 起步；ELRS TX 失去连接后会轮换尝试多种波特率自动识别（见模块 05）。
## 9. 📦 参考实现 crsf-mini（完整源码）
> **crsf-mini 源码包**：`crsf.h` / `crsf.c`（库本体）、`test_crsf.c`（63 项单元测试 + 测试向量）、`examples/`（STM32 HAL DMA、ESP32 Arduino）、`Makefile`、`README.md`。在 PC 上执行 `make test` 即可验证；移植时只需把 `crsf.h` / `crsf.c` 加入工程。
[下载 crsf-mini 源码包](../assets/crsf-mini.zip)
crsf-mini.zip — C99 参考实现源码包（MIT License）
<details>
<summary>📄 crsf.h — 常量、结构体与 API</summary>

```c
/**
 * crsf.h - Minimal, portable CRSF (Crossfire Serial Protocol) library
 *
 * Reference implementation for learning / prototyping. Based on:
 *   - TBS CRSF Protocol Specification  https://github.com/tbs-fpv/tbs-crsf-spec
 *   - Behaviour of ExpressLRS / Betaflight (420000 baud RX<->FC, 0.1 V battery units, ...)
 *
 * Features
 *   - C99, no malloc, no OS/HAL dependency, ISR friendly
 *   - stream parser: sync filter + length check + CRC8 + timeout + resync
 *   - pack/unpack of the 16 x 11 bit RC channels
 *   - encoders/decoders for common telemetry and extended frames
 *
 * Frame layout
 *   broadcast : [sync][len][type][payload ...][crc]
 *   extended  : [sync][len][type][dest][orig][payload ...][crc]
 *   len = number of bytes after the len field (type + payload + crc), valid 2..62
 *   crc = CRC-8, poly 0xD5, init 0, over type + payload (NOT sync, NOT len)
 *   multi-byte fields are big endian (only the RC channel bit packing is LSB first)
 *
 * License: MIT
 */
#ifndef CRSF_H
#define CRSF_H

#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

/* ------------------------------------------------------------------ limits */
#define CRSF_FRAME_SIZE_MAX      64u  /* whole frame, sync .. crc             */
#define CRSF_LEN_MIN              2u  /* len field minimum: type + crc        */
#define CRSF_LEN_MAX             62u  /* len field maximum: 64 - sync - len   */
#define CRSF_PAYLOAD_MAX         60u  /* broadcast payload (64 - 4)           */
#define CRSF_EXT_PAYLOAD_MAX     58u  /* extended payload  (64 - 6)           */

/* -------------------------------------------------------------- baud rates */
#define CRSF_BAUD_SPEC_RX_FC  416666u /* TBS spec, RX <-> FC                  */
#define CRSF_BAUD_ELRS_RX     420000u /* ExpressLRS default, RX <-> FC        */
#define CRSF_BAUD_HANDSET     400000u /* TBS default, handset <-> TX module   */

/* --------------------------------------------------------------- addresses */
#define CRSF_ADDR_BROADCAST    0x00u
#define CRSF_ADDR_USB          0x10u
#define CRSF_ADDR_BT_WIFI      0x12u
#define CRSF_ADDR_OSD          0x80u
#define CRSF_ADDR_ESC_1        0x90u
#define CRSF_ADDR_CURRENT      0xC0u  /* voltage / current sensor            */
#define CRSF_ADDR_GPS          0xC2u
#define CRSF_ADDR_BLACKBOX     0xC4u
#define CRSF_ADDR_FC           0xC8u  /* flight controller                   */
#define CRSF_ADDR_VTX          0xCEu
#define CRSF_ADDR_RADIO        0xEAu  /* remote control / handset            */
#define CRSF_ADDR_RX           0xECu  /* receiver                            */
#define CRSF_ADDR_TX           0xEEu  /* transmitter module                  */
#define CRSF_SYNC_BYTE         0xC8u  /* default sync byte                   */

/* ------------------------------------------------------------- frame types */
/* broadcast frames: no dest / orig */
#define CRSF_TYPE_GPS          0x02u
#define CRSF_TYPE_VARIO        0x07u
#define CRSF_TYPE_BATTERY      0x08u
#define CRSF_TYPE_BARO_ALT     0x09u
#define CRSF_TYPE_HEARTBEAT    0x0Bu
#define CRSF_TYPE_LINK_STATS   0x14u
#define CRSF_TYPE_RC_CHANNELS  0x16u
#define CRSF_TYPE_SUBSET_RC    0x17u
#define CRSF_TYPE_LINK_RX_ID   0x1Cu
#define CRSF_TYPE_LINK_TX_ID   0x1Du
#define CRSF_TYPE_ATTITUDE     0x1Eu
#define CRSF_TYPE_FLIGHT_MODE  0x21u
/* extended frames: [dest][orig] follow the type byte */
#define CRSF_TYPE_PING         0x28u
#define CRSF_TYPE_DEVICE_INFO  0x29u
#define CRSF_TYPE_PARAM_ENTRY  0x2Bu
#define CRSF_TYPE_PARAM_READ   0x2Cu
#define CRSF_TYPE_PARAM_WRITE  0x2Du
#define CRSF_TYPE_COMMAND      0x32u
#define CRSF_TYPE_RADIO_ID     0x3Au
#define CRSF_TYPE_MSP_REQ      0x7Au
#define CRSF_TYPE_MSP_RESP     0x7Bu

/* Rule of thumb from the spec: types >= 0x28 are extended (carry dest/orig).
 * Known exceptions with the short (broadcast) header: 0x34 logging,
 * 0x80 ArduPilot passthrough, 0x81/0x82 mLRS, 0xAA MAVLink envelope.
 * Always check the spec entry of other vendor specific types. */
static inline bool crsf_is_extended(uint8_t t)
{
    return t >= 0x28u && t != 0x34u && (t < 0x80u || t > 0x82u) && t != 0xAAu;
}
#define CRSF_IS_EXTENDED(type) crsf_is_extended((uint8_t)(type))

/* -------------------------------------------- 0x32 command ids (subset) */
#define CRSF_CMD_FC            0x01u
#define CRSF_CMD_OSD           0x05u
#define CRSF_CMD_VTX           0x08u
#define CRSF_CMD_LED           0x09u
#define CRSF_CMD_GENERAL       0x0Au  /* 0x70 speed proposal / 0x71 response */
#define CRSF_CMD_CROSSFIRE     0x10u  /* 0x01 bind, 0x02 cancel bind, ...    */
#define CRSF_CMD_ACK           0xFFu

/* ------------------------------------------------------------ RC channels */
#define CRSF_NUM_CHANNELS      16u
#define CRSF_RC_PAYLOAD_SIZE   22u    /* 16 x 11 bit                          */
#define CRSF_CH_MIN            172u   /*  988 us (ExpressLRS / Betaflight)    */
#define CRSF_CH_MID            992u   /* 1500 us                              */
#define CRSF_CH_MAX            1811u  /* 2012 us                              */
/* conversions as given in the TBS spec (integer math, truncates toward 0) */
#define CRSF_TICKS_TO_US(x)  (((((int32_t)(x)) - 992) * 5) / 8 + 1500)
#define CRSF_US_TO_TICKS(x)  (((((int32_t)(x)) - 1500) * 8) / 5 + 992)

/* ---------------------------------------------------------------- structs */
typedef struct {
    uint8_t up_rssi_ant1;   /* uplink RSSI antenna 1, dBm * -1 (60 = -60 dBm) */
    uint8_t up_rssi_ant2;   /* uplink RSSI antenna 2, dBm * -1                */
    uint8_t up_lq;          /* uplink link quality, 0..100 %                  */
    int8_t  up_snr;         /* uplink SNR, dB                                 */
    uint8_t active_antenna; /* 0 = antenna 1, 1 = antenna 2                   */
    uint8_t rf_profile;     /* rate / mode index - meaning is vendor specific */
    uint8_t up_rf_power;    /* TX power index, see crsf_rf_power_mw()         */
    uint8_t down_rssi;      /* downlink (telemetry) RSSI, dBm * -1            */
    uint8_t down_lq;        /* downlink link quality, %                       */
    int8_t  down_snr;       /* downlink SNR, dB                               */
} crsf_link_stats_t;

typedef struct {
    uint16_t voltage_dv;    /* 0.1 V (Betaflight / ExpressLRS / EdgeTX usage) */
    uint16_t current_da;    /* 0.1 A                                          */
    uint32_t capacity_mah;  /* used capacity, 24 bit on the wire              */
    uint8_t  remaining;     /* remaining, %                                   */
} crsf_battery_t;

typedef struct {
    int32_t  lat_e7;        /* degree * 1e7                                   */
    int32_t  lon_e7;        /* degree * 1e7                                   */
    uint16_t speed_dkmh;    /* ground speed, 0.1 km/h (Betaflight practice)   */
    uint16_t heading_cdeg;  /* degree * 100                                   */
    int32_t  altitude_m;    /* metres; the encoder adds the +1000 m offset    */
    uint8_t  satellites;
} crsf_gps_t;

#define CRSF_DEVICE_NAME_MAX 44u   /* 58 - 14 fixed bytes, including '\0'   */
typedef struct {
    char     name[CRSF_DEVICE_NAME_MAX];
    uint32_t serial;        /* ExpressLRS reports 0x454C5253 ("ELRS")         */
    uint32_t hw_id;
    uint32_t fw_id;
    uint8_t  param_count;
    uint8_t  param_version;
} crsf_device_info_t;

/* ------------------------------------------------------------------ parser */
/* Called for every complete frame with a valid CRC. 'frame' points to
 * [sync][len][type]...[crc] and is only valid during the callback. */
typedef void (*crsf_frame_cb_t)(void *user, const uint8_t *frame, uint8_t frame_len);

typedef struct {
    uint8_t         buf[CRSF_FRAME_SIZE_MAX];
    uint8_t         pos;
    uint8_t         sync_map[32];   /* bitmap of accepted sync bytes          */
    uint32_t        frame_start_us;
    uint32_t        timeout_us;     /* 0 = disabled                           */
    crsf_frame_cb_t on_frame;
    void           *user;
    /* statistics */
    uint32_t        frames_ok;
    uint32_t        crc_errors;
    uint32_t        len_errors;
    uint32_t        timeouts;
} crsf_parser_t;

/* baud: used to derive the partial-frame timeout (0 -> 420000). Accepts 0xC8
 * as sync byte; add more with crsf_parser_accept() (e.g. 0xEA on a handset,
 * 0xEE on a TX module). */
void crsf_parser_init(crsf_parser_t *p, uint32_t baud, crsf_frame_cb_t cb, void *user);
void crsf_parser_accept(crsf_parser_t *p, uint8_t sync);
/* Feed received bytes. now_us = (ideally) arrival time of this chunk. */
void crsf_parser_feed(crsf_parser_t *p, const uint8_t *data, size_t len, uint32_t now_us);

/* ---------------------------------------------------------- frame helpers */
static inline uint8_t crsf_frame_type(const uint8_t *f)  { return f[2]; }
static inline uint8_t crsf_frame_size(const uint8_t *f)  { return (uint8_t)(f[1] + 2u); }
static inline const uint8_t *crsf_payload(const uint8_t *f) { return &f[3]; }
static inline uint8_t crsf_payload_len(const uint8_t *f)
{ return (uint8_t)(f[1] >= 2u ? f[1] - 2u : 0u); }
static inline uint8_t crsf_ext_dest(const uint8_t *f)    { return f[3]; }
static inline uint8_t crsf_ext_orig(const uint8_t *f)    { return f[4]; }
static inline const uint8_t *crsf_ext_payload(const uint8_t *f) { return &f[5]; }
static inline uint8_t crsf_ext_payload_len(const uint8_t *f)
{ return (uint8_t)(f[1] >= 4u ? f[1] - 4u : 0u); }

/* -------------------------------------------------------------------- CRC */
uint8_t crsf_crc8(const uint8_t *data, size_t len);      /* poly 0xD5          */
uint8_t crsf_crc8_cmd(const uint8_t *data, size_t len);  /* poly 0xBA (0x32)   */
/* true if buf[0..len-1] is exactly one well-formed frame with a valid CRC */
bool    crsf_frame_check(const uint8_t *buf, size_t len);

/* ----------------------------------------------------------- RC channels */
void crsf_pack_channels(const uint16_t ch[16], uint8_t out[22]);
void crsf_unpack_channels(const uint8_t in[22], uint16_t ch[16]);

/* --------------------------------------------- decoders (frame = sync..crc)
 * Return false if the type does not match or the payload is too short.
 * Longer payloads are accepted (extra fields are ignored, as the spec asks). */
bool crsf_decode_rc_channels(const uint8_t *frame, uint16_t ch[16]);
bool crsf_decode_link_stats(const uint8_t *frame, crsf_link_stats_t *ls);
bool crsf_decode_battery(const uint8_t *frame, crsf_battery_t *bat);
bool crsf_decode_device_info(const uint8_t *frame, crsf_device_info_t *info);
uint16_t crsf_rf_power_mw(uint8_t index);   /* 0x14 up_rf_power -> mW */

/* ---------------------------------------- encoders (out >= 64 bytes)
 * Return the total frame size in bytes, or 0 if the payload is too long. */
uint8_t crsf_encode_frame(uint8_t *out, uint8_t sync, uint8_t type,
                          const uint8_t *payload, uint8_t len);
uint8_t crsf_encode_ext_frame(uint8_t *out, uint8_t sync, uint8_t type,
                              uint8_t dest, uint8_t orig,
                              const uint8_t *payload, uint8_t len);
/* 0x32 direct command: payload = sub command + data; adds the inner command CRC */
uint8_t crsf_encode_command(uint8_t *out, uint8_t sync, uint8_t dest, uint8_t orig,
                            uint8_t cmd, const uint8_t *payload, uint8_t len);
uint8_t crsf_encode_rc_channels(uint8_t *out, uint8_t sync, const uint16_t ch[16]);
uint8_t crsf_encode_link_stats(uint8_t *out, const crsf_link_stats_t *ls);
uint8_t crsf_encode_battery(uint8_t *out, const crsf_battery_t *bat);
uint8_t crsf_encode_gps(uint8_t *out, const crsf_gps_t *gps);
uint8_t crsf_encode_vario(uint8_t *out, int16_t vspeed_cms);
/* attitude in 100 urad units (rad * 10000), range +-180 deg */
uint8_t crsf_encode_attitude(uint8_t *out, int16_t pitch, int16_t roll, int16_t yaw);
uint8_t crsf_encode_flight_mode(uint8_t *out, const char *mode);
uint8_t crsf_encode_ping(uint8_t *out, uint8_t sync, uint8_t dest, uint8_t orig);
uint8_t crsf_encode_device_info(uint8_t *out, uint8_t sync, uint8_t dest, uint8_t orig,
                                const crsf_device_info_t *info);
uint8_t crsf_encode_param_read(uint8_t *out, uint8_t sync, uint8_t dest, uint8_t orig,
                               uint8_t param, uint8_t chunk);

#ifdef __cplusplus
}
#endif

#endif /* CRSF_H */
```

</details>
<details>
<summary>📄 crsf.c — CRC、流式解析器、通道打包与帧编解码</summary>

```c
/**
 * crsf.c - Minimal, portable CRSF library (see crsf.h)
 * License: MIT
 */
#include "crsf.h"

#include <string.h>

/* =================================================================== CRC */
static uint8_t crc8_calc(const uint8_t *d, size_t n, uint8_t poly)
{
    uint8_t crc = 0;
    while (n--) {
        uint8_t i;
        crc ^= *d++;
        for (i = 0; i < 8u; i++) {
            if (crc & 0x80u) crc = (uint8_t)((crc << 1) ^ poly);
            else             crc = (uint8_t)(crc << 1);
        }
    }
    return crc;
}

/* CRC-8/DVB-S2. A 256 byte lookup table (see the spec) is faster if needed. */
uint8_t crsf_crc8(const uint8_t *data, size_t len)     { return crc8_calc(data, len, 0xD5u); }
uint8_t crsf_crc8_cmd(const uint8_t *data, size_t len) { return crc8_calc(data, len, 0xBAu); }

bool crsf_frame_check(const uint8_t *f, size_t n)
{
    uint8_t len;
    if (n < 4u) return false;
    len = f[1];
    if (len < CRSF_LEN_MIN || len > CRSF_LEN_MAX || (size_t)len + 2u != n) return false;
    return crsf_crc8(&f[2], (size_t)len - 1u) == f[n - 1u];
}

/* ================================================================ parser */
static bool is_sync(const crsf_parser_t *p, uint8_t b)
{
    return (((unsigned)p->sync_map[b >> 3] >> (b & 7u)) & 1u) != 0u;
}

void crsf_parser_accept(crsf_parser_t *p, uint8_t sync)
{
    p->sync_map[sync >> 3] |= (uint8_t)(1u << (sync & 7u));
}

void crsf_parser_init(crsf_parser_t *p, uint32_t baud, crsf_frame_cb_t cb, void *user)
{
    memset(p, 0, sizeof(*p));
    if (baud == 0u) baud = CRSF_BAUD_ELRS_RX;
    /* air time of a maximum frame (64 bytes * 10 bit, 8N1) + 15 % margin:
     * 420000 baud -> 1752 us, 400000 baud -> 1840 us */
    p->timeout_us = (uint32_t)((uint64_t)CRSF_FRAME_SIZE_MAX * 10u * 1150000u / baud);
    p->on_frame = cb;
    p->user = user;
    crsf_parser_accept(p, CRSF_SYNC_BYTE);
}

static void drop(crsf_parser_t *p, uint8_t n)
{
    if (n >= p->pos) { p->pos = 0; return; }
    memmove(p->buf, p->buf + n, (size_t)(p->pos - n));
    p->pos = (uint8_t)(p->pos - n);
}

/* Extract as many frames as possible from buf[]. Returns when the buffer is
 * empty or holds a plausible but still incomplete frame. On any error only
 * ONE byte is dropped, so a real frame hidden behind a false sync byte is
 * found again (resync without losing data). */
static void scan(crsf_parser_t *p)
{
    while (p->pos > 0u) {
        uint8_t len, total;
        if (!is_sync(p, p->buf[0])) {                 /* hunt for a sync byte */
            uint8_t i = 1;
            while (i < p->pos && !is_sync(p, p->buf[i])) i++;
            drop(p, i);
            continue;
        }
        if (p->pos < 2u) return;                      /* need the len byte   */
        len = p->buf[1];
        if (len < CRSF_LEN_MIN || len > CRSF_LEN_MAX) {
            p->len_errors++;
            drop(p, 1);
            continue;
        }
        total = (uint8_t)(len + 2u);
        if (p->pos < total) return;                   /* need more bytes     */
        if (crsf_crc8(&p->buf[2], (size_t)len - 1u) == p->buf[total - 1u]) {
            p->frames_ok++;
            if (p->on_frame) p->on_frame(p->user, p->buf, total);
            drop(p, total);
        } else {
            p->crc_errors++;
            drop(p, 1);
        }
    }
}

void crsf_parser_feed(crsf_parser_t *p, const uint8_t *data, size_t len, uint32_t now_us)
{
    size_t i;
    for (i = 0; i < len; i++) {
        uint8_t before;
        if (p->pos > 0u && p->timeout_us != 0u &&
            (uint32_t)(now_us - p->frame_start_us) > p->timeout_us) {
            /* stale partial frame: salvage complete frames, drop the rest */
            p->timeouts++;
            while (p->pos > 0u) { drop(p, 1); scan(p); }
        }
        if (p->pos == 0u) {
            if (!is_sync(p, data[i])) continue;       /* idle: wait for sync */
            p->frame_start_us = now_us;
        }
        p->buf[p->pos++] = data[i];
        before = p->pos;
        scan(p);
        if (p->pos > 0u && p->pos < before)           /* resynced mid-buffer */
            p->frame_start_us = now_us;
    }
}

/* =========================================================== RC channels */
void crsf_pack_channels(const uint16_t ch[16], uint8_t out[22])
{
    uint32_t bits = 0;
    uint8_t nbits = 0, o = 0, i;
    for (i = 0; i < CRSF_NUM_CHANNELS; i++) {
        bits |= (uint32_t)(ch[i] & 0x7FFu) << nbits;  /* LSB first */
        nbits = (uint8_t)(nbits + 11u);
        while (nbits >= 8u) {
            out[o++] = (uint8_t)bits;
            bits >>= 8;
            nbits = (uint8_t)(nbits - 8u);
        }
    }
}

void crsf_unpack_channels(const uint8_t in[22], uint16_t ch[16])
{
    uint32_t bits = 0;
    uint8_t nbits = 0, c = 0, i;
    for (i = 0; i < CRSF_RC_PAYLOAD_SIZE; i++) {
        bits |= (uint32_t)in[i] << nbits;
        nbits = (uint8_t)(nbits + 8u);
        while (nbits >= 11u && c < CRSF_NUM_CHANNELS) {
            ch[c++] = (uint16_t)(bits & 0x7FFu);
            bits >>= 11;
            nbits = (uint8_t)(nbits - 11u);
        }
    }
}

/* ======================================================= byte order I/O */
static uint16_t rd16(const uint8_t *p) { return (uint16_t)(((uint16_t)p[0] << 8) | p[1]); }
static uint32_t rd24(const uint8_t *p)
{ return ((uint32_t)p[0] << 16) | ((uint32_t)p[1] << 8) | p[2]; }
static uint32_t rd32(const uint8_t *p)
{ return ((uint32_t)p[0] << 24) | ((uint32_t)p[1] << 16) | ((uint32_t)p[2] << 8) | p[3]; }
static void wr16(uint8_t *p, uint16_t v) { p[0] = (uint8_t)(v >> 8); p[1] = (uint8_t)v; }
static void wr24(uint8_t *p, uint32_t v)
{ p[0] = (uint8_t)(v >> 16); p[1] = (uint8_t)(v >> 8); p[2] = (uint8_t)v; }
static void wr32(uint8_t *p, uint32_t v)
{ p[0] = (uint8_t)(v >> 24); p[1] = (uint8_t)(v >> 16); p[2] = (uint8_t)(v >> 8); p[3] = (uint8_t)v; }

/* ============================================================== decoders */
bool crsf_decode_rc_channels(const uint8_t *f, uint16_t ch[16])
{
    if (f[2] != CRSF_TYPE_RC_CHANNELS || crsf_payload_len(f) < CRSF_RC_PAYLOAD_SIZE)
        return false;
    crsf_unpack_channels(&f[3], ch);
    return true;
}

bool crsf_decode_link_stats(const uint8_t *f, crsf_link_stats_t *ls)
{
    const uint8_t *p = &f[3];
    if (f[2] != CRSF_TYPE_LINK_STATS || crsf_payload_len(f) < 10u) return false;
    ls->up_rssi_ant1   = p[0];
    ls->up_rssi_ant2   = p[1];
    ls->up_lq          = p[2];
    ls->up_snr         = (int8_t)p[3];
    ls->active_antenna = p[4];
    ls->rf_profile     = p[5];
    ls->up_rf_power    = p[6];
    ls->down_rssi      = p[7];
    ls->down_lq        = p[8];
    ls->down_snr       = (int8_t)p[9];
    return true;
}

bool crsf_decode_battery(const uint8_t *f, crsf_battery_t *b)
{
    const uint8_t *p = &f[3];
    if (f[2] != CRSF_TYPE_BATTERY || crsf_payload_len(f) < 8u) return false;
    b->voltage_dv   = rd16(&p[0]);
    b->current_da   = rd16(&p[2]);
    b->capacity_mah = rd24(&p[4]);
    b->remaining    = p[7];
    return true;
}

bool crsf_decode_device_info(const uint8_t *f, crsf_device_info_t *info)
{
    const uint8_t *p = crsf_ext_payload(f);
    uint8_t n = crsf_ext_payload_len(f), i = 0, copy;
    if (f[2] != CRSF_TYPE_DEVICE_INFO) return false;
    while (i < n && p[i] != 0u) i++;                  /* end of name string */
    if (i >= n || (uint8_t)(n - i - 1u) < 14u) return false;
    copy = (i < CRSF_DEVICE_NAME_MAX) ? i : (uint8_t)(CRSF_DEVICE_NAME_MAX - 1u);
    memcpy(info->name, p, copy);
    info->name[copy] = '\0';
    p += i + 1u;
    info->serial        = rd32(&p[0]);
    info->hw_id         = rd32(&p[4]);
    info->fw_id         = rd32(&p[8]);
    info->param_count   = p[12];
    info->param_version = p[13];
    return true;
}

uint16_t crsf_rf_power_mw(uint8_t index)
{
    static const uint16_t mw[] = { 0, 10, 25, 100, 500, 1000, 2000, 250, 50 };
    return (index < sizeof(mw) / sizeof(mw[0])) ? mw[index] : 0u;
}

/* ============================================================== encoders */
uint8_t crsf_encode_frame(uint8_t *out, uint8_t sync, uint8_t type,
                          const uint8_t *payload, uint8_t len)
{
    if (len > CRSF_PAYLOAD_MAX) return 0;
    out[0] = sync;
    out[1] = (uint8_t)(len + 2u);                      /* type + payload + crc */
    out[2] = type;
    if (len) memcpy(&out[3], payload, len);
    out[3u + len] = crsf_crc8(&out[2], (size_t)len + 1u);
    return (uint8_t)(len + 4u);
}

uint8_t crsf_encode_ext_frame(uint8_t *out, uint8_t sync, uint8_t type,
                              uint8_t dest, uint8_t orig,
                              const uint8_t *payload, uint8_t len)
{
    uint8_t tmp[CRSF_PAYLOAD_MAX];
    if (len > CRSF_EXT_PAYLOAD_MAX) return 0;
    tmp[0] = dest;
    tmp[1] = orig;
    if (len) memcpy(&tmp[2], payload, len);
    return crsf_encode_frame(out, sync, type, tmp, (uint8_t)(len + 2u));
}

uint8_t crsf_encode_command(uint8_t *out, uint8_t sync, uint8_t dest, uint8_t orig,
                            uint8_t cmd, const uint8_t *payload, uint8_t len)
{
    /* extended payload = [cmd][payload ...][cmd_crc]
     * cmd_crc (poly 0xBA) covers: type 0x32, dest, orig, cmd, payload */
    uint8_t tmp[4u + CRSF_EXT_PAYLOAD_MAX];
    uint8_t n;
    if ((unsigned)len + 2u > CRSF_EXT_PAYLOAD_MAX) return 0;
    tmp[0] = CRSF_TYPE_COMMAND;
    tmp[1] = dest;
    tmp[2] = orig;
    tmp[3] = cmd;
    if (len) memcpy(&tmp[4], payload, len);
    n = (uint8_t)(4u + len);
    tmp[n] = crsf_crc8_cmd(tmp, n);
    return crsf_encode_ext_frame(out, sync, CRSF_TYPE_COMMAND, dest, orig,
                                 &tmp[3], (uint8_t)(len + 2u));
}

uint8_t crsf_encode_rc_channels(uint8_t *out, uint8_t sync, const uint16_t ch[16])
{
    uint8_t packed[CRSF_RC_PAYLOAD_SIZE];
    crsf_pack_channels(ch, packed);
    return crsf_encode_frame(out, sync, CRSF_TYPE_RC_CHANNELS, packed, CRSF_RC_PAYLOAD_SIZE);
}

uint8_t crsf_encode_link_stats(uint8_t *out, const crsf_link_stats_t *ls)
{
    uint8_t p[10];
    p[0] = ls->up_rssi_ant1;   p[1] = ls->up_rssi_ant2;
    p[2] = ls->up_lq;          p[3] = (uint8_t)ls->up_snr;
    p[4] = ls->active_antenna; p[5] = ls->rf_profile;
    p[6] = ls->up_rf_power;    p[7] = ls->down_rssi;
    p[8] = ls->down_lq;        p[9] = (uint8_t)ls->down_snr;
    return crsf_encode_frame(out, CRSF_SYNC_BYTE, CRSF_TYPE_LINK_STATS, p, 10);
}

uint8_t crsf_encode_battery(uint8_t *out, const crsf_battery_t *b)
{
    uint8_t p[8];
    wr16(&p[0], b->voltage_dv);
    wr16(&p[2], b->current_da);
    wr24(&p[4], b->capacity_mah);
    p[7] = b->remaining;
    return crsf_encode_frame(out, CRSF_SYNC_BYTE, CRSF_TYPE_BATTERY, p, 8);
}

uint8_t crsf_encode_gps(uint8_t *out, const crsf_gps_t *g)
{
    uint8_t p[15];
    int32_t alt = g->altitude_m + 1000;                /* +1000 m offset */
    if (alt < 0) alt = 0;
    if (alt > 0xFFFF) alt = 0xFFFF;
    wr32(&p[0], (uint32_t)g->lat_e7);
    wr32(&p[4], (uint32_t)g->lon_e7);
    wr16(&p[8], g->speed_dkmh);
    wr16(&p[10], g->heading_cdeg);
    wr16(&p[12], (uint16_t)alt);
    p[14] = g->satellites;
    return crsf_encode_frame(out, CRSF_SYNC_BYTE, CRSF_TYPE_GPS, p, 15);
}

uint8_t crsf_encode_vario(uint8_t *out, int16_t vspeed_cms)
{
    uint8_t p[2];
    wr16(p, (uint16_t)vspeed_cms);
    return crsf_encode_frame(out, CRSF_SYNC_BYTE, CRSF_TYPE_VARIO, p, 2);
}

uint8_t crsf_encode_attitude(uint8_t *out, int16_t pitch, int16_t roll, int16_t yaw)
{
    uint8_t p[6];
    wr16(&p[0], (uint16_t)pitch);
    wr16(&p[2], (uint16_t)roll);
    wr16(&p[4], (uint16_t)yaw);
    return crsf_encode_frame(out, CRSF_SYNC_BYTE, CRSF_TYPE_ATTITUDE, p, 6);
}

uint8_t crsf_encode_flight_mode(uint8_t *out, const char *mode)
{
    uint8_t p[CRSF_PAYLOAD_MAX];
    uint8_t n = 0;
    while (mode[n] != '\0' && n < CRSF_PAYLOAD_MAX - 1u) { p[n] = (uint8_t)mode[n]; n++; }
    p[n++] = 0;                                        /* null terminated */
    return crsf_encode_frame(out, CRSF_SYNC_BYTE, CRSF_TYPE_FLIGHT_MODE, p, n);
}

uint8_t crsf_encode_ping(uint8_t *out, uint8_t sync, uint8_t dest, uint8_t orig)
{
    return crsf_encode_ext_frame(out, sync, CRSF_TYPE_PING, dest, orig, NULL, 0);
}

uint8_t crsf_encode_device_info(uint8_t *out, uint8_t sync, uint8_t dest, uint8_t orig,
                                const crsf_device_info_t *info)
{
    uint8_t p[CRSF_EXT_PAYLOAD_MAX];
    uint8_t n = 0;
    while (n < CRSF_DEVICE_NAME_MAX - 1u && info->name[n] != '\0') {
        p[n] = (uint8_t)info->name[n];
        n++;
    }
    p[n++] = 0;
    wr32(&p[n], info->serial); n = (uint8_t)(n + 4u);
    wr32(&p[n], info->hw_id);  n = (uint8_t)(n + 4u);
    wr32(&p[n], info->fw_id);  n = (uint8_t)(n + 4u);
    p[n++] = info->param_count;
    p[n++] = info->param_version;
    return crsf_encode_ext_frame(out, sync, CRSF_TYPE_DEVICE_INFO, dest, orig, p, n);
}

uint8_t crsf_encode_param_read(uint8_t *out, uint8_t sync, uint8_t dest, uint8_t orig,
                               uint8_t param, uint8_t chunk)
{
    uint8_t p[2];
    p[0] = param;
    p[1] = chunk;
    return crsf_encode_ext_frame(out, sync, CRSF_TYPE_PARAM_READ, dest, orig, p, 2);
}
```

</details>
<details>
<summary>📄 examples/stm32_hal_dma.c — STM32 HAL：DMA 循环 + IDLE 接收、遥测发送、失控判断</summary>

```c
/**
 * stm32_hal_dma.c - CRSF receiver input on STM32 (HAL), illustrative example
 *
 * ELRS / Crossfire RX  -> USART1, 420000 baud (TBS spec: 416666), 8N1,
 * NOT inverted, full duplex (RX.TX -> MCU.RX, RX.RX <- MCU.TX).
 *
 * CubeMX: USART1 async 420000 8N1, DMA RX channel in CIRCULAR mode,
 *         USART1 global interrupt enabled; TIM2 = free running 32 bit, 1 MHz.
 * Not part of the host test build - adapt handles and names to your board.
 */
#include "main.h"
#include "crsf.h"

extern UART_HandleTypeDef huart1;
extern TIM_HandleTypeDef  htim2;

#define RX_DMA_SIZE 128u
static uint8_t       rx_dma[RX_DMA_SIZE];
static uint16_t      rx_old;                 /* last processed DMA position */
static crsf_parser_t parser;

/* data shared with the main loop */
static volatile uint16_t rc_us[CRSF_NUM_CHANNELS];
static volatile uint32_t rc_last_ms;
static volatile uint8_t  link_lq, link_rssi_dbm_neg;

static void on_crsf_frame(void *user, const uint8_t *f, uint8_t len)
{
    (void)user; (void)len;
    switch (crsf_frame_type(f)) {
    case CRSF_TYPE_RC_CHANNELS: {
        uint16_t ch[CRSF_NUM_CHANNELS];
        uint32_t i;
        if (f[0] != CRSF_ADDR_FC) break;        /* RC frames to the FC use 0xC8 */
        if (!crsf_decode_rc_channels(f, ch)) break;
        for (i = 0; i < CRSF_NUM_CHANNELS; i++)
            rc_us[i] = (uint16_t)CRSF_TICKS_TO_US(ch[i]);
        rc_last_ms = HAL_GetTick();
        break;
    }
    case CRSF_TYPE_LINK_STATS: {
        crsf_link_stats_t ls;
        if (crsf_decode_link_stats(f, &ls)) {
            link_lq = ls.up_lq;
            link_rssi_dbm_neg = ls.active_antenna ? ls.up_rssi_ant2 : ls.up_rssi_ant1;
        }
        break;
    }
    default:
        break;                                  /* ignore unknown types */
    }
}

void crsf_input_start(void)
{
    crsf_parser_init(&parser, CRSF_BAUD_ELRS_RX, on_crsf_frame, NULL);
    HAL_TIM_Base_Start(&htim2);
    rx_old = 0;
    HAL_UARTEx_ReceiveToIdle_DMA(&huart1, rx_dma, RX_DMA_SIZE);
}

/* HAL calls this on IDLE line, DMA half transfer and DMA transfer complete.
 * In circular mode 'pos' is the current write position inside rx_dma[]. */
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t pos)
{
    uint32_t now = __HAL_TIM_GET_COUNTER(&htim2);   /* 1 MHz timestamp */
    if (huart != &huart1 || pos == rx_old) return;
    if (pos > rx_old) {
        crsf_parser_feed(&parser, &rx_dma[rx_old], (size_t)(pos - rx_old), now);
    } else {                                        /* wrapped around */
        crsf_parser_feed(&parser, &rx_dma[rx_old], (size_t)(RX_DMA_SIZE - rx_old), now);
        crsf_parser_feed(&parser, &rx_dma[0], pos, now);
    }
    rx_old = (pos == RX_DMA_SIZE) ? 0u : pos;
}

/* With DMA reception enabled, HAL treats every UART error (noise, framing,
 * overrun - e.g. while plugging in the receiver) as blocking and aborts the
 * reception. Restart it, otherwise CRSF input stops for good. */
void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart)
{
    if (huart != &huart1 || huart->RxState != HAL_UART_STATE_READY) return;
    rx_old = 0;
    HAL_UARTEx_ReceiveToIdle_DMA(&huart1, rx_dma, RX_DMA_SIZE);
}

/* Send one telemetry frame (FC -> RX). The buffer must stay valid until the
 * DMA transfer is done, therefore it is static. */
bool crsf_send_battery(uint16_t mv, uint16_t ma, uint32_t mah, uint8_t pct)
{
    static uint8_t tx[CRSF_FRAME_SIZE_MAX];
    crsf_battery_t b;
    uint8_t n;
    if (huart1.gState != HAL_UART_STATE_READY) return false;  /* still busy */
    b.voltage_dv = (uint16_t)(mv / 100u);    /* 0.1 V */
    b.current_da = (uint16_t)(ma / 100u);    /* 0.1 A */
    b.capacity_mah = mah;
    b.remaining = pct;
    n = crsf_encode_battery(tx, &b);
    return HAL_UART_Transmit_DMA(&huart1, tx, n) == HAL_OK;
}

/* main loop helper: failsafe if no valid RC frame for too long */
bool crsf_rc_lost(uint32_t timeout_ms)
{
    return (HAL_GetTick() - rc_last_ms) > timeout_ms;
}
```

</details>
<details>
<summary>📄 examples/esp32_arduino.ino — ESP32 Arduino：读取通道与 LQ / RSSI</summary>

```c++
// esp32_arduino.ino - read CRSF from an ExpressLRS receiver with an ESP32 (Arduino core)
// Wiring (full duplex, not inverted): ELRS RX "TX" -> GPIO16, ELRS RX "RX" <- GPIO17, GND, 5V
// Copy crsf.h / crsf.c into the sketch folder.
#include <Arduino.h>
#include "crsf.h"

static crsf_parser_t parser;
static uint16_t ch[CRSF_NUM_CHANNELS];
static uint32_t lastRcMs;
static crsf_link_stats_t linkStats;

static void onFrame(void *user, const uint8_t *f, uint8_t len) {
  (void)user; (void)len;
  if (crsf_frame_type(f) == CRSF_TYPE_RC_CHANNELS && crsf_decode_rc_channels(f, ch)) {
    lastRcMs = millis();
  } else if (crsf_frame_type(f) == CRSF_TYPE_LINK_STATS) {
    crsf_decode_link_stats(f, &linkStats);
  }
}

void setup() {
  Serial.begin(115200);
  Serial1.begin(420000, SERIAL_8N1, /*rx=*/16, /*tx=*/17);
  crsf_parser_init(&parser, 420000, onFrame, nullptr);
  parser.timeout_us = 0;  // polled reads have no reliable arrival time: rely on CRC + resync
}

void loop() {
  uint8_t buf[64];
  size_t n = Serial1.available();
  if (n > sizeof(buf)) n = sizeof(buf);
  if (n) {
    n = Serial1.readBytes(buf, n);
    crsf_parser_feed(&parser, buf, n, micros());
  }

  static uint32_t lastPrint;
  if (millis() - lastPrint >= 200) {
    lastPrint = millis();
    if (millis() - lastRcMs > 1000) {
      Serial.println("FAILSAFE: no RC frames");
    } else {
      Serial.printf("CH1-4 %4ld %4ld %4ld %4ld us | ARM(CH5) %4ld | LQ %u%% RSSI -%u dBm\n",
                    (long)CRSF_TICKS_TO_US(ch[0]), (long)CRSF_TICKS_TO_US(ch[1]),
                    (long)CRSF_TICKS_TO_US(ch[2]), (long)CRSF_TICKS_TO_US(ch[3]),
                    (long)CRSF_TICKS_TO_US(ch[4]), linkStats.up_lq, linkStats.up_rssi_ant1);
    }
  }
}
```

</details>
## 10. 移植与优化建议
- **CRC 实现**：crsf-mini 用逐位计算（省 Flash，420k 下足够快）；多串口或高包率时可换成 256 字节查表（规范附有表格）；
- **不要用 packed struct 直接 memcpy 帧**：大小端、对齐、位域布局都不可移植。Betaflight 用位域结构体解包通道，依赖 GCC + 小端；crsf-mini 全部按字节和移位处理；
- **多端口**：每个 UART 一个解析器实例；如果设备要做路由（如飞控连接多个 CRSF 设备），按 Dest 转发不是发给自己的扩展帧；
- **通道数据**：中断写、主循环读时用双缓冲或关中断拷贝；
- **低延迟**：收到 RC 帧后尽快（例如 1 ms 内）更新输出，避免再额外等一个主循环周期；
- **测试**：在 PC 上用测试向量做单元测试，并注入随机噪声、截断帧、假 Sync；上板后用逻辑分析仪测量"帧结束 → 回调 → 输出更新"的延迟；
- **资源**：解析器实例约 130 字节 RAM（32 位 MCU），回调里的局部变量（16 路通道 32 字节）很小，适合在中断栈里运行。
> **本章要点**
> - 只有驱动层与硬件相关；帧层、协议层是纯 C，先在 PC 上用测试向量跑通
> - RX → MCU：420000（或 416666）8N1、不反相、全双工；硬件 UART + DMA 循环 + IDLE
> - STM32 HAL 在 UART 错误时会中止 DMA 接收，必须在 ErrorCallback 里重启
> - 解析器四关：Sync → Len（2–62）→ CRC → 回调；出错只丢 1 字节重扫；帧内超时约 1.75 ms
> - 回调要短，frame 指针只在回调内有效；RC 帧只认 Sync 0xC8；未知类型、多余字段都忽略
> - 失控以"最后一帧有效 RC 帧的时间"判断（规范建议约 1 s），恢复后要求重新解锁
> - 遥测：static 发送缓冲 + 忙检查 + 轮转调度，每类约 10 Hz；电池按 0.1 V / 0.1 A
> - 设备侧：Ping（Dest 为自己或 0x00）→ 回 Device Info，交换 Dest / Orig
> - 遥控器侧：单线半双工、主机定时发送、发送完成后立即切回接收
📖 出处：[TBS CRSF 规范](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [Betaflight rx/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/rx/crsf.c) · [Betaflight telemetry/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/telemetry/crsf.c) · [ELRS 配置飞控](https://www.expresslrs.org/quick-start/receivers/configuring-fc) · [ELRS 遥测带宽](https://www.expresslrs.org/info/telem-bandwidth/) · [ExpressLRS CRSFHandset.cpp](https://github.com/ExpressLRS/ExpressLRS/blob/master/src/lib/Handset/CRSFHandset.cpp)
