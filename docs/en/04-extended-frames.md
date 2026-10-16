# 🛠️ 04 Extended Frames: Discovery, Parameters, and Commands

[中文](../04-extended-frames.md) | **English** · [Guide home](../../README.en.md)

> **Chapter summary:** Extended frames add Dest/Orig after Type, enabling addressed requests and forwarding through multi-port devices. This chapter covers discovery (0x28/0x29), parameters (0x2B/0x2C/0x2D, including chunks and types), direct commands (0x32 with two CRCs), RC synchronization (0x3A), MSP over CRSF (0x7A/0x7B), and vendor extensions. These frames underpin ELRS Lua menus, handset binding controls, and remote VTX configuration.

## 1. Extended-frame review

- Layout: `[Sync][Len][Type][Dest][Orig][Payload][CRC]`, generally Type ≥ 0x28. Len = payload size + 4 for Type, Dest, Orig, and CRC. Maximum payload: 58 bytes.
- The **Host**, usually handset 0xEA, sends requests to a **Device**, such as TX module 0xEE, receiver 0xEC, or FC 0xC8. Responses **swap Dest and Orig**.
- Exceptions include 0x34 Logging, 0x80 ArduPilot, 0x81/0x82 mLRS, and 0xAA MAVLink encapsulation: these use a **short header** despite Type ≥ 0x28. Check the specification for the particular type.

## 2. Discovery: 0x28 Ping → 0x29 Device Information

1. The Host sends **0x28**, with no payload, to a device address or broadcast Dest 0x00.
2. Each receiving device replies with **0x29**:

| Field | Type | Meaning |
| --- | --- | --- |
| Device_name | Null-terminated string | For example, "ELRS RX" |
| Serial_number | uint32 | ExpressLRS uses 0x454C5253, ASCII "ELRS"; its Lua script recognizes ELRS devices from this value |
| Hardware_ID | uint32 | Hardware identifier |
| Firmware_ID | uint32 | Firmware version identifier |
| Parameters_total | uint8 | Number of parameters |
| Parameter_version_number | uint8 | Parameter-table version |

Test vectors:

| Frame | Bytes |
| --- | --- |
| Handset broadcast Ping: Sync 0xEE to the TX module, Dest 0x00, Orig 0xEA | `EE 04 28 00 EA 54` |
| RX Device Info: Dest 0xEA, Orig 0xEC, "ELRS RX", 12 parameters | `EA 1A 29 EA EC 45 4C 52 53 20 52 58 00 45 4C 52 53 00 00 00 00 00 04 01 00 0C 00 9C` |

## 3. Parameter system behind the Lua menus

### 3.1 A complete interaction

```text
Host (handset 0xEA)                          Device (TX module 0xEE)
  0x28 Ping, Dest=0x00          ------------>
                                <------------  0x29 Device Info (name, parameter count N)
  0x2C Read [param 0][chunk 0]  ------------>
                                <------------  0x2B [0][remaining 0] FOLDER "ROOT" + child list
  0x2C Read [param 1][chunk 0]  ------------>
                                <------------  0x2B [1][remaining 1] first chunk
  0x2C Read [param 1][chunk 1]  ------------>
                                <------------  0x2B [1][remaining 0] last chunk; reassembly complete
  ... read parameters 1..N in order ...
  0x2D Write [param k][new value]    ------------>
                                <------------  0x2D echoes new value (FLOAT / TEXT_SELECTION / STRING)
                                               or full 0x2B entry (COMMAND)
```

### 3.2 0x2C read requests and chunking

- Payload: `[Parameter_number][Chunk_number]`; chunks start at 0.
- The Host **always reads chunk 0 first**. The device supplies the remaining-chunk count in 0x2B. If nonzero, increment the chunk number and continue reading the **same parameter**. Read the next parameter only after the count reaches zero.
- Each response carries at most **56 bytes** of entry data: 58 minus parameter number and remaining count. The Host can repeat 0x2C to request a lost chunk.
- Test vector: `EE 06 2C EE EA 01 00 86`, reading TX module parameter 1, chunk 0.

### 3.3 0x2B parameter entries

Each frame's payload is `[Parameter_number][Chunks_remaining][Data chunk]`. Concatenate the chunks in order to obtain:

`[Parent_folder][Data_type][Name\0][Type-specific fields...]`

- Data_type **bit 7 is the hidden flag**; 1 means hidden. Parameters can be shown or hidden dynamically. Bits 0–6 contain the type.
- **Parameter 0 is the root folder**, which must be a FOLDER, conventionally named "ROOT". A device that does not respond to parameter 0 does not support the root-folder structure; the Host should then read 1 through Parameters_total.

| Value | Type | Type-specific fields | Data written with 0x2D |
| --- | --- | --- | --- |
| 0–5 | UINT8 … INT32 | Deprecated in favor of FLOAT | — |
| 8 | FLOAT | int32 Value / Min / Max / Default, uint8 decimal places, int32 step, null-terminated Unit | int32, 4 bytes |
| 9 | TEXT_SELECTION | Null-terminated Options separated by semicolons; uint8 Value / Min / Max / Default; null-terminated Unit | uint8 option index, 1 byte |
| 10 | STRING | Null-terminated Value, uint8 maximum length | New string |
| 11 | FOLDER | Child parameter numbers ending in 0xFF | Read-only |
| 12 | INFO | Null-terminated Info; read-only, no maximum-length field | Read-only |
| 13 | COMMAND | uint8 Status, uint8 Timeout; specification comment “ms * 100”; null-terminated Info | uint8 Status |
| 127 | OUT_OF_RANGE | Requested number out of range / end of parameter list | — |

FLOAT uses an integer and decimal-place count: Value = 1234 with Decimal_point = 2 means 12.34.

### 3.4 0x2D writes and acknowledgments

- Payload: `[Parameter_number][Data]`.
- The device **must acknowledge** writes. FLOAT / TEXT_SELECTION / STRING return **0x2D**, with the same parameter number and accepted value. COMMAND returns a full **0x2B** entry containing updated Status and Info for progress display.

### 3.5 COMMAND state machine: binding example

| Value | State | Direction | Meaning |
| --- | --- | --- | --- |
| 0 | READY | Device→Host | Idle / finished |
| 1 | START | Host→Device | Start execution |
| 2 | PROGRESS | Device→Host | Running; Info describes progress |
| 3 | CONFIRMATION_NEEDED | Device→Host | Host shows a confirm/cancel dialog |
| 4 | CONFIRM | Host→Device | User confirms |
| 5 | CANCEL | Host→Device | User cancels |
| 6 | POLL | Host→Device | Request the latest 0x2B status |

```text
Host: 0x2C read parameter "Bind"            Device: 0x2B COMMAND  Status=READY     Info=""
Host: 0x2D Status=START             Device: 0x2B COMMAND  Status=PROGRESS  Info="Binding"
Host: 0x2D Status=POLL (repeat as needed)     Device: 0x2B current status
                                    Device: 0x2B COMMAND  Status=READY     Info="OK" (done)
```

The specification explicitly requires the Host to send POLL when checking commands such as Bind on the TX module (0xEE); otherwise the latest status will not be returned.

### 3.6 Minimum features for a Lua-configurable device

- Respond to 0x28 addressed to yourself or 0x00 with 0x29, reporting the parameter count.
- Maintain a parameter table: parameter 0 is ROOT with its child list; serialize each other entry according to its type.
- Handle 0x2C by dividing entry data into 56-byte chunks and reporting the correct remaining count.
- Handle 0x2D by validating bounds, saving the value, and acknowledging it. Debounce or coalesce changes before writing flash.
- The handset Lua script reaches remote devices through the TX module and air link. Attach your device to CRSF and respond to the correct Dest so it appears under ELRS Lua's "Other Devices".

## 4. 0x32 Direct Commands

- Layout: `[Sync][Len][0x32][Dest][Orig][Command_ID][Subcommand / Data][Cmd_CRC8][CRC8]`.
- **Command CRC:** polynomial **0xBA**, covering 0x32, Dest, Orig, Command_ID, and command payload.
- **Outer CRC:** polynomial **0xD5**, covering Type through the command CRC. Calculate both.

Betaflight sends this frame to place a receiver in binding mode:

```text
C8 07 32 EC C8 10 01 9E E8
|  |  |  |  |  |  |  |  '-- Outer CRC8 (poly 0xD5, covers 32 ... 9E)
|  |  |  |  |  |  |  '----- Command CRC8 (poly 0xBA, covers 32 EC C8 10 01)
|  |  |  |  |  |  '-------- Subcommand 0x01 = enter binding
|  |  |  |  |  '----------- Command_ID 0x10 = Crossfire (receiver command)
|  |  |  |  '-------------- Orig 0xC8 = FC
|  |  |  '----------------- Dest 0xEC = receiver
|  |  '-------------------- Type 0x32 = Direct Commands
|  '----------------------- Len = 7
'-------------------------- Sync 0xC8
```

The specification's example `C8 07 32 EE 13 13 01 66 06` tells a TX module to enable RC over WiFi.

| Command_ID | Category | Main subcommands |
| --- | --- | --- |
| 0xFF | Command ACK | Command_ID, SubCommand_ID, Action: 1 executed / 0 unsupported, and Info |
| 0x01 | FC | 0x01 Force Disarm; 0x02 Scale Channel |
| 0x03 | Bluetooth | 0x01 reset; 0x02 enable/disable; 0x64 echo |
| 0x05 | OSD | 0x01 key input: bit7 Enter, bit6 up, bit5 down, bit4 left, bit3 right |
| 0x08 | VTX | 0x02 frequency in MHz; 0x04 enter PitMode on power-up; 0x05 exit PitMode; 0x06 dynamic power, renewed at 1 Hz with fallback after 3 s; 0x08 power in dBm |
| 0x09 | LED | Restore defaults; override color, breathing, blinking, gradients using packed HSV |
| 0x0A | General | 0x70 baud proposal: uint8 port_id + uint32 baud; 0x71 response: port_id + 1 accept / 0 reject |
| 0x10 | Crossfire | 0x01 bind; 0x02 cancel bind; 0x03 binding ID; 0x05 select model; 0x06/0x07 query/report model; 0x0A/0x0B RX telemetry on/off; 0x0C factory reset; 0x0D restore factory firmware |
| 0x13 | RC over WiFi | 0x01 enable; 0x02 disable |
| 0x20 | Flow Control | 0x01 subscribe: frame type + maximum interval in ms; 0x02 unsubscribe |
| 0x22 | Screen | 0x01 popup; 0x02 selection response |

### 4.1 Baud negotiation: 0x32 / 0x0A / 0x70 and 0x71

- Host proposes a higher rate → Device accepts or rejects → both switch if accepted.
- Betaflight returns to the default rate after **three consecutive frame errors** following the switch.
- ExpressLRS does **not use this negotiation**. RX⇄FC uses a fixed rate, default 420000, configurable in WebUI. Handset⇄TX baud is selected in EdgeTX and detected automatically by the ELRS module.

## 5. 0x3A.0x10 RC synchronization: Timing Correction / CRSF Shot

- Direction: TX module→handset on the half-duplex link. Layout: `EA 0D 3A EA EE 10 [update_interval] [offset] CRC`.
- update_interval: uint32, 100 ns units, representing the air-packet period. At 250 Hz: 4 ms = 40000.
- offset: int32, 100 ns units. **Positive means handset data is too early; negative means too late.**
- The handset adjusts mixer/transmit timing so the RC frame arrives just before air transmission, reducing latency and jitter. ExpressLRS uses this to synchronize the EdgeTX mixer to the packet rate; EdgeTX handles it at 1 µs precision.

## 6. MSP over CRSF: 0x7A request / 0x7B response

- This tunnels MSP over the radio link for remote FC access through tools such as Betaflight Lua and VTX configuration.
- Remove the MSP header, `$` + M/X + direction, and MSP checksum; transport the MSP body.
- Prefix each payload with one **Status** byte: bits 0–3 sequence counter; bit 4 first chunk of a new MSP packet; bits 5–6 MSP version, 1 or 2; bit 7 error, responses only.
- Each frame carries up to **57 bytes** of MSP chunk data. Split longer packets. MSP CRC is omitted because CRSF CRC protects the transport.
- Responses **swap Dest/Orig**; for example, Betaflight replies with Dest 0xEA and Orig 0xC8.
- MSP requires telemetry downlink. It cannot work with ELRS telemetry set to Off and shares downlink bandwidth with sensor telemetry; see chapter 05.

## 7. Other extended and vendor-specific frames

| Type | Name | Notes |
| --- | --- | --- |
| 0x34 | Logging | Debug logs; short header |
| 0x3C | Game | Scoring and command codes |
| 0x80 | ArduPilot Passthrough | Short header; subtype 0xF0 single packet, 0xF2 multiple packets, up to 9, 0xF1 status text |
| 0x81 / 0x82 | mLRS | Encapsulates mBridge for its Lua configuration script |
| 0x88 | Rotorflight telemetry encapsulation | Format varies by Rotorflight version |
| 0xAA | MAVLink encapsulation | total_chunks:4 / current_chunk:4, both counted from 0; data_size up to 58 |
| 0xAC | MAVLink system status | See the specification |

> **Key takeaways**
> - Extended headers add Dest/Orig; responses swap them. Some vendor types ≥0x28 use short headers.
> - Discovery is 0x28 Ping → 0x29 Device Info, including name, serial number, and parameter count. ELRS serial is ASCII "ELRS".
> - Parameters: 0x2C read with parameter/chunk numbers → 0x2B entry with remaining count, ≤56 bytes per chunk → 0x2D write with mandatory acknowledgment.
> - Parameter 0 is ROOT. Common types: FLOAT, TEXT_SELECTION, STRING, FOLDER, INFO, COMMAND.
> - 0x32 has inner 0xBA and outer 0xD5 CRCs.
> - 0x3A.0x10 synchronizes handset/TX timing; MSP over CRSF requires telemetry downlink.

📖 Sources: [TBS CRSF specification: Extended Frame Types](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md) · [Betaflight rx/crsf.c](https://github.com/betaflight/betaflight/blob/master/src/main/rx/crsf.c) · [ExpressLRS Lua](https://www.expresslrs.org/quick-start/transmitters/lua-howto/) · [ExpressLRS CRSFHandset.cpp](https://github.com/ExpressLRS/ExpressLRS/blob/master/src/lib/Handset/CRSFHandset.cpp)

---

[← Previous](03-frame-types.md) · [Contents](../../README.en.md#-knowledge-map) · [Next →](05-expresslrs.md)
