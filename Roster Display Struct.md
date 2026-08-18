# Roster Display Struct

The per-party-member roster display struct the formation screen builds, and the job→sprite/palette resolvers it feeds.

## Points

- **The roster display struct has base `0x801C8638`, stride `0x128` (≤20 slots), indexed via a pointer array at `0x801CD5EC`; `+0x24` holds the JOB ID, not the level; it is built by `FUN_801210E8` → `FUN_80120BB0`, which copies from the unit struct and derives the display fields.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §7 (struct base/stride/pointer array, `+0x24` = JOB ID, builder chain)
  - D: live RAM reads across roster entry confirming the layout
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **Roster body-sprite resolution: job → descriptor table → a 24×40 cell at tpage `0x64` (VRAM (256,0)); the general indexing rule is `idx = job·2 − 0x7C`, +1 when the unit's `+0x70 & 0x40` (female) — so each unit renders one body per gender-appropriate job.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §7/§13 (descriptor table, `idx = job*2 − 0x7C` + female +1 rule)
  - D: live descriptor reads matching the drawn 24×40 tpage-0x64 cells
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/; see also [[Formation Element Placement]]'s 24×40 tpage-0x64 point)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **Per-unit body CLUT selection goes through the `palId` table at `0x8018A168`: `GetClut((palId&3)·0x10 + 0x140, (palId>>2) + 0xE0)` — i.e. the palId byte splits into a 2-bit CLUT-column index and a VRAM row offset.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §13 (palId table + `GetClut` split, live-confirmed per unit)
  - D: live `palId` reads per party member reproducing each body's drawn CLUT
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The per-cell status-marker icon is emitted by `FUN_80118244` (descriptors @ `0x8018C768`/`0x7A4`/`0x7CC`; crystal/treasure/chocobo markers), with the marker id picked from the `+0x11F` pose/status byte (bit0/bit1) combined with `+0x70` status bits (marker ids 9/10/0xB/0x8C).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §13.4 (status-marker builder + id derivation)
  - D: oracle per-cell marker captures matched against the `+0x11F`/`+0x70` bit patterns
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`

## Notes

(empty — user territory)

## Related

- [[Formation Element Placement]]
- [[Formation Screen Compositing]]
- [[Change Job Screen]]
