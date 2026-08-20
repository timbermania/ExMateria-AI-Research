# Show Graphic Opcode

The PSX event opcode `{7D} ShowGraphic(xGR)` fades a full-screen graphic in, holds it, then fades it out, with the script gating on completion via `{E5} WaitForInstruction(0x3D)` / `{8F}`. Its task (kind `0x3D`) uploads the card and draws it through the `ETC.OUT` overlay (shared slot `0x801BF000`) as two semi-transparent POLY_GT4 passes — additive white text + subtractive drop shadow — with a left→right reveal, an 80-frame hold, and a 128-frame global grey fade-out, dithered by the PSX 4×4 ordered dither. As of 2026-07-10 the whole mechanism is decoded (dispatch, task/kind wiring, ID→file map via ETC.OUT's 13-row descriptor table, the four graphic `.BIN` formats, blend/reveal/fade/dither) and every parameter is ISO-deterministic from `EVENT/ETC.OUT` + the graphic `.BIN`s — no savestate needed. The Godot port has landed: deterministic parser (`tools/parse_show_graphics.py` + tests), dual-pass reveal shaders with PSX dither, and kind-`0x3D` WFI liveness in the ScenarioVM.

## Points

- **`{7D} ShowGraphic(xGR)` is a 1-byte-operand full-screen graphic: it fades the graphic in, holds it, then fades it out before the event ends; the `0x7D` dispatch case at `0x8014532C` grabs a task slot via `FUN_80149bec(0x10)`, spawns body `FUN_8013c710` through `event_task_spawn`, and stashes the 1-byte graphic ID in the task param block via `FUN_8014ca38(handle, ID, 0, 0)` (size table `0x8014D170[0x7D]=0x01`).** — `[S·D·R] 3/3`
  - S: dispatch case `0x8014532C`, task body `FUN_8013c710` @`0x8013C710`, size table `0x8014D170` (`battle_disassembly.txt`)
  - D: scenario 8 capture — park at PC40 on a read BP on the opcode byte in the loaded bytecode at RAM `0x8004A7A5` (`7D 01`), pre-dispatch (savestate `reference-assets/scenario_8_pc_38_park.sstate`, 2026-07-10)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`{7D}` handler → `ScenarioShowGraphic`) + `godot-learning/tests/ScenarioShowGraphicTest.gd` (`_test_decode_operand`, `_test_handler_registered_not_skip`)
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **The `{7D}` task body `FUN_8013c710` stamps task kind `0x3D`, takes the graphics-cmd mailbox via `FUN_8013bc14(0xC)` (blocks on `DAT_80166004`), then jumps into the overlay render worker `SUB_801c05d4` — the direct parallel of `{76} DarkScreen` = `FUN_8013BD94` (kind `0x36`, mailbox cmd 6); `{7D}` is kind `0x3D`, cmd `0xC`.** — `[S·D·R] 3/3`
  - S: `battle:0x8013C710` — `event_task_set_kind(0x3D)` + `FUN_8013bc14(0xC)` + `jal SUB_801c05d4` (`battle_disassembly.txt` + ETC.OUT decompile)
  - D: scenario 8 capture — the kind-`0x3D` task is live in slot 2 of the task table (base `0x8016986C`, stride `0x400`, slot base `0x8016A06C` this run) while the card is up (2026-07-10)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`TASK_SHOWGRAPHIC := 61`, kind-`0x3D` liveness in the WFI registry) + `godot-learning/tests/ScenarioShowGraphicTest.gd` (`_test_barrier_predicate_live_then_clear`)
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **`{E5} WaitForInstruction(0x3D, 0x00)` blocks while any task slot's kind byte (`+0x4C`) is `0x3D` — `evt_wfi_E5_poll_case @ 0x80145964` loops the task slots (base `0x8016986c`, stride `0x400`) — and opcode `0x8F` (`LAB_80145cfc`) is a dedicated wait-for-graphic that spins on kind `0x3D` too; the graphic task stays live across fade-in + hold + fade-out, so the barriers release only when the card is fully faded.** — `[S·D·R] 3/3`
  - S: `0x80145964`, `LAB_80145cfc`, `event_task_set_kind @ 0x80149D48` (kind written to slot `+0x4C`), `FUN_80149cbc` find-by-kind (`battle_disassembly.txt`)
  - D: scenario 8 capture — the VM advances THROUGH the two `Wait For Instruction [Task=61]` barriers (PC41/42) exactly when the card's fade-out completes, then halts at `Show Map Title` @PC43 (2026-07-10)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_task_liveness[TASK_SHOWGRAPHIC]` + `godot-learning/tests/ScenarioShowGraphicTest.gd` (`_test_barrier_predicate_live_then_clear`, `_test_barrier_holds_through_hold_phase`); `{8F}` not implemented — `UNKNOWN_0X8F` auto-skipped in `godot-learning/src/scenarios/EventInstruction.gd`
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **`{17} GraphicTimeout` is a NO-OP in BATTLE.BIN: `LAB_80144acc` just advances (size byte 0x05 skipped) — its real fade-out-timeout logic lives in the WORLD.BIN world-map interpreter (not exported), so in battle scenarios it is modelable as advance-only; do not confuse it with opcode `0x37` = a graphic-descriptor fade-flag toggler (`FUN_80149b38`, array `0x8016e43c`).** — `[S] 1/3`
  - S: `LAB_80144acc`, `FUN_80149b38`, array `0x8016e43c` (`battle_disassembly.txt`)
  - R: `godot-learning/src/scenarios/EventInstruction.gd` `UNKNOWN_0X17` + the unnamed-opcode auto-skip in `ScenarioVM.gd` (advance-only, no visual)
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **The graphic operand maps to a file via `row = operand − 1`: `0x01–0x04` → `CHAPTER1–4.BIN`, `0x05–0x06` → `CHAPTER1.BIN` (filler rows), `0x07` → `GAMEOVER.BIN`, `0x08–0x0C` → `END1–5.BIN`, and `0x10–0x91` → `WLDBK.BIN` entry `operand−0x10` — the last range is NOT in ETC.OUT's battle table, it is the world-map path in WORLD.BIN.** — `[S·D·R] 3/3`
  - S: ETC.OUT per-ID descriptor table, file `0x18A8` (VA `0x801C08A8`), re-verified against raw bytes; id map reconciles with the ffhacktics wiki map
  - D: scenario 8 capture — operand `1` loaded `chapter1.bin` (texture + CLUT byte-matched) (2026-07-10)
  - R: `godot-learning/tools/parse_show_graphics.py` (`row_for_operand`) + `godot-learning/tools/test_parse_show_graphics.py` (`test_operand_maps_to_row_minus_one`)
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **The ShowGraphic render worker is ETC.OUT, not ATTACK.OUT: `0x801BF000` is a shared overlay slot, and while the card is up the small `EVENT/ETC.OUT` (7548 B) is loaded over the low part of the slot — the worker at `0x801BF77C` is ETC.OUT code (entry `SUB_801c05d4` = ETC.OUT file `0x15D4`) — and the per-graphic descriptor table read live at `0x801CA0C8` is ATTACK.OUT-residual data (slot offset `0xB0C8` beyond ETC.OUT's 7548 B) that the ShowGraphic worker does not use.** — `[S·D] 2/3`
  - S: ETC.OUT decompile at base `0x801BF000` — worker byte-match; ETC.OUT spans `0x801BF000..0x801C0D7C`; `0x801CA0C8` residual proven by the 32-byte signature match against all `EVENT/*.OUT`
  - D: scenario 8 capture — resident signature at `0x801BF77C` = `f8febd27…` at park (matches ATTACK.OUT) vs `00008290…` while the card is up (matches ETC.OUT) (2026-07-10)
  - R: none — runtime overlay loading not present in godot-learning (probed `godot-learning/src/` + `godot-learning/tests/`; the port renders from the parsed `.BIN` assets directly, with ETC.OUT constants absorbed via `tools/parse_show_graphics.py`)
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **ETC.OUT's master per-graphic table (file `0x18A8`, VA `0x801C08A8`, stride `0x20`, 13 live rows) holds `{RECT*, template*, aux ptrs, format (0=CHAPTER, 1=END-still, 2=movie), filename*, disc sector LBA, size}` — ETC.OUT indexes it with `DAT_801c08b8 + row*0x20` where `row = operand − 1` (listing `0x801c05f0` `sll v0,s0,0x5`) — so for any battle ShowGraphic id the entire effect is computable from `ETC.OUT` + the graphic `.BIN` with no framebuffer or savestate.** — `[S·D·R] 3/3`
  - S: ETC.OUT decompile + field layout re-verified against raw `ETC.OUT` bytes
  - D: scenario 8 capture — operand `1` → row 0 = chapter1.bin, byte-matched (2026-07-10)
  - R: `godot-learning/tools/parse_show_graphics.py` (`DESC_TABLE_OFF = 0x18A8`, `parse_descriptor_table`, `deref_rect`) + `godot-learning/tools/test_parse_show_graphics.py` (`test_table_has_13_rows_and_row0_is_chapter1`, `test_rect_derefs_to_upload_rectangle`)
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **`CHAPTER1–4.BIN` (EVENT/, 8192 B) is a sparse 4bpp indexed strip: pixels `[0x000,0x800)`, zero-fill `[0x800,0x1FE0)`, 16-entry RGB555 CLUT at `[0x1FE0,0x2000)` (last 32 bytes) — 256×16, 4bpp, low-nibble = left pixel; the CLUT is a 16-step grayscale ramp (`0x0000` black → `0xFFFF` white), i.e. anti-aliased white serif text on transparent black.** — `[S·R] 2/3`
  - S: extracted `CHAPTER1.BIN` byte layout + ETC.OUT table row 0 (0x2000 B read to RECT `(384,0,64,64)`)
  - R: `godot-learning/tools/parse_show_graphics.py` (`decode_chapter`, `parse_chapter_clut`) + `godot-learning/tools/test_parse_show_graphics.py` (`test_clut_is_grayscale_ramp_idx0_transparent`, `test_chapter1_render_params`)
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **The remaining graphic formats: `END1–5.BIN` (131072 B) = 256×256 16bpp RGB555 raw framebuffer (bit15 = STP/mask, no header/CLUT); `GAMEOVER.BIN` (65536 B) = 256×256 8bpp indexed with an external CLUT (exact palette unknown — grayscale approx; 16bpp interpretation is garbage); `WLDBK.BIN` (16097280 B) = a headerless raw framebuffer container of exactly 131 back-to-back 256×240 16bpp RGB555 frames at fixed stride `0x1E000` (122880 B = 256×240×2, remainder 0).** — `[S·R] 2/3`
  - S: extracted `END1.BIN` / `GAMEOVER.BIN` / `WLDBK.BIN` byte layouts (131 × 0x1E000 = 16097280 exactly; visually verified: END1 = "Fort Zeakden Fire", WLDBK entry 44 = painted airship-deck background)
  - R: `godot-learning/tools/parse_show_graphics.py` (`decode_rgb555`, `decode_gameover`, WLDBK stride decode) + `godot-learning/tools/test_parse_show_graphics.py`
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **The fmt-0 (CHAPTER) render recipe is fully ISO-sourced: the 8192 B `.BIN` is exactly a 64×64 16bpp VRAM cell, uploaded *whole* via `GsLoadImage` to RECT `(384,0,64,64)` and read back as the 256×16 4bpp texture; the embedded CLUT (file `0x1FE0`) lands at VRAM `(432,63)` automatically — the builder hardcodes the clut word `0x0FDB`; on-screen quads come from the template at VA `0x801C0668` (stride `0xC` records `{x:s8, y:s8, w:s16, h:s16, U:s16, V:s16}`, card content width 248, split into y 0 / y 32 halves) with a constant screen draw-offset `(+256 X, +128 Y)`.** — `[S·D·R] 3/3`
  - S: ETC.OUT builder `0x801BF3E0` + RECT constant `0x801C0648` + quad template `0x801C0668` + table rows (file `0x18A8`)
  - D: scenario 8 capture — uploaded texture at VRAM texpage `x[384..448), y[0..17)`, CLUT at `(432,63)` verified as the 16-step grayscale ramp, on-screen span x≈[11..244] / y≈[78..93] after the GPU draw-offset (2026-07-10)
  - R: `godot-learning/tools/parse_show_graphics.py` (`chapter_render_params`) + `godot-learning/tools/test_parse_show_graphics.py` (`test_chapter1_render_params`, `test_template_grow_limit_is_248`)
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **The ETC.OUT entry `SUB_801c05d4` branches on the row's `format`: fmt 0 (CHAPTER/GAMEOVER) → anim loop `0x801C017C` + builder `0x801BF3E0` (dual-pass gouraud strip); fmt 1 (END1 still) → anim loop `0x801C03AC` + builder `0x801BFC58` (vertical-strip GT4, 256×256 direct colour + CLUT-strip upload); fmt 2 (END2–5) → a per-frame movie stream with the frames inside `end*.bin` — only fmt 0 is dynamically verified, and GAMEOVER shares the fmt-0 dispatch but its table `size` (`0x10000`) is inconsistent with the 8192-B `(384,0,64,64)` rect, so its upload geometry remains unverified.** — `[S·R] 2/3`
  - S: ETC.OUT decompile — `format` field at row `+0x10`; loop/builder addresses; structurally decoded, not yet captured
  - R: `godot-learning/tools/parse_show_graphics.py` (decodes all four graphic classes, emits the per-ID `show_graphics.json` manifest) + `godot-learning/tools/test_parse_show_graphics.py`
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **The card is drawn as two semi-transparent POLY_GT4 passes — an ADDITIVE (ABR 1, B+F) pass for the white text and a SUBTRACTIVE (ABR 2, B−F) pass offset ~(1,1) px for a dark drop shadow — with CLUT index 0 (`0x0000`) as the transparent background (the scene shows straight through); the tpage words are computed by the builder as `(fmt&3)<<7 | 0x26` / `0x46` (additive `0x26`, subtractive `0x46`), so the blend mode is runtime packet data invisible to a static decompile.** — `[S·D·R] 3/3`
  - S: built GPU primitives in the linked list @ `0x801C0BDC` (code byte `0x3E` = POLY_GT4 `0x3C` | abe `0x02`; tpage → ABR 1 and ABR 2, texpage XY `(384,0)`) + ETC.OUT builder `0x801BF3E0` tpage computation
  - D: scenario 8 capture — the clean 67-word VRAM diff at `02_reveal_start` proves B+F, not averaging (e.g. `(12,318): scene(8,7,3) → (29,28,24)`, delta +21,+21,+21 — averaging would need F>31, impossible); all output pixels have STP=1; covered pixels flip only the STP bit at the reveal front (`(33,320): 0064 → 8064`) (2026-07-10)
  - R: `godot-learning/assets/shaders/show_graphic.gdshader` (additive text pass) + `godot-learning/assets/shaders/show_graphic_shadow.gdshader` (subtractive drop-shadow, +1,+1) driven by the dual pass in `godot-learning/src/scenarios/ScenarioShowGraphic.gd`; `godot-learning/tests/ScenarioShowGraphicTest.gd`
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **The fade-in is a left→right WIPE, not a global alpha fade: a moving vertical front with a soft ~22–32 px additive leading edge sweeps the card at dynamic `front_x ≈ 0.88·frame + 31` (~240 frames ≈ 4.0 s @ 60 Hz), behind the front the card is at full uniform brightness; the static constants are grow step = BATTLE global `_DAT_80165f88` (static-init 1 px/frame), grow-limit 248 (template `+0x1C`), soft-edge kernel `min(0x20, remaining)` = 32 px, interior vertex grey `0x80` and moving-edge vertices `0`.** — `[S·D·R] 3/3`
  - S: ETC.OUT anim loop `0x801C017C` / builder `0x801BF3E0` constants (grey 128, edge 32, hold 0x50, shrink 0x80) + `_DAT_80165f88`
  - D: scenario 8 capture — front x at frames 2 / 62 / 242 ≈ 33 / 89 / 244 (full); linear fit 0.88 px/frame; per-column peak luminance flat at 248 once revealed (no residual gradient) (2026-07-10)
  - R: `godot-learning/src/scenarios/ScenarioShowGraphic.gd` (`_reveal_front`, `grow_step`/`grow_limit`/`edge`) + `godot-learning/assets/shaders/show_graphic_reveal.gdshaderinc` (soft L→R ramp `g = clamp((reveal_front − x)/EDGE, 0, 1)`) + `godot-learning/tests/ScenarioShowGraphicTest.gd` (`_test_reveal_wipes_left_to_right`)
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **The full ShowGraphic timeline (absolute frames, per-vblank clock): reveal ~240 frames → hold at uniform full brightness 0x50 (80) frames (dynamic ~85) → fade-out 0x80 (128) frames → task frees and releases the kind-`0x3D` liveness; ETC.OUT's anim loops (`0x801C017C`/`0x801C03AC`) tick once per BATTLE vsync `0x8014ca80`, so the frame counters are a per-vblank clock; the task slot state holds the per-phase frame counter at `+0x14` (resets between reveal and fade-out), the fade level at `+0x18`, the absolute lifetime counter at `+0x1C`, and a pointer into the overlay at `+0x10` (the `0x801C0BDC` primitive buffer).** — `[S·D·R] 3/3`
  - S: ETC.OUT anim loops `0x801C017C`/`0x801C03AC` (phase lengths ~grow / hold 0x50 / shrink 0x80) + task slot layout (kind @ slot `+0x4C`, table base `0x8016986C`, stride `0x400`)
  - D: scenario 8 capture — absolute counter `+0x1C` tracks reveal→hold→fade at ~243/~328/~455; `+0x14` resets at fade-out start; barriers at PC41/42 release at ~455 (2026-07-10)
  - R: `godot-learning/src/scenarios/ScenarioShowGraphic.gd` phase machine (reveal → hold `hold_frames = 0x50` → fade `fade_frames = 128` → `is_live()` false) + `godot-learning/tests/ScenarioShowGraphicTest.gd`
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **The fade-out is a GLOBAL UNIFORM level ramp, not a reverse wipe: all vertices' grey level ramps uniformly from `0x80` (128) down to 0 over ~127–128 frames, the per-pixel contribution is `base_additive × (level/127)` with peaks un-clamping as it falls, the task's `+0x18` mirrors `127 − phase_frame`, and at level 0 the task frees itself — releasing the `{E5}(0x3D)` / `{8F}` barriers.** — `[S·D·R] 3/3`
  - S: ETC.OUT anim loop (fade phase 0x80, grey 128→0) + `+0x18` field
  - D: scenario 8 capture — uniform −1 step/frame across the whole card width (x[11..244]); at level 43 (phase 84) the rule-line mean peak fell 31.0 → 15.6 (ratio 0.503, matching ~50 %); Δphase 80 == Δlevel 80 (2026-07-10)
  - R: `godot-learning/src/scenarios/ScenarioShowGraphic.gd` (`_grey` uniform ramp, fade phase) + `godot-learning/tests/ScenarioShowGraphicTest.gd` (`_test_holds_then_fades_grey_to_zero`)
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`
- **The card's additive gouraud quads are drawn with the PSX GPU's 4×4 ordered dither enabled (a global draw-env setting, not a per-primitive bit) and quantized into RGB555 with the standard matrix (`-4 0 -3 1 / 2 -2 3 -1 / -3 1 -4 0 / 3 -1 2 -2`, added to the 8-bit value then `>>3` to 5-bit, keyed on screen `(x%4, y%4)`); the dither is visible only at mid fade levels (sub-5-bit contributions) — invisible at full saturation and at zero.** — `[S·D·R] 3/3`
  - S: ETC.OUT draw-env (dither bit in the texpage/GP0 word) + absence of a per-packet dither bit in the built primitives @ `0x801C0BDC`
  - D: scenario 8 capture — clear period-4 column pattern at ~50 % fade: mean R by `x%4` = [16.5, 15.4, 16.1, 14.2], adjacent columns ±1–2 in a 4-px cycle (2026-07-10)
  - R: `godot-learning/assets/shaders/psx_dither.gdshaderinc` (`PSX_DITHER_MATRIX`) applied at the 5-bit quantization step via `godot-learning/assets/shaders/show_graphic_reveal.gdshaderinc` (`ordered_dither`) in both card passes
  - src: `research/working_documents/scenario_1_captures/show_graphic_op7d_decode.md`

## Notes

(empty — user territory)

## Related

- [[DarkScreen Opcode]]
- [[Event Opcode Catalog]]
- [[PSX GPU Primitives]]
- [[PSX Texture Page Register]]
- [[Scenario Wait Semantics]]
- [[Reveal Opcode]]
