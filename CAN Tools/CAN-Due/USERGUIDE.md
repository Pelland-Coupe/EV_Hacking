# GVRET Firmware User Guide

**Generalized Electric Vehicle Reverse Engineering Tool**
Firmware version: `GVRET alpha 2017-11-09` — build **343**, EEPROM version `0x17`
Authors: Collin Kidder, Michael Neuweiler, Charles Galpin (MIT license)

This guide documents how to *use* the firmware as it exists in this source tree,
including every function that can be triggered by a signal on a physical pin and
every signal the firmware can generate on a pin. It was produced by analysis of
the source code; file/line references are given where they matter.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Supported Hardware](#2-supported-hardware)
3. [Quick Start](#3-quick-start)
4. [Pin Reference](#4-pin-reference)
   - 4.1 [Inputs — functions triggered by signals on pins](#41-inputs--functions-triggered-by-signals-on-pins)
   - 4.2 [Outputs — signals the firmware creates on pins](#42-outputs--signals-the-firmware-creates-on-pins)
5. [Text Console Reference](#5-text-console-reference)
6. [Lawicel / SLCAN Mode](#6-lawicel--slcan-mode)
7. [Binary GVRET Protocol](#7-binary-gvret-protocol)
8. [SD Card Logging](#8-sd-card-logging)
9. [EEPROM Settings & Factory Reset](#9-eeprom-settings--factory-reset)
10. [Known Quirks & Bugs in This Build](#10-known-quirks--bugs-in-this-build)
11. [Verifying the Firmware on Your Hardware](#11-verifying-the-firmware-on-your-hardware)

---

## 1. Overview

GVRET turns an Arduino-Due-class board into an autonomous CAN bus
reverse-engineering tool. After configuration it runs stand-alone: plug it into
a vehicle bus and it logs traffic to an SD card with no host connected.

Main capabilities:

* **Two high-speed CAN channels** (CAN0, CAN1) via the Due's built-in CAN
  controllers, plus a **single-wire CAN (SWCAN)** option:
  * On CANDue ≤ 2.1 boards, CAN1's transceiver can be switched into single-wire
    mode (bus 1 doubles as SWCAN).
  * On CANDue 2.2+ boards a dedicated MCP2515-based SWCAN channel exists
    (reported as bus 2).
* **Automatic frame pass-through** between CAN0 and CAN1 (hardware gateway),
  gated by jumper pins (see §4.1).
* **SD card logging** in three formats: Binary, GVRET CSV, CRTD (§8).
* **USB serial interface** with three personalities:
  * human-readable text console + frame dump (default),
  * Lawicel/SLCAN-compatible command set (§6),
  * the binary GVRET protocol spoken by SavvyCAN (§7).
* **Digital I/O features**: 4 digital inputs, 4 analog inputs, 8 digital
  outputs, and a configurable "digital toggle" bridge between one Arduino pin
  and one CAN identifier in either direction (§4).
* All settings persist in EEPROM; factory reset via console command.

---

## 2. Supported Hardware

The board type is stored in EEPROM as `SYSTYPE` and selects pin maps, LED
assignments and bus count (`loadSettings()` in `GVRET-gvret.ino`).

| SYSTYPE | Board | CAN buses | SD card | Notes |
|--------:|-------|----------:|:-------:|-------|
| 0 | CANDue (original) | 2 (+SW via CAN1) | yes | Factory default |
| 1 | GEVCU | 2 | no | Transceivers have no enable pins |
| 2 | CANDue 1.3 – 2.1 | 2 (+SW via CAN1) | yes | |
| 3 | CANDue 2.2 | 3 (dedicated SWCAN) | yes | MCP2515 on SPI for SWCAN |

### Per-board pin assignments (`config.h`)

| Function | CANDue (0) | GEVCU (1) | CANDue 1.3–2.1 (2) | CANDue 2.2 (3) |
|---|---:|---:|---:|---:|
| EEPROM write-protect | 18 | 19 | 18 | 18 |
| CAN0 transceiver enable | 50 | n/a | 50 | 50 |
| CAN1 transceiver enable | 48 | n/a | 48 | 48 |
| SWCAN mode pin 0 | 46 | n/a | 46 | 46 |
| SWCAN mode pin 1 | 44 | n/a | 44 | 44 |
| SD card chip-select | 10 | n/a | 10 | 10 |
| LED CAN TX | 73 | 13 | 13 | 13 |
| LED CAN RX | 72 | 13 | 13 | 13 |
| LED logging | 13 | none | 13 | 13 |

CANDue 2.2 additional MCP2515 (SWCAN) pins: RESET = 32, CS = 34, INT = 38
(interrupt-driven, falling edge), RXB0 = 41, RXB1 = 40.

### Required libraries to compile (from README.md)

`due_can`, `can_common`, `MCP2515`, `due_wire`, `SdFat` (beta),
`DueFlashStorage`, `Wire_EEPROM` — installed in the Arduino libraries folder
with exactly these names (no `-master` suffixes). Build in Arduino IDE 1.5.4+.

---

## 3. Quick Start

1. Set `SYSTYPE` correctly once (text console: `SYSTYPE=2`, then power-cycle).
2. Wire CAN0-H/L (and CAN1-H/L if used) to the vehicle bus. The bus must be
   terminated at both ends (~120 Ω each end) — the device does not add
   termination.
3. Connect the USB **native port** to a PC. `SerialUSB` is the main interface;
   the programming port (`Serial`, 115200 baud) only carries early debug text.
4. Open a serial terminal (or SavvyCAN). Send `h` + line ending to see the menu
   and build number. Line endings (LF, CR, CRLF) are required for all
   text commands.
5. Typical first-time setup:

   ```
   CAN0SPEED=500000        set bus speed
   CAN0EN=1                enable CAN0
   FILETYPE=2              log as GVRET CSV
   FILEBASE=CANBUS         base file name
   s                       start logging to SD
   MARK=Key on             annotate the log
   S                       stop logging
   ```

6. For SavvyCAN: simply connect over serial USB — SavvyCAN sends `0xE7`,
   which switches the device into binary protocol mode automatically (§7).

---

## 4. Pin Reference

### 4.1 Inputs — functions triggered by signals on pins

#### Pass-through gate pins 11 and 12 (all boards)

`ENABLE_PASS_0TO1_PIN = 11`, `ENABLE_PASS_1TO0_PIN = 12`
(`config.h`, used in `loop()` at `GVRET-gvret.ino:628` and `:637`).

Both are inputs with internal pull-ups enabled at boot, so **floating =
function active**.

| Pin | Signal condition | Effect |
|----:|------------------|--------|
| 11 | HIGH (default, pull-up) | Every frame received on **CAN0 is re-transmitted on CAN1** |
| 11 | LOW (shorted to GND) | CAN0 → CAN1 forwarding disabled |
| 12 | HIGH (default, pull-up) | Every frame received on **CAN1 is re-transmitted on CAN0** |
| 12 | LOW (shorted to GND) | CAN1 → CAN0 forwarding disabled |

This lets you use the board as a wire-in gateway/recorder and cut the
forwarding path with a jumper without touching software. There is **no
debouncing** on these pins (they are intended as static jumpers).

> Warning: forwarded frames are re-transmitted as-is. Forwarding between two
> real buses merges their traffic; make sure that is what you want.

#### Digital toggle system — pin-to-CAN direction (mode 0)

Configured by the `DIGTOG*` console commands (§5). When
`DIGTOGEN=1` and `DIGTOGMODE=0`, the pin number given by `DIGTOGPIN` becomes a
digital input watched by the main loop (`GVRET-gvret.ino:654-672`):

* The pin is polled every loop iteration; a transition is only accepted after
  **4 consecutive identical opposite readings** (simple debounce).
* On **every accepted transition (both rising and falling edges)** the firmware
  transmits the configured frame — ID `DIGTOGID`, payload `DIGTOGPAYLOAD`,
  length `DIGTOGLEN` — on each bus flagged with `DIGTOGCAN0` / `DIGTOGCAN1`.
* The same payload is sent regardless of edge direction; the receiver cannot
  tell rising from falling.
* If `DIGTOGID` > 0x7FF the frame is sent as 29-bit extended, otherwise 11-bit.

Use case: wire a switch/sensor/ECU pin to an Arduino pin and record its state
changes as CAN frames on the bus, synchronized with logged traffic.

#### Digital inputs 0–3 (Arduino pins 48, 49, 50, 51)

Four digital inputs mapped in `sys_early_setup()` (`sys_io.cpp:64-67`). They
are **active-low**: `getDigital()` returns `true` when the pin reads LOW
(`sys_io.cpp:161`). They are read on demand by binary protocol command
`0x02` (§7) which returns them packed as bits 0–3 of one byte.

> **Hardware conflict warning:** on CANDue boards, pins 48 and 50 are also the
> CAN1/CAN0 transceiver-enable outputs (§4.2). Reading "digital inputs" 0 and 2
> on those boards actually reports the transceiver enable lines. The four
> digital inputs are only meaningfully usable on GEVCU hardware (or boards
> wired accordingly).

#### Analog inputs A0–A3

Four ADC channels are continuously sampled by a DMA-driven ADC
(`setupFastADC()`, `sys_io.cpp`) with heavy averaging (~3 ms window).

They are returned by binary protocol command `0x03` as four 16-bit values.

> **This build's analog inputs are non-functional:** the call to
> `sys_io_adc_poll()` is commented out in `loop()` (`GVRET-gvret.ino:1157`),
> so the processed values never update and command `0x03` always returns
> zeroes. See §10.

#### Commented-out feature: mark trigger on digital input 0

`loop()` contains a disabled block (`GVRET-gvret.ino:611-622`) that would have
inserted a "MARK TRIGGERED" event when digital input 0 changed. It is not
functional in this build.

### 4.2 Outputs — signals the firmware creates on pins

#### Eight general-purpose outputs (Arduino pins 4, 5, 6, 7, 2, 3, 8, 9)

Mapped in `sys_early_setup()` (`sys_io.cpp:76-83`) as outputs 0–7, initialized
LOW at boot. Two ways to drive them:

| Method | How | Detail |
|---|---|---|
| Binary protocol `SET_DIG_OUT` (cmd 0x04) | one bitmask byte | bit *c* = output *c*: 1 = HIGH, 0 = LOW. Sets all 8 at once |
| Text console short commands | `K` / `J` | `K` = all 8 HIGH, `J` = all 8 LOW |

Typical use: driving external relays/LEDs or biasing circuits under test while
recording bus reactions.

#### Digital toggle system — CAN-to-pin direction (mode 1)

When `DIGTOGEN=1` and `DIGTOGMODE=1`, `DIGTOGPIN` becomes an **output**
(`GVRET-gvret.ino:309-325`):

* Startup level: if `DIGTOGLEVEL=1` (mode bit 7 set) the pin is driven **LOW**
  at boot; if `DIGTOGLEVEL=0` it is driven **HIGH**. (Note: this is the
  *opposite* of what the comment in `config.h` says — see §10.)
* While the loop runs, every received frame on the participating buses is
  compared against the configured filter:
  * participates in CAN0 only if `DIGTOGCAN0=1`, in CAN1 only if
    `DIGTOGCAN1=1`;
  * the CAN ID must equal `DIGTOGID`;
  * if `DIGTOGLEN` > 0, the first `DIGTOGLEN` data bytes must equal
    `DIGTOGPAYLOAD`; if `DIGTOGLEN=0`, any payload matches.
* On a match the pin **toggles** (HIGH→LOW→HIGH…) — `processDigToggleFrame()`,
  `GVRET-gvret.ino:542-563`.

Use case: detect on a physical pin (scope/logic analyzer/LED) that a specific
CAN message with specific content occurred — e.g. trigger a capture or drive
an actuator when a particular gear/state message appears.

#### Single-wire CAN mode pins 46 and 44 (CANDue boards)

`SWCANMode0Pin` / `SWCANMode1Pin` control the single-wire transceiver
(`setSWCANSleep()` / `setSWCANEnabled()` / `setSWCANWakeup()`,
`GVRET-gvret.ino:245-265`). You normally do not drive these yourself; the
firmware sets them according to `SINGLEWIRE` state:

| State | Pin 46 (mode0) | Pin 44 (mode1) | CAN1 enable pin (48) |
|---|---|---|---|
| SWCAN sleeping (normal high-speed CAN1) | LOW | LOW | HIGH |
| SWCAN enabled | HIGH | HIGH | LOW |
| SWCAN wakeup pulse | LOW | HIGH | unchanged |

On CANDue 2.2 (dedicated SWCAN) the CAN1 enable pin is left alone; the
MCP2515 channel is used instead.

#### Special case: SWCAN wakeup frame (ID 0x100)

If single-wire mode is enabled and you transmit a frame with **ID 0x100** on
the SWCAN bus (bus 1 on shared-transceiver boards, bus 2 on dedicated), the
firmware emits a wakeup sequence around the frame:
`setSWCANWakeup()` → 1 ms delay → transmit → 1 ms delay → `setSWCANEnabled()`
(`GVRET-gvret.ino:879-899`). This mirrors the wake-up pattern required by many
GM/Daimler single-wire nodes.

#### CAN transceiver enable pins 50 / 48

Passed to `Can0.begin(speed, pin)` / `Can1.begin(speed, pin)`; the due_can
library drives them to enable the physical transceivers. Do not use these pins
for anything else (see digital-input conflict warning above).

#### Status LEDs

| LED | Driven when |
|---|---|
| LED_CANRX | toggles on **every received CAN frame** (`toggleRXLED()`) |
| LED_CANTX | toggles whenever a frame is sent via `CAN0SEND`/Lawicel/console transmit |
| LED_LOGGING | toggles on **every SD card flush** (≈ once per second while logging) |
| Blink LED (73) | set HIGH when `setup()` completes successfully |

Per-board pin numbers: see table in §2. On original CANDue, pins 72/73 LEDs
are active-low; pin 13 LED is active-high.

#### Pins driven internally (not user functions)

EEPROM write-protect (18/19), SD card SPI chip-select (10), MCP2515 CS/RESET/
INT (34/32/38) — listed for completeness so they aren't repurposed by accident.

---

## 5. Text Console Reference

Line-oriented, max 80 characters, terminated by LF, CR or CRLF. Dispatch rule
(`SerialConsole.cpp:146-160`):

* single character → **short command**
* contains `=` → **config command** (`NAME=value`)
* anything else → **Lawicel command** (§6)

### Short commands

| Command | Action |
|---|---|
| `h`, `H`, `?` | Print help menu + build number |
| `K` | Set all 8 outputs HIGH |
| `J` | Set all 8 outputs LOW |
| `R` | Invalidate EEPROM (factory defaults after power cycle) |
| `s` | Start logging to SD file |
| `S` | Stop logging to SD file |
| `O` | Lawicel: open CAN0, enter Lawicel mode |
| `C` | Lawicel: close CAN0 |
| `L` | Lawicel: open CAN0 (listen-only **not** actually implemented) |
| `P` | Lawicel: poll one frame (CR if none) |
| `A` | Lawicel: poll all frames (CR if none) |
| `F` | Lawicel: status → `F00` |
| `V` | Lawicel: version → `V1013`, enters Lawicel mode |
| `N` | Lawicel: serial number → `ND00D`, enters Lawicel mode |

### Config commands (`NAME=value`)

Values are parsed with `strtol(..., 0)` — decimal or `0x`-prefixed hex.
Unless noted, the new value is saved to EEPROM immediately.

| Command | Range | Description |
|---|---|---|
| `LOGLEVEL` | 0–4 | 0=debug, 1=info, 2=warn, 3=error, 4=off (console diagnostics) |
| `SYSTYPE` | 0–3 | Board type; **power cycle to apply** |
| `CAN0EN` / `CAN1EN` | 0/1 | Enable/disable the channel (re-begins or disables controller) |
| `CAN0SPEED` / `CAN1SPEED` | 1–1000000 | Bit rate in baud; applies immediately |
| `SWSPEED` | 1–1000000 | SWCAN bit rate (stored; used at MCP2515 init) |
| `CAN0LISTENONLY` / `CAN1LISTENONLY` | 0/1 | Listen-only (no ACK/dominate) mode |
| `CAN0FILTER0..7` | id,mask,ext,en | Hardware acceptance filter, e.g. `CAN0FILTER0=0x7FF,0x7FF,0,1`. mask 0 = accept all |
| `CAN1FILTER0..7` | id,mask,ext,en | Same for CAN1 |
| `CAN0SEND` | id,len,d0,d1,… | Transmit on CAN0, e.g. `CAN0SEND=0x200,4,1,2,3,4`. ID > 0x7FF ⇒ extended frame |
| `CAN1SEND` | id,len,d0,… | Transmit on CAN1 |
| `SWSEND` | id,len,d0,… | Transmit on SWCAN (MCP2515) |
| `MARK=text` | free text | Insert annotation into log (GVRET format: `Mark:` line; CRTD: `CEV` event); echoed to console |
| `SINGLEWIRE` | 0/1 | Single-wire mode flag (pin states change on next bus (re)start) |
| `BINSERIAL` | 0/1 | Persist binary-serial preference (normally managed automatically) |
| `FILETYPE` | 0–3 | Log format: 0=none, 1=binary, 2=GVRET CSV, 3=CRTD |
| `FILEBASE` | string | Base file name (default `CANBUS`) |
| `FILEEXT` | string | File extension (default `TXT`) |
| `FILENUM` | 0–65535 | Incrementing number used when not appending |
| `FILEAPPEND` | 0/1 | 0 = new numbered file each start, 1 = append to `BASE.EXT` |
| `FILEAUTO` | 0/1 | Start logging automatically at power-up |
| `DIGTOGEN` | 0/1 | Enable digital toggle system |
| `DIGTOGMODE` | 0/1 | 0 = watch pin, send CAN; 1 = watch CAN, toggle pin |
| `DIGTOGLEVEL` | 0/1 | Default/startup level (see polarity caveat §10) |
| `DIGTOGPIN` | 0–77 | Arduino pin used by the toggle system |
| `DIGTOGID` | < 0x40000000 | CAN ID for the toggle system |
| `DIGTOGCAN0` / `DIGTOGCAN1` | 0/1 | Which buses the toggle system listens/talks on |
| `DIGTOGLEN` | 0–8 | Payload length to send (TX) or validate (RX) |
| `DIGTOGPAYLOAD` | b0,b1,…,b7 | Payload bytes, comma separated |

Factory defaults (written when EEPROM version mismatches):
CAN0/CAN1 @ 500000 enabled, SWCAN 33333 disabled, filters accept-all,
`FILETYPE=3` (CRTD), text serial, no auto-log, `LOGLEVEL=1`.

---

## 6. Lawicel / SLCAN Mode

A subset of the Lawicel CANUSB command set is understood directly on the same
serial line (no mode switch needed for TX commands). Entering Lawicel mode
(via `O`, `L`, `V` or `N`) makes received frames print as `t`/`T` lines
terminated by CR.

| Command | Support | Behavior here |
|---|---|---|
| `t<III><L><DD…>` | yes | Transmit standard frame on **CAN0** |
| `T<IIIIIIII><L><DD…>` | yes | Transmit extended frame on CAN0 (note: `extended` flag bug — see §10) |
| `S<n>` | yes | Preset CAN0 speed: 0=10k, 1=20k, 2=50k, 3=100k, 4=125k, 5=250k, 6=500k, 7=800k, 8=1M |
| `X1` / `X0` | yes | Auto-poll flag (stored but RX gating is disabled in this build — frames stream anyway) |
| `Z1` / `Z0` | yes | Append 16-bit millisecond timestamp to RX lines |
| `O` / `L` / `C` | partial | Open / "listen-only" / close CAN0 |
| `P` / `A` / `F` | yes | Poll / poll-all / status |
| `V` / `N` | yes | Version `V1013`, serial `ND00D` |
| `U<n>` | ignored | USB CDC baud is fixed |
| `r`, `R`, `W`, `m`, `M`, `Q` | stubs | Accepted, no action |

Received-frame format (CR terminated):

```
t<id 3 hex><len><data bytes 2 hex each>[<timestamp 4 hex>]
T<id 8 hex><len><data bytes 2 hex each>[<timestamp 4 hex>]
```

To leave Lawicel-style streaming, either send `0xE7` (switches to binary
protocol) or power-cycle with `BINSERIAL=0`.

---

## 7. Binary GVRET Protocol

This is the protocol SavvyCAN speaks. Entry: host sends byte **0xE7** — the
device then enables binary comm, disables Lawicel mode and sets all CAN
filters to promiscuous (accept everything).

### Framing

```
0xF1 | CMD | payload… | CHECKSUM
```

* `0xF1` is the sync byte; bytes between a checksum and the next `0xF1` are
  discarded.
* CHECKSUM = XOR of all preceding bytes of the message.
* **In this build the device never verifies the incoming checksum, and its own
  CAN-frame messages carry a constant `0x00` checksum byte. Several reply
  messages omit the checksum entirely** (marked below). Parsers should rely on
  the documented lengths.

### Host → device commands

| Cmd | Name | Payload | Reply |
|----:|---|---|---|
| 0x00 | BUILD_CAN_FRAME | id (4 B LE, bit31 = extended), bus (1 B, 0/1/2), len (1 B, low nibble), data ×len, checksum | frame transmitted on selected bus |
| 0x01 | TIME_SYNC | (any 1 byte) | `F1 01 <micros 4 B LE>` immediately; host then sends 1 filler byte |
| 0x02 | DIG_INPUTS | — | `F1 02 <inputs bitfield> <cksum>` (bits 0–3 = digital inputs 0–3, 1 = pin LOW) |
| 0x03 | ANA_INPUTS | — | `F1 03 <A0 u16LE> <A1> <A2> <A3> <cksum>` (**always zero in this build**) |
| 0x04 | SET_DIG_OUT | bitmask (1 B) | outputs 0–7 set per bits (pins 4,5,6,7,2,3,8,9) |
| 0x05 | SETUP_CANBUS | CAN0 speed (4 B LE), CAN1 speed (4 B LE) | applies + saves EEPROM; see flag bits below |
| 0x06 | GET_CANBUS_PARAMS | — | `F1 06 <st0> <spd0 4B> <st1> <spd1 4B>` (12 B, **no checksum**); st = enabled \| listenOnly<<4 \| singleWire<<6 (bus 1) |
| 0x07 | GET_DEV_INFO | — | `F1 07 <build 2B LE> <eepromver> <filetype> <autostart> <singlewire>` (8 B, **no checksum**) |
| 0x08 | SET_SW_MODE | 1 B: `0x10` = enable SWCAN, anything else = disable | saved to EEPROM |
| 0x09 | KEEPALIVE | — | `F1 09 DE AD` (4 B, **no checksum**) |
| 0x0A | SET_SYSTYPE | 1 B type 0–3 | saved, settings reloaded |
| 0x0B | ECHO_CAN_FRAME | same layout as 0x00 | frame echoed to host over USB instead of transmitted |
| 0x0C | GET_NUMBUSES | — | `F1 0C 03` (always reports 3) |
| 0x0D | GET_EXT_BUSES | — | `F1 0D <sw st> <sw spd 4B> <bus4 st+spd 5B=0> <bus5 st+spd 5B=0>` (17 B, **no checksum**) |
| 0x0E | SET_EXT_BUSES | SW flags/speed (4 B), LIN1 (4 B, ignored), LIN2 (4 B, ignored) | SW part applied + saved |

**Speed word flag bits** (commands 0x05 / 0x0E, value > 0):

| Bit | Meaning |
|---|---|
| 31 | flags are valid (else old behavior: just enable) |
| 30 | channel enabled (0 = disabled) |
| 29 | listen-only mode |
| 19–0 | bit rate, capped at 1 000 000 (SWCAN: 100 000) |

Speed value 0 disables the channel.

**Bus numbering:** 0 = CAN0, 1 = CAN1, 2 = dedicated SWCAN (MCP2515, CANDue
2.2 only). On shared-transceiver boards, bus 1 with `SINGLEWIRE=1` *is* the
SWCAN bus.

### Device → host: received CAN frames

Binary mode (`sendFrameToUSB()`, `GVRET-gvret.ino:456-475`):

```
F1 00 | timestamp µs (4 B LE) | id (4 B LE, bit31 = extended) |
len | (bus << 4) packed into len byte | data ×len | 0x00
```

Text mode equivalent (when `BINSERIAL=0`, not Lawicel):

```
<micros> - <id HEX> X|S <bus> <len> <d0> <d1> …
```

Timestamps are the Due `micros()` counter (32-bit, wraps ≈ 71.6 min).

---

## 8. SD Card Logging

Requires a card present at boot (`sd.begin()` on chip-select 10). Toggle with
`s` / `S`, or `FILEAUTO=1` for automatic start.

### Formats (`FILETYPE`)

**1 — BINARYFILE** (record, little-endian):

```
timestamp µs (4 B) | id (4 B, bit31 = extended) | len | (bus << 4) | data ×len
```

**2 — GVRET CSV** (what SavvyCAN imports as "GVRET" log):

```
<millis>,<id hex>,<extended 0/1>,<bus>,<len>,<d0 hex>,<d1 hex>,…
```

CRLF-terminated, no header row, millisecond resolution.

**3 — CRTD** (default):

```
<seconds.fraction> R11|R29 <id hex> <d0 hex> <d1 hex> …
```

Marks are written as `<seconds> CEV <text>`.

### Marks

`MARK=<text>` inserts an annotation. In GVRET format it writes
`Mark: <text>`; in CRTD a `CEV` event line. Use liberally during captures to
correlate log positions with real-world actions.

### File naming

| `FILEAPPEND` | File opened |
|---:|---|
| 1 | `BASE.EXT` — appended, never truncated |
| 0 | `BASE<FILENUM>.EXT` — created/truncated, `FILENUM` incremented and saved |

Defaults: `CANBUS`, `TXT`, starting number 1.

### Write buffering

An 8 KB buffer is flushed when nearly full (> BUF_SIZE − 40 bytes pending) or
at least once per second with pending data (`Logger::loop()`). The logging LED
toggles on each flush. A failed write disables further file logging until
reboot.

---

## 9. EEPROM Settings & Factory Reset

* EEPROM page 275: main settings (`EEPROMSettings`), validated by version
  field (`EEPROM_VER = 0x17`). Mismatch ⇒ factory defaults written.
* EEPROM page 276: digital toggle settings. `mode == 255` ⇒ defaults
  (disabled, pin 1, ID 0x700, length 0).
* Nearly every config command writes through immediately; `SYSTYPE` needs a
  power cycle.
* **Factory reset:** send `R`, then power-cycle. Restores: CAN0/CAN1 500k
  enabled, SWCAN 33333 disabled, accept-all filters, CRTD file format, text
  serial, auto-log off, log level info, sysType 0 (CANDue).

---

## 10. Known Quirks & Bugs in This Build

Found by code inspection; worth knowing before relying on a feature.

1. **Analog inputs always read 0.** `sys_io_adc_poll()` is commented out in
   `loop()` (`GVRET-gvret.ino:1157`). The DMA ADC still runs but results are
   never processed. Re-add the call to revive analog inputs.
2. **Checksums are decorative.** Incoming checksums are never verified
   (checks are commented out in `BUILD_CAN_FRAME`/`ECHO_CAN_FRAME`), and
   outgoing CAN-frame messages always carry `0x00`. Several replies
   (TIME_SYNC, GET_CANBUS_PARAMS, GET_DEV_INFO, KEEPALIVE, GET_NUMBUSES,
   GET_EXT_BUSES) have no checksum byte at all.
3. **Lawicel `T` (extended TX) sets `extended = false`**
   (`SerialConsole.cpp:185`) — extended frames sent via Lawicel `T` go out
   mislabeled. Use `CAN0SEND=` instead.
4. **Lawicel `S` falls through to `s`** (`SerialConsole.cpp:194-227`) —
   harmless today because `s` is empty, but the speed *is* applied.
5. **`L` is not listen-only.** The Lawicel listen-only open just reopens CAN0
   normally.
6. **Lawicel auto-poll gating is disabled.** The block that suppressed RX
   output unless polled is commented out in `loop()`, so frames stream
   continuously regardless of `X1`.
7. **`DIGTOGLEVEL` polarity is inverted relative to the comment in
   `config.h`.** With mode bit 7 set (=1) the toggle pin is driven **LOW** at
   startup, and with 0 it is driven HIGH (`GVRET-gvret.ino:310-324`).
8. **Digital inputs 0 and 2 collide with transceiver enables** on CANDue
   boards (pins 48/50 double as CAN1/CAN0 enable outputs). See §4.1.
9. **`GET_NUMBUSES` always answers 3**, even on hardware without the
   dedicated SWCAN channel.
10. **Toggle TX sends the same payload for both pin edges** — rising and
    falling are indistinguishable on the bus.
11. **No debounce on pass-through gate pins 11/12** (fine for jumpers, avoid
    using pushbuttons).
12. **Mark trigger via digital input 0 is stubbed out** (commented block in
    `loop()`).
13. `handleCANSend` treats IDs ≥ 0x7FF as extended, so standard ID 0x7FF
    itself remains standard while anything larger becomes a 29-bit frame.
14. Debug `Serial` (programming port) is opened at 115200 but carries almost
    nothing; all real interaction happens on `SerialUSB` (native port).

---

## 11. Verifying the Firmware on Your Hardware

This guide describes the source in this folder (build 343). Upstream
`collin80/GVRET` master matches it byte-for-byte in protocol and pin logic,
but a physical device may run an older release or a modified fork. Verify
before trusting §4.1 behavior.

### 11.1 Software fingerprint (non-invasive)

| Method | What to check |
|---|---|
| Text console `h` | `Build number:` line (343 here) and menu contents — presence of `DIGTOG*`, `SWSPEED`, `SWSEND`, `FILEAUTO` entries is a feature fingerprint |
| SavvyCAN | Auto-detects GVRET devices, shows build number in connection status |
| Binary `GET_DEV_INFO` (`F1 07`) | Bytes 2–3 = build number (little-endian), byte 4 = `EEPROM_VER` (`0x17` here) |

A matching build number identifies the version but not local modifications.
If pin behavior contradicts this guide, assume a fork regardless of the
reported build.

### 11.2 Bench test of the pass-through pins (definitive)

Equipment: the GVRET board plus one dual-port USB-CAN adapter (or two single
adapters). Each bus segment must contain at least one **active** node — a lone
transmitter never receives an ACK and retries forever.

1. CAN0 ↔ port A on bus 1; CAN1 ↔ port B on bus 2; common ground; ~120 Ω
   termination per segment.
2. Open a serial terminal on the native USB port, send `h` to confirm alive.
3. Inject frame `0x123` on bus 1 → expect a copy on bus 2
   (pin 11 floating = pulled HIGH = forwarding ON in this build).
4. Jumper Arduino pin 11 to GND, inject again → forwarding must stop.
5. Repeat in the other direction: inject on bus 2, watch bus 1, jumper pin 12.

| Observation | Conclusion |
|---|---|
| Forwards by default, stops when pin grounded | Matches this build |
| Silent by default, forwards when pin grounded | Inverted fork/older logic |

### 11.3 Single-bus echo test (no second adapter)

Wire **both** CAN0 and CAN1 onto the same bus segment: every external frame is
received by one channel and re-transmitted by the other, so duplicates flood
the capture. Grounding pin 11 removes the 0→1 echoes, pin 12 the 1→0 echoes.
This doubles bus load — fine on a bench node, use briefly on a vehicle.

### 11.4 Guaranteed parity

Compile and flash this exact source tree (library list in §2). The device then
matches this guide by construction.

---

*Guide generated from source analysis of this repository (build 343).*
