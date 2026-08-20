# Show Map Title Opcode

The PSX event opcode `{91} ShowMapTitle(X, Y, Speed)` shows the pre-battle location-name text strip from `EVENT/MAPTITLE.BIN`, selected by the current battle map (strip = map_id − 1), not by the opcode operand. It is revealed with a left→right additive wipe, held for 110 frames, then erased with a second left→right wipe (left end dies first) — not the `{7D}` global brightness fade. The opcode is blocking: the BATTLE.BIN dispatch case ends in `FUN_8014c9d0`, which yields the event VM until the ATTACK.OUT task goes inactive, with no following `{E5}`. The worker, all placement/timing constants, and the external 16-entry CLUT are ROM-sourced from `EVENT/ATTACK.OUT`; the mechanism was decoded from static disassembly cross-checked against a live frame-by-frame scenario-8 capture (2026-07-10). The Godot port has landed: `tools/parse_map_titles.py` (256×20/4bpp decode, ROM-derived CLUT, manifest), `ScenarioMapTitle.gd` (dual-pass additive text + subtractive shadow, ROM-sourced render params), and `ScenarioMapTitleTest.gd` (including the VM-blocks-on-title guard).

## Points

- **`{91} ShowMapTitle` is a 3-operand instruction (X, Y, Speed — one byte each, instruction size 4 per size table `0x8014D170[0x91]=0x03`) that shows the pre-battle location-name text strip; the operand carries NO image index, so the location image is selected by map/scenario context.** — `[S·D·R] 3/3`
  - S: size-table byte `0x8014D201` (`[0x91]=0x03`), dispatch case `0x80145E10`, `project-assets/fft-rom/battle_disassembly.txt` + `battle_decompilation.c`
  - D: scenario 8 live capture (2026-07-10) — raw instruction `91 00 00 01` at PC43
  - R: `godot-learning/src/scenarios/EventInstruction.gd` (`SHOW_MAP_TITLE = 0x91`) + `ScenarioDecode.show_map_title` intent; guard `godot-learning/tests/ScenarioMapTitleTest.gd` ("X/Y/Speed decoded" asserts)
  - src: `research/working_documents/scenario_1_captures/show_map_title_op91_decode.md`
- **The `{91}` case spawns its task from the resident ATTACK.OUT overlay instead of swapping ETC.OUT like `{7D}`: it grabs a free slot (`FUN_80149bec(0x10)`), posts graphics-mailbox cmd `0x0D` (`FUN_8013bc14 @ 0x8013BC14`, word `DAT_80166004`), inlines worker `SUB_801c9ec0` (ATTACK.OUT file off `0xAEC0`), spawns task body `0x801C9D68` (file off `0xAD68`), and stashes X/Y/Speed sign-extended into slot +0x00/+0x04/+0x08 via `FUN_8014ca38` — ATTACK.OUT (scene-actor renderer) is already resident at park in the shared overlay slot `0x801BF000` (spans `0x801BF000..0x801DDC44`), so no overlay swap is needed.** — `[S·D·R] 3/3`
  - S: case body `0x80145E10`–`0x80145E5C`, param stash `FUN_8014CA38`, `battle_disassembly.txt`; ATTACK.OUT file extents `0xAEC0`/`0xAD68` vs ETC.OUT `0x1D7C`
  - D: scenario 8 PC38→PC43 drive (2026-07-10) — both VRAM uploads fire from `SUB_801c9ec0` (`ra` `0x801C9F54` strip / `0x801C9F6C` CLUT, GsLoadImage log)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`_op_show_map_title` @2875 → `ScenarioApply.show_map_title` @551 → `ScenarioMapTitle` node); guard `godot-learning/tests/ScenarioMapTitleTest.gd` (handler bound, "map title started + live")
  - src: `research/working_documents/scenario_1_captures/show_map_title_op91_decode.md`
- **`{91}` is a BLOCKING opcode: the dispatch case ends with `FUN_8014c9d0(slot)` (`0x8014C9D0`), which yields the main event context until the spawned task's active byte (slot+0x48) clears — the VM does not advance past `{91}` until the whole reveal→hold→erase completes (~591 frames at Speed 1); no following `{E5}` is required, in contrast to `{7D}`'s two external `{E5}(0x3D)` barriers.** — `[S·D·R] 3/3`
  - S: call site `0x80145E5C`, `FUN_8014c9d0 @ 0x8014C9D0`, `battle_disassembly.txt`
  - D: scenario 8 live capture (2026-07-10) — sepia tint stays up through the whole title, event cursor sits in the ATTACK.OUT worker / at `{91}` until the strip erases, then scene restore + PC54 dialogue
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_show_map_title` arms `_arm_wait_until(not is_live())` on the main context; guard `godot-learning/tests/ScenarioMapTitleTest.gd` ("VM main context BLOCKS on the map title (not fire-and-forget)", `wait_until` releases after full erase)
  - src: `research/working_documents/scenario_1_captures/show_map_title_op91_decode.md`
- **`EVENT/MAPTITLE.BIN` is a flat fixed-stride array — 307,200 B = 120 strips × 2,560 B, each strip one 256×20 4bpp indexed image (low nibble = left pixel, no in-block header, no in-block CLUT, index 0 = transparent background) — with 115 inked strips carrying location names (the Shishi/FFTacText "Pre-battle location text images" group); strip 23 (`0x17`) @ file offset `0xE800` is "Military Academy's Auditorium".** — `[S·D·R] 3/3`
  - S: `EVENT/MAPTITLE.BIN` static layout (307,200 B = 120×2560; strip 23 @ file off `0xE800`)
  - D: scenario 8 live capture (2026-07-10) — captured VRAM strip decodes to a unique byte-exact match of MAPTITLE strip 23, 5120/5120 texels equal (runner-up block 3134/4096); upload rect (448,224,64,20)
  - R: `godot-learning/tools/parse_map_titles.py` (`decode_title` 256×20/4bpp low-nibble-left, `is_blank` pad detection) + emitted manifest; guard `godot-learning/tests/ScenarioMapTitleTest.gd`
  - src: `research/working_documents/scenario_1_captures/show_map_title_op91_decode.md`
- **The MAPTITLE strip index = map_id − 1: the ATTACK.OUT worker reads event var `0x33` (the battle map_id) and uploads strip = var − 1 (block = strip/4, col = strip%4), so MAPTITLE.BIN is a flat map_id−1-ordered array — verified beyond scenario 8: map 104 → strip 103 ("The Beoulve Residence", scenario 14 silent-no-op case) and every independently-decoded KNOWN_NAMES slot 0..29 equals `map_names[slot+1]`.** — `[S·D·R] 3/3`
  - S: worker `attackout_map_title_worker_entry @ 0x801C9EC0` (ATTACK.OUT file off `0xAEC0`): `ori $a0,0x33` @ `0x801C9ED8`, `jal get_event_var` @ `0x801C9EDC`, `addiu $v1,$v0,-1` @ `0x801C9EE4` (static disasm, 2026-07-12)
  - D: scenario 8 byte-match map 24→23 (2026-07-10); scenario 14 map 104→103 strip image
  - R: `godot-learning/tools/parse_map_titles.py` manifest `map_id_to_slot` + `ScenarioMapTitle.slot_for_map`; guard `godot-learning/tests/ScenarioMapTitleTest.gd` ("map 24 -> MAPTITLE slot 23", "unknown map -> slot -1")
  - src: `research/working_documents/scenario_1_captures/show_map_title_op91_decode.md`
- **The title phase model is reveal (LEFT→RIGHT wipe) → hold → erase (a SECOND left→right wipe whose front sweeps rightward, so the left end vanishes first) — NOT `{7D}`'s global uniform brightness fade; the ATTACK.OUT task self-drives all phases on the per-vblank tick, with `Speed` scaling the per-frame wipe step.** — `[S·D·R] 3/3`
  - S: `EVENT/ATTACK.OUT` file offs (VA `0x801BF000`-relative): erase flag `a3=1` @ `0xAE68` (VA `0x801C9E68`), erase limit `slti 249` @ `0xAE88` (VA `0x801C9E88`), reveal grow-limit `slti 248` @ `0xADD8` (VA `0x801C9DD8`)
  - D: scenario 8 frame captures (2026-07-10): s2 reveal front x[45..56] (only "M…" in), s3 fully revealed x[45..211], s4 mid-erase x[70..211] (left end x[45..69] already gone)
  - R: `godot-learning/src/scenarios/ScenarioMapTitle.gd` (`reveal_front`/`erase_front` L→R sweeps, erase front static during reveal + hold); guard `godot-learning/tests/ScenarioMapTitleTest.gd` ("erase front swept through the mid range (L->R wipe)")
  - src: `research/working_documents/scenario_1_captures/show_map_title_op91_decode.md`
- **The strip renders in two passes: an ADDITIVE grey text pass (scene STP 0→1, `result = scene + (g,g,g)` with equal R=G=B deltas, PSX semi-trans mode 1 B+F; grey = the CLUT texel value) plus a SUBTRACTIVE drop-shadow at ~(+1,+1) px (PSX abr 2, B−F) gated to the glyph negative space; CLUT index 0 (0x0000) is transparent. The subtractive component is VRAM-confirmed but its exact ROM draw-path is NOT statically pinned — the `{7D}` `0x26/0x46` abr immediates are not literal in the ATTACK.OUT map-title builder region.** — `[D·R] 2/3`
  - D: scenario 8 s1-vs-s3 framebuffer diff (2026-07-10): 808 brighter (additive text) and 426 darker (shadow, many driven to (0,0,0)) pixels; best overlap 345/808 at offset (+1,+1)
  - R: `godot-learning/src/scenarios/ScenarioMapTitle.gd` `_build_render` (two passes: additive text quad + subtractive `show_map_title_shadow.gdshader` at offset (+1,+1), render priorities 101/102) + `godot-learning/tests/ScenarioScreenOverlayTest.gd` (ScenarioMapTitle node)
  - src: `research/working_documents/scenario_1_captures/show_map_title_op91_decode.md`
- **The MAPTITLE palette is external to the .BIN — a fixed 16-entry grey ramp embedded in `EVENT/ATTACK.OUT` at file off `0x16FF0` (VA `0x801D5FF0`), uploaded by the worker to VRAM (448,255) (CLUT word `0x3FDC` = `(255<<6)|(448>>4)`); idx0 = 0x0000 (transparent), idx15 = 0xFFFF (white), monotone ramp between; ROM source byte-identical to the live capture (distinct from, though visually similar to, the CHAPTER card ramp).** — `[S·D·R] 3/3`
  - S: `EVENT/ATTACK.OUT` file off `0x16FF0` (= VA `0x801D5FF0`) + upload descriptor file off `0x16FC8`
  - D: scenario 8 live capture (2026-07-10) — live VRAM CLUT @ `0x801D5FF0` byte-matches the ROM ramp; CLUT upload rect (448,255,16,1)
  - R: `godot-learning/tools/parse_map_titles.py` `read_maptitle_clut` (reads ATTACK.OUT @0x16FF0, cross-checks against the baked copy, raises on divergence) + manifest `clut_source`
  - src: `research/working_documents/scenario_1_captures/show_map_title_op91_decode.md`
- **Every placement/timing constant of the title quad is ROM-sourced from ATTACK.OUT immediates — prim base X+128 / Y+96, quad height 20, reveal grow start 8 → limit 248, hold 110 frames, erase limit 249, interior grey 128, soft-edge kernel 32 px, Speed step = slot+0x08 clamped to min 1 — and the quad (built into BSS scratch, no file-resident geometry template) maps to the observed screen band x[45..211] / y[100..108] via one global battle draw-env offset (NOT in ATTACK.OUT, GAP-6.2).** — `[S·D·R] 3/3`
  - S: `EVENT/ATTACK.OUT` file offs: `0xA9A0` (`80000225`, +128), `0xAA68` (`6000c324`, +96), `0xAA9C` (+20), `0xAD98` (start 8), `0xADD8` (248), `0xAE24` (`6e00222a`, 110), `0xAE88` (`f900222a`, 249), `0xADC4` (`80000634`, grey 128), `0xABD4` (`20000434`, edge 32), `0xAD8C` (speed min-1)
  - D: scenario 8 capture (2026-07-10) observed band x[45..211] / y[100..108 (displayed buffer y[0..240])
  - R: `godot-learning/src/scenarios/ScenarioMapTitle.gd` `configure_render` adopts the manifest `render` block + `parse_map_titles.py` `render_source`; guard `godot-learning/tests/ScenarioMapTitleTest.gd` ("grow_limit 248 (ATTACK.OUT 0xADD8)", "hold 110 frames (ATTACK.OUT 0xAE24)", "edge 32px (ATTACK.OUT 0xABD4)", "prim_y_base 96 (ATTACK.OUT 0xAA68)")
  - src: `research/working_documents/scenario_1_captures/show_map_title_op91_decode.md`

## Notes

(empty — user territory)

## Related

- [[Show Graphic Opcode]]
- [[Event Opcode Catalog]]
- [[Scenario Wait Semantics]]
- [[Web-psx Cross-Validation]]
- [[DarkScreen Opcode]]
