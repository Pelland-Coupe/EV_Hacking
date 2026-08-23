# Identification of unknown BMU/BMS IDs (0x3E2–0x3EC and related)

**Date:** 2026-08-23
**Sources:** 10 bridge logs (`Bridge-logs/`) + `23-08-26 P-CAN-long_log.csv` (131.8 s, direct P-CAN)
**Correction to earlier note:** no UDS diagnostics were running during these captures. The BMU
(Battery Management Unit, aka BMS) transmits these IDs as part of its normal broadcast.

---

## 1. Headline finding

The "unknown" IDs are the **BMS cell-voltage telemetry pages**, plus housekeeping counters/flags.
Together with the already-stubbed `UNK_3B9/3BA/3BB` and `UNK_3D0–3DE` messages they form one
coherent 1 Hz family:

| Group | IDs | Msgs × words | Values seen |
|---|---|---|---|
| Voltage block A | 0x3B9, 0x3BA, 0x3BB | 3 × 4 = 12 | 3898–3904 mV |
| Voltage block B | 0x3D0–0x3DE | 15 × 4 = 60 | 3898–3906 mV |
| Voltage block C | 0x3E2, 0x3E3, 0x3E5–0x3E8 | 6 × 4 = 24 | 3901–3905 mV |
| Mixed block | 0x3E4 words 0–1 | 2 | 3902–3905 mV |
| **Total cell values** | | | **98 — matches the Kona EV 98S pack and the 0x3E9 counter wrap** |

Every voltage word is **little-endian, 16-bit, scale 1 mV** (e.g. `3E 0F` → 0x0F3E = 3902 →
3.902 V), consistent with a healthy parked NCM pack.

## 2. Per-ID identification

| CAN-ID | Proposed content | Confidence | Evidence |
|---|---|---|---|
| 0x3B9–0x3BB, 0x3D0–0x3DE | Cell voltages, 4 × LE u16 mV per msg | High | mV-plausible static values; completes 98-cell arithmetic |
| 0x3E2, 0x3E3, 0x3E5–0x3E8 | Cell voltages, 4 × LE u16 mV per msg | High | same |
| 0x3E4 D0–D3 | Cell voltages (2 cells) | High | same |
| 0x3E4 D4–D7 | **Unknown gauges** (see §4) | Low | slow byte-counters, tick +256 rarely |
| 0x3E9 D0 | Cycle/cell index counter, u8, wraps **98→1** at 1 Hz | High | exact 98-wrap observed in long log |
| 0x3EA D0 | BMS state flag: 1 = pre-ready monitoring, 2 = (ignition-on?) | Medium | flipped 1→2 once at +61 s in long log; stayed 1 in all short logs |
| 0x3EB D0 | Sequence/page counter mod 10 (1…10 repeating) | High | 11 full cycles observed |
| 0x3EC | Reserved / all-clear status (always 0x00) | Medium | constant across all 11 captures |

## 3. Lifecycle

- All 29 IDs broadcast at exactly **1 Hz from bus wake** (12 V on).
- Short bridge logs: family stops **8–13 s after wake**, coinciding with the S/S press that takes
  the car toward READY (stop time varies per log exactly as the human press timing does; CCM log,
  with slowest press, ran longest at 13 s). No other bus traffic changes at that moment.
- Long log (no start attempt for >110 s): **never stops**.
- Conclusion: this page-set runs while the BMS is awake-but-pre-drive; it ceases on transition to
  HV-run state. No payload change anywhere else on the bus marks the stop.

## 4. Open questions

1. **0x3E4 D4–D7**: two independent 2-byte fields. Observed values cluster ~3084–3087 and
   ~3340–3343, with rare +256 jumps (high byte ticks: D5 ticked ~11:01, D7 between captures).
   Candidates: cumulative energy/charge-throughput gauges, min/max-derived stats, or non-mV
   temperatures with unusual scaling. A charge session will disambiguate immediately.
2. **Cell ordering**: which message/word maps to which physical cell index is unproven. The
   0x3E9 counter may sequence through cells 1–98 but payloads do not visibly rotate with it.
3. **0x3EA = 2 trigger**: flip at +61 s in long log coincided only with routine changes elsewhere;
   candidates include 0x597 bit7 clearing, 0x497 nibble change (both BMS-range IDs).

## 5. External cross-reference

- projectgus/kona-ev-dbc `pcan.dbc`: **no coverage** for 0x3B9/0x3Dx/0x3Ex (its BMS coverage is
  0x491…0x5D8 range). Our identification extends beyond public knowledge.
- uhi22 Ioniq28Motor.dbc: no cell-voltage definitions.
- opendbc hyundai_kia_generic: ADAS-focused, no coverage.

## 6. Recommended follow-up captures

1. **READY, ≥5 min, no gear change**: confirm family stops at READY and stays stopped; watch
   0x3EA value at the transition (expect 2? or freeze?).
2. **Charge session (AC, ≥10 min)**: cell voltages will climb smoothly — confirms mV scale
   definitively and reveals whether 0x3E4 gauges track energy (kWh) or temperature.
3. **Drive with hard acceleration**: max/min cell words spread wider under load; identifies
   weakest-cell behaviour if 0x3E4 holds extremes.
4. Keep using the **direct P-CAN tap** (like the long log): the bridge setup adds no value here
   since BMU attribution is already established.

## 7. Deliverables

`My_pcan_draft_addendum.dbc` next to this file contains draft `BO_/SG_` definitions for the 11
missing IDs (0x3E2–0x3EC), ready to merge into `My_pcan.dbc` (IDs 994–1004 decimal, no clashes).
Validated with cantools: loads cleanly and decodes live frames (e.g. `0x3E2` → 4 × 3.902 V).

> Note: `id_dbc_coverage.csv` was regenerated after fixing a little-endian byte-mapping bug in
> the coverage script (verified signal-by-signal against cantools across all 131 DBC messages).
> New totals: **14 Full / 83 Partial / 11 Missing** (previously reported 13/84/11); 42 rows
> changed their note-of-coverage details.

### Paste-in signal blocks for the existing UNK stubs

Add these `SG_` lines under the matching `BO_` already present in `My_pcan.dbc`
(`UNK_3B9`=2147484601, `UNK_3BA`=2147484602, `UNK_3BB`=2147484603,
`UNK_3D0…UNK_3DE`=2147484624…2147484638):

```
 SG_ CellVoltage_1 : 0|16@1+ (0.001,0) [0|65.534] "V" XXX
 SG_ CellVoltage_2 : 16|16@1+ (0.001,0) [0|65.534] "V" XXX
 SG_ CellVoltage_3 : 32|16@1+ (0.001,0) [0|65.534] "V" XXX
 SG_ CellVoltage_4 : 48|16@1+ (0.001,0) [0|65.534] "V" XXX
```

(Identical block for each of the 18 stubbed messages; consider renaming the messages to
`BMS_3B9_CellVoltages` etc. per repo convention at merge time.)
