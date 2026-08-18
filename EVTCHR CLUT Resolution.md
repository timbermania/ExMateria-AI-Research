# EVTCHR CLUT Resolution

How FFT PSX colors event-script / EVTCHR cinematic unit sprites, along two independently-selected axes: the CLUT *address* (a slot-indexed VRAM coordinate — X = roster slot × 16, Y = fixed 483 — written by `evtchr_unit_clut_writer` from the allocator-assigned slot, with no unit-struct byte feeding it) and the CLUT *content* (which of the unit's own 16-row SPR sub-palettes gets uploaded, chosen by the ENTD palette byte at record offset `0x17`). Monsters ignore the ENTD byte and take their row from the job (SCUS `0x2E`); generic humans pass the byte through; named unique units author only body row 0, so a stale byte must clamp. Godot's unified `SpritePaletteResolver` (2026-07-05) implements the content-axis rule, validated by two test suites plus a headful scenario-6 run; the PSX-side content-upload trace remains an open live-PSX item.

## Points

- **A cinematic unit's CLUT coordinate derives from the roster slot alone: `evtchr_unit_clut_writer` reads `slot[+0x4]` (lbu @ `0x80087a98`), computes CLUT_X = slot × 16 (`sll a0,s6,0x4` @ `0x80087b98`), takes CLUT_Y as a fixed immediate 483 (`ori a1,zero,0x1e3` @ `0x80087ba0`), and stores the word `(483<<6)|slot` at `slot[+0x10]` via GsGetClut (`sh v0,0x10(s2)` @ `0x80087ba4`); the word then flows `slot[+0x10]` → `render_state[+0x6]` (`FUN_80083e10` @ `0x80083e48`/`0x80083e54`) → `POLY_FT4 packet[+0x0e]` (`poly_ft4_packet_builder` @ `0x8007b2f8`) — no unit-struct byte feeds the coordinate, so the ENTD palette byte does not move the CLUT address.** — `[S·D] 2/3`
  - S: `evtchr_unit_clut_writer` sites `0x80087a98`/`0x80087b98`/`0x80087ba0`/`0x80087ba4`, `FUN_80083e10` @ `0x80083e48`/`0x80083e54`, `poly_ft4_packet_builder` `0x8007b2f8` (BATTLE.BIN disassembly)
  - D: live VRAM bit-match, roster slots 0/1/12 → words `0x78c0`/`0x78c1`/`0x78cc` (source doc §1.1, 2026-07-05; same words as the 2026-06-28 chapel VRAM dump)
  - D: live scn6 "letgo" carry beat (2026-07-09, per `EVTCHR_FRAME_RESOLUTION.md` §3): unit-array index (`0x800B7308`, stride `0x440`) → Ovelia slot 12 → `0x78CC`, Delita slot 5 → `0x78C5`
  - R: none — no `(483<<6)|slot` staging CLUT word in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/` for `0x78c0`/`roster_slot`/`483`); the Godot atlas omits the PSX staging address by design
  - src: `research/working_documents/EVTCHR_CLUT_RESOLUTION.md`
- **`evtchr_unit_queue_drain` @ `0x80088e04` is the sole caller of `evtchr_unit_clut_writer`, so Orbonne-chapel units and scenario-6 carry units share ONE CLUT path — there is no separate cinematic CLUT renderer.** — `[S] 1/3`
  - S: xref `evtchr_unit_queue_drain` `0x80088e04` → `evtchr_unit_clut_writer` `0x80087a28` (BATTLE.BIN disassembly)
  - R: none — the sole-caller CLUT-writer path not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`; only a PSX-address comment at `ScenarioVM.gd:777`)
  - src: `research/working_documents/EVTCHR_CLUT_RESOLUTION.md`
- **The ENTD per-deployed-unit "Palette" byte (record offset `0x17` / `bytes[23]`) is a raw sub-palette ROW index into the unit's own SPR — not an RGB recolor and not the team-ownership field (team is a separate 2-bit field, byte `0x18` mask `0x30`); FFTPatcher models the byte as a raw hex spinner with no enum.** — `[D·R] 2/3`
  - D: live CLUT bit-match, generic humans Blue=0/Red=2 (chapel capture 2026-06-28, per [[Cinematic Palette Pipeline]] bit-match point; re-cited by source doc §3)
  - R: `godot-learning/tools/parse_entd.py` (palette field @ `0x17`) + `godot-learning/src/scenarios/ScenarioPlayerScene.gd` `_resolve_body_palette_row` (passes `slot.palette`, :830), validated by `godot-learning/tests/ResolveBodyPaletteRowTest.gd` (19/0)
  - src: `research/working_documents/EVTCHR_CLUT_RESOLUTION.md`
- **EVTCHR cinematic pixels are colored with the acting unit's OWN SPR palette, NOT the EVTCHR segment's embedded palette: the segment's `+0x0780` embedded palettes exist but the default render path leaves them unused, and `{11}` UnitAnim carries no palette parameter.** — `[S·R] 2/3`
  - S: `evtchr_unit_clut_writer` `0x80087a28` (BATTLE.BIN disassembly); no palette operand in the `{11}` UnitAnim param block
  - R: `godot-learning/src/animation/SpriteLayerManager.gd` cinematic mode (V14: swaps only the pixel atlas, keeps the SPR row bound) + `godot-learning/src/data/SpritePaletteResolver.gd`, validated by headful scenario-6 carry render (2026-07-05) + `godot-learning/tests/ResolveBodyPaletteRowTest.gd` (19/0)
  - src: `research/working_documents/EVTCHR_CLUT_RESOLUTION.md`
- **An SPR's palette section is a 512-byte header at offset 0: 16 sub-palettes × 16 BGR555 colors (1-bit STP + 5-bit B/G/R per color) — the 16 rows the ENTD palette byte indexes.** — `[R] 1/3`
  - R: `godot-learning/tools/extract_spr.py` (`PALETTE_SIZE = 512` at offset 0, `bgr555_to_rgba` bit layout: bits 0–4 R, 5–9 G, 10–14 B, bit 15 STP; no named test)
  - src: `research/working_documents/EVTCHR_CLUT_RESOLUTION.md`
- **A monster's body palette row is a JOB attribute (SCUS `0x2E`), not the ENTD palette byte: the PSX picks the monster row from the job (branch @ `0x8005b770`), and live scenario-6 verified a chocobo (unit `0x8B`, job `0x5E` Yellow) rendering row 0 despite `palette: 2`.** — `[S·D·R] 3/3`
  - S: `0x8005b770` job branch (BATTLE.BIN disassembly); `extract_fft_data.py:294`
  - D: live scenario-6 verification — unit `0x8B` job `0x5E` → yellow, ENTD byte 2 ignored (2026-07-05, source doc §3 matrix)
  - R: `godot-learning/src/data/SpritePaletteResolver.gd` (`resolve_body_palette_row` monster branch + `job_body_palette_row`), validated by `godot-learning/tests/ResolveBodyPaletteRowTest.gd` (19/0) + `godot-learning/tests/CombatBodyPaletteRowTest.gd` (8/0)
  - src: `research/working_documents/EVTCHR_CLUT_RESOLUTION.md`
- **ENTD record 387 slot 12 (Delita, the scenario-6 carry unit) = `{sprite_set: 0x05 (DILY2.SPR "Delita Ch2-3"), palette: 2, job: 0x05}`; `05.palette.tga` populated rows = `{0, 1, 8}` (row 8 = portrait; body rows 0–7 = `{0, 1}`) and row 2 — the ENTD byte value — is all-black, so the resolver clamps to row 0; headful scenario-6 verified `body_palette_row` 2→0 on both `05.tga` and the EVTCHR `segment_001.tga` cinematic atlas (V14 keeps the row bound).** — `[R] 1/3`
  - R: `godot-learning/tools/bake_populated_rows.py` → `godot-learning/assets/sprites/populated_rows.json` (`05` → `[0, 1]`) + `godot-learning/src/data/SpritePaletteResolver.gd`, validated by `godot-learning/tests/ResolveBodyPaletteRowTest.gd` (19/0, Delita clamp case) + headful scenario-6 (2026-07-05)
  - src: `research/working_documents/EVTCHR_CLUT_RESOLUTION.md`
- **A runtime team-recolor path exists: "Palette modification based on team" at `0x00068534`–`0x00068630` (reads `unit+0x5 & 0x30`) is gated by the "modified palette byte" at `unit+0x13e` — `FUN_800822bc` skips the recolor at `0x800822c4`/`0x800822cc` if that byte is nonzero — and it is a candidate mechanism for enemy-squad recolor.** — `[S] 1/3`
  - S: `0x00068534`–`0x00068630`, `FUN_800822bc` skip sites `0x800822c4`/`0x800822cc` (BATTLE.BIN disassembly)
  - R: none — team-recolor path / `unit+0x13e` modified-palette byte not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/` for `0x13e`/`team_recolor`/`0x68534`)
  - src: `research/working_documents/EVTCHR_CLUT_RESOLUTION.md`
- **Godot's unified `SpritePaletteResolver.resolve_body_palette_row(slot, sprite_id)` collapses the four historical body-palette-row sources into one rule: monster → the job's `body_palette_row` (unclamped); generic human → the ENTD byte passes through when that row is populated in the SPR (generic SPRs author rows 0–4, so Blue=0/Red=2 survive); named unique SPR → clamp to row 0 when the byte points at an unpopulated row; the populated rows are a bake-time fact in `assets/sprites/populated_rows.json` scanned from the ISO-derived `NN.palette.tga` (body rows 0–7; `05`→`[0,1]`, generics `60`/`62`→`[0,1,2,3,4]`, named `01`/`0C`→`[0]`).** — `[R] 1/3`
  - R: `godot-learning/src/data/SpritePaletteResolver.gd` + `godot-learning/tools/bake_populated_rows.py`, validated by `godot-learning/tests/ResolveBodyPaletteRowTest.gd` (19/0) + `bake_populated_rows.py --check` pre-flight in `godot-learning/tests/run_all_tests.sh`
  - src: `research/working_documents/EVTCHR_CLUT_RESOLUTION.md`
- **The JOB (CONTENT) palette axis has one owner, `SpritePaletteResolver.job_body_palette_row(job_id_hex)` — the raw jobs.json `body_palette_row`, unclamped — consumed by `Unit.change_job`, `BaseRoster.spawn_unit`, and the resolver's monster branch; `is_monster` is `kind=="monster"` ONLY, so special non-humanoid Holy Dragon (job `0x48`) keeps its authored row 3 instead of being clamped 3→0 by the humanoid branch.** — `[R] 1/3`
  - R: `godot-learning/src/data/SpritePaletteResolver.gd` `job_body_palette_row`, consumed at `godot-learning/src/units/Unit.gd` (change_job) and `godot-learning/src/roster/BaseRoster.gd` (spawn_unit), validated by `godot-learning/tests/CombatBodyPaletteRowTest.gd` (8/0)
  - src: `research/working_documents/EVTCHR_CLUT_RESOLUTION.md`
- **A `body_palette_row` set before `add_child` never reached the shader (the setter's material did not exist yet), so roster-spawned units rendered uncolored; `Unit._load_initial_sprite_texture` now re-seeds the palette-row uniform once `sprite_layers` is live, mirroring the sprite-texture re-application.** — `[R] 1/3`
  - R: `godot-learning/src/units/Unit.gd` `_load_initial_sprite_texture` re-seed + `godot-learning/src/roster/BaseRoster.gd` `spawn_unit` (row set before `add_child`), validated by `godot-learning/tests/CombatBodyPaletteRowTest.gd` (8/0, real `BaseRoster.spawn_unit` path propagates the row onto a live Unit)
  - src: `research/working_documents/EVTCHR_CLUT_RESOLUTION.md`

## Notes

(empty — user territory)

## Related

- [[Cinematic Palette Pipeline]]
- [[ENTD Unit Deployment Table]]
- [[Cinematic Sprite Renderer]]
- [[Sprite Set Resolution]]
- [[Reset Palette Opcode]]
- [[Unit Anim Opcode]]
- [[EVTCHR Frame Resolution]]
