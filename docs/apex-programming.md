# Apex output programming (`prog`)

Companion reference for the `prog` field carried by output-config endpoints in
the [Apex Fusion OpenAPI description](../openapi.yaml).

> ⚠️ **Unofficial and conservative.** Like the rest of this repository, this
> document is reverse-engineered from live captures of the Fusion web app. It
> is **not** a replacement for Neptune's programming reference. It documents
> *observed behavior only*; anything not directly verified from captures is
> marked as such. It is intentionally incomplete and hardware-dependent.
>
> Not affiliated with or endorsed by Neptune Systems.

## What `prog` is

Every configurable output has a **control program** — the source of the Apex
programming language that decides the output's state. In this API it appears
as the `prog` string on:

- `GET /api/apex/{apexId}/config/outputs` — per output, inside each `ConfigOutput`
- `GET /api/apex/{apexId}/config/outputs/{did}` — a single output
- `PUT /api/apex/{apexId}/config/outputs/{did}` — **writing** `prog` changes
  controller programming

A program is a newline-separated (`\n`) list of statements, evaluated top to
bottom; the last matching statement wins. A minimal program is a single line
such as `Set OFF`.

Two distinct things live in this string, and they are documented very
differently below:

1. **Programming-language statements** — human-authored control logic
   (`Fallback`, `Set`, `If … Then …`, etc.). Well documented by Neptune.
2. **`tdata` schedule lines** — machine-generated breakpoints the Fusion app
   emits for dosing, pump-flow, and light schedules. **Not** documented by
   Neptune, and only partially understood here.

## 1. The programming language

The Apex programming language is fully documented by Neptune in the
**AquaController / Apex Comprehensive Reference Manual**. For command
semantics, operator behavior, and the authoritative list of statements, use
that manual — do not treat this section as complete.

What follows is only the set of statement *forms* observed in captured
programs, provided so a client can recognize and round-trip existing `prog`
strings.

### Observed statement forms

| Form | Observed examples |
|------|-------------------|
| `Fallback ON` / `Fallback OFF` | `Fallback OFF` |
| `Set ON` / `Set OFF` | `Set ON`, `Set OFF` |
| `Set PFn` (profile reference) | `Set PF1`, `Set PF2` |
| `If <input> <op> <value> Then <state>` | `If pH > 8.37 Then OFF`, `If ORP < 330 Then ON`, `If Tmp > 81.7 Then OFF`, `If L_SUMP < 6.3 Then ON` |
| `If <switch> CLOSED\|OPEN Then <state>` | `If LEAK CLOSED Then OFF`, `If ATO_LO OPEN Then OFF` |
| `If Output <name> = ON\|OFF Then <state>` | `If Output RO_REFILL = ON Then ON` |
| `If Output <name> Watts <op> <value> Then <state>` | `If Output P_OZONE Watts < 1 Then OFF` |
| `If Error <name> Then <state>` | `If Error P_UV Then OFF` |
| `If Time <HH:MM> to <HH:MM> Then <state>` | `If Time 16:00 to 11:00 Then ON` |
| `If Feed{A\|B\|C\|D} <nnn> Then <state>` | `If FeedA 000 Then OFF` |
| `If Moon <nnn>/<nnn> Then <state>` | `If Moon 000/000 Then ON` |
| `Defer <mmm:ss> Then <state>` | `Defer 000:10 Then ON`, `Defer 045:00 Then ON` |
| `Min Time <mmm:ss> Then <state>` | `Min Time 030:00 Then OFF` |
| `When On > <mmm:ss> Then <state>` | `When On > 007:00 Then OFF` |
| `OSC <mmm:ss>/<mmm:ss>/<mmm:ss> Then <state>` | `OSC 000:00/060:00/120:00 Then ON` |

Notes and caveats:

- `<state>` is `ON`, `OFF`, or a profile reference like `PF1`.
- The keyword casing observed in the wild is inconsistent (`Fallback` vs
  `FALLBACK`); treat statements case-insensitively when parsing.
- Duration arguments (`Defer`, `Min Time`, `OSC`, `When On`) use a `mmm:ss`
  form — **minutes:seconds**, so `030:00` is 30 minutes, not 30 hours.
- The **parameter order** of multi-argument statements (notably `OSC`) is
  **not verified here** — consult Neptune's manual before relying on it.
- Input/switch/output names (`pH`, `L_SUMP`, `LEAK`, `RO_REFILL`, …) are
  controller-specific and come from the input/module configuration.

## 2. The `tdata` schedule-line encoding

Schedules that the Fusion UI presents as tables — dosing amounts and times,
pump-flow ramps, light ramps — are **not** stored as discrete API fields.
They are encoded as one or more `tdata` lines inside `prog`. This encoding is
not documented by Neptune; the description below is what can be verified from
captures, and no more.

### What is verifiable

Each schedule line has the shape:

```
tdata HH:MM:SS,<i1>,<i2>,<i3>,<i4>,<i5>,<i6>,<i7>,<i8>,<i9>,<i10>,<i11>,<i12>,<i13>
```

- A literal `tdata`, then a **`HH:MM:SS` breakpoint time**, then **exactly 13
  comma-separated integers**. This structure is stable across every captured
  example.
- Programs may contain **multiple `tdata` lines**, one per breakpoint,
  ordered by time (e.g. a pump ramp with six breakpoints across the day).
- **Field 1 correlates with output class:** it is `1` on dosers (`type` of
  `dos` / `dqd`) and `0` on pumps and other outputs in every observed sample.
  This is an *observed correlation only* — its meaning is not confirmed.

### What is NOT decoded

**The per-field meaning of the 13 integers is only partially understood, and no
official specification exists.** One early hypothesis — "field 2 = pump mode id"
— was **refuted** by the captures: two `MXMPump|AI|Orbit4K` pumps (identical
hardware) show field 2 = `16` vs `0`, and mode-like values fall in different
positions across output types. Do not build a complete field table from the
examples below; treat unexplained integers as opaque unless you have verified
them against your own controller.

Neptune has never published these values and, per community tutorials relaying
Neptune's own guidance, they are "not meant to be human-readable" and should
not be hand-edited — the intended workflow is to edit schedules in the Fusion
Wizard, which regenerates `tdata` automatically.

If you write a `tdata` line whose fields you do not understand, you may change
dosing volumes or pump/light output in ways that are unsafe for livestock. When
in doubt, read the current `prog` back, change only the language statements, and
leave existing `tdata` lines byte-for-byte intact.

### Field-position observations

The following are *observed correlations*, drawn from this repository's captures
and corroborated where noted by community-reported schedules (see [Examples
reported by community sources](#examples-reported-by-community-sources)). They
are **not** confirmed field definitions.

| Field | Observation | Evidence |
|-------|-------------|----------|
| time | `HH:MM:SS` breakpoint; lines are ordered by time | all captures |
| f1 | `1` on dosers (`dos`/`dqd`), `0` on pumps and lights | captures + community DOS/light examples |
| f2 | small integer that varies *within the same hardware* (Orbit4K: `16` vs `0`) — **not** a stable mode id | captures |
| f3 | tracks an **intensity / speed percentage** on lights and pumps (ramps toward `0`–`100`); on dosers it is a small value, not a percentage | community Kessil examples + pump captures |
| f4… | vary by output type. On lights, community examples show f4+ carrying per-**spectrum/color channel** values; on dosers/pumps their meaning is unverified | community light examples; captures |
| f13 | `0` in every observed line | all captures + community examples |

### Formal examples (this repository's captures)

Each row is one `tdata` breakpoint with its 13 integers split into columns
`f1`–`f13`, grouped by output `type`. One representative output is shown per
type.

#### `dos` — doser (output `MAG`)

| output | time | f1 | f2 | f3 | f4 | f5 | f6 | f7 | f8 | f9 | f10 | f11 | f12 | f13 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| MAG | 00:00:00 | 1 | 21 | 0 | 2 | 4 | 116 | 75 | 0 | 2 | 3 | 72 | 19 | 0 |

#### `dqd` — doser (output `SW_NEW_L`)

| output | time | f1 | f2 | f3 | f4 | f5 | f6 | f7 | f8 | f9 | f10 | f11 | f12 | f13 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| SW_NEW_L | 00:00:00 | 1 | 19 | 7 | 98 | 2 | 88 | 143 | 7 | 128 | 2 | 28 | 10 | 0 |

#### `MXMPump|AI|Orbit4K` — pump (output `Gyre_Back_Lo`)

| output | time | f1 | f2 | f3 | f4 | f5 | f6 | f7 | f8 | f9 | f10 | f11 | f12 | f13 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Gyre_Back_Lo | 00:00:00 | 0 | 16 | 0 | 17 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| Gyre_Back_Lo | 03:30:00 | 0 | 16 | 5 | 17 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| Gyre_Back_Lo | 07:50:00 | 0 | 16 | 15 | 19 | 120 | 120 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| Gyre_Back_Lo | 13:25:00 | 0 | 16 | 15 | 18 | 2 | 10 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| Gyre_Back_Lo | 18:57:00 | 0 | 16 | 2 | 17 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| Gyre_Back_Lo | 23:59:00 | 0 | 16 | 0 | 17 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

#### `MXMPump|Ecotech|Vortech` — pump (output `PWH_LEFT`)

| output | time | f1 | f2 | f3 | f4 | f5 | f6 | f7 | f8 | f9 | f10 | f11 | f12 | f13 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| PWH_LEFT | 00:00:00 | 0 | 0 | 2 | 2 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| PWH_LEFT | 05:33:00 | 0 | 0 | 8 | 3 | 224 | 46 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| PWH_LEFT | 09:58:00 | 0 | 0 | 21 | 8 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| PWH_LEFT | 13:01:00 | 0 | 0 | 43 | 7 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| PWH_LEFT | 17:29:00 | 0 | 0 | 38 | 2 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| PWH_LEFT | 23:59:00 | 0 | 0 | 8 | 2 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

#### `MXMPump|Ecotech|Vectra` — pump (output `Return1`, constant flow)

| output | time | f1 | f2 | f3 | f4 | f5 | f6 | f7 | f8 | f9 | f10 | f11 | f12 | f13 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Return1 | 00:00:00 | 0 | 0 | 52 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| Return1 | 23:59:00 | 0 | 0 | 52 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

For reference, the surrounding program for a doser (KALK) looks like:

```
Fallback OFF
tdata 00:00:00,1,21,1,28,3,132,95,1,58,3,72,15,0
If pH > 8.37 Then OFF
If Alkx28 > 10.00 Then OFF
Min Time 030:00 Then OFF
```

### Examples reported by community sources

These lines are **not** from this repository's captures — they are transcribed
from public forum posts and included to broaden the sample, especially for
**lighting** outputs (which this repo has not captured directly). The lighting
examples are what corroborate the "f3 = intensity, f4+ = color channels"
observation above.

Kessil brightness-only light schedule (intensity in f3; color channels `0`),
reported on [Reef2Reef](https://www.reef2reef.com/threads/apex-lights-programming.303087/):

```
tdata 12:00:00,0,0,50,0,0,0,0,0,0,0,0,0,0
tdata 13:30:00,0,0,100,0,0,0,0,0,0,0,0,0,0
tdata 16:30:00,0,0,100,0,0,0,0,0,0,0,0,0,0
tdata 17:30:00,0,0,40,0,0,0,0,0,0,0,0,0,0
```

Kessil A500X full-color light schedule (f3 intensity, f4+ per-channel),
reported on [Reef2Reef](https://www.reef2reef.com/threads/kessil-ap9x-apex-advanced-control.893956/):

```
tdata 12:00:00,0,0,80,80,0,50,25,27,0,0,0,0,0
tdata 14:00:00,0,0,100,100,100,100,100,100,0,0,0,0,0
tdata 15:00:00,0,0,100,100,100,100,100,100,0,0,0,0,0
```

DOS automatic-water-change dose (f1 = `1`, doser), from the
[Reef2Reef DOS tutorial](https://www.reef2reef.com/ams/neptune-apex-tutorials-part-9-dos.849/):

```
Fallback OFF
tdata 00:00:00,1,17,10,87,2,88,143,10,107,2,28,10,0
If Output H2O_CONTROL = OFF Then OFF
```

## Safe editing guidance

Because `PUT …/config/outputs/{did}` replaces the whole program:

1. `GET` the output first and preserve its existing `prog` verbatim.
2. Edit only the statement lines you understand.
3. Leave `tdata` lines untouched unless you have independently verified their
   encoding on your own hardware.

## Prior art

As of this writing, no public source provides a complete field-by-field decode
of `tdata`; Neptune treats the encoding as opaque and the community consensus is
to edit schedules only through the Wizard. The partial observations above (e.g.
f3 as intensity/speed for lights and pumps) come from forum posts, not
documentation. Existing open-source Apex clients target the **local
controller's `status.xml`** rather than the Fusion cloud API and do not model
the `prog`/`tdata` encoding:

- [`maxvitek/apex`](https://github.com/maxvitek/apex) — Python; reads status,
  probes, outlets from the local controller.
- [`reefassistant/apex`](https://github.com/reefassistant/apex) — Go; local
  network only, explicitly **not** for Fusion.
- [telegraf Neptune Apex input plugin](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/neptune_apex)
  — Go; parses the local `status.xml`.

If you find or produce a verified `tdata` field decode, contributions are
welcome (see the repository README).

## See also

- Neptune's **Apex Comprehensive Reference Manual** for authoritative
  programming-language documentation.
- Reef2Reef, ["Neptune Apex Tutorials Part 9: DOS"](https://www.reef2reef.com/ams/neptune-apex-tutorials-part-9-dos.849/)
  — community explanation of the DOS Schedule Wizard and `tdata`.
- [`openapi.yaml`](../openapi.yaml) — the `ConfigOutput` schema and the
  output-config endpoints that carry `prog`.
