# Encounter C64 — Reverse Engineering Notes

# Aaron's notes
- marking up the PRG is not helpful since it gets expanded into memory elsewhere.
- next step: ditch Regenerator and move to a full RAM debugger
- question: what is a good program to annotate all the locations?
- what does Martin Piper use?

## Memory Map

| Address | Contents |
|---------|----------|
| `$0801` | BASIC stub — line 1001 (empty, no SYS in bytecode) |
| `$0806–$080F` | ASCII "1001 SYS 2066" — human-readable label, skipped by BASIC |
| `$0810–$0811` | `$0000` — end of BASIC program |
| `$0812` | **Machine code entry point** — `SEI / LDX #$30` |
| `$2B0E` | Sound effect data — "saucer move" (8 bytes) |
| `$5726` | End of loaded program |

Launch: `SYS 2066` from BASIC prompt, or VICE `-autostart`.

---

## Sound Engine

### Playback Routine: `$4E13`

The engine reads from a **runtime table at `$80A0`** and writes to SID Voice 3 frequency high byte (`$D40F`):

```asm
$4E13  ldy e0048      ; load loop-point index into Y
$4E15  beq b4E28      ; Y=0 → sound not active, skip
$4E17  inc a49        ; advance position counter
$4E19  ldx a49        ; X = current position
$4E1B  lda f80A0,x   ; load byte from $80A0+X  ← runtime table
$4E1E  bne b4E25      ; non-zero → write frequency
$4E20  sty a49        ; zero hit → reset position to loop-point
$4E22  lda f80A0,y   ; reload from loop-point
$4E25  sta $d40f      ; write to SID Voice 3 freq high byte
```

Key behaviours:
- **Zero-terminated** — a `$00` byte in the table signals end-of-effect
- **Loop point** — stored in Y before the routine runs; when `$00` is hit, position resets to Y and continues looping from there
- **Initial trigger** — sets position counter (`a49`) to `loop_point - 1`; first `inc` brings it to `loop_point`, so the first byte played is `$80A0 + loop_point`

### How Sound Data Gets to $80A0

There is **no separate copy routine**. The depacker at `$0340–$036F` (running in the tape buffer) decompresses ALL game data — code, tables, and sound effects — directly to their final runtime addresses in one pass. The sound table lands at `$80A0` simply because that's where the depacker writes it.

Confirmed by watchpoint on write to `$80A0`: hit at `$0360: STA ($FE),Y` with `$FE/$FF = $9D $80`, `Y = $03` — the depacker's own store instruction, mid-decompression.

### Runtime Sound Table Structure (`$80A0`)

The table is a **flat byte array**, indexed directly by Y (the loop-point). Y=0 is hardwired as "inactive" (BEQ guard at engine entry). The first byte at `$80A0` is always `$00` to serve as that sentinel. All other loop-point values are byte offsets into the table hardcoded by the game.

The block decompressed by the depacker starts at `$809D` (3 zero bytes precede `$80A0`).

Table ends at `$8100` (confirmed: `$8100–$811F` are all zero). Data from `$8120` onward is a different structure (alternating `$80/$81` pairs, then address-like bytes — probably display or object tables).

#### Confirmed sound effects

| Loop pt Y | Runtime addr | Bytes | Hz sequence | Shape |
|-----------|-------------|-------|-------------|-------|
| Y=1 | `$80A1` | `40 30 20 30` | 976→720→464→720 | V-sweep ("saucer wait") |
| Y=$10 | `$80B0` | `40 38 30 28 20 28 30 38` | 976→848→720→592→464→592→720→848 | U-sweep ("saucer move") |

#### Probable single-note effects (`$80D0–$80FF`)

Each is one frequency byte immediately followed by `$00` — plays a constant repeating tone. Loop point Y = the offset of that byte.

| Loop pt Y | Addr  | Byte | Hz   |
|-----------|-------|------|------|
| Y=$30 | `$80D0` | `$C0` | 3024 |
| Y=$38 | `$80D8` | `$80` | 2000 |
| Y=$40 | `$80E0` | `$40` |  976 |
| Y=$48 | `$80E8` | `$20` |  464 |
| Y=$50 | `$80F0` | `$10` |  208 |
| Y=$58 | `$80F8` | `$08` |   80 |
| Y=$5F | `$80FF` | `$0F` |  192 |

These form a descending pitch bank — likely used for hit/explosion/event sounds where the game just picks a note.

#### Ambiguous region (`$80C0–$80CF`)

`01 01 01 01 01 00 01 00 00 08 FF 00 00 00 F0 F0`

Freq `$01` = −32 Hz under the formula (sub-audible). This region may belong to the noise or bit-shift voice paths rather than the table-driven engine. Needs gameplay observation to resolve — watch what Y values are actually written to the loop-point variable during play.

### File vs Runtime Addresses

The Regenerator disassembly addresses (e.g. `$4E13` for the playback routine, `$2B0E` for sound data) are **file offsets in the compressed PRG**, not runtime addresses. After decompression the runtime layout is different. Confirmed: checking `$4E10` in live RAM shows non-code bytes. Runtime PC during gameplay observed at `$9E31`.

### Frequency Formula

Each byte in the table maps to a frequency:

```
freq_hz = -48 + (16 × decimal_value)
```

Example: `$40` = 64 decimal → `-48 + (16 × 64)` = **976 Hz**

- Duration: ~22ms per byte
- Waveform: triangle
- SID register: Voice 3 (`$D40F` = freq high byte, `$D40E` = freq low byte)

### "Saucer Move" Sound Effect (`$2B0E`)

8 frequency bytes — a U-shaped sweep down then back up:

| # | Hex | Dec | Hz |
|---|-----|-----|----|
| 0 | $40 | 64  | 976 |
| 1 | $38 | 56  | 848 |
| 2 | $30 | 48  | 720 |
| 3 | $28 | 40  | 592 |
| 4 | $20 | 32  | 464 |
| 5 | $28 | 40  | 592 |
| 6 | $30 | 48  | 720 |
| 7 | $38 | 56  | 848 |

Preview tool: `sfx-preview.html` (open in browser).

### Other `$D40F` Write Sites

| Address | What it does |
|---------|-------------|
| `$3F9F` | `STX #$11` — hardcoded value, probably init/reset |
| `$3FB9` | Sets control reg to `$81` (noise waveform) — different sound type |
| `$43B0` | Computes frequency via bit-shifting (ASL/LSR/ROR) — not table-driven |

---

## Open Questions

- ~~Where is the copy routine that moves sound data from `$2B0E` (file) to `$80A0` (runtime)?~~ **Answered**: the depacker IS the mechanism — no separate copy.
- What is the full set of sound effects? V-sweep (Y=1) and saucer move (Y=$10) confirmed. Seven probable single-note effects at Y=$30–$5F. Region Y=$20 (`$80C0`) ambiguous — needs gameplay observation of actual Y values used.
- How is the loop-point index (Y) set before the playback routine runs? (What triggers a sound and sets `e0048`/Y?)
- Regenerator file addresses (`$4E13`, `$2B0E`, etc.) don't match runtime layout. What are the true runtime addresses for the sound engine and other key routines?
