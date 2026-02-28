# Encounter C64 — Reverse Engineering Notes

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
- **Runtime address** — `$80A0` is not in the loaded file. Data is copied there from the file at startup (copy routine not yet found)

### File vs Runtime Addresses

Sound effect data is stored in the file around `$2B0E`. At startup the game copies it to `$80A0` in RAM. A previous investigation found data at `$80B0` — that is `$80A0 + $10`, an indexed offset into the same runtime table. Both observations are consistent.

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

- Where is the copy routine that moves sound data from `$2B0E` (file) to `$80A0` (runtime)?
- What is the full structure of the sound effect table region starting at `$2B0E`?
- How is the loop-point index (loaded into Y at `$4E13`) set before the routine runs?
