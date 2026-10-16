# 🧩 06 Embedded Implementation Guide with C Reference Code

[中文](../06-embedded-implementation.md) | **English** · [Guide home](../../README.en.md)

> **Chapter summary:** Implement CRSF on an MCU: layered architecture → UART configuration → circular DMA/IDLE reception → stream parsing with Sync filtering, length checks, CRC, timeout, and resynchronization → failsafe → telemetry scheduling → device Ping responses → handset-side half-duplex. The portable C99 **crsf-mini** library uses no dynamic allocation or platform dependencies and includes 63 unit tests. It targets platforms such as STM32, ESP32, and RP2040. Full source appears in section 9; the source notes report passing tests, which were not rerun for this translation.

## 1. Layered architecture

```text
Application  Channel mapping, failsafe, arming, telemetry sources, parameter menus
   ^ v
Protocol     crsf_decode_*() / crsf_encode_*(): field decoding and unit conversion
   ^ v
Framing      crsf_parser_feed(): Sync filter -> Len check -> CRC -> callback
   ^ v
Driver       UART + circular DMA + IDLE events + microsecond timestamps + DMA TX
```

| Layer | Responsibility | Hardware-specific? | PC unit testing? |
| --- | --- | --- | --- |
| Driver | Receive/transmit bytes and timestamp them | Yes | Substitute recordings or vectors |
| Framing | Assemble frames, verify CRC, time out, resynchronize | No | Yes |
| Protocol | Encode/decode fields and convert units | No | Yes |
| Application | Failsafe, arming, outputs, telemetry sources | Partly | Partly |

**Only the driver touches hardware.** Write framing/protocol code in plain C, verify it on a PC with chapter 07's vectors, then port it to the MCU. Most CRSF logic problems can be found before hardware testing.

## 2. UART configuration

| Role | Baud rate | Format | Voltage / polarity | Duplex and wiring |
| --- | --- | --- | --- | --- |
| MCU acts as FC, connected to receiver | 420000 ELRS default or 416666 TBS specification | 8N1 | 3.3 V, **non-inverted** | Full-duplex: receiver TX → MCU RX, receiver RX ← MCU TX, common ground |
| MCU acts as handset, connected to TX module | Start at 400000; ELRS supports higher rates | 8N1 | 3.3 V; some external module ports are inverted | Single-wire half-duplex; MCU is master |

- 416666 and 420000 differ by about 0.8% and can interoperate within UART tolerance. The combined endpoint error should generally stay within approximately 2%.
- Check the actual baud your MCU generates. At an STM32 USART clock of 84 MHz, 84 MHz ÷ 420000 = 200, giving zero error. At 72 MHz, the resulting rate is approximately 421053, +0.25%. CubeMX shows the error.
- Use a **hardware UART**. At 420k, each byte takes only about 23.8 µs, making software serial unreliable.
- RX⇄FC uses **normal polarity**. Do not copy SBUS inversion settings.
- For channel reception only, receiver TX→MCU RX is sufficient. Telemetry or MSP requires both signal directions.

## 3. Reception: circular DMA + IDLE events

At 420k, a 26-byte RC frame takes about 619 µs. At F1000, one frame arrives every millisecond, generating about 26,000 byte interrupts per second. Circular DMA plus UART IDLE can reduce this to roughly one callback per frame, while buffering data through callback delays. Frame length varies, so ordinary fixed-length DMA is unsuitable.

```text
UART RX --DMA(circular)--> rx_dma[128] --IDLE / HT / TC event--> HAL_UARTEx_RxEventCallback(pos)
                                                                   |
                         crsf_parser_feed(new bytes old..pos) <----+
                                       |
                                       v
                                  on_frame()  (valid CRC only)
```

1. In CubeMX, configure asynchronous USART at 420000 8N1, RX DMA in **Circular** mode, and enable the USART global interrupt. Start continuous reception with `HAL_UARTEx_ReceiveToIdle_DMA(&huart1, rx_dma, 128)`.
2. HAL calls `HAL_UARTEx_RxEventCallback(huart, pos)` for IDLE, half-transfer (HT), and transfer-complete (TC) events. pos is the current DMA write position.
3. Process only bytes between the previous position rx_old and pos. If pos is less than rx_old, the buffer wrapped: feed the parser in two parts.
4. Allocate room for at least two maximum-size frames, **128 bytes**.
5. **HAL pitfall:** UART noise, framing, or overrun errors, often during receiver hot-plugging, can abort DMA reception. Restart it in `HAL_UART_ErrorCallback()` or CRSF input may stop permanently.
6. For timestamps, configure TIM2 as a 32-bit free-running 1 MHz counter, prescaler = timer clock ÷ 1 MHz − 1, or use `DWT->CYCCNT`. Unsigned subtraction handles counter rollover.

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

Other platforms:

| Platform | Suggested approach |
| --- | --- |
| ESP32 Arduino | Use `Serial1.begin(420000, SERIAL_8N1, rx, tx)` and read in loop(). The driver buffers data, so byte timestamps may be inaccurate: set parser.timeout_us=0 and rely on CRC/resynchronization. Arduino-ESP32 2.x and later also provide onReceive()/setRxTimeout() for IDLE-like callbacks. |
| ESP32 ESP-IDF | Use uart_driver_install() with an event queue and uart_set_rx_timeout() for a timeout of 1–2 character times. |
| RP2040 | The PL011 receive timeout triggers with a nonempty FIFO after approximately 32 bit times without new data, serving a similar role to IDLE; DMA can also be used. |
| Small MCU without DMA | Byte ISR writes a ring buffer; the main loop feeds the parser. Record timestamps in the ISR. |

## 4. Parser: validation gates and resynchronization

```text
WAIT_SYNC ---> READ_LEN ---> READ_BODY ---> CHECK_CRC ---> on_frame()
    ^             | bad Len      | timeout      | bad CRC
    +-------------+--------------+--------------+
        drop 1 byte and rescan from the next byte (timeout: salvage first)
```

1. **Sync filter:** recognize only accepted Sync bytes as candidate headers. Default is 0xC8. The specification includes 0xC8, 0x00, and the receiving device's address; add them with crsf_parser_accept(), including 0xEA on a handset.
2. **Length check:** Len must be 2–62, otherwise increment len_errors.
3. **Collect the complete frame:** total length = Len + 2.
4. **CRC:** polynomial 0xD5, covering Type through payload end. Increment crc_errors on failure.
5. **Resynchronize:** on error, **drop only one byte** and search again. A real frame consumed behind a false Sync, such as 0xC8 inside a payload, remains recoverable in the buffer.
6. **In-frame timeout:** maximum frame duration plus 15% is 64 × 10 ÷ 420000 × 1.15 ≈ 1752 µs; Betaflight uses about 1.75 ms. If incomplete after this interval, increment timeouts, first recover any complete frames in the buffer, then discard the partial frame. Set timeout_us=0 to disable this.
7. **Statistics:** frames_ok, crc_errors, len_errors, and timeouts. Print once per second during debugging.

Callback rules:

- The frame pointer is valid **only during the callback**. Copy any data that must survive.
- A callback may run in **interrupt context** when feed is called from DMA/IDLE callbacks. Decode, copy, and set flags only; avoid printf and blocking transmission.
- For RC channels, accept `frame[0] == 0xC8`, matching Betaflight.
- Ignore unknown types. For long frames, decode known fields and ignore extras, including the optional extra ELRS 4.x 0x16 byte; require payload ≥22. Reject decoding if required fields are missing.

Minimal usage:

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

## 5. Failsafe handling

- Use **the timestamp of the last valid RC frame**. The receiver stops 0x16 on signal loss; the specification suggests about 1 s before FC failsafe.
- LQ, RSSI, and individual channel values are useful for warnings, not as the sole failsafe criterion.
- After recovery, if the arm switch remains armed, require the user to disarm and arm again so motors cannot restart immediately when the link returns.
- When an ISR writes channel data and the main loop reads it, use an interrupt-protected copy or double buffering to avoid mixing old and new channel values.

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

## 6. Telemetry scheduling: MCU → receiver

- RX⇄FC is full-duplex. The MCU can transmit without waiting for an RC frame. Use Sync 0xC8 for return frames, as Betaflight does.
- A DMA transmission buffer must remain valid until completion. Use **static storage** and check that the UART is idle before starting.
- ELRS packet rate and telemetry ratio constrain the downlink, down to about 8 bps at 50 Hz and 1:128. Faster serial transmission cannot increase that bandwidth. Follow Betaflight's approach of rotating telemetry types on a 100 ms cycle, with about 10 Hz per type.
- After discovering sensors in EdgeTX, look for RxBt/Curr/Capa/Bat% for battery, Ptch/Roll/Yaw for attitude, FM for flight mode, VSpd, and GPS/GSpd/Hdg/Alt/Sats.

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

The battery encoder above should produce `C8 0A 08 00 A8 00 7B 00 04 D2 38 09`, using 0.1 V / 0.1 A units. Use this as a self-check.

## 7. Device side: Ping and Device Info

Configuration tools such as ELRS Lua's Other Devices use broadcast Ping to find devices. To make your MCU discoverable, respond to 0x28 addressed to itself or 0x00 with 0x29, **swapping Dest/Orig**:

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

Self-check: send `C8 04 28 00 EA 54` through USB-TTL. Expected response:

`C8 1F 29 EA C8 4D 59 20 43 52 53 46 20 4E 4F 44 45 00 4D 59 4E 44 00 00 00 01 00 01 00 00 00 00 11`

If param_count is greater than zero, also implement 0x2C reads with chunked 0x2B responses, and 0x2D writes; see chapter 04.

## 8. Handset side: single-wire half-duplex

When an MCU acts as a handset to drive a TX module, for a custom radio, ground station, or simulator adapter:

- **Half-duplex UART:** on STM32, initialize with HAL_HalfDuplex_Init(), enable the transmitter before sending, and immediately enable the receiver when transmission is complete. Use the **TC flag, not TXE**. The signal line needs a pull-up.
- **Echo:** circuits that join TX/RX through resistors may receive their own transmitted bytes. Discard that echo.
- **Master cadence:** send 0x16 periodically at the module's packet rate, for example every 4 ms at 250 Hz. The module replies in the gap with link statistics, telemetry, and parameter responses.
- **Address:** add `crsf_parser_accept(&p, CRSF_ADDR_RADIO)`; module frames addressed to the handset begin with 0xEA.
- **Optional synchronization:** the module can send 0x3A.0x10 with packet interval and phase offset so the handset adjusts its transmit timing to reduce jitter; see chapter 04.
- **Polarity:** external module ports are often inverted. USARTs on STM32F3/F7/G4/H7 support TXINV/RXINV. F1/F4 need external inversion circuitry.
- **Baud:** start at 400000. ELRS TX cycles through supported rates after losing communication; see chapter 05.

## 9. 📦 crsf-mini reference implementation: full source

> **Source archive:** crsf.h and crsf.c contain the library; test_crsf.c contains 63 unit tests and vectors; examples/ contains STM32 HAL DMA and ESP32 Arduino integrations; Makefile and README.md are included. Run make test on a PC. For porting, add crsf.h and crsf.c to your project.

[Download the original crsf-mini source archive](../../assets/crsf-mini.zip)

crsf-mini.zip — C99 reference implementation, MIT License.

<details>
<summary>📄 crsf.h — constants, structures, and API</summary>

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
<summary>📄 crsf.c — CRC, stream parser, channel packing, frame encoders and decoders</summary>

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
<summary>📄 examples/stm32_hal_dma.c — circular DMA/IDLE, telemetry, and failsafe</summary>

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
<summary>📄 examples/esp32_arduino.ino — channel and LQ/RSSI reception</summary>

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

## 10. Porting and optimization

- **CRC:** crsf-mini uses a bitwise implementation to save flash; it is fast enough at 420k. For multiple UARTs or high rates, use a 256-byte lookup table such as the specification's table.
- **Avoid memcpy directly into packed frame structs:** endianness, alignment, and bitfield layout are not portable. Betaflight's channel bitfields rely on GCC/little-endian. crsf-mini uses byte operations and shifts.
- **Multiple ports:** give each UART its own parser. A routing device should forward extended frames addressed to other destinations.
- **Shared channels:** use double buffering or an interrupt-protected copy between ISR writers and main-loop readers.
- **Low latency:** update outputs promptly after RC reception, for example within 1 ms, instead of waiting an extra main-loop cycle.
- **Testing:** run PC unit tests with vectors plus random noise, truncated frames, and false Sync bytes. On hardware, measure frame end → callback → output update with a logic analyzer.
- **Resources:** a parser instance uses approximately 130 bytes of RAM on a 32-bit MCU. A callback's 16-channel array occupies 32 bytes, suitable for modest interrupt stacks.

> **Key takeaways**
> - Only the driver is hardware-specific. Validate plain-C framing and protocol code on a PC first.
> - RX→MCU: 420000 or 416666, 8N1, non-inverted, full-duplex. Use hardware UART and circular DMA/IDLE where available.
> - STM32 HAL can abort DMA reception on UART errors; restart in ErrorCallback.
> - Parser gates: Sync → Len 2–62 → CRC → callback. Drop one byte on error and rescan; timeout is about 1.75 ms.
> - Keep callbacks short. The frame pointer is temporary. Accept RC Sync 0xC8; ignore unknown types and extra fields.
> - Failsafe uses the last valid RC timestamp, with approximately 1 s suggested by the specification; require re-arming after recovery.
> - Telemetry uses static buffers, busy checks, and round-robin scheduling, about 10 Hz per type. Battery units are 0.1 V / 0.1 A.
> - Devices answer Ping to themselves or broadcast with Device Info, swapping addresses.
> - A handset is the half-duplex master, transmits periodically, then immediately returns to receive mode.

📖 Sources: [TBS CRSF specification](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [Betaflight rx/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/rx/crsf.c) · [Betaflight telemetry/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/telemetry/crsf.c) · [ELRS FC configuration](https://www.expresslrs.org/quick-start/receivers/configuring-fc) · [ELRS telemetry bandwidth](https://www.expresslrs.org/info/telem-bandwidth/) · [ExpressLRS CRSFHandset.cpp](https://github.com/ExpressLRS/ExpressLRS/blob/master/src/lib/Handset/CRSFHandset.cpp)

---

[← Previous](05-expresslrs.md) · [Contents](../../README.en.md#-knowledge-map) · [Next →](07-debugging-and-tests.md)
