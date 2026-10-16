# 🏆 08 Hands-on Projects and Self-Assessment

[中文](../08-projects-and-quiz.md) | **English** · [Guide home](../../README.en.md)

> **Chapter summary:** Five progressively harder projects turn knowledge into practical skills: a PC decoder, capture visualization, CRSF-to-PWM conversion, a telemetry sensor node, and a Lua-configurable CRSF device. Each includes goals, steps, acceptance criteria, and relevant chapters. Fifteen expandable questions and answers check understanding.

## 1. Project overview

| # | Project | Difficulty | Time | Hardware | Relevant chapters |
| --- | --- | --- | --- | --- | --- |
| 1 | PC CRSF decoder | ⭐ | Half a day | None | 02, 03, section 4 of 06 |
| 2 | Serial capture and visualization | ⭐⭐ | 1 day | USB-TTL + ELRS receiver | 05, 07 |
| 3 | CRSF-to-PWM servo output | ⭐⭐⭐ | 2–3 days | STM32/ESP32 + servo | Sections 2–5 of 06 |
| 4 | Telemetry sensor node | ⭐⭐⭐ | 2–3 days | MCU + voltage divider + IMU | 03, 05, section 6 of 06 |
| 5 | Lua-configurable CRSF device | ⭐⭐⭐⭐ | 1 week | MCU + complete ELRS system | 04 |

## 2. Project 1 · PC CRSF decoder

**Goal:** get framing and protocol logic correct before using hardware.

1. Build crsf-mini on a PC and run `make test`, or implement your own library using it as a reference.
2. Write a CLI tool that reads hexadecimal strings from standard input and prints frame types and decoded fields.
3. Combine all chapter 07 vectors with random bytes, truncated frames, and false Sync bytes. Feed the stream to the parser in random-sized chunks.

**Acceptance:** all vectors decode correctly; no real frames are lost and no false frames appear in the noise stream. crc_errors/len_errors counts match expectations.

**Extra challenge:** compile with `-fsanitize=address,undefined` and fuzz with 100 million random bytes without crashing.

## 3. Project 2 · Serial capture and visualization

**Goal:** observe real ELRS traffic.

1. Connect USB-TTL RX to receiver TX and ground; run crsf_sniff.py.
2. Extend the script with matplotlib to plot 16 channel bars and LQ/RSSI curves in real time.
3. Count frames per second by type, including RC and link statistics.

**Acceptance:** plots follow stick movement; RC frame rate changes with packet-rate settings; LQ/RSSI respond to attenuation or distance.

**Questions:** is RC frame rate exactly the packet rate? How frequently do statistics arrive? Does one lost air packet stop RC output?

## 4. Project 3 · CRSF-to-PWM servo output

**Goal:** build a small CRSF→PWM converter.

1. Receive and parse RC frames using chapter 06's UART DMA/IDLE approach.
2. Generate four 50 Hz servo PWM outputs. Use `CRSF_TICKS_TO_US(ch)`, clamped to 1000–2000 µs.
3. After more than 1 s without valid RC frames, output preset failsafe positions; resume tracking when the signal returns, subject to the application's recovery/arming safeguards.
4. Update timer compare registers promptly after receiving RC data instead of waiting an extra main-loop cycle.

**Acceptance:** servos follow smoothly; after turning off the handset, they reach failsafe positions in about 1 s. Measure RC frame end → PWM update latency with a logic analyzer.

**Extra challenge:** output DShot/OneShot, or use CH15 LQ to drive a signal-quality LED in a mode that reserves CH15 for LQ.

## 5. Project 4 · Telemetry sensor node

**Goal:** show your MCU's data on the handset.

1. Make the MCU act as an FC at address 0xC8, with both UART directions connected to the receiver.
2. Measure battery voltage through an ADC and divider, then send 0x08 in 0.1 V units. Estimate attitude with an IMU such as MPU6050 and send 0x1E in 100 µrad units. Send status text with 0x21.
3. Rotate telemetry types as in chapter 06 section 6, about 10 Hz per type.

**Acceptance:** EdgeTX sensor discovery finds RxBt, Ptch, Roll, Yaw, and FM. Values match a multimeter or actual orientation. Changing the telemetry ratio from 1:2 to 1:128 makes refresh slower.

**Experiment:** send a counter incrementing every 100 ms as a 0x21 string. At a low telemetry ratio, check whether displayed values skip ahead or arrive in sequence. This reveals whether the receiver drops old values or queues data beyond the available bandwidth.

## 6. Project 5 · Lua-configurable CRSF device

**Goal:** implement discovery and the parameter protocol.

1. Answer 0x28 with 0x29 and `param_count = 3`.
2. Create a parameter tree: parameter 0 FOLDER "ROOT"; parameter 1 FLOAT "LED Brightness", 0–100%, step 1; parameter 2 TEXT_SELECTION "Mode", options "Off;Blink;On"; parameter 3 COMMAND "Save".
3. Support 0x2C chunk reads with up to 56 entry bytes per response, 0x2D writes, and the COMMAND state machine START→PROGRESS→READY. Persist parameters in flash.

**Acceptance:**

1. First emulate the handset with a PC script sending 0x28/0x2C/0x2D and printing responses; verify the full flow offline.
2. Connect to the real ELRS system. Find the device in Lua's Other Devices, change values, and observe immediate responses.
3. Verify that parameters survive a power cycle.

**Extra challenge:** use an entry longer than 56 bytes to test chunking; deliberately drop one chunk and verify that the Host requests it again.

## 7. Further directions

- CRSF→SBUS/PPM converter; remember SBUS is 100000 baud, 8E2, inverted.
- CRSF→USB HID game controller, connecting receiver output to a PC flight simulator.
- MSP over CRSF, 0x7A/0x7B, to read FC version or configure VTX from the handset.
- An MCU driving an ELRS TX module as the handset: half-duplex, master cadence, 0x3A.0x10 synchronization.
- Dual-receiver redundancy, switching between two CRSF inputs using LQ and frame-arrival information.

## 8. Self-assessment: expand to see answers

<details>
<summary>Q1 A frame begins with C8 18 16. How many bytes are in the frame and payload?</summary>

Len = 0x18 = 24. Total frame size is Len + 2 = **26 bytes**. Payload size is Len − 2, excluding Type and CRC, = **22 bytes**, exactly 16 × 11 bits.

</details>

<details>
<summary>Q2 What are the CRC8 polynomial, initial value, and coverage? What is special about 0x32?</summary>

Polynomial **0xD5**, CRC-8/DVB-S2, initial value 0. Coverage runs from **Type through payload end**, excluding Sync, Len, and CRC itself. A 0x32 command has an additional **inner CRC8**, polynomial **0xBA**, covering 0x32, Dest, Orig, Command ID, and command payload.

</details>

<details>
<summary>Q3 Why is the channel payload 22 bytes?</summary>

16 channels × 11 bits = 176 bits = 22 bytes. Values are packed continuously, LSB-first, without padding.

</details>

<details>
<summary>Q4 What pulse widths correspond to 172 / 992 / 1811, and how are they converted?</summary>

Approximately **988 / 1500 / 2012 µs**. Use µs = (ticks − 992) × 5 / 8 + 1500, or ticks = (µs − 1500) × 8 / 5 + 992. The exact arithmetic value for 1811 is 2011.875 µs; integer truncation gives 2011.

</details>

<details>
<summary>Q5 How do broadcast and extended frames differ?</summary>

Types below 0x28 generally use `[Sync][Len][Type][Payload][CRC]`, without Dest/Orig; Len = payload size + 2. Extended frames add Dest/Orig after Type; Len = payload size + 4, and responses swap those addresses. Some types ≥0x28, including 0x34, 0x80–0x82, and 0xAA, are exceptions with short headers.

</details>

<details>
<summary>Q6 Where are 400000, 416666, and 420000 baud used?</summary>

**400000** is the TBS default for handset⇄TX. **416666** is the TBS specification rate for RX⇄FC. **420000** is the ExpressLRS default for RX⇄FC. The latter two differ by about 0.8% and can interoperate within UART tolerance.

</details>

<details>
<summary>Q7 Why are ELRS external modules at 400K limited to 250 Hz?</summary>

The single-wire half-duplex period must fit two maximum 64-byte frames: 2 × 64 × 10 = 1280 bits. At 400 kbaud this takes 3.2 ms, approximately 312 Hz. A 250 Hz period is 4 ms and fits; 333 Hz is about 3 ms and does not. Higher packet rates need more baud.

</details>

<details>
<summary>Q8 Uplink RSSI Ant.1 is 0x50 in a 0x14 frame. What does it mean?</summary>

0x50 = 80. The field stores the negative of dBm, so the RSSI is **−80 dBm**.

</details>

<details>
<summary>Q9 How should the parser handle an ELRS 4.x 0x16 frame with Len = 0x19?</summary>

Accept it. Require Len ≥0x18 and decode the first 22 channel bytes. Ignore or decode the optional extra armStatus byte. Requiring exactly 0x18 can break channel reception after an upgrade.

</details>

<details>
<summary>Q10 How can 0x29 Device Info identify an ExpressLRS device?</summary>

Check Serial_number: ExpressLRS uses **0x454C5253**, ASCII "ELRS". Its Lua script uses the same identifier.

</details>

<details>
<summary>Q11 What if a parameter entry does not fit in one frame?</summary>

Split it into chunks. Each 0x2B reports the remaining count and carries up to 56 entry bytes, 58 minus parameter number and remaining count. The Host reads chunk 0 first, then requests successive chunks of the same parameter with 0x2C until the count is zero. Concatenate in order and rerequest lost chunks.

</details>

<details>
<summary>Q12 What is the execution flow for a COMMAND parameter such as Bind?</summary>

Host writes **START** with 0x2D → device responds with a 0x2B entry reporting **PROGRESS**. If confirmation is needed, it reports CONFIRMATION_NEEDED and the Host sends CONFIRM or CANCEL. During execution, the Host sends **POLL** for updates. On completion the device returns to **READY**. TX module commands require POLL to obtain the latest state.

</details>

<details>
<summary>Q13 What should an FC do when RC frames stop?</summary>

Use the absence of valid RC frames as the signal-loss criterion. In cut mode, the receiver stops 0x16. The specification suggests waiting about **1 s** before the FC starts its failsafe procedure. Do not rely on LQ=0 or a particular channel value. Require re-arming after recovery.

</details>

<details>
<summary>Q14 Why should the arm switch be AUX1 / CH5 in ELRS?</summary>

ELRS uses CH5 as the arm state for safety and performance behavior, for example stopping telemetry while armed in Race mode. Approximately 1000 µs is disarmed and 2000 µs armed. Remap a three-position switch to two positions in EdgeTX. Channel update rates still depend on the selected switch mode; see chapter 05.

</details>

<details>
<summary>Q15 What bandwidth is available at 50 Hz with telemetry ratio 1:128, and what follows from it?</summary>

About **8 bps**, enough only for infrequent updates of a few sensor values. MSP shares this bandwidth, so practical remote configuration is severely constrained. Increase packet rate or telemetry allocation when needed; for example, 250 Hz at 1:2 provides about 2500 bps without burst.

</details>

Scoring: 12 or more correct answers indicates a solid grasp of the core protocol and readiness for projects 4–5. Fewer than 8 suggests revisiting chapters 02, 03, and 05.

> **Key takeaways**
> - Validate parsing and codecs on a PC first, then move to hardware.
> - Use measurable, repeatable acceptance criteria for each project.
> - Project 3 covers reception/failsafe, project 4 telemetry, and project 5 extended frames/parameters.
> - The questions cover frame structure, CRC, channel conversion, baud rates, statistics, ELRS conventions, and parameters. Revisit the corresponding chapter for missed answers.

📖 Sources: [TBS CRSF specification](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [ELRS FC configuration](https://www.expresslrs.org/quick-start/receivers/configuring-fc) · [Switch modes](https://www.expresslrs.org/software/switch-config/) · [Telemetry bandwidth](https://www.expresslrs.org/info/telem-bandwidth/) · [Lua](https://www.expresslrs.org/quick-start/transmitters/lua-howto/)

---

[← Previous](07-debugging-and-tests.md) · [Contents](../../README.en.md#-knowledge-map)
