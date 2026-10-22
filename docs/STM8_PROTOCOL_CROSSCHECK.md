# STM8 protocol cross-check

## 1. Source selection, provenance and limits

| Item | Pinned evidence |
|---|---|
| ESP32 | `TSDZ2-ESP32/TSDZ2-ESP32-Main`, `25c0ac430e10d93546891635caa76534ee2ecf27`, version1.2.2 |
| Selected STM8 | `TSDZ2-ESP32/TSDZ2-Smart-EBike` |
| Branch | `master` at inspection; no tag points at selected commit |
| Exact STM8 commit | `3df3083ad220856eb416692aa6d5ad73f5fdd1fc` |
| Version | `src/main.h:15`: `FW_VERSION 17`; commit explicitly says **WIP v17** |
| Previous tagged candidate | `v16` = `9400b82f5b41cddb914f921a9ee5e471d4b5a147` |
| Scope | Source-level protocol match, not a verified binary release pairing or hardware certification |

Provenance chain, rather than selection of an unrelated fork:

1. Pinned ESP32 `README.md:1–6` describes its Open Source Firmware interface and
   links the old `TSDZ-ESP32/TSDZ-ESP32-wiki` URL. STM8 `README.md:1–6` describes
   the motor controller and links the same old URL. Current organization is
   [TSDZ2-ESP32](https://github.com/TSDZ2-ESP32); its five repositories include this
   controller, ESP32, Android, PCB and Wiki. The old spelling is recorded, not
   treated as an independent source pin.
2. [Original author's current project wiki](https://github.com/TSDZ2-ESP32/TSDZ2-ESP32-Wiki/wiki)
   identifies the custom controller firmware as derived from Open Source Firmware
   0.20.0-beta1, with a faster controller loop and redesigned motor control. It
   links the historical OpenSource-EBike-firmware/TSDZ2_wiki (now OpenSourceEBike).
   Organization page identifies the controller repository as a fork of
   `leon927/TSDZ2-Smart-EBike`. This establishes lineage, not equivalence to every fork.
3. [Controller flash wiki](https://github.com/TSDZ2-ESP32/TSDZ2-ESP32-Wiki/wiki/TSDZ-Controller-Flash)
   points to the project's own `bin/STM8` distribution and states ROM bootloader
   must be enabled via option bytes for OTA. No binary-to-source checksum manifest
   or exact ESP32↔STM8 pairing manifest was established in this audit.
4. Both pinned ESP32 and selected STM8 trees contain no `.gitmodules`/gitlinks;
   STM8 is not pinned as a submodule by ESP32. ESP32 history was read directly from
   fetched upstream commit objects; STM8 was cloned separately **outside this
   monorepo** for audit. No new controller source tree is imported here.
5. Coordinated history supports selection: STM8 `2efab1f5c704a8248394adc8a70c94b6611a997b`
   (2022-03-28) introduced the new status structure/Torque Smooth alongside ESP32
   `cc2bcf5` that day; STM8 `e31cf54fda322bacdabdfc1986adca1885266d7e`
   (2021-07-26) introduced watchdog OTA alongside ESP32 `8691354`.
6. Selected STM8 commit (mspider65, 2023-10-19 13:09:02 +0200) adds controller
   UART error bit and revised communications scheduling. Pinned ESP32 commit
   (same author, 13:01:40 that day) adds parsing of that UART error and revised
   transmission. Full SHA evidence: [STM8 commit](https://github.com/TSDZ2-ESP32/TSDZ2-Smart-EBike/commit/3df3083ad220856eb416692aa6d5ad73f5fdd1fc),
   [ESP32 commit](https://github.com/TSDZ2-ESP32/TSDZ2-ESP32-Main/commit/25c0ac430e10d93546891635caa76534ee2ecf27).

The actual reciprocal field assignments below, not date alone, establish protocol
compatibility. v16 is a historical candidate with older timeout policy and no new
RX-error status bit; it is not silently substituted for v17. **Important WIP
limitation:** v17 disables real brake input (section9). This audit does not
recommend flashing it or claim a stable released v17 image.

## 2. Evidence vocabulary and source map

- **VERIFIED BOTH SIDES**: matching ESP32 producer/STM8 consumer or converse.
- **ESP32-SIDE ONLY**: gateway-local policy/conversion not executed in STM8.
- **STM8-SIDE ONLY**: controller computation/policy proved here, not defined by ESP32.
- **UNKNOWN**: not established by inspected source; no invented units/protocol.

Unless stated otherwise, `S:` references mean the selected STM8 commit's
`src/ebike_app.c`. Its relevant functions are `UART2_RX_IRQHandler` (1527–1544),
`communications_controller` (1550–1571), `uart_receive_package` (1579–1775),
`uart_send_package` (1793–1980), `UART2_TX_IRQHandler` (1512–1522).
`E:` means pinned ESP32 `main/tsdz_data.c`: `getControllerMessage` (503–721),
`processControllerMessage` (368–501). ESP32 UART references mean
`main/tsdz_uart.c`; full original audit tables remain in `PROTOCOL_CONTRACT.md`.

Immutable STM8 source links:

- [ebike_app.c](https://github.com/TSDZ2-ESP32/TSDZ2-Smart-EBike/blob/3df3083ad220856eb416692aa6d5ad73f5fdd1fc/src/ebike_app.c)
- [motor.c](https://github.com/TSDZ2-ESP32/TSDZ2-Smart-EBike/blob/3df3083ad220856eb416692aa6d5ad73f5fdd1fc/src/motor.c)
- [main.c](https://github.com/TSDZ2-ESP32/TSDZ2-Smart-EBike/blob/3df3083ad220856eb416692aa6d5ad73f5fdd1fc/src/main.c),
  [main.h](https://github.com/TSDZ2-ESP32/TSDZ2-Smart-EBike/blob/3df3083ad220856eb416692aa6d5ad73f5fdd1fc/src/main.h)
- [common.c](https://github.com/TSDZ2-ESP32/TSDZ2-Smart-EBike/blob/3df3083ad220856eb416692aa6d5ad73f5fdd1fc/src/common.c),
  [common.h](https://github.com/TSDZ2-ESP32/TSDZ2-Smart-EBike/blob/3df3083ad220856eb416692aa6d5ad73f5fdd1fc/src/common.h)
- [uart.c](https://github.com/TSDZ2-ESP32/TSDZ2-Smart-EBike/blob/3df3083ad220856eb416692aa6d5ad73f5fdd1fc/src/uart.c),
  [timers.c](https://github.com/TSDZ2-ESP32/TSDZ2-Smart-EBike/blob/3df3083ad220856eb416692aa6d5ad73f5fdd1fc/src/timers.c),
  [adc.c](https://github.com/TSDZ2-ESP32/TSDZ2-Smart-EBike/blob/3df3083ad220856eb416692aa6d5ad73f5fdd1fc/src/adc.c),
  [pins.h](https://github.com/TSDZ2-ESP32/TSDZ2-Smart-EBike/blob/3df3083ad220856eb416692aa6d5ad73f5fdd1fc/src/pins.h)

## 3. Receive framing, UART and mode values

**VERIFIED BOTH SIDES:** STM8 `src/uart.c:17–31`, `uart2_init`, configures UART2
9600 8N1, clock disabled, TX/RX enabled. Matches ESP32 UART2 format. STM8 pins
PD5 TX/PD6 RX are defined in `src/pins.h:93–96`; ESP32 pins remain TX25/RX27.
Hardware electrical levels/tolerance remain UNKNOWN.

S:308–309 declares RX13/TX29. RX ISR seeks0x59, collects13 bytes, sets complete
flag and disables RX interrupt; it re-enables after main-loop processing, including
bad CRC (S:1527–1544,1769–1773). No inter-byte framing timeout is implemented.
Receiver validates CRC over0..10 with11 low/12 high. Valid CRC clears missing-RX
counter/error and consumes mode, assist, lights before ID dispatch (S:1581–1610).
Bad CRC consumes neither fields nor resets safety counter. Unknown ID has no
payload case but a valid CRC still updates common fields and resets RX timeout.
This matches ESP32's13-byte construction (E:505–717).

**VERIFIED BOTH SIDES:** `src/common.h:13–20` matches ESP32 `main/main.h:17–24`:
OFF0, POWER1, TORQUE2, CADENCE3, eMTB4, WALK5, CRUISE6, MOTOR_CALIBRATION7.
No street bit is expected on wire mode2; that flag is ESP32-local.

### Assist parameter: actual controller meaning

All calculations below are **STM8-SIDE ONLY** refinements of a **VERIFIED BOTH
SIDES** byte3 mapping (E:522–567 ↔ S:1599–1600). Preserve integer rounding/types.

| Mode | Controller behavior and scale | STM8 function/source |
|---|---|---|
| OFF0 | No mode handler; control targets start0 each cycle | `ebike_control_motor`, S:392–425 |
| POWER1 | Nominal2% steps; actual `power_x100=(cadence*parameter*torque_x100)>>9`, an approximation to divisor478, not an exact2% multiplier | `apply_power_assist`, S:523–555 |
| TORQUE2 | Battery current target in ADC steps = torque_delta_ADC × parameter /110, subject to cadence/current/duty gates | `apply_torque_assist`, S:608–627,658–672 |
| CADENCE3 | Target power = parameter×2W, parameter capped200 (400W), zero when cadence0 | `apply_power_assist`, S:507–520 |
| eMTB4 | Parameter1..20 selects nonlinear lookup tables160,165,…255 using torque_delta_ADC+10; no percent/physical unit | `apply_emtb_assist`, S:696–798; tables S:83–298 |
| WALK5 | Duty target=min(parameter,80), wheel speed<8km/h; current target min(80ADC steps,configured max) | `apply_walk_assist`, S:847–869 |
| CRUISE6 | Parameter×10 gives target speed_x10 (km/h); parameter0 captures current speed on entry. ESP32 sends0 | `apply_cruise`, S:872–910; E:562–565 |
| CALIBRATION7 | Raw duty target; values≥254 clamp253; uses configured max current | `apply_calibration_assist`, S:676–693 |

## 4. IDs0–5: configuration semantics

Each payload position corresponds to the unchanged ESP32 tables in contract B.
The mappings are **VERIFIED BOTH SIDES**; controller use/formula is **STM8-SIDE
ONLY**. Wire multi-byte values remain LE. Safe calibrated physical values are not
inferred from byte ranges.

| ID / offsets | Actual STM8 use | Paired source |
|---|---|---|
| 0 /5–6 | Voltage_x10 converted to cutoff ADC: raw×100/87; voltage ADC conversion87mV/step | E:576–578 ↔ S:1612–1613; `src/main.h:179` |
| 0 /7 | Max current in A → ADC steps `raw*100/16` | E:580–588 ↔ S:1615–1621; `src/main.h:194` |
| 0 /8 | Limit×25W → current_x100=`raw*2500000/voltage_mV`; divide16 to ADC; choose min(current limit,power limit) | E:590–595 ↔ S:1618–1626 |
| 0 /9 | FOC coefficient capped50, used in phase-current based angle calculation; not direct inductance in mH | E:597–598 ↔ S:1628–1632; `motor.c:684–712` |
| 0 /10 | Acceleration adjustment0..100 maps to inverse PWM ramp step DEFAULT→MIN; `map_ui8` clamps endpoints | E:600–601 ↔ S:1634–1641; `common.c:36–50`; `main.h:54–55` |
| 1 /5 | Unused normally. Under DEADTIME_TEST and calibration, deadtime register receives clamp(uint8(24+raw),14,32); otherwise24 | E:605–612 ↔ S:1644–1663 |
| 1 /6 bit0,bit7 | Field weakening enable / OTA watchdog reset requests only acted on while `!ui8_motor_enabled`; other bits unused | E:617–623 ↔ S:1665–1682 |
| 1 /7–8 | Wheel perimeter in mm, speed computation uses it | E:625–627 ↔ S:1684–1685,1036–1045 |
| 1 /9 | Max km/h; current ramps to0 across max−2..max+2km/h. Value0 disables this limiter | E:629–633 ↔ S:1687–1688,1025–1033 |
| 1 /10 | Nonzero threshold≤100 allows synthetic cadence at rest if torque_delta_ADC >110−threshold. >100 resets threshold0 | E:635–636 ↔ S:1690–1693,525–529,613–617,701–705 |
| 2 /5 | Entire byte optional ADC function0 unused/1 temperature/2 throttle, not masked despite comment | E:639–644 ↔ S:1696–1698,427–435; `common.h:35–37` |
| 2 /6,7 | Temperature min/max in°C. If min≥max target current0; otherwise taper current between min×10 and max×10 | E:646–649 ↔ S:1700–1704,1000–1022 |
| 2 /8,9 low7 | 15-bit torque zero offset assigned only if byte9 bit7 set; also updates coaster threshold offset−40 floored0. Clearing enable bit does not restore/recalculate offset here | E:651–656 ↔ S:1706–1713; `main.h:120` |
| 2 /10 | Torque conversion factor, Nm×100 per torque delta ADC step; torque_x100=delta×factor | E:658–659 ↔ S:1715–1716,1241–1242; `main.h:168–175` |
| 3 /5–10 | Hall reference array order6,2,3,1,5,4; byte5=0 selects all default phase references, otherwise copies all6 bytes | E:662–685 ↔ S:1719–1736; `motor.c:241–312`; `main.h:25–31` |
| 4 /5–10 | Same Hall order, interpolation counter offsets in4µs ticks; byte5=0 selects all default UP/DOWN offsets | E:687–694 ↔ S:1739–1755; `timers.c:54–63`; `main.h:111–113` |
| 5 /5 | Torque Smooth enable, any nonzero enables in controller; ESP32 sends0/1 | E:697–702 ↔ S:1758–1759,1223 |
| 5 /6 | Minimum torque delta ADC threshold; smoothing requires clipped delta strictly greater than min | E:703 ↔ S:1760,1224 |
| 5 /7 | Max absolute difference from previous stroke same-position ADC sample; smoothing requires difference strictly less than max | E:704 ↔ S:1761,1225–1232 |
| 5 /8,9,10 | **No semantic reads. Explicit Unused comments; ignored by ID5 parser.** Included in generic CRC, so “unused” does not mean CRC-exempt | E:696–705 ↔ S:1758–1766; CRC loop S:1585–1586 |

Torque offset startup calibration runs over cycles161..199 and adds6 at200
(nominal4–5sec, S:1106–1107,1192–1207). A flagged configuration assignment is not
a persistent mode variable preventing that startup calculation. Hall angles use
256 codes/electrical revolution: one code=360/256 degrees; they are circular codes,
not literal degrees (`motor.c:24`,241–252,377; `main.h:25–31`). FOC angle uses the
same angular code added at `motor.c:377`. FOC coefficient computes estimated phase
current from `(filtered_current<<8)/duty`, multiplies coefficient/256, caps15 and
filters (active assembly `motor.c:682–730`); it is a dimensionless tuning coefficient.
Physical inductance calibration remains UNKNOWN. Hall counter tick4µs is proved
by16MHz/prescaler64 in `timers.c:57–63`.

### Lights byte4

**VERIFIED BOTH SIDES:** bit7 requests on/off; bits6:0 are one enum, not seven
independent flags (E:569–572 ↔ S:1602–1607). **STM8-SIDE ONLY:**
`ebike_control_lights`, S:1322–1485:

| Config | Enabled, no brake | Enabled, brake | Disabled, brake |
|---|---|---|---|
| 0 | On | On | Off |
| 1 | Normal flash | Normal flash | Off |
| 2 | On | Brake flash | Off |
| 3 | Normal flash | On | Off |
| 4 | Normal flash | Brake flash | Off |
| 5 | On | On | On |
| 6 | On | Brake flash | Brake flash |
| 7 | Normal flash | On | On |
| 8 | Normal flash | Brake flash | Brake flash |
| Other | On | On | Off |

Disabled/no brake always off. Counters advance every25ms controller call, not
100ms despite stale comments: normal flash ON4 calls/OFF2, brake flash ON2/OFF2
(S:1323–1370; `main.c:102–111`). These are nominal100/50ms and50/50ms phases.
However selected WIP sets brake0, so brake columns are dormant in normal execution.

## 5. Status generation and actual telemetry sources

**VERIFIED BOTH SIDES:** S:1824–1974 ↔ E:368–500 agrees on29 bytes, header0x43,
packed byte1, all common fields, normal/Hall overlay and CRC. Below are controller
origins (**STM8-SIDE ONLY** refinements); physical calibration accuracy remains
unmeasured. All multi-byte wire fields are LE.

| Offset | Controller value/origin and actual scaling | STM8 source/function |
|---|---|---|
| 1 bits1:0 | 1 when mode7, else0; never emits2/3 | `uart_send_package`, S:1831–1836 |
| 1 bit2 | `ui8_rx_error` from command loss counter, not direct UART peripheral parity/overrun report | S:1841–1843,1550–1570 |
| 1 bit3 | `ui8_brake_state`; selected WIP forces0 | S:1839–1840; `motor.c:742–753` |
| 1 high nibble | `ui8_system_state<<4`, defined codes0/1/2/7 | S:1844–1845; `common.h:23–26` |
| 2–3 | Filtered battery mV: accumulator−(acc>>2)+ADC_voltage, then `(acc>>2)*87`, narrowed16-bit | S:1086–1102 `get_battery_voltage`,1847–1849; `main.h:179` |
| 4 | Filtered battery current in0.1A: filtered_ADC×16/10 | S:364–365,1851–1852; `motor.c:671–679` current filter; `main.h:194` |
| 5–6 bits2:0 | Speed_x10=`perimeter_mm*((PWM_CYCLES_SECOND/1000)*36)/wheel_ticks`; zero if no tick interval; packed low11 bits | S:1036–1045 `calc_wheel_speed`,1854–1859 |
| 6 bits7:3 | FOC angular code,360/256deg per code; packed5-bit capacity, controller computes/clamps filtered input≤15 | S:1859; `motor.c:377,682–730` |
| 7 | Cadence RPM=`PWM_CYCLES_SECOND*3/cadence_ticks`,0 without interval; derived from20 magnets; startup/synthetic cadence possible | S:1049–1083 `calc_cadence`,1862; `motor.c:893–909` |
| 8–9 | Pedal torque Nm×100=nonnegative deltaADC×conversion factor, with startup/smoothing behavior | S:1192–1242 `get_pedal_torque`,1864–1866; `main.h:168–175` |
| 10 | Whole°C, filtered temp_x10/10. ADC temp_x10=`filtered_optional_ADC*10000/2048`; updated only in temp-control function | S:1000–1010 `apply_temperature_limiting`,1868–1869 |
| 11 | Raw duty register control code, normal target cap254; fraction used as /256 in phase modulation, not a direct percent | S:1872,457–459; `main.h:79`; `motor.c:414–469` (active multiplication/high-byte assembly) |
| 12–13 | ERPS=`(250000>>2)/(hall_total_ticks>>2)` unless high sign bit of interval set, then0 | S:347–353,1874–1876; `main.h:33` |
| 14 | Field weakening counter offset,4µs per tick; current code max6 (24µs) | S:1878–1879; `main.h:113`; `motor.c:796–806` |
| 15 | Normal mode numeric FW_VERSION17; no packed major/minor | S:1883–1885; `main.h:15` |
| 16–17 | Raw right-aligned10-bit torque ADC channel4 | S:1887–1890; `adc.c:26`; `motor.c:667–668` |
| 18 | Optional ADC channel7 right-aligned10-bit value>>2, thus8-bit | S:1892–1893; `motor.c:669–670`; `adc.c:26` |
| 19 | Only THROTTLE_CONTROL: optional_ADC>>2 clamped/mapped47..176→0..255; all other functions transmit0 (not temperature control level) | S:1895–1899,959–965 `apply_throttle`; `main.h:136–137` |
| 20 | Cached Torque Smooth application percentage: `(smooth_count*10)>>3`, count reset each20 normal TXs; new percentage assigned after current frame field copy | S:1901–1902,1936–1941;1223–1232 |
| 21–26 | Selector data below, or Hall overlay in mode7 | S:1904–1962 |
| 27–28 | CRC over0..26, low then high | S:1966–1974 |

ADC hardware is10-bit (`STM8S_StdPeriph_Lib/inc/stm8s_adc1.h` ADC API; right alignment
`adc.c:26`); battery voltage/torque/optional are copied from buffered ADC registers
in active assembly `motor.c:665–670`. Current uses the low register byte then
filters in8-bit state; high current behavior must not be inferred from an ideal
unbounded10-bit measurement. Calibration constants are firmware assumptions,
not verified sensor readings. Speed uses integer `PWM_CYCLES_SECOND/1000`,
explicitly noted by source; do not silently replace it with floating point.

### Extended selector0/1/2/3

**VERIFIED BOTH SIDES** field layouts: E:459–488 ↔ S:1904–1933. Normal packets
alternate selector0 (odd incremented send count) with selector1 by default,
selector2 if TIME_DEBUG, otherwise selector3 if HALL_DEBUG. They do not cycle
through all four selectors at runtime. Both debug macros are commented out in
`src/main.h:12–13`; TIME_DEBUG takes precedence if both externally defined.

| Selector | Controller meaning (**STM8-SIDE ONLY**) | Source |
|---|---|---|
| 0 /22–24 | Wheel sensor count: low20 bits of32-bit counter; byte24 masked0x0f (high nibble always0). ESP32 accepts24 bits | S:1906–1909; `motor.c:830–845` accepted rising edges |
| 0 /25–26 | Low16 bits of crank_x20/20, whole crank revolutions | S:1911–1914; `motor.c:893–895`; `main.h:148` |
| 1 /22,23,24 | AVG/MIN/MAX torque delta ADC, not Nm; prior completed stroke from80 transition samples. Samples clipped255 for min/max/buffer; sum uses unclipped delta, average narrowed8-bit | S:1122–1185 `new_torque_sample`,1930–1932 |
| 1 /25,26 | Not assigned by this branch; stale buffer values, often prior selector0 crank bytes | S:1928–1933 |
| 2 /22 | Max ebike-loop execution time from TIM4 counter, nominal ms (stale comment says PWM50µs) | S:1919; `main.c:105–116` |
| 2 /23–24 | LE16 maximum PWM down ISR timing expression | S:1804–1811,1920–1921 |
| 2 /25–26 | LE16 maximum PWM up ISR timing expression | S:1813–1820,1922–1923 |
| 3 /22,23 | Hall invalid-state error count / unexpected-sequence count, unsigned8-bit wrapping | S:1925–1927; `motor.c:259–321` |
| 3 /24–26 | Not assigned in this branch, stale values | S:1924–1927 |

TIME_DEBUG formulas use direction-tagged TIM1 counts and hardcoded210/630;
counter clock16MHz implies62.5ns/count, but current PWM max444 differs from old
420-era comments/constants. Also `main.c:114` lacks a semicolon inside TIME_DEBUG.
That disabled configuration was **not build-verified**; do not treat those maxima
as calibrated duration. Exact usable physical debug durations remain UNKNOWN.

### Hall calibration packet

**VERIFIED BOTH SIDES:** mode7 switches type1, replaces15..26 with six LE16 Hall
intervals (S:1831–1833,1944–1962 ↔ E:491–498). **STM8-SIDE ONLY**: each interval
is last transition counter minus previous transition counter, modulo16-bit, in4µs
ticks. Order indices0..5 is Hall states6,2,3,1,5,4 (motor.c:241–312), i.e. intervals
ending at those states, not absolute rotor angles. Normal selectors/version do
not coexist in that overlay.

## 6. CRC cross-check

**VERIFIED BOTH SIDES:** STM8 `src/common.c:92–104`, `crc16`, and ESP32
`main/tsdz_utils.c:86–97` implement the same byte XOR / eight right-shift loop
with0xA001. Both callers initialize0xffff. Parameters: width16, canonical
poly0x8005, refin/refout=true, xorout0, no final reversal/augmentation; equivalent
CRC-16/MODBUS. RX13 covers0..10 and expects LE CRC11/12; TX29 covers0..26 and
appends LE CRC27/28 (S:1583–1591,1966–1974; E:708–717; ESP32 UART:243–256).

Both original C function bodies were extracted with balanced-brace scanning,
renamed to avoid symbol collisions, compiled together to wasm with local Clang,
and executed under Node. Only standalone iteration wrappers were added outside
this repository. The three contract vectors passed for **both** implementations:

| Input | CRC | Low/high wire bytes |
|---|---|---|
| ASCII `123456789` | 0x4B37 | `37 4B` |
| `59 00 00 00 00 22 01 0F 14 1B 19` | 0x119A | `9A 11` |
| `43` followed by26 zero bytes | 0x1198 | `98 11` |

Additional1024 deterministic inputs matched: for n=0..1023 use length n%257;
byte[j]=(n*73+j*29)&255; init0xffff. These checks prove function agreement,
not whole-firmware execution or device timing. The original contract reproduction
can be run for STM8 by extracting `src/common.c` through its matching closing
brace rather than assuming it is the last function.

## 7. Status transmit timing

**STM8-SIDE ONLY:** `src/main.c:101–111` runs `ebike_app_controller` every nominal
25ms; `src/timers.c:71–90` supplies TIM4 ticks. S:374 calls communications every
controller iteration. S:1556–1558 attempts TX every fourth call, including first
call because counter starts0: nominal100ms/10Hz, independent of receiving commands.
`uart_send_package` returns early if previous TX counter≤29 (S:1797–1799), and
UART TX ISR sends29 bytes then disables itself and increments counter to30
(S:1512–1522,1976–1979). No per-command immediate response is required.

These are source scheduling intervals, not measured wall-clock frequency; interrupt
latency, tick quantization and blocking can alter them. Selector0 and the selected
alternate are each nominal5Hz. Hall mode uses the same TX scheduler.

## 8. Command loss/timeout

**STM8-SIDE ONLY**, S:`communications_controller`:1546–1570:

- Increment uint8 missing-RX counter every25ms call, then process available packet.
  Any CRC-valid13-byte frame resets counter and RX error (S:1592–1594).
- At counter>20 (first21st missed call, nominal525ms), set `ui8_rx_error=1`.
  POWER1, TORQUE2, CADENCE3 and eMTB4 **retain their mode**, assist and configuration.
  Other modes (WALK/CRUISE/CALIBRATION/unknown) are set OFF.
- Status TX is attempted **before** that timeout check, so newly-set error/mode
  may first appear on a later scheduled frame. Valid reception clears error before
  the send step and can report recovery that iteration.
- Counter wraps modulo256 (nominal6.4sec). Error is not cleared by wrap because
  only valid RX clears it; normal retained modes are not changed by this branch.
- Bad CRC does not reset counter. This “controller UART RX error” is missing valid
  command traffic, not a direct hardware CRC/parity IRQ indicator.

**ESP32-SIDE ONLY:** its independent >500ms controller-status timeout sets local
system bit7. Pausing Fake STM8 TX exercises that gateway timeout; pausing ESP32
commands exercises the controller behavior above. Do not conflate them or assume
all assist stops on loss. This policy changed from v16's older all-mode OFF logic.

## 9. Controller status/error and conflicts

**VERIFIED BOTH SIDES:** wire byte1 type bits1:0, RX error bit2, brake bit3,
error high nibble match E:369–383. **STM8-SIDE ONLY:** only four system-state
codes are defined (`src/common.h:23–26`, S:`check_system`:1250–1319):

| Code | Name / real trigger |
|---|---|
| 0 | NO_ERROR |
| 1 | MOTOR_BLOCKED: current>5.0A and ERPS<10 for >20 controller checks; reset after >200 checks (nominal525ms/5.025sec) |
| 2 | TORQUE_SENSOR: offset>300 or<10 or raw torqueADC>500, only modesPOWER/TORQUE/eMTB; clears when predicate ceases |
| 7 | BATTERY_OVERCURRENT: raw ADC low byte≥156, latched because check returns early while code7 |

Other nibble values are unassigned by this pinned controller source, not invented
faults. ESP32 codes8 overvoltage,9 temperature limit,10 temperature max are
**ESP32-SIDE ONLY** synthesized after a zero received nibble (E:419–427), not STM8
defined/emitted errors in this revision. Command loss uses bit2, not another nibble.

No framing/CRC/ID byte-map conflict was found. The following asymmetries/behavior
limitations must remain explicit rather than calling all semantics identical:

1. **WIP brake behavior:** `motor.c:742–753` comments out physical/coaster brake
   detection and writes `ui8_brake_state=0`; `main.c:82–85` also disables startup
   brake hold. Wire bit exists and gateway parses it, but real brake assertion
   will not appear from this selected WIP. Commit message explicitly acknowledges
   temporary brake bypass. No source was repaired here.
2. **Optional field semantic correction:** ESP32 E:451–453 comment says throttle
   or temperature mapped0..255; STM8 S:1895–1899 sends throttle only, else0.
3. **Width asymmetry:** ESP32 selector0 decodes24-bit wheel counter; STM8 masks it
   to20 bits. Compatible representation but effective wrap range2^20 counts.
4. **Controller safety policy:** v17 retains normal assist modes on command loss.
   ESP32 does not guarantee STM8 accepted OTA flags; only !motor_enabled STM8
   processes OTA. Thus cached “stationary” values plus gateway handoff cannot
   prove STM8 actually reset.
5. **Disabled debug configuration:** TIME_DEBUG missing semicolon and stale timing
   constants prevent treating it as a validated real-time diagnostic interface.
6. **Hall zero semantics:** STM8 treats first payload byte0 as reset of the whole
   reference/offset set. ESP32 validates references, then adds phase adjustment;
   an all-zero reference set plus nonzero adjustment no longer encodes a zero reset
   (E:663–685). This is a reachable semantic edge, not a different byte layout.

## 10. OTA reset and available flashing source

**VERIFIED BOTH SIDES:** ESP32 ID1 byte6 bit7 requests reset twice while its status
counter advances, then UART waits TX complete and suspends main task to run OTA
(contract G; ESP32 UART:90–103; E:617–620). STM8 S:1665–1682 acts only while
`!ui8_motor_enabled`. Each valid flagged ID1 increments `wwdg_reset_counter`;
at≥2 it writes `WWDG->CR=WWDG_CR_WDGA` to trigger watchdog reset. Valid ID1 without
bit7 while disabled resets this counter0. Intervening IDs0/2/3/4/5 do not reset it;
motor-enabled ID1 does not process the flag/counter at all. This is not “any two
adjacent packets”. Field weakening bit0 is gated by the same disabled condition.

**STM8-SIDE ONLY:** watchdog reset trigger source is present; custom bootloader
server source is **not found** in current tracked tree or boot/ota-named history
paths. The project's flash wiki requires enabling chip bootloader option bytes.
Do not invent a software jump to a custom loader or claim ROM source was audited.

**ESP32-SIDE ONLY:** existing flashing client is present in
`esp32-gateway/main/tsdz_ota_stm8.c` and `main/stm8s/E_W_ROUTINEs_32K_ver_*.h`:

| Stage | Existing client behavior / source function |
|---|---|
| Entry | `stm8_ota_task`:725–737 waits300ms after handoff, then syncs |
| Sync | `bsl_sync`:124–178 sends0x7f, waits ACK0x79/NACK0x1f with retry/echo handling, flushes and sendsACK; returns1 only forACK despite comment |
| Receive mode | `receive_port`:86–111, mode2 echoes each received byte; not the normal29-byte status protocol |
| Identify | `bsl_getInfo`:313–416 probes address0x00ffff via READ0x11, requests GET0x00; supported BSL versions0x10/12/13/14 select matching RAM payloads (831–855) |
| Upload/write | `bsl_memWrite`:433–599 sends WRITE0x31+complement,32-bit BE address+XOR, length−1+data+XOR, checkingACK at each stage; blocks≤128 aligned to128 |
| Image flow | `stm8_ota_task`:740–873 downloads≤32KiB, uploads matching RAM routine, then writes firmware at0x8000; no separate ERASE command issued in this path |
| Start app | `bsl_jumpTo`:612–691 sends GO0x21+complement, BE32 address+XOR; caller uses0x8000 at876; success/error notifications then ESP32 restart |

RAM erase/write payload headers contain byte arrays and start/end addresses,
not human-readable source of ROM bootloader or those routines. Generic ST flash
peripheral library files do not establish a bootloader UART server. Actual ROM
internals, selected hardware BSL version, option-byte state, acceptance and timing
remain **UNKNOWN** without further authoritative source/device verification.
No new OTA protocol, bootloader code or flashing attempt was made.

## 11. Fake STM8 compatibility requirements

Future specification only; **no simulator implementation in this change**.

- Match pinned revision explicitly: advertise numericFW17 when normal v17 profile
  selected, not a guessed dotted version. Label it a WIP-derived protocol profile.
- Receive0x59/13-byte framing, CRC-check all11 prefix bytes, parse common fields
  and IDs0..5 with correct bit/LE semantics. Ignore ID5 trailing8..10 semantically
  while retaining them for CRC/logging. Do not enforce a fictitious all-zero tail.
- Model consumer clamps, units and sentinel/reset behavior in section4, including
  first Hall byte0 resetting whole arrays and flagged torque offset assignment.
  eMTB is LUT selection, WALK/CALIBRATION parameters are duty codes, not percentages.
- Send29-byte status nominally every4×25ms, independent of received commands;
  avoid overlapping sends. Normal packets alternate selector0 and configured
  selector1/2/3, not a four-selector round robin. Hall type1 replaces15..26 in mode7.
- Preserve source truncations: speed11-bit, wheel count20-bit, whole crank16-bit,
  optionalADC>>2, temperature whole°C, throttle-only byte19, raw duty/angle codes.
  Use Hall transition intervals in4µs ticks and state order6,2,3,1,5,4.
- Distinguish sensor rawADC, converted telemetry, gateway voltage compensation and
  error synthesis. Keep torque ADC statistics separate from Nm×100 torque. Model
  Torque Smooth update every20 normal sends with cached percentage timing.
- Implement controller command-loss behavior separately from gateway-status loss:
  after21 missing nominal25ms checks, set wireRX-error bit and OFF only non-normal
  modes; retain normal modes1..4. Valid CRC command clears error; note counter wrap
  and send-before-timeout-check ordering. A stricter test mode must be labeled as
  an intentional deviation rather than claimed authentic v17 behavior.
- For source-faithful v17 mode, default brake bit0 because WIP bypasses brakes.
  Explicit brake fault injection should remain available as a **test override**
  for gateway parser testing, clearly distinguished from authentic default output.
- Inject named controller codes1/2/7 and separate RX-error/brake bits; local gateway
  8/9/10 and timeout0x80 are not controller nibble definitions. Provide deliberate
  invalid CRC/drop/partial packets, pause statusTX and serial timestamp/raw/decoded logs.
- Recognize OTA reset flag only while simulated motor disabled and after two
  flagged ID1 receptions; do not reset count on unrelated IDs. Log/stop normal
  telemetry for simulated reset as a controlled future test; do not claim or
  implement a guessed ROM flashing server.

## 12. Remaining unknowns and validation

**UNKNOWN:** electrical tolerance/wiring acceptance; calibrated sensor accuracy,
real device scheduler timings and motor behavior; physical DEADTIME_TEST timing;
physical inductance represented by FOC coefficient; stable released-v17 binary correspondence; ROM bootloader
source/behavior and current hardware option bytes; usable physical TIME_DEBUG
durations in its disabled/unbuilt configuration. Unsupported types2/3, selectors
outside0..3 and undefined error nibble values have no semantics in this source.
Unused bytes have no assigned field meaning; this is resolved non-use, not a
license to invent a meaning. All claims are scoped to the two pinned revisions.

Audit evidence: reciprocal source review, full STM8 RX/TX paths and callers,
historical diffs against v16, CRC code execution (three shared vectors plus1024
inputs). No motor/controller firmware build, device test or OTA flash is claimed.
Only the two documentation files may differ from prior audit commit
`4addc25bf37ad2f6232863c9b7517b695a290e99`; existing audit history is preserved.
