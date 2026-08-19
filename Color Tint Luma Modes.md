# Color Tint Luma Modes

FFT's `{32}` Color Unit, `{33}` Color Field, and `{1A}` Map Darkness all funnel into one BATTLE.BIN blend engine — `color_tint_blend_apply @ 0x8008F710` — which composes each palette entry from signed 5-bit per-channel deltas and a mode-selected source (current, halved current, base, halved base, or a base/current luma scalar). The luma modes 2/3/6/7 reduce each entry to a single weighted-luminance scalar (2·R+3·G+B, divided by 6 or 12) and broadcast it to all three channels — a channel mix no per-channel affine map can express — producing the desaturated "brown paper" sepia of scenario 8's flashback. The mechanism is proven statically from the decompilation, dynamically verified against a live scenario-8 capture (byte-exact 227/227 base→out seal against the resident base strip), and reimplemented in `godot-learning` (`ScenarioColorTint.luma_out5` plus unit/field shader luma branches, per-view unit-over-field precedence, and a mode-8 cross-fade to base). The same view engine carries Orbonne scenario-4's lightning map blue-hue (view-0 blue CLUT content staged to VRAM row Y=494), and `{66}` Commit Palette bakes the settled view strip into the base palette, making a tint permanent.

## Points

- **`{32}` Color Unit, `{33}` Color Field, and `{1A}` Map Darkness all funnel into a single blend backend — `color_tint_blend_apply(mode, Time, view, unit_pal, broadcast, dR, dG, dB)` at `0x8008F710` (size 0x938) — whose RGB operands are signed 5-bit deltas.** — `[S] 1/3`
  - S: `color_tint_blend_apply @ 0x8008F710` (`battle_decompilation.c`)
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **The engine's mode switch has 11 cases — 0=`cur+d`, 1=`(cur>>1)+d`, 2=`(2·curR+3·curG+curB)/6+d`, 3=`(2·curR+3·curG+curB)/12+d`, 4/9=`pal+d`, 5=`(pal>>1)+d`, 6=`(2·palR+3·palG+palB)/6+d`, 7=`(2·palR+3·palG+palB)/12+d`, 8=`pal` (absolute, no delta), 10=reset — where `cur` is the tween-current channel and `pal` the base-palette channel.** — `[S·D·R] 3/3`
  - S: mode switch inside `color_tint_blend_apply @ 0x8008F710` (`battle_decompilation.c`)
  - D: Raise E005 CLUT capture (2026-07-12, effect-editor session `raise`): mode-5 dim plateau at b3i8 (bank 3, CLUT idx 8) reads `(3,1,0)` = `(14,10,5)>>1 + (−4,−4,−4)` byte-exact, and the post-effect mode-8 snap returns exactly the `(14,10,5)` base
  - R: `godot-learning/src/core/ColorRecipe.gd` `from_mode` (all 11 modes, `param_max` bit-depth-agnostic), validated by `godot-learning/tests/ColorRecipeTest.gd` (22 tests) + the `PaletteSubsystemTest.gd` fold tests
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
  - src: `research/working_documents/MAP_COLOR_SUBSYSTEM_PARITY_RAISE_E005.md`
- **The luma modes 2/3/6/7 reduce each entry to a single scalar `luma = ⌊(2·R+3·G+B)/div⌋` (div 6 or 12, read from base or current) and broadcast it to all three channels (`out_R = luma+dR`, `out_G = luma+dG`, `out_B = luma+dB`), so R/G/B become equal except for the constant delta — a channel mix that a per-channel `scale·base+bias` affine map cannot reproduce.** — `[S·D·R] 3/3`
  - S: luma broadcast `LAB_8008f9b0` inside `color_tint_blend_apply @ 0x8008F710` (`battle_decompilation.c`)
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10): broadcast-luma fingerprint `(R−G, G−B)=(1,2)` on 157 view-0 entries, recovered luma identical across channels 23/23 sampled; measured mode-7 vectors (delta (4,3,1): luma/12=3→(7,6,4), =9→(13,12,10)) match live CLUT bytes
  - R: `godot-learning/src/scenarios/ScenarioColorTint.gd` `luma_out5(base5, div, delta5)` (integer core mirrored in `godot-learning/assets/shaders/unit.gdshader` and `godot-learning/assets/shaders/indexed_color.gdshader`), validated by `godot-learning/tests/ScenarioLumaTintTest.gd`
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **Each palette view (stride 0x982) holds a tween-current array `DAT_80099676` (7 B/entry: 5-bit cur RGB, STP/mask byte, three DDA deltas), a base palette `DAT_80099d76` (=+0x700, 2 B/entry BGR555), and DDA header `DAT_800995f6`/`…fa`; each composited entry commits as a BGR555 word `R + G·0x20 + B·0x400 + STP·0x8000` to `dynamic_clut_view_strip[view·0x100 + i]`, DMA'd to the VRAM CLUT.** — `[S·D] 2/3`
  - S: `DAT_80099676`, `DAT_80099d76`, `DAT_800995f6` (`battle_decompilation.c`)
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10): palette views walked in `DAT_80099676` at stride 0x982; VRAM row 504 shows the committed strip — brown ramp interleaved with grayscale dialog CLUT
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **Each output channel clamps to `[0,31]`, and if all three channels land at 0, B is forced to 1 to keep the entry non-zero/opaque.** — `[S·D·R] 3/3`
  - S: clamp path inside `color_tint_blend_apply @ 0x8008F710` (`battle_decompilation.c`)
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10): view-3 unit entry at luma/12=0 with delta (4,2,−1) reads (4,2,0) — B clamps −1→0 in the live CLUT
  - R: `godot-learning/src/scenarios/ScenarioColorTint.gd` `luma_out5` clamp, validated by `godot-learning/tests/ScenarioLumaTintTest.gd`
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **`Time==0` snaps (writes cur and the committed strip directly) while `Time!=0` arms a per-entry DDA (`[4..6] = (target−cur)+0x1F`) walked by the per-frame ramp driver (`cur_ch += curve_table[step + delta_idx·width]`, then re-commit); `Time<4` uses the fast 8-wide table (one step/frame, 8 frames) and `Time≥4` the slow 32-wide table (one step every `Time>>2` frames, `32·(Time>>2)` frames), and on completion the entry re-latches via a mode-8/9 re-apply (`DAT_800995fa` 1↔2).** — `[S·R] 2/3`
  - S: ramp loop ~`0x80091c38`, `color_ramp_curve_table @ 0x800956e4`, throttle `DAT_800995f8`/`DAT_800995fa` (`battle_decompilation.c`)
  - R: `godot-learning/src/core/ColorRecipe.gd` `ramp_frames_for_time` (Time<4 → 8 frames, one step/frame; Time≥4 → `32·(time>>2)` frames), validated by `PaletteSubsystemTest._test_time_value_drives_the_ramp`
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
  - src: `research/working_documents/MAP_COLOR_SUBSYSTEM_PARITY_RAISE_E005.md`
- **The slow 32-wide DDA curve table at `0x800956e4` is LINEAR — each row (`delta_idx = (target−cur)+0x1F`, valid Δ∈[−31,+31]) is an even Bresenham spread of `(target−cur)` with no easing — so a linear lerp is byte-faithful and no curve data needs parsing.** — `[S] 1/3`
  - S: `color_ramp_curve_table @ 0x800956e4` (`battle_decompilation.c`)
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **The `broadcast` parameter selects the write target: `broadcast==1` writes the whole 0x100-entry view (used by `{33}` Color Field and `{1A}` Map Darkness) while `broadcast!=1` writes a single 16-entry unit palette at `unit_pal·0x70`, whose result lands in the sprite object's own field via `FUN_8007a6e4` (the door-fade `{32}` path).** — `[S] 1/3`
  - S: `param_5` branch + `FUN_8007a6e4` inside `color_tint_blend_apply @ 0x8008F710` (`battle_decompilation.c`)
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **`{32}` Color Unit is a per-unit fanout (`broadcast=0`) — each targeted unit gets two `color_tint_blend_apply` calls, view 3 (additive delta) and view 4 (same delta sign-flipped when a terrain-height/STP test fails, `tile_height_lookup` vs the unit's `+0x42`) — while `{33}` Color Field writes the whole 0x100-entry view-0 page (`broadcast=1`), so last-write-per-view wins: in scn8 the sprites keep the unit op's (4,2,−1) (PC29) and the map keeps the field op's (4,3,1) (PC28).** — `[S·D] 2/3`
  - S: `color_unit_fanout @ 0x800933C4` → `color_unit_set_per_unit @ 0x800931C4` (`battle_decompilation.c`)
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10): view 0 ends on delta (4,3,1) while view 3 ends on (4,2,−1)
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **Scenario 8's "brown paper" flashback is mode-7 luma tint on the full palette — `{33}` PC28 delta (4,3,1) Time 0 and `{32}` PC29 delta (4,2,−1) Time 0 (RGB operands signed 5-bit: 255→−1), re-applied at PC38/39 as (2,1,−1)/(2,0,−3) Time-4 ramps — pinning every entry to `luma/12 + delta` (luma/12 caps at 15), a desaturated half-brightness sepia.** — `[S·D] 2/3`
  - S: mode-7 branch of `color_tint_blend_apply @ 0x8008F710` (`battle_decompilation.c`); chunk operands in `godot-learning/assets/scenarios/chunks/scenario_008_chunk.json`
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10, PCSX-Redux, `scenario_id @ 0x8016A014 == 8`): view-0 fingerprint (1,2) on 157 entries 23/23 luma-consistent, view-3 fingerprint (2,3) on 81 entries 22/22, views 1&2 pure grayscale (0,0), VRAM row 504 committed strip brown ramp
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **`out_ch = clamp(⌊(2·baseR+3·baseG+baseB)/12⌋ + delta_ch, 0, 31)` reproduces every non-transparent entry of the resident field palette — 227/227, 100% — against the base strip `DAT_80099d76` (view 0); the base is resident and preserved, and mode 7 reads it fresh each apply (idempotent), so a post-tint snapshot still reproduces the transform.** — `[S·D] 2/3`
  - S: base strip `DAT_80099d76` (`battle_decompilation.c`)
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10): base→out seal closed 227/227 (snapshot sat past PC38, so the matched delta was (2,1,−1))
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **Across all 500 scenario chunks the luma modes are: mode 7 (base luma /12) 16 uses (scn 8, 14, 203, 285 — sepia flashbacks; deltas (4,3,1) (4,2,−1) (2,0,−2) (2,−1,−4) (0,−2,−6) (2,1,−1)), mode 3 (current luma /12) 6 uses (scn54 δ(0,0,0), scn396 δ(−1,−1,−1)), mode 6 (base luma /6) 1 use (scn112 δ(−1,0,2)), mode 2 0 uses — so `(div∈{6,12}, source∈{base,current}, delta)` covers 100% of shipped luma usage, with negative deltas common enough that clamp-to-0 matters.** — `[S] 1/3`
  - S: static scan of every `{32}/{33}/{1A}` op in `godot-learning/assets/scenarios/chunks/*` (parsed from TEST.EVT)
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **In Godot, a unit's own `{32}` luma wins over the `{33}` field luma (per-view last-write order — `{32}` fires after `{33}`), the field luma still applies to units with no own `{32}`, and affine (Orbonne prayer) composition is untouched.** — `[S·D·R] 3/3`
  - S: per-view last-write routing — `color_unit_fanout @ 0x800933C4` (view 3) after the view-0 field write, PC29 after PC28 (`battle_decompilation.c`, `scenario_008_chunk.json`)
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10): tinted units show the unit op's (4,2,−1), not the field's (4,3,1)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_effective_unit_tint`, validated by `godot-learning/tests/ScenarioLumaTintTest.gd` `_test_unit_luma_wins_over_field_luma`
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **In Godot, a mode-8 (restore-to-base) apply from a live luma state with `time>0` is a cross-fade: `ScenarioColorTint` keeps the old luma spec alive and ramps a `luma_mix` scalar 1→0 over the ramp frame count (`ramp_frames_for_time(8) = 32·(8>>2) = 64` frames for scn8 PC47/48's Time=8), collapsing to `luma_div=0` only on the final frame; both shaders compute `mix(base_affine, luma_out, luma_mix)` (`unit_luma_mix` in unit.gdshader, `field_luma_mix` in indexed_color.gdshader, pushed via `MapComposer.set_field_luma`).** — `[S·R] 2/3`
  - S: scn8 PC47 `{33}` Color=8 Time=8 + PC48 `{32}` Color=8 Time=8 (`godot-learning/assets/scenarios/chunks/scenario_008_chunk.json`); ramp timing `ramp_frames_for_time` (`godot-learning/src/scenarios/ScenarioColorTint.gd`)
  - R: `godot-learning/src/scenarios/ScenarioColorTint.gd` `luma_mix`, validated by `godot-learning/tests/ScenarioLumaTintTest.gd` `_test_luma_fade_out_cross_fades_to_base`
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **The "base" that modes 6/7 read is the palette as currently loaded — after weather/map-state selection and any earlier `{33}`/`{66}` commits — so a byte-faithful reimplementation must apply those prior palette ops in VM order first, and every implementation value (luma formula, mode table, chunk operands, base palettes, ramp cadence) is ISO-derived (decompiled BATTLE.BIN + parsed disc data) with RAM used only to verify.** — `[S·D·R] 3/3`
  - S: base read of `color_tint_blend_apply @ 0x8008F710` from `DAT_80099d76` (`battle_decompilation.c`)
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10): base strip resident and preserved — the 227/227 seal holds against a post-tint snapshot
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` applies palette ops in VM order; `godot-learning/src/map/MapComposer.gd` bakes the field palette from parsed ISO sources
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`

- **The Orbonne scenario-4 "map blue hue" during the lightning flash is view-0's blue CLUT content (staging `0x800e4ea4` → VRAM row Y=494) ramping up and decaying frame-for-frame in lockstep with the `{2E}` gradient flash, armed by the transient `{33}` Color Fields paired 1:1 with each `{2E}` (handler `FUN_80093170` sign-extends R/G/B before calling `color_tint_blend_apply @0x8008f710`; flash operands as signed bytes are (−8,0,+4) / (−1,0,+2) = red-down/blue-up; the PC31/39 mode-8 `{33}`s restore to the committed base).** — `[S·D·R] 3/3`
  - S: `FUN_80093170`, `color_tint_blend_apply @0x8008f710`, staging `0x800e4ea4` (`battle_disassembly.txt`); `{33}`/`{2E}` pairing insts 27→28, 30→31, 35→36, 38→39 in `scenario_004_chunk.json`
  - D: correlation trace, Orbonne reference savestate (2026-07-05): view-0 blue 5→9→7→6→5 against gradTopR 48→184→48; chapel region mean R38.9/G54.0/B63.2 → R31.8/G67.8/B82.8 (B/R 1.62→2.60 — red drops, blue rises)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_color_field` + `ScenarioColorTint` (sign-extends R/G/B via `_sb`, DDA ramp) — the `{33}` map hue was already faithful; headful-verified scenario 4 (2026-07-05)
  - src: `research/working_documents/LIGHTNING_FLASH_OPCODE_2E_BACKGROUND.md`
- **`{66}` Commit Palette is a strip→base commit: `FUN_8008f63c` copies `dynamic_clut_view_strip` into the base copy `DAT_80099d76`, making a transient view tint permanent — in Orbonne scenario 4 the PC12 `{33}` mode-4 (−3,−1,+3) map blue is made permanent by the PC14 `0x66`.** — `[S·D·R] 3/3`
  - S: the `0x66` case `@0x80145194`, `FUN_8008f63c`, base copy `DAT_80099d76` (`battle_disassembly.txt`)
  - D: write-watchpoint on the base `0x80099d78` caught writer `pc=0x8008f68c` from the `0x66` case (2026-07-09)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_commit_palette` (bound `COMMIT_PALETTE = 0x66`; bakes the settled field tint into the committed base via `MapComposer.commit_field_tint`), validated by `godot-learning/tests/ScenarioCommitPaletteTest.gd`
  - src: `research/working_documents/LIGHTNING_FLASH_OPCODE_2E_BACKGROUND.md`
- **`{33}` Color Field operand layout + semantics (cross-source, HIGH confidence): first byte is a `xPR` preset-color selector — the one documented structural difference from `{1A}`'s `xBM` blend-mode selector — followed by signed `Red/Green/Blue` + unsigned `Time`, the same tail as `{1A}` (and `{31}` ColorBGBeta shares `{33}`'s exact layout); no unit-target fields, so the effect is map-wide (contrast `{32} Color Unit`, which adds `Units,Multi`); scale anchors `(-128,-128,-128)`=black, `(+127,+127,+127)`=white, `(0,0,0)`=restore original map colour. `{3E}` Color Screen carries two RGB triples (initial+target ramped), distinguishing it from single-delta `{1A}`/`{33}`, and `{2E}` uses unsigned RGB (scn4 operands >127) while `{33}` sign-extends — proof the two are distinct channels.** — `[S·R] 2/3`
  - S: FFTPatcher `EventCommands.xml` (`hex=33` "Color Field": Color(1B) + signed R/G/B + unsigned Time(1B), no unit target) + ffhacktics `ColorField(xPR, +RED, +GRN, +BLU, TIM)` scale anchors + GaneshaDx corroboration (deep-research pass, doc §4b)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `color_field` → `ScenarioColorTint.apply(mode, r, g, b, time)` (whole-field broadcast, signed RGB via `ScenarioDecode.color_field`), validated by `godot-learning/tests/ScenarioColorFieldTest.gd`
  - src: `research/working_documents/MAP_FLASH_SCENARIO4_PERSISTENT_DARK.md`
- **The map samples the *tinted* CLUT, not the raw base palette: `dynamic_clut_view_strip` (`0x800e4ea4`) is a RAM CLUT staging strip tinted per-frame by the colour engine and DMA'd to VRAM (0,494) by `flush_clut_view_strip` (`0x80092f98`); the map colour texture is 16 palettes × 16 colours, each a 16-bit little-endian BGR555 word (`0000h`=transparent) — the palette content a faithful `{33}` CLUT-content fade must ramp.** — `[S·D] 2/3`
  - S: `dynamic_clut_view_strip @ 0x800e4ea4`, `flush_clut_view_strip @ 0x80092f98`, warm base `0x800f6ab0` (doc §4b Q3; full RE in `research/working_documents/MAP_HUE_WEATHER_STATE_CLUT_BAKE.md`)
  - D: warm base `0x800f6ab0` vs cool tinted strip `0x800e4ea4` == VRAM Y494, Δ(−3,−1,+3) 5-bit (2026-07-09)
  - R: none — per-frame tinted view strip not present in godot-learning (probed `godot-learning/src/` + `godot-learning/tests/`; `ScenarioColorTint` models the `{32}`/`{33}` tints as a uniform shader-side scale/bias over the base palette instead)
  - src: `research/working_documents/MAP_FLASH_SCENARIO4_PERSISTENT_DARK.md`

## Notes

(empty — user territory)

## Related

- [[Event Unit Selector]]
- [[Reset Palette Opcode]]
- [[Color Screen Opcode]]
- [[Color Track Interpolation]]
- [[Map Tint]]
- [[Background Opcode]]
- [[Map Darkness Opcode]]
