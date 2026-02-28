# Encounter (C64) — Reverse Engineering Project

## Project Goal
Reverse engineer the Commodore 64 game "Encounter" for learning and exploration.

## Current Findings
**Read `notes.md` for all current reverse engineering findings.** It is the canonical record of what has been discovered and should be kept up to date. When the user says "record this" or "write this to md", update both `notes.md` (human-readable findings) and this file if project-level context changes.

## Tools
- **VICE Emulator** — installed locally, run with `x64sc -autostart encounter.prg`
- **Regenerator 2000** — 6502 disassembler, MCP server on localhost:3000
- **SFX preview tool** — `sfx-preview.html`, open in browser

## Files
| File | Description |
|------|-------------|
| `encounter.p00` | Original — PC64 Universal Loader format (20,289 bytes) |
| `encounter.prg` | Extracted PRG — use this with Regenerator (20,263 bytes) |
| `notes.md` | Reverse engineering findings — **read this first** |
| `sfx-preview.html` | Browser tool for aurally previewing SFX byte sequences |

## P00 Format
Header is **26 bytes**: `"C64File\0"` (8) + filename PETSCII padded (16) + record size (2). PRG extracted by stripping first 26 bytes.

## Regenerator 2000 — Tool Notes
- **Merged byte lines**: adjacent bytes auto-merge into one `.byte` line; only the first byte's side comment is shown. Write side comments to describe the whole group, not just the first byte.
- **Phantom xrefs**: branch instructions in data regions generate fake xrefs. Always check `get_cross_references` on both ends — if neither the source nor the target has real inbound xrefs from known code, it's a phantom.
- **No xref = indirect pointer**: if a data region has no xrefs at all, the code accesses it via a computed/zero-page pointer, not a hardcoded address. Search for the address bytes (little-endian) in data regions, or trace from known I/O writes.

## User Preferences
- Always update `notes.md` when recording findings — it is what the user reads
- Keep explanations clear and educational (user is learning reverse engineering)
