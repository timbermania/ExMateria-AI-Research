# Crystal Status Visual

How the FFT PSX renders `{92}` Inflict Status SS=1 ("turn to crystal"): a dedicated animated diamond graphic keyed on the Crystal bit (`+0x58 & 0x40`), NOT a palette recolor — a live SS=1 probe showed the poison-style status-recolor pass (`FUN_800927bc`, offsets 0/8/0) does not fire (writer filtered on `ra=0x800822AC` → zero writes) and the unit's own sprite fields (`+0x06` sprite-set, `+0x1E0` SHP frame) are unchanged. The asset is 8 frames in `BATTLE/OTHER.SPR` at y=101..122 with palette = OTHER.SPR sub-palette row 10 (matched against the live VRAM crystal CLUT at Y=483 up to a +(−1,0,+1) rounding); `godot-learning/tools/extract_crystal.py` extracts the sheet deterministically, and the Godot port draws it as an 8-frame billboard (`CrystalSprite3D`) that replaces the body. The OTHER.SEQ playback cadence and the spawn/render path that substitutes the graphic on the crystal bit were not decoded in the source doc.

## Points

- **Crystal (SS=1) is a dedicated animated graphic, not a status-recolor tint: the shared status-recolor dispatch's green pass (`FUN_800927bc`, offsets 0/8/0) does NOT fire on SS=1 — the writer filtered on `ra=0x800822AC` got zero writes, only the near-identity pass-1 palette load runs — and the unit's sprite fields (`+0x06` sprite-set, `+0x1E0` SHP frame) are unchanged; the crystal is drawn as a separate graphic keyed on the crystal bit (`+0x58 & 0x40`) and shimmers via its sprite frames (the palette itself is static over time).** — `[S·D·R] 3/3`
  - S: writer `0x800822AC` (filtered, zero writes), `FUN_800927bc`, unit-struct offsets `+0x06`/`+0x1E0`/`+0x58` (doc §12.7)
  - D: op92 `ss_01` live capture + filtered-writer probe (2026-07-05)
  - R: `godot-learning/src/units/Unit.gd` `scenario_inflict_crystal` (hides the body mesh, spawns the billboard) + `src/units/CrystalSprite3D.gd` — validated by `tests/ScenarioInflictStatusTest.gd` `_test_status1_crystallizes_and_hides_body` / `_test_status1_does_not_halt`
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`
- **The crystal asset is 8 frames in `BATTLE/OTHER.SPR`'s top uncompressed 256×256 4bpp section at y = 101..122 (22px tall, x-starts {3,19,34,50,66,82,98,115}, ~16px pitch, widths 11–14; a detailed diamond → bright white/cyan diamond pulse) with palette = OTHER.SPR sub-palette row 10 — confirmed against the live VRAM crystal CLUT (Y=483) as row 10 with a +(−1,0,+1) rounding from the palette-load pass, and recoloring the OTHER.SPR diamonds with row 10 reproduces the on-screen crystal exactly.** — `[S·D·R] 3/3`
  - S: `BATTLE/OTHER.SPR` top section y=101..122 + sub-palette row 10 (ISO-derived; doc §12.7)
  - D: match against the live VRAM crystal CLUT Y=483, op92 `ss_01` (2026-07-05)
  - R: `godot-learning/tools/extract_crystal.py` (deterministic, pure file read) → `res://assets/sprites/textures/crystal/crystal_sheet.png` consumed by `src/units/CrystalSprite3D.gd` — validated by `tests/ScenarioInflictStatusTest.gd` `_test_status1_crystallizes_and_hides_body`
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`

## Notes

(empty — user territory)

## Related

- [[Inflict Status Opcode]]
- [[Poison Visual Recolor]]
- [[Event VM Index]]
