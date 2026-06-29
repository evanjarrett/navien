# NPE-240A2 decode notes (model-specific findings)

This page records protocol-decode findings from an **NPE-240A2** (device_type
`11` = `NPE2`, firmware "Ver 17.0" in the NaviLink app), validated against ~2
months of full-resolution Home Assistant recorder history plus Navien-app
ground-truth screenshots.

Terminology: **DHW** = Domestic Hot Water (potable hot water drawn at taps and
showers); **SH** = Space Heating (the heating-loop water on combi/boiler models).

The unit is **DHW-only tankless** (no space-heating loop). Most of the existing
decode in `navien_proto.h` was reverse-engineered on **NCB-H combi** units, so a
number of fields that are correct on NCB-H carry different data here. This page
documents what is confirmed, what differs by model, and what is still open.

See also: [payload.md](./payload.md), [240.md](./240.md).

## Confirmed correct on NPE-240A2

Independent validation of existing fields against the Navien app:

| Field (gas bytes, little-endian) | Decode | App value | Verdict |
|---|---|---|---|
| `total_operating_time` (36/37) | `hi*256+lo` = 275 | "Total Operating Time 275.0 hr" | ✅ units = **hours** |
| `days_since_install` (28/29) | `hi*256+lo` = 1211 | ≈ install age (~3.3 yr) | ✅ |
| `accumulated_gas_usage` (24/25) | ×0.1 m³ → 3349.7 m³ | "Total Gas 1,214 Therms" | ✅ |
| `error_code` / `error_level` (water 14/15/16) | 0 | no faults | ✅ all zero over 2 months |
| recirc: `heating_mode == 0x08` | active recirculation | matches weekly schedule (Fri ON 07:30 / OFF 23:00) exactly | ✅ |
| `recirculation_enabled & 0x02` | scheduled-recirc active | value `2` dominates the recirc window | ✅ |
| `operating_state` (water 10) | the `OPERATING_STATE` enum (STANDBY 0x14 … ACTIVE_COMBUSTION 0x33 …) | observed ignition→high ramp 0x3C→0x2C→0x2D→0x33 | ✅ enum confirmed on NPE2 |

## Fix: Domestic Usage Count is ×10 (gas 30/31)

The proto comment already says the counter is *"in 10 usage increments"* but the
code published the raw 16-bit value. Apply ×10.

- Raw `(hi*256+lo)` = 4116 over 1211 days = 3.4 draws/day — implausibly low.
- ×10 = **41,160** → ~34 draws/day, matching the app's "Domestic Usage Cnt".

Because ×10 of a 16-bit raw can exceed 65,535, the corresponding state fields
were widened to `uint32_t`. Implemented in `navien.cpp` for both
`total_dhw_usage` and `cumulative_domestic_usage_cnt`.

### Example frame (cumulative-counter region, gas status packet)

Gas status header (see [payload.md](./payload.md)):

```
F7 05 50 0F 90 ...
```

Cumulative-counter tail bytes from the dominant subframe, with the app-confirmed
decode:

```
offset : 24   25   28   29   30   31   36   37
raw    : D9   82   BB   04   14   10   13   01
                                  └ usage 30/31
decode :
  accumulated_gas_usage (24/25) = 0x82D9 = 33497  → ×0.1 m³ = 3349.7 m³   (app: 1,214 Therms)
  days_since_install   (28/29) = 0x04BB =  1211   → days                  (≈ 3.3 yr)
  cumulative_domestic_usage_cnt (30/31) = 0x1014 = 4116 → ×10 = 41,160    (app: "Domestic Usage Cnt", ~34/day)
  total_operating_time (36/37) = 0x0113 =   275   → hours                 (app: 275.0 hr)
```

## Model-specific: gate space-heating fields on DHW-only units

These were derived on NCB-H combi units and do **not** decode correctly on a
DHW-only NPE-240A2:

| Field | gas bytes | NPE-240A2 behavior |
|---|---|---|
| `cumulative_dwh_usage_hours` | 38/39 | reads **0** despite the app showing DHW usage time → quantity lives elsewhere on this model |
| `cumulative_sh_usage_hours` | 40/41 | **fast-varying garbage** (250+ distinct values) — there is no space heating to count |
| `sh_set_temp` / `sh_outlet_temp` / `sh_return_temp` | 17/18 (+ water) | no space-heating loop on this unit |

The component now gates publication of these sensors with
`Navien::is_dhw_only(device_type)` (true for the NPE/NPN tankless families and
their cascade variants). On those models the sensors are simply not published
rather than emitting misleading values. Combi/boiler models are unaffected.

`outdoor_temp` (gas 19) is also space-heating-related and is suspect on this unit
(no outdoor-reset probe); it is left published because its decode already has a
no-probe sentinel (`0x9E`), but treat it as unverified on NPE2.

## Refined byte meanings (NPE-240A2; previously `unknown_*`)

Data-driven, no app label — treat as tentative pending post-reflash raw capture:

| Bytes | Old note | NPE-240A2 finding |
|---|---|---|
| water 19 | "0x00 on NCB-H" | flow-gated categorical bracket (0/44/51/63/66), not a magnitude (r≈0 vs flow) |
| water 20–23 | "per-model constants" | a single 4-byte status block (move in lockstep, ~25 distinct tuples); byte 23 = phase (0/0x20/0x3F) |
| water 25–26 | unknown | mirror `system_status` (24); byte 25 toggles 0↔1 every packet = sequence/handshake bit |
| water 27 | `boiler_active` boolean | NOT boolean here — 87 distinct values 0–249 (likely frame-blended) |
| water 30/31 | "Counter B (0xFF on NCB-H)" | **cumulative gas-volume odometer, ~0.25 m³/tick** (16-bit LE). Confirmed gas-locked: r=0.9998 vs cumulative gas; flat overnight while the burner is off, ticks only while gas flows; a 16h/+1.2 m³ capture advanced it +4 (pred +4.7). Decoded as the `gas_odometer` sensor. The earlier "~90-min timer" guess is refuted. |
| gas 26/27 | "always 0x00" | vary on NPE2 — a paired state word, moves in lockstep during heating |
| gas 33 | "Counter C ÷12.015" | coarse cumulative gas-volume odometer **~1.0 m³/tick** (r=0.99, locks 4:1 with water 30/31). MEDIUM confidence — needs a multi-m³ sample to tick. The "÷12" is draws/tick, not the unit. |
| gas 35 | "0x00" | 3-state firing level: 0x42 = max fire (kcal≈16138), 0x00 = moderate, 0x3F = low-fire |
| gas 42–47 | undecoded tail | byte 42 = sub-frame address; 44–47 = 60-day-constant device-identity params (0x5180…) in the dominant subframe |

## Open question: frame blending (≥2 variants share one header)

The heater appears to emit **more than one frame variant under the same header
signature**, and a single-struct parser blends them. Direct evidence: `gbyte_32`
shows two different values *at the same one-second timestamp* (e.g. 132 then 189)
while `gbyte_15` (outlet temp) in the same window is a smooth ramp — so the
RS-485 link is clean; the byte genuinely differs between interleaved frames.

Consequences:
- The clean, low-cardinality bytes (temps, flow, capacity, `operating_state`, the
  16-bit counters) decode fine.
- The tail/status bytes can't be fully resolved until frames are separated.
- Candidate sub-frame discriminators: water 29/32, gas 32, gas 42.

Recommended next step: parse per-frame on-device (post-reflash) and re-classify
each variant separately before trusting tail-byte fields. The HA per-byte sensor
reconstruction used for this page **blends** subframes by construction, which is
why the cumulative counters must be read from the dominant subframe.

### Raw-frame text sensors (for continued reverse-engineering)

To support exactly this analysis, the component can expose each parsed frame as a
hex string via two optional `text_sensor` flags:

```yaml
text_sensor:
  - platform: navien
    navien: navien_main
    name: "Water Raw Frame"
    water_raw: true
  - platform: navien
    navien: navien_main
    name: "Gas Raw Frame"
    gas_raw: true
```

Each publishes one **fully-parsed, frame-aligned** payload as space-separated
hex — so unlike the old per-byte HA sensors, the bytes within a value are
guaranteed to come from the *same* frame (no blending). The first hex pair is
packet offset 6 (the start of the `WATER_DATA` / `GAS_DATA` struct), so packet
byte `N` is hex pair index `N - 6`. Record these in HA history and split per byte
in SQL (`split`/`substr`) to study how the unknown positions move with events —
and to finally separate the sub-frame variants the blending hid.

For eyeballing in HA, any individual byte can also be exposed as a **numeric**
sensor via `gas_byte_N` / `water_byte_N` keys on the hub `sensor` platform
(N = absolute packet offset, gas 6–47 / water 6–39):

```yaml
sensor:
  - platform: navien
    id: navien_main
    # …
    gas_byte_32:
      name: "GByte 32"
    water_byte_19:
      name: "WByte 19"
```

These are also frame-aligned. The hex dumps remain the better bulk-analysis
input (one row = one whole frame); the per-byte sensors are for live dashboards.

## Note: panel/controller version vs the app

The app's "Software Update Ver. 17.0" is the **NaviLink/app** firmware, not the
unit's panel/controller version byte. On this unit the controller/panel version
bytes are 12/19 → "1.2"/"1.9", which is expected and does not need to match the
app's number.
