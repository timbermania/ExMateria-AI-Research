# Selection Orb And Floor Spotlight

The per-unit focus marker: the additive "selection orb" (pulsing on the selected unit) and the floor spotlight it sits on — brightness and falloff sourced from the same `orb_spatial_falloff`.

## Points

- **The selection orb is a 12×12 sprite at sheet (243,73)–(254,84) of the shared FRAME/range sheet (LBA 0xE68 +0x1000, 256×256 4bpp == RANGETILE, tpage 0x3F → VRAM (960,256)) — radially symmetric: transparent corners, dark-blue ring, bright-cyan core; its CLUT is 16 BGR555 colours uploaded to VRAM (624,508) as clut-id `0x7F27`, disc source `EVENT/FRAME.BIN @ 0x9280` (the FRAME.BIN palette tail: `0x9160` bars, `0x91C0` background stone, `0x9280` orb) — a separate palette from the range-tile CLUT column; the prim is an additive `POLY_FT4` (command `0x2E`, ABR mode 1 from tpage bits 5–6) and the pulse is an additive-gouraud brightness modulation, NOT a palette cycle: CLUT and texel are byte-static, only the prim's R=G=B gouraud ramps.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §10.1–10.2 (texel cell, CLUT bytes + upload target, prim packet layout at RAM `0x801F10D8`, rock-constant `clut=0x7F27`/`tpage=0x003F`/`cmd=0x2E` under prim-watch)
  - D: full-VRAM diff across 4 pulse phases shows zero change outside the framebuffer (CLUT + texel static); framebuffer captures of the 12×12 orb region
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/; the earlier port orb shaders live on the import-godot-game branch)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The orb's brightness is a discrete symmetric ping-pong with an 84-frame DISPLAYED period (~1.40 s): ±4 per 2 frames, held 2 frames per level (4 at the endpoints) — the older "~42f" reading was the write-BP sampling the code at ~2× display rate; base brightness is `orb_spatial_falloff = clamp(200 − sqrt(dx² + 4dy²)/64, 80, 128)` (writer `0x8018BAD0`) with (dx,dy) = cell centre − cursor for non-selected units, and `base + triangle(phase)` for the selected one (cell0 pulses 88↔164, 16 distinct levels).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §10.2–10.3 (ping-pong ramp table, 84-frame correction, spatial-falloff writer confirmed by live sampling)
  - D: per-frame brightness sampling of the selected vs non-selected cells (88↔164 selected; [80,128] clamp on non-selected)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The formation floor is a STATIC grid of 64×32 textured cobblestone cells (POLY_G4 with 4 gouraud corner colours; builder `8012cfd4` @ `0x8012cfd4`, corner brightness fed from `FUN_80116e74`'s `local_70`/`local_6b` arrays, ultimately `orb_spatial_falloff`) — the tiles/geometry never move; only the per-vertex brightness does: a horizontal SPOTLIGHT that tracks the selected unit's COLUMN (brightest under the selected column, ~half-luma at the far side, ~2:1 luma swing, fully reversible), while the vertical top→bottom falloff is fixed — moving the selection down within the same column leaves the floor byte-identical; on the equipment sub-screen the pool centre instead chases the sliding unit to ~(164,177), so the sweep must sample the unit's live box centre, not the docked cell; the exact per-corner arithmetic was still OPEN (needs raw-disasm of `FUN_80116e74`).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §14.7.3 (round-5 LIVE correction: static texture + moving column-keyed brightness; unreliable decompile drops the column-offset arg) + §15.24 (pool retarget `(164,177)` on the equip slide)
  - D: 4-column floor-luma sampling with the D-pad driven (left selected: 96/93/73/47; right selected: 46/73/94/98 — the bright band flips L↔R); same-column vertical move → byte-identical floor
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/; see also [[Formation Screen Compositing]]'s port-fit point for `spot_base 1.72`)
  - src: `research/working_documents/FORMATION_SCREEN.md`

## Notes

(empty — user territory)

## Related

- [[Formation Screen Compositing]]
- [[Gold Selection Box]]
- [[Equip Sub Screen]]
