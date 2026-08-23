# P-CAN Startup Sequence Signal Report

**Log:** `23-08-26 P-CAN-long_log.csv` (GVRET CSV, 309,450 frames, 122.6 s – 254.4 s)
**DBC:** `../My_pcan.dbc`
**Capture sequence** (per `Bridge-Logs.txt`): 12V battery on → press brake → press S/S → wait until running → press S/S → wait for shutdown → 12V off.

All timestamps below are log-relative seconds (log starts at 122.600 s device time).

## Event timeline

| Time (s) | Event | Evidence |
|---|---|---|
| 122.60 | Log start, 12V already on | BCM wake at 125.70 |
| ~129.29 | Brake pressed (already down when IEB wakes) | CYL_PRES ≈ 61.8 Bar held from first frame |
| 129.37 | S/S pressed (#1) | BCM_593 byte0 1→20 |
| 129.38 | Ignition "Starting" | IgnitionSw 0→4 |
| 129.47–129.49 | VCU + BMS begin transmitting | first frames of 0x595 / 0x58F / 0x5A3 / 0x540 |
| 129.59 | Request for HV turn on | VCU_58F.Unk_Status 0→1 |
| 129.66 | HV request corroborated | VCU_200.UnkStatus 16→23 |
| 129.86–130.16 | Precharge window | PrechargeRelay pulse (~300 ms) |
| 130.16 | HV precharge completed | ContactorClosed 0→1 |
| 131.11 | RUN / Ready | IgnitionSw 4→3 |
| 130.97 / 131.41 | Brake released | pressure-valid drops, DriverBraking→0 |
| 242.74 | S/S pressed (#2, power off) | BCM_593 byte0 20→0 |
| 242.76–242.79 | Ignition off | Ign1→0, IgnitionSw→0 |
| 243.15 | Contactors open | ContactorClosed 1→0 |

## Findings per event

### 1. Brake press

| Message | ID | Signal | Definition | Behaviour in log |
|---|---|---|---|---|
| StabilityControl | 0x220 | `CYL_PRES_STAT` | bit 38\|1@1+ | =1 while pedal pressed; →0 at release (130.971) |
| StabilityControl | 0x220 | `CYL_PRES` | bits 26\|12@1+ (0.1 Bar) | ≈61.8 Bar held while pressed |
| TractionControlMed | 0x390 | `DriverBraking` | bit 55\|1@1+ | =1 while braking; →0 at 131.412 |
| IEB_2A2 | 0x2A2 | `BrakePedalForce` | bits 24\|16@1+ | ≈4500 held; ramps down during release (131.09–131.30) |

> **Caveat:** the rising edge of the brake press is not captured — IEB's first frame
> (129.291 s) already shows full pedal pressure. Only the *held* and *released*
> states are present in this capture.

### 2. S/S (start/stop) button pressed

Best P-CAN indicator: **`BCM_593.Unknown1`** (ID 0x593, byte 0) — it changes *before*
any other ignition-related signal and flips at both presses:

| Time (s) | Value change | Interpretation |
|---|---|---|
| 125.696 | 0→1 | BCM awake, ignition off |
| **129.366** | **1→20 (0x14)** | S/S press #1 — leads IGPM reaction by 17 ms |
| **242.741** | **20→0** | S/S press #2 — leads IGPM reaction by 20 ms |
| 244.741 | 0→1 | back to idle state |

Resulting ignition state on `BodyState` (ID 0x541):
- `IgnitionSw`: 0=Off → **4=Starting** (@129.383) → **3=On/RUN** (@131.111) → 0 (@242.789)
- `Ign1` set @129.383, cleared @242.761; `Ign2` set @131.111, cleared @242.789

### 3. Request for HV turn on

Best candidate: **`VCU_58F.Unk_Status`** (ID 0x58F, Motorola bit 43, 4-bit):

| Time (s) | Value | Notes |
|---|---|---|
| 129.592 | 0→1 | 209 ms after S/S press, **269 ms before precharge relay closes** |
| 129.892 | →15 | precharge starting |
| 129.992 | →14 | transient during contactor close |
| 130.092 | →15 | steady while powered |
| 242.790 | →0 | exactly at shutdown |

Corroborating evidence:
- VCU/BMS wake and start transmitting at 129.470–129.490 s.
- `VCU_200.UnkStatus` (ID 0x200) sets its low bits 16→23 @129.656; reaches 59 (0x3B,
  documented "Ready") @131.006.

### 4. HV precharge completed

| Message | ID | Signal | Behaviour |
|---|---|---|---|
| Batt_HV_Status | 0x595 | `PrechargeRelay` | high 129.861→130.161 (~300 ms precharge pulse), then opens |
| BMS_5A3 | 0x5A3 | `ContactorClosed` | **0→1 @130.164** — main contactors fully closed = precharge complete; holds until 243.153 |
| BatteryLimits | 0x594 | `Sig594_BMS_Bit34`, `Sig594_BMS_Bit38` set / `Bit61` cleared | status flags flip @130.158 |
| BMS_597 | 0x597 | `Active` | 0→1 @130.163 (Ready or charging) |

> Note: `UNK_520.StartIndicator` (whose DBC comment says it pulses at end of
> precharge) stayed 0 for this entire capture.

## Method

Frames were decoded with `cantools` against `My_pcan.dbc` (with an in-memory fixup:
signals named `594_BMS_Bit*` renamed to satisfy DBC identifier rules). Transitions of
every signal were collected across the full log, then grouped into event windows around
the known sequence; low-frequency signals (≤3 changes per window) were kept and
counter/checksum-like signals filtered out. A raw byte-level diff between the OFF state
(124–129.3 s) and the HV-request window (129.55–129.85 s) confirmed the VCU-side findings.
