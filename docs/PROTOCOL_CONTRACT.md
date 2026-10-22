# TSDZ2 UART Protocol Contract

## Scope and evidence

Baseline: `08b4020eec73e463b3c16262f69b92c7c00e16e4`.
ESP32 upstream: `TSDZ2-ESP32/TSDZ2-ESP32-Main`, commit
`25c0ac430e10d93546891635caa76534ee2ecf27`, version `1.2.2`.
Android handoff evidence: imported commit `58692d5d0dd4209d9c96d91a3336998a6e9a3357`.

This is a source audit, not a hardware acceptance test. All controller wire-field
meanings are **ESP32-side verified only**: they describe what this gateway writes
or interprets. Actual STM8 processing, control laws, electrical levels and
bootloader compatibility are not independently proven by this baseline.

References below are baseline line numbers relative to `esp32-gateway/main/`:

| Key | File and principal functions |
|---|---|
| U | `tsdz_uart.c`: `tsdz_uart_init`, `tsdz_uart_task`, `ctMessageReceived`, `checkCRC16` |
| UH | `tsdz_uart.h`: UART/header/length macros |
| D | `tsdz_data.c`: `getControllerMessage` (503–721), `processControllerMessage` (368–501), `processLcdMessage` (179–270), configuration initializer (85–126), `validHallRefAngles` (723–746), `tsdz_update_cfg` (749–774) |
| DH | `tsdz_data.h`: `struct_tsdz_cfg`, `struct_tsdz_data`, `struct_tsdz_hall` |
| C/CH | `tsdz_commands.c/.h`: `exec_command`, `command_ota`, `command_motor_calibration`, constants |
| O/OH | `tsdz_ota_stm8.c/.h`: `ota_stm8_init`, `ota_stm8_start`, `stm8_ota_task`, BSL helpers, `stm8_ota_status` |
| M/MH | `main.c`: `app_main`, `mainTask`; `main.h`: mode/error/ADC constants |
| CRC | `tsdz_utils.c:79–97`: `crc16`; declaration `tsdz_utils.h:18` |

All eleven requested files were read in full, including the full OTA implementation.
`main.c`, relevant BLE dispatch and Android OTA call sites were followed as well.

## A. UART physical/configuration and timing

| Item | Verified behavior | Source |
|---|---|---|
| Controller port | `CT_UART = UART_NUM_2` | UH:14 |
| ESP32 TX | GPIO25, macro `CT_RX_PIN`, passed in TX argument of `uart_set_pin` | UH:16–17; U:78 `tsdz_uart_init` |
| ESP32 RX | GPIO27, macro `CT_TX_PIN`, passed in RX argument | UH:16–17; U:78 |
| Format | 9600 baud, 8 data bits, parity disabled, 1 stop bit: 8N1 | U:51–57,74 |
| Flow control | Disabled; RTS/CTS pins unchanged | U:56,78 |
| Buffering | RX256/TX256 bytes, no event queue | U:82 |
| Separate OEM LCD port | UART1; ESP32 TX33 (`LCD_RX_PIN`), RX26 (`LCD_TX_PIN`); same format | UH:13,18–19; U:60–69 |
| Startup | 1000ms delay before UART init; comment says avoid STM8 bootloader activation | M:41–60 `app_main` |
| Poll | Nominal20ms `vTaskDelayUntil` loop; UART task each iteration | M:33,79–98 `mainTask` |
| Transmit | After a valid OEM LCD frame, or elapsed time since last controller send **strictly >100ms** | U:108–128 `tsdz_uart_task` |
| Controller timeout | **Strictly >500ms** since complete CRC-valid controller frame sets local0x80; otherwise clears it | U:130–157 |
| LCD timeout | >500ms sets local0x40 | U:158–161 |
| Framing | Search0x43 then consume29 total bytes, return after one complete frame per call; no inter-byte timeout in parser | U:201–234 `ctMessageReceived` |

Names are remote-endpoint-oriented: do not wire ESP32 TX27/RX25 by reading macro
names alone. Electrical voltage levels/tolerance are **UNKNOWN**.
U:88–89 comments describe10msg/sec, but actual code is LCD-event-driven plus a
>100ms fallback sampled by the20ms loop, not an exact100ms timer. STM8 TX cadence
is not established. Calculated serialization at9600 8N1 is13.54ms for13 bytes and
30.21ms for29 bytes; these are calculations, not measured timings.
Bad CRC increments `ui8_rxc_errors` without refreshing the timeout timestamp
(U:130–143). Parser state persists across polling calls.

## B. ESP32 → STM8: 13-byte packet

UH:22,27 defines `LCD_MSG_ID=0x59`, `LCD_OS_MSG_BYTES=13`; D:`getControllerMessage`
builds the packet and U:113–125 transmits it. Offsets are zero-based, bit0 is LSB.
LE means low byte first; `—` endian means a single byte/bit field. Unless stated,
unsigned N-bit storage permits0..2^N−1, not a proven safe controller setting range.

### Common envelope

| Offset | Bits | Width | Endian | Field | Unit/scaling, value meaning | Source |
|---|---|---|---|---|---|---|
| 0 | 7:0 | 8 | — | Header | Fixed0x59 | D:505–506 `getControllerMessage`; UH:22 |
| 1 | 7:0 | 8 | — | Message ID | 0→1→2→3→4→5→0, starts0; advances per builder call, no ACK | D:139,508–509,719–720 |
| 2 | 7:0 | 8 | — | Riding mode | 0..7 in F; locked→OFF, else app calibration→7, else level0→OFF, else internal mode &0x7f | D:511–520 |
| 3 | 7:0 | 8 | — | Assist parameter | Mode-dependent table below; not raw LCD assist level | D:522–567 |
| 4 | 6:0 | 7 | — | Lights config | `ui8_lights_configuration &0x7f`; individual meanings UNKNOWN | D:569–572 |
| 4 | 7 | 1 | — | Lights state | 1 when OEM lights on | D:250–251 `processLcdMessage`; D:571–572 |
| 5..10 | 7:0 each | 6×8 | Per ID | Payload | Every byte mapped below | D:574–706 |
| 11 | 7:0 | 8 | LE low | CRC | Over bytes0..10 | D:708–717 |
| 12 | 7:0 | 8 | LE high | CRC | Same CRC high byte | D:716–717 |

All D references in the following outgoing tables refer to `getControllerMessage`
unless another function is named.

### Byte3 assist parameter

| Mode | Byte3 source/value | Proven scaling/qualification | Source |
|---|---|---|---|
| OFF0 | 0 | No parameter | D:562–565 |
| POWER1 | `ui8_power_assist_level[level-1]` | Initializer comment `%/2`:25→50% | D:116,524–530 |
| TORQUE2 | `ui8_torque_assist_level[level-1]` | Raw; physical conversion UNKNOWN | D:117,538–544 |
| CADENCE3 | `ui8_cadence_assist_level[level-1]` | Initializer comment `W/2`:50→100W | D:115,531–537 |
| eMTB4 | `ui8_eMTB_assist_sensitivity[level-1]` | Raw sensitivity; conversion UNKNOWN | D:118,545–551 |
| WALK5 | `ui8_walk_assist_level[level-1]` | Raw; conversion UNKNOWN | D:119,552–558 |
| CRUISE6 | 0 | No parameter | D:562–565 |
| MOTOR_CALIBRATION7 | `ui8_app_assist_parameter` | Raw command byte; physical conversion UNKNOWN | D:559–561; C:181–197 `command_motor_calibration` |

Array modes write0 for level0. Normal OEM input establishes levels0..4
(D:180–196 `processLcdMessage`; DH OEM masks); builder has no additional array
upper-bound check. These values are not universally percentages.

### ID0

| Offset | Bits | Width | Endian | Field | Unit/scaling, value meaning | Source |
|---|---|---|---|---|---|---|
| 5 | 7:0 | 8 | LE low | Battery low voltage cutoff | Combined5–6 `ui16_battery_low_voltage_cut_off_x10`, /10V | D:576–578 |
| 6 | 7:0 | 8 | LE high | Cutoff high | Same16-bit value | D:577–578 |
| 7 | 7:0 | 8 | — | Max battery current | Amps; configured current or temperature-mapped current→0 between min/max when ESP32 temp control on | D:580–588; `tsdz_utils.c:53–65` `map` |
| 8 | 7:0 | 8 | — | Battery power limit | `*_div25`, name-derived ×25W; street power if internal street flag and limit enabled, otherwise target max | D:590–595 |
| 9 | 7:0 | 8 | — | FOC multiplicator / inductance setting | Raw; inductance conversion UNKNOWN; config update rejects >50 | D:597–598,754 `tsdz_update_cfg` |
| 10 | 7:0 | 8 | — | Motor acceleration | Raw; physical/time units UNKNOWN | D:600–601 |

### ID1

| Offset | Bits | Width | Endian | Field | Unit/scaling, value meaning | Source |
|---|---|---|---|---|---|---|
| 5 | 7:0 | 8 | — | Free/future use | Normally0; if `DEADTIME_TEST` and mode7, app rotor adjustment instead; units UNKNOWN | D:605–612 |
| 6 | 0 | 1 | — | Field weakening enable | Config flags bit1 (`0x02`) | D:617,622–623 |
| 6 | 6:1 | 6 | — | Unset | Explicitly zero in this sender | D:617–623 |
| 6 | 7 | 1 | — | OTA reboot request | Set if `stm8_ota_status>0`, increments status each such ID1 | D:618–620 |
| 7 | 7:0 | 8 | LE low | Wheel perimeter | Combined7–8 raw16-bit; mm corroborated by Android `res/values/strings.xml:170` | D:625–627 |
| 8 | 7:0 | 8 | LE high | Perimeter high | Same value | D:626–627 |
| 9 | 7:0 | 8 | — | Wheel max speed | km/h inferred from gateway config usage; street max if internal street flag, else max | D:629–633; D:253–267 `processLcdMessage` |
| 10 | 7:0 | 8 | — | Assist without pedal rotation threshold | Raw; physical scale UNKNOWN | D:635–636 |

Correction: D:614 comment says byte5 for flags, but actual assignments use **byte6**.

### ID2

| Offset | Bits | Width | Endian | Field | Unit/scaling, value meaning | Source |
|---|---|---|---|---|---|---|
| 5 | 7:0 | 8 | — | Optional ADC function | 0 unused,1 temperature,2 throttle; force0 if street prohibits throttle | D:639–644; MH:11–14 |
| 6 | 7:0 | 8 | — | Temp limiting min | Whole°C under gateway ×10 temperature convention | D:646–647; D:423–426 `processControllerMessage` |
| 7 | 7:0 | 8 | — | Temp limiting max | Same convention | D:648–649; D:423–426 |
| 8 | 7:0 | 8 | LE low | Torque offset low | Raw ADC offset low8 bits | D:651–653 |
| 9 | 6:0 | 7 | LE high | Torque offset high | Combined15-bit value,0..32767 | D:652–653 |
| 9 | 7 | 1 | — | Torque offset fix enable | Config flags bit0 | D:654–656 |
| 10 | 7:0 | 8 | — | Torque ADC conversion | `ui8_pedal_torque_per_10_bit_ADC_step_x100`; name records ×100/per10-bit ADC step; exact STM8 physical calibration UNKNOWN | D:658–659 |

### ID3

Reference array order is Hall states **6,2,3,1,5,4** (D:122 initializer). These are
raw circular angle codes, not proven degrees. Valid references yield reference+adj,
narrowed to uint8_t (modulo256); invalid references yield six zeros. With no
`DEADTIME_TEST`, mode7 uses app rotor adjustment; otherwise config phase adjustment
(D:663–685). No local definition of that build macro was found; an external build
could define it. Source for rows below: D:`getControllerMessage`.

| Offset | Bits | Width | Endian | Field | Unit/scaling, value meaning | Source |
|---|---|---|---|---|---|---|
| 5 | 7:0 | 8 | — | Hall6 reference | `ui8_hall_ref_angles[0]+adj` or0 | D:672,679 |
| 6 | 7:0 | 8 | — | Hall2 reference | `[1]+adj` or0 | D:673,680 |
| 7 | 7:0 | 8 | — | Hall3 reference | `[2]+adj` or0 | D:674,681 |
| 8 | 7:0 | 8 | — | Hall1 reference | `[3]+adj` or0 | D:675,682 |
| 9 | 7:0 | 8 | — | Hall5 reference | `[4]+adj` or0 | D:676,683 |
| 10 | 7:0 | 8 | — | Hall4 reference | `[5]+adj` or0 | D:677,684 |

`validHallRefAngles` accepts all-zero references (so they still receive adj), or
first value213..233 plus all six circular differences33..53 (D:723–745).
Config update allows phase/hall offset adjustment0..10 or246..255 (D:754–761),
without proving STM8 signed interpretation.

### ID4

Raw Hall counter offsets; physical time unit UNKNOWN. Same Hall array order
(D:123). Each sum narrows modulo256. Source: D:`getControllerMessage`.

| Offset | Bits | Width | Endian | Field | Unit/scaling, value meaning | Source |
|---|---|---|---|---|---|---|
| 5 | 7:0 | 8 | — | Hall6 offset | `ui8_hall_offsets[0]+ui8_hall_offset_adj` | D:689 |
| 6 | 7:0 | 8 | — | Hall2 offset | `[1]+ui8_hall_offset_adj` | D:690 |
| 7 | 7:0 | 8 | — | Hall3 offset | `[2]+ui8_hall_offset_adj` | D:691 |
| 8 | 7:0 | 8 | — | Hall1 offset | `[3]+ui8_hall_offset_adj` | D:692 |
| 9 | 7:0 | 8 | — | Hall5 offset | `[4]+ui8_hall_offset_adj` | D:693 |
| 10 | 7:0 | 8 | — | Hall4 offset | `[5]+ui8_hall_offset_adj` | D:694 |

### ID5

| Offset | Bits | Width | Endian | Field | Unit/scaling, value meaning | Source |
|---|---|---|---|---|---|---|
| 5 | 7:0 | 8 | — | Torque smooth enable | Exactly0/1 from config flags bit2 | D:697–702 `getControllerMessage` |
| 6 | 7:0 | 8 | — | Torque smooth min | Raw; Android UI calls minimum ADC threshold; STM8 formula UNKNOWN | D:703; Android `res/values/strings.xml:164` |
| 7 | 7:0 | 8 | — | Torque smooth max | Raw; Android UI calls maximum ADC difference threshold; formula UNKNOWN | D:704; Android `res/values/strings.xml:165` |
| 8 | 7:0 | 8 | — | UNKNOWN / UNUSED by ID5 builder | Unassigned, not guaranteed0 | D:696–705; U:33 |
| 9 | 7:0 | 8 | — | UNKNOWN / UNUSED by ID5 builder | Unassigned, not guaranteed0 | D:696–705; U:33 |
| 10 | 7:0 | 8 | — | UNKNOWN / UNUSED by ID5 builder | Unassigned, not guaranteed0 | D:696–705; U:33 |

U:33 defines one static zero-initialized `ct_send_msg`, reused by both send sites.
In normal0..5 rotation, ID5 bytes8..10 retain ID4 bytes8..10 (Hall offsets indices3..5
plus adjustment). This is a storage side effect, **not** a defined ID5 semantic or
proof STM8 ignores those bytes. CRC covers them. Simulator must not reject valid
ID5 merely because these bytes are nonzero.

## C. STM8 → ESP32: 29-byte packet

UH:21,25 and U:`ctMessageReceived` establish header0x43 and length29.
`checkCRC16` validates bytes0..26 against27 low/28 high before D's parser runs.
Unsigned multi-byte values are LE. No semantic range checks beyond framing/CRC
are applied here. Units describe gateway conventions, **ESP32-side verified only**.
All D references in this section mean `processControllerMessage`.

| Offset | Bits | Width | Endian | Field | Unit/scaling, value meaning | Source |
|---|---|---|---|---|---|---|
| 0 | 7:0 | 8 | — | Header | Fixed0x43 | UH:21; U:211–217 |
| 1 | 1:0 | 2 | — | Message type | 0 normal;1 Hall calibration;2/3 unhandled/UNKNOWN | D:369–373,439–500 |
| 1 | 2 | 1 | — | Controller UART RX error | 1→local bit4 | D:378–381 |
| 1 | 3 | 1 | — | Brake | 1→local bit5 | D:374–377 |
| 1 | 7:4 | 4 | — | Controller error nibble | →local bits3:0 | D:382–383 |
| 2 | 7:0 | 8 | LE low | Battery voltage low | Combined2–3 /1000V; gateway adds compensation below | D:388–391 |
| 3 | 7:0 | 8 | LE high | Battery voltage high | Same16-bit value | D:391 |
| 4 | 7:0 | 8 | — | Battery current | /10A, representable0..25.5A | D:385–386 |
| 5 | 7:0 | 8 | LE low | Wheel speed low | Combined5 + ((6 &7)<<8), /10km/h | D:397–400 |
| 6 | 2:0 | 3 | LE high | Wheel speed high | 11-bit total0..2047, /10km/h | D:398–400 |
| 6 | 7:3 | 5 | — | FOC angle | Raw0..31; physical angular scale UNKNOWN | D:399–402 |
| 7 | 7:0 | 8 | — | Cadence | RPM | D:404–405 |
| 8 | 7:0 | 8 | LE low | Pedal torque low | Combined8–9 /100 under torque_x100 convention; Nm not explicitly established by ESP32 arithmetic | D:407–410 |
| 9 | 7:0 | 8 | LE high | Pedal torque high | Same16-bit value | D:408 |
| 10 | 7:0 | 8 | — | Motor temperature | Whole°C under gateway convention; stored×10 only if ESP32 temp control off and optional ADC=1; otherwise ignored | D:412–414,423–426 |
| 11 | 7:0 | 8 | — | PWM duty | Raw0..255; percentage normalization/max duty UNKNOWN | D:430–431 |
| 12 | 7:0 | 8 | LE low | Motor ERPS low | Combined12–13 electrical rotations/sec by source comment, no multiplier | D:433–434 |
| 13 | 7:0 | 8 | LE high | Motor ERPS high | Same16-bit value | D:434 |
| 14 | 7:0 | 8 | — | Field weakening Hall counter offset | Raw; time unit UNKNOWN | D:436–437 |
| 15 | 7:0 | 8 | — | Firmware version (type0) | Numeric byte, packing UNKNOWN; cached initial255 means not yet received | D:128,443; C:131 `get_app_version` |
| 16 | 7:0 | 8 | LE low | Torque ADC (type0) | Combined16–17 rawADC; effective resolution UNKNOWN | D:445–446 |
| 17 | 7:0 | 8 | LE high | Torque ADC high | Same16-bit value | D:446 |
| 18 | 7:0 | 8 | — | Optional ADC (type0) | Raw8-bit `ui8_adc_throttle`; calibration UNKNOWN | D:448–449 |
| 19 | 7:0 | 8 | — | Throttle/temp mapped field (type0) | Comment says mapped0..255 and unused by Android; physical mapping UNKNOWN | D:451–453 |
| 20 | 7:0 | 8 | — | Torque Smooth (type0) | Comment: active percentage over last2sec; copied without range check | D:455–456 |
| 21 | 7:0 | 8 | — | Extended selector (type0) | 0..3 handled; others UNKNOWN | D:458–488 |
| 22 | 7:0 | 8 | Per selector | Extended byte0 | Table below | D:459–488 |
| 23 | 7:0 | 8 | Per selector | Extended byte1 | Table below | D:459–488 |
| 24 | 7:0 | 8 | Per selector | Extended byte2 | Table below | D:459–488 |
| 25 | 7:0 | 8 | Per selector | Extended byte3 | Table below | D:459–488 |
| 26 | 7:0 | 8 | Per selector | Extended byte4 | Table below | D:459–488 |
| 27 | 7:0 | 8 | LE low | CRC | Over0..26 | U:243–256 `checkCRC16` |
| 28 | 7:0 | 8 | LE high | CRC | Same CRC | U:253 |

Voltage transformation: `tmp = current_x10 * battery_pack_resistance_x1000 /10`
(integer division); stored voltage_x1000 = raw_LE16 + `(uint16_t)tmp`, narrowed to
uint16_t (D:390–391). Do not encode already-compensated voltage twice.
Bytes1..14 are processed before the type switch even for types1/2/3. Unknown types
still count as valid reception if CRC passes (U:132–138). Type0 clears Hall-valid flag.

### Extended data for normal type0 (selector byte21)

All byte fields use bits7:0. UNUSED means unread by this gateway branch, not a
proven STM8 reservation. Source function: D:`processControllerMessage`.

| Selector | Offsets | Width | Endian | Field/scaling | Source |
|---|---|---|---|---|---|
| 0 | 22 low,23 middle,24 high | 24 | LE | Wheel revolutions, raw count | D:460–464 |
| 0 | 25 low,26 high | 16 | LE | Crank revolutions, raw count | D:466–467 |
| 1 | 22 | 8 | — | TorqueAVG; physical unit UNKNOWN | D:469–473 |
| 1 | 23 | 8 | — | TorqueMin; physical unit UNKNOWN | D:472 |
| 1 | 24 | 8 | — | TorqueMax; physical unit UNKNOWN | D:473 |
| 1 | 25,26 | 2×8 | — | UNKNOWN / UNUSED | D:469–474 |
| 2 | 22,23,24,25,26 | 5×8 | — | PWM time debug1..5; individual meanings/units UNKNOWN | D:475–481 |
| 3 | 22,23 | 2×8 | — | Hall error debug1..2; individual meanings UNKNOWN | D:483–486 |
| 3 | 24,25,26 | 3×8 | — | UNKNOWN / UNUSED | D:483–487 |
| Other | 22..26 | 5×8 | UNKNOWN | No default decoder | D:459–488 |

Selector1 sets debugFlags0;2 sets0x20;3 sets0x40;0 does not reset flags.
Unassigned state fields retain previous values (D:459–488).

### Type1 Hall calibration overlay

Bytes0..14 and27..28 remain common; bytes15..26 replace **all** normal-specific
fields including version and extended selector. Values are raw counters; physical
time unit and mapping to physical Hall states are UNKNOWN.

| Low offset | High offset | Bits each | Width | Endian | Field | Source |
|---|---|---|---|---|---|---|
| 15 | 16 | 7:0 | 16 | LE | ui16_hall_1 | D:492 |
| 17 | 18 | 7:0 | 16 | LE | ui16_hall_2 | D:493 |
| 19 | 20 | 7:0 | 16 | LE | ui16_hall_3 | D:494 |
| 21 | 22 | 7:0 | 16 | LE | ui16_hall_4 | D:495 |
| 23 | 24 | 7:0 | 16 | LE | ui16_hall_5 | D:496 |
| 25 | 26 | 7:0 | 16 | LE | ui16_hall_6 | D:497 |

D:498 sets `hall_calib_data_valid=1`; M:`mainTask`:100–104 notifies packed
`tsdz_hall` over BLE then clears the flag. Do not infer incoming counter order from
outgoing Hall-reference array order.

## D. Exact CRC implementation and executed vectors

CRC:`crc16` is bitwise, reflected:

- Width16; initial **0xFFFF** (D:709 and U:244).
- Canonical polynomial **0x8005**, x^16+x^15+x^2+1; right-shift constant **0xA001**.
- `refin=true`, `refout=true`; reflected register used directly, no extra final reversal.
- XOR each input byte into register low byte, then eight LSB-first right shifts;
  XOR0xA001 after a shift if prior bit0 was1 (CRC:90–95).
- `xorout=0x0000`; no extra zero-byte augmentation or final transform.
- Equivalent **CRC-16/MODBUS** parameter set, derived from implementation.
- TX13 covers bytes0..10; RX29 covers0..26 including headers and unused bytes.
- Transmit **low byte then high byte**, independently proven at D:712–717 and U:247–253.

The original C function body was extracted unchanged, compiled to WebAssembly
with local Clang, and executed under Node. Only an initializing/iteration wrapper
was added outside the repository. All three results came from that original
implementation. These are CRC fixtures, not physically valid telemetry or safe
motor settings.

| Input hex (CRC excluded) | Length | Expected CRC | Transmitted low/high |
|---|---|---|---|
| `31 32 33 34 35 36 37 38 39` (ASCII123456789) | 9 | 0x4B37 | `37 4B` |
| `59 00 00 00 00 22 01 0F 14 1B 19` | 11 | 0x119A | `9A 11` |
| `43 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` | 27 | 0x1198 | `98 11` |

Reproduce from repository root with Python3, Clang+wasm linker, Node (use full
installed clang path if absent from PATH). This uses a temporary directory:

```python
from pathlib import Path
import tempfile, subprocess
s = Path('esp32-gateway/main/tsdz_utils.c').read_text()
fn = s[s.index('void crc16('):]  # last function in this baseline file
wrapper = '''
unsigned int audit_crc(const unsigned char *p, unsigned int n) {
 uint16_t c=0xffff;
 for(unsigned int i=0;i<n;i++) crc16(p[i], &c);
 return c;
}
'''
js = '''
const fs=require('fs');
(async()=>{
 const {instance}=await WebAssembly.instantiate(fs.readFileSync(process.argv[2]));
 const e=instance.exports;
 for(const [h,want] of [['313233343536373839',0x4b37],
                       ['590000000022010f141b19',0x119a],
                       ['43'+'00'.repeat(26),0x1198]]) {
  const p=Buffer.from(h,'hex');
  new Uint8Array(e.memory.buffer,4096,p.length).set(p);
  const c=e.audit_crc(4096,p.length);
  if(c!==want) throw Error('CRC mismatch');
  console.log(h,c.toString(16),[c&255,c>>>8]);
 }
})();
'''
with tempfile.TemporaryDirectory() as td:
 p=Path(td)
 (p/'crc.c').write_text('typedef unsigned char uint8_t;\n'
                       'typedef unsigned short uint16_t;\n'+fn+wrapper)
 (p/'run.cjs').write_text(js)
 subprocess.run(['clang','--target=wasm32','-nostdlib','-Wl,--no-entry',
                 '-Wl,--export=audit_crc','-Wl,--export-memory',
                 str(p/'crc.c'),'-o',str(p/'crc.wasm')],check=True)
 subprocess.run(['node',str(p/'run.cjs'),str(p/'crc.wasm')],check=True)
```

The separately named OEM `crc8` is merely addition modulo256
(`tsdz_utils.c:69–75`); do not confuse it with the STM8-link CRC.

## E. Controller status/error

Wire byte1 differs from the local system-state byte (D:369–383
`processControllerMessage`; MH:31–43):

| Wire bits | Meaning | Local state |
|---|---|---|
| 1:0 | Type0 normal,1 calibration,2/3 UNKNOWN | Not copied as error bits |
| 2 | Controller UART RX error | bit4,0x10 |
| 3 | Brake active | bit5,0x20 |
| 7:4 | Error nibble | bits3:0,mask0x0f |
| No wire bit | LCD missing >500ms | bit6,0x40 (U:158–161) |
| No wire bit | Controller missing >500ms | bit7,0x80 (U:154–157) |

| Value | Known symbol | Qualification |
|---|---|---|
| 0 | NO_ERROR | No controller nibble error |
| 1 | ERROR_MOTOR_BLOCKED | Defined MH:33 |
| 2 | ERROR_TORQUE_SENSOR | Defined MH:34 |
| 7 | ERROR_BATTERY_OVERCURRENT | Defined MH:35 |
| 8 | ERROR_OVERVOLTAGE | Defined MH:36; may be synthesized locally |
| 9 | ERROR_TEMPERATURE_LIMIT | Defined MH:37; may be synthesized locally |
| 10 | ERROR_TEMPERATURE_MAX | Defined MH:38; may be synthesized locally |
| 3–6,11–15 | UNKNOWN | No named meanings here |

D:419–427 synthesizes8/9/10 only if the received nibble was0, after battery/temp
checks. Local display of these codes does not prove STM8 emitted them.
CRC error counter, controller-reported RX-error bit, and local timeout are distinct.
Correction: MH:28–30 comments about bits0–5/brake6/status7–8 are stale; actual
masks/parser prove the layout above. OEM display error translation is a separate
layer in D:`getLCDMessage`:301–342, not these wire error values.

## F. Riding modes

Actual UART values from MH:16–24 and D:`getControllerMessage`:511–567:

| Mode | Numeric value |
|---|---|
| OFF_MODE | 0 |
| POWER_ASSIST_MODE | 1 |
| TORQUE_ASSIST_MODE | 2 |
| CADENCE_ASSIST_MODE | 3 |
| eMTB_ASSIST_MODE | 4 |
| WALK_ASSIST_MODE | 5 |
| CRUISE_MODE | 6 |
| MOTOR_CALIBRATION_MODE | 7 |

Do not substitute app override values (MH:68–73). Internal street bit7 is masked
out of wire byte2 and instead influences limits/throttle. OEM/app mode selection
is in D:`processLcdMessage`:201–267, including walk below8.0km/h and cruise
above9.0km/h under its conditions (MH:75–76). STM8 control laws remain unverified.

## G. Existing OTA handoff

1. Android `android-app/app/src/main/java/spider65/ebike/tsdz2_esp32/ota/Stm8_Ota.java:394–415`,
   `startUpdate`, starts HTTP server and writes command0x03 + `ssid|password|port`.
   `TSDZConst.java:23` defines command; `TSDZBTService.java:443–450`, `writeCommand`,
   writes the command characteristic.
2. ESP32 `tsdz_bt.c:555–571`, `handle_write_value`, calls C:`exec_command`:38–45 →
   `command_ota`:101–117. Wheel speed/cadence/ERPS >0 rejects with status2. These
   are cached telemetry checks, not independent proof the motor is stationary.
3. O:`ota_stm8_init`:904–930 parses parameters, rejects NULL SSID/password or port0,
   calls `wifi_set_data`, sets `stm8_ota_status=1`. Success acknowledges setup,
   not flashing completion.
4. Normal UART continues. ID1 byte6 bit7 is set while status>0 and each such builder
   call increments1→2→3 (D:617–620). IDs still rotate0..5. “Two consecutive times”
   means two ID1 occurrences with intervening IDs, not adjacent packets. The code
   counts builder calls, not confirmed STM8 receptions.
5. A later U:`tsdz_uart_task`:90–103 sees status>2, waits up to100ms for CT TX done.
   Success calls `ota_stm8_start`; failure resets status0 and resumes normal work.
   There is no STM8 acceptance acknowledgment at this handoff.
6. O:`ota_stm8_start`:897–901 creates `stm8_ota_task` (stack3072,priority5), then
   `vTaskSuspend(NULL)` suspends caller main task. Normal UART and other main-loop
   services stop there. Task-creation success is not checked.
7. O:`stm8_ota_task`:725–737 waits300ms then `bsl_sync(CT_UART)`. That helper flushes
   input and syncs; actual return1 means success despite its contrary return-value
   comment (O:124–178). No baud/parity reconfiguration occurs in this OTA file.
8. Successful sync permits Wi-Fi and firmware download (O:740–813,32KiB checks),
   then `bsl_getInfo` (819–824). BSL versions0x10/12/13/14 select matching routines;
   other versions fail (831–855). Only then `bsl_memWrite` uploads RAM routines
   (857), writes firmware starting0x8000 (866–873), and `bsl_jumpTo` starts0x8000
   (876). These are existing call conditions, not a newly designed OTA protocol.
9. Existing command0x06 reports phases0 success,1 download done,2 RAM sent,3 write
   progress,4 error (CH:30–38; O:815–894). Success/error paths restart after1sec.
   Android `Stm8_Ota.java:487–527`, `onMessageEvent`, consumes those notifications.

Actual STM8 reboot reaction, bootloader window, ROM compatibility and physical
flashing are **ESP32-side verified only / not hardware verified**. O:728–730's
one-second window is a comment, not measurement or independent STM8 proof.
No bootloader simulator or full independently verified OTA wire contract is specified.

## Corrections and UNKNOWN inventory

No prior committed packet contract exists here. Corrections concern source/comment
readings, not a claim that a prior simulator existed:

- ESP32 TX25/RX27, despite opposite-looking macro names.
- ID1 flags are byte6 despite byte5 comment; ID5 bytes8..10 are stale/unassigned,
  not fixed zero or new semantic fields.
- “10msg/sec” is not a strict timer guarantee; actual >100ms/LCD event paths govern.
- Wire/local status differ; executable masks supersede stale MH comments.
- OTA has no STM8 acceptance check before suspension; `bsl_sync` return1 is success.

All controller semantics are **ESP32-side verified only**. Explicit unknowns:

- Electrical levels/tolerance, actual STM8 UART settings, normal STM8 TX cadence,
  timing tolerances, controller processing/control laws and safe allowed ranges.
- ID5 bytes8..10 purpose; individual lights configurations; TORQUE/eMTB/WALK and
  calibration assist physical scales; inductance conversion; acceleration units;
  no-pedal threshold scale; torque ADC calibration formula; angle-code/counter
  units and signed interpretation.
- Message types2/3; selectors outside0..3; unused selector1 bytes25..26 and selector3
  bytes24..26; individual debug meanings/scales; torque statistics units; torque
  physical calibration; FOC angular scale; PWM percent normalization; ADC effective
  resolution/calibration; optional mapped field conversion; firmware version packing;
  incoming Hall counter units/physical state order.
- Error codes3–6/11–15 and which named errors STM8 actually emits. Locally synthesized
  errors must be distinguished from received errors.
- STM8 OTA request acceptance, bootloader compatibility and measured timings.

## Validation scope

Both packet lengths and all byte positions are accounted for, retaining UNKNOWN
labels. CRC implementation, coverage and order were traced in producer/validator;
three vectors executed original C code; the embedded reproduction script was also
executed successfully with the installed Clang path. No production build or device
test is claimed. Before commit, both imported tree IDs were verified against their
upstream commits; all 264 imported files plus `fake-stm8/README.md` were compared
byte-for-byte with baseline blobs and matched. Git diff showed no tracked-file
changes and the only new file was this document. The baseline branch remained at
`08b4020eec73e463b3c16262f69b92c7c00e16e4`.

## Fake STM8 v1 minimum behavior

Future requirements only; no simulator source is included:

- Receive13-byte packets with0x59 header; validate CRC0..10 against11/12 low/high;
  reject/log corrupt frames without treating them as commands.
- Decode IDs0..5, riding mode/assist, flags and mixed bitfields per B/F. Preserve/log
  unknown bytes; do not require ID5 bytes8..10 to be zero or invent their meaning.
- Generate legal29-byte type0 status with0x43 header, packed speed/FOC, LE fields,
  selector0/1/2/3 payload shapes and generated CRC27/28 low/high.
- Periodic configurable telemetry sufficient to avoid >500ms gateway timeout;
  do not call a chosen simulator cadence a proven STM8 requirement.
- Fault injection: bad CRC, dropped/partial frames, brake state, controller UART
  receive-error bit and known controller error nibble, independently controllable.
- Pause TX to trigger >500ms communication timeout, then resume valid telemetry
  and verify recovery; distinguish it from UART-error bit and CRC-error counter.
- Serial logging: timestamps, raw TX/RX, ID/mode/assist, CRC results, unknown bytes,
  fault settings and pause/resume events.
- If later testing Hall calibration, use type1 overlay without invented units.
  Log OTA handoff requests; do not implement or guess STM8 flashing as part of
  minimum normal-UART simulation.
