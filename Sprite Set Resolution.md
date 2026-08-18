# Sprite Set Resolution

How FFT turns an ENTD unit's `sprite_set` byte and `job` byte into an actual SPR file. The two bytes are independent fields: `sprite_set < 0x80` is a direct SPR file index for named/story units (job irrelevant to art), while `0x80`/`0x81`/`0x82` are the only defined high-range marker values (Generic Male / Generic Female / Monster) whose sprite is derived from `job` by fixed per-class formulas. The marker semantics apply only in the ENTD sprite-set context — the same numbers are real human SPR files elsewhere — and the resolver must key on the `sprite_set` value, not the `flags1` monster bit. In the runtime (cinematic) context, sprite_sets can additionally carry high bit 0x200, in which case the real SPR id is the low byte (`ss & 0xff`).

## Points

- **`sprite_set` (ENTD +0x00) and `job` (ENTD +0x0A) are independent fields, not redundant: `sprite_set` selects whose art a unit wears, `job` selects its class (stats/abilities/AI), and the two have separate 256-entry name tables in FFTPatcher; for the ~200 named/story slots the sprite comes straight from `sprite_set` and job is ignored for rendering, so apparent `sprite_set == job` on some special-character slots is an authoring coincidence (counter-example: Ovelia carries `sprite_set = 0x0C` with `job = 0x5E` Princess).** — `[S·R] 2/3`
  - S: FFTPatcher `PatcherLib/Resources/PSX-US/SpriteSets.xml` (separate 256-entry table) + `Datatypes/ENTD/EventUnit.cs` (`SpriteSet = bytes[0]`, `Job = bytes[10]`)
  - R: `godot-learning/src/scenarios/ScenarioPlayerScene.gd` `_resolve_sprite_set` (returns the byte unchanged for `< 0x80`, job ignored), validated by `godot-learning/tests/ResolveSpriteSetTest.gd` (named passthrough + Ovelia trap cases)
  - src: `research/key_documents/SPRITE_SET_RESOLUTION.md`
- **The resolution threshold is `sprite_set >= 0x80`: values below 0x80 are direct SPR file indices, and only `0x80` (Generic Male), `0x81` (Generic Female), `0x82` (Monster) are defined in the high range — `0x83`–`0x9F` are undefined in the ENTD sprite-set namespace (beyond them only `0xFE`/`0xFF` filler names).** — `[S·R] 2/3`
  - S: FFTPatcher `PatcherLib/Resources/PSX-US/SpriteSets.xml` — no `0x83`–`0x9F` entries
  - R: `godot-learning/src/scenarios/ScenarioPlayerScene.gd` `_resolve_sprite_set` implements the `< 0x80` split, validated by `godot-learning/tests/ResolveSpriteSetTest.gd`
  - src: `research/key_documents/SPRITE_SET_RESOLUTION.md`
- **Generic humans (jobs `0x4A`–`0x5D`) derive their SPR from job as `SPR = 0x60 + (job - 0x4A) * 2`, +1 for the female variant — e.g. `0x80` + job `0x4C` (Knight) → `0x64`, `0x81` + job `0x4C` → `0x65`.** — `[S·R·D] 3/3`
  - S: FFTPatcher `PatcherLib/ISOPatching/PspIso.cs` (`JobFormationSpritesJobCheckID`, `JobFormationSprites1` = 0x4A bytes); formula in `research/lua_scripts/lua_scripts/roster_editor.lua`
  - R: `godot-learning/src/data/JobDatabase.gd` `get_sprite_id` (flat dict resolved at extract time by `tools/extract_fft_data.py`), validated by `godot-learning/tests/ResolveSpriteSetTest.gd` (0x80/0x81 job 0x4C → 0x64/0x65)
  - R: `godot-learning/src/scenarios/ScenarioPlayerScene.gd` `_resolve_sprite_set` (0x80/0x81 → `JobDatabase.get_sprite_id(job, gender from flags1_decoded.female)`, landed 2026-06-28), validated by headful chapel spawn log (bit-exact) + `godot-learning/tests/ScenarioPaletteResolutionTest.gd` (8/8) (2026-06-28), per `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`
  - D: chapel PSX live roster, full-cast scene (scenario_id=4, 2026-06-28): the ENTD record-256 generic-soldier slots render as runtime sprite_sets 0x0060/0x0065/0x0262/0x0264/0x0266 — bit-exact to the formula (high bit 0x200 set on the three Red soldiers), per `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`
  - src: `research/key_documents/SPRITE_SET_RESOLUTION.md`
- **Monsters (job `>= 0x5E`) derive their SPR as `SPR = 0x86 + floor((job - 0x5E) / 3)` — each 3-job monster family shares one sheet, and Chocobo `job = 0x5E` → `SPR = 0x86`.** — `[S·R] 2/3`
  - S: FFTPatcher `PatcherLib/ISOPatching/PspIso.cs` `JobFormationSprites2` (74 × 2-byte LE, contains word `0x0086`)
  - R: `godot-learning/src/data/JobDatabase.gd` `get_sprite_id("5e")` returns `0x86`; Chocobo `SPR 0x86` is parsed asset `godot-learning/assets/sprites/textures/86.tga`; validated by `godot-learning/tests/ResolveSpriteSetTest.gd` (0x82 + job 0x5E → 0x86)
  - src: `research/key_documents/SPRITE_SET_RESOLUTION.md`
- **The marker values collide with real sprite IDs: `0x80`/`0x81`/`0x82` are also real human SPR files (Male/Female Calculator, Male Bard) — marker semantics apply only in the ENTD `sprite_set` context, and treating `sprite_set = 0x82` as a raw ID loads `82.SPR` = "Male Bard" (a human) instead of a monster.** — `[S·R] 2/3`
  - S: FFTPatcher `PatcherLib/Resources/PSX-US/SpriteSets.xml` — the same numeric values are defined as real sprite names
  - R: `godot-learning/src/scenarios/ScenarioPlayerScene.gd` `_resolve_sprite_set` + `godot-learning/tests/ResolveSpriteSetTest.gd` (pins 0x82 + job 0x5E resolving to 0x86, not raw 0x82 "Male Bard")
  - src: `research/key_documents/SPRITE_SET_RESOLUTION.md`
- **Only the three marker values occur at `sprite_set >= 0x80` in the shipped ENTD, and the Chocobo slots (ENTD records 297 `0x129` / 425 `0x1A9`: `sprite_set 0x82`, `job 0x5E`) carry `monster=false` and `female=true` — so resolution must key on the `sprite_set` VALUE, not the `flags1` monster bit; a flag-gated resolver leaks those slots through as raw `0x82`.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/ScenarioPlayerScene.gd` `_resolve_sprite_set` keys on the value, validated by `godot-learning/tests/ResolveSpriteSetTest.gd` (0x82 with monster=false regression case, pinned from ENTD records 297/425; confirmed by 2026-07-05 code review against real data)
  - src: `research/key_documents/SPRITE_SET_RESOLUTION.md`
- **Runtime sprite_sets can carry high bit 0x200 (observed 0x262/0x264/0x266 on the chapel Red soldiers at runtime); the real SPR id is the low byte `ss & 0xff` (0x62 ITEM_M, 0x64 KNIGHT_M, 0x66 YUMI_M), while `sprite_files.json` only maps 0x01–0x9A.** — `[D] 1/3`
  - D: chapel full-cast live roster + per-slot VRAM CLUT bit-match (scenario_id=4, 2026-06-28; `_bitmatch.py` / `_clut_live.json` in the doc dir)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`

## Notes

(empty — user territory)

## Related

- [[ENTD Unit Deployment Table]]
- [[Add Ghost Unit Opcode]]
- [[Cinematic Palette Pipeline]]
