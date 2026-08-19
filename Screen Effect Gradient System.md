# Screen Effect Gradient System

During spell effects FFT dims and colors the screen by drawing the background as a Gouraud-shaded fullscreen quad (GP0 0x2C/0x38) whose four vertex colors are driven by the screen track — Track 3 of the timeline color-track system. The track holds 33 keyframes (int16 time, interleaved start/end RGB, control byte); each keyframe either fades linearly from start to end RGB, or tints via one of 11 blend modes in the applier at 0x80090258, then ramps linearly to the computed target. All gradient state lives in fixed RAM blocks in 16.16 fixed point (primary 0x800a1b18, secondary 0x800a1b7c, current 0x800f5b40), and screen-background interpolation is pure linear per-frame delta accumulation — the dither-curve table at 0x800956e4 belongs to the palette color tracks (unit tinting), not to the screen background. A 2026-08 E005/Raise pass closed the open "V-shape" thread: the dim→recover→re-dim V on the background gradient is authored (framebuffer-confirmed on the PSX) and Godot reproduces it continuously with no seam pop; the combat arming path is `FUN_80090258` on this same gradient block (the subtractive `screen_tint_apply` quad is scenario-only), and `ScreenSubsystem` has since migrated onto the unified `ColorStack`/`ColorRecipe` model, folding TOP+BOTTOM through one continuous op stream.

## Points

- **The spell-effect background is rendered as a Gouraud-shaded fullscreen quad — GP0 0x2C (shaded textured quad) or GP0 0x38 (untextured shaded quad) with 4 vertices carrying independent RGB colors that the PSX rasterizer interpolates; GPU color-register configuration happens at 0x800e8190.** — `[S] 1/3`
  - S: 0x800e8190 (GPU setup, color register configuration), per `research/wiki_articles/screen_effect_gradient_system.md`
  - src: `research/wiki_articles/screen_effect_gradient_system.md`
- **Gradient state is held in fixed RAM addresses in 16.16 fixed point: primary gradient (2 RGB colors + deltas) at 0x800a1b18–0x800a1b38, secondary gradient at 0x800a1b7c–0x800a1b94, current per-frame RGB at 0x800f5b40–0x800f5b48, and byte override registers at 0x800a1b48–0x800a1b4a.** — `[S] 1/3`
  - S: RAM addresses 0x800a1b18/0x800a1b7c/0x800f5b40/0x800a1b48, per `research/wiki_articles/screen_effect_gradient_system.md`
  - src: `research/wiki_articles/screen_effect_gradient_system.md`
- **Gradient/tint entry points: init_screen_fade at 0x80090048 sets up fade colors + duration, init_screen_fade_alt at 0x80090c48 is the secondary fade path, and the per-frame color update / tint applier is at 0x80090258.** — `[S] 1/3`
  - S: 0x80090048, 0x80090c48, 0x80090258, per `research/wiki_articles/screen_effect_gradient_system.md`
  - src: `research/wiki_articles/screen_effect_gradient_system.md`
- **The screen tint applier at 0x80090258 implements an 11-mode blend switch (mode = ctrl & 0x7F): 0 direct add (current+param), 1 current/2+param, 2 luma (2R+3G+B)/6 + param, 3 luma/12 + param, 4 byte-register+param, 5 byte-register/2+param, 6 byte luma/6 + param, 7 byte luma/12 + param, 8 byte-register direct, 9 pulse (mode 4 + pulse arming), 10 stop; results are clamped to the 0–0xFF color range and, when duration != 0, ramped with delta = (target − current)/(duration × 8).** — `[S·D·R] 3/3`
  - S: 11-mode switch at 0x80090258, per `research/wiki_articles/screen_effect_gradient_system.md`; re-verified against `battle_decompilation.c` (modes 4–8 read byte baseline `0x800a1b48–4a`, mode 5 = `byte·0x8000 + param·0x10000`, results clamped to `[0,0xff]·0x10000`, DDA = `(target−cur)/(Time<<3)`)
  - D: Raise E005 live gradient readout (2026-07-12): TOP baseline `(32,64,124)`, mode-5 dim floor `(16,32,62)` = baseline/2 + param 0 — byte-exact; BOTTOM `(108,120,112)`→`(54,60,56)`
  - R: `godot-learning/src/effects/ScreenSubsystem.gd` (8-bit `ColorRecipe`/`ColorStack` mirror, `quantize=false`), validated by `godot-learning/tests/ScreenSubsystemTest.gd` `_test_fold_top_mode5_over_baseline` / `_test_fold_bottom_mode5_over_baseline` (branch `import-godot-game`)
  - src: `research/wiki_articles/screen_effect_gradient_system.md`
  - src: `research/working_documents/MAP_COLOR_SUBSYSTEM_PARITY_RAISE_E005.md`
- **Mode 9 arms pulsing: it sets pulse flag DAT_800a1b14 = 1 and stores the pulse color at DAT_800a1b15–0x800a1b17; the per-frame update then alternates between mode-8 and mode-9 targets every 32 frames (oscillating glow) until mode 10 clears the flag (DAT_800a1b14 = 0) and disables the animation.** — `[S] 1/3`
  - S: DAT_800a1b14–0x800a1b17, per `research/wiki_articles/screen_effect_gradient_system.md`
  - src: `research/wiki_articles/screen_effect_gradient_system.md`
- **The Screen Effect is Track 3 of the color-track system, with its track at fixed offsets from timeline_section_ptr: 0x1036 (300 bytes) Phase 1 and 0x13BA (300 bytes) Phase 2 for 3-phase effects, 0x057E (298 bytes) for 1-phase/for-each effects.** — `[S] 1/3`
  - S: offsets 0x1036/0x13BA/0x057E from timeline_section_ptr (0x801BC0C8), per `research/wiki_articles/screen_effect_gradient_system.md`
  - src: `research/wiki_articles/screen_effect_gradient_system.md`
- **The 298-byte screen track holds 33 keyframes: int16 time values at 0x000 (66 bytes), interleaved start-RGB triplets at 0x042 (99 bytes), interleaved end-RGB triplets at 0x0A5 (99 bytes), one control byte per keyframe at 0x108 (33 bytes), plus 1 padding byte.** — `[S] 1/3`
  - S: track layout consumed by advance_screen_color_track at 0x801A45C8, per `research/wiki_articles/screen_effect_gradient_system.md`
  - src: `research/wiki_articles/screen_effect_gradient_system.md`
- **Keyframe duration is 1 frame when the time value is 0, otherwise time_value × 8 (<< 3) — valid durations are 8, 16, 24, 32, … frames.** — `[S] 1/3`
  - S: duration calculation in advance_screen_color_track at 0x801A45C8, per `research/wiki_articles/screen_effect_gradient_system.md`
  - src: `research/wiki_articles/screen_effect_gradient_system.md`
- **The keyframe control byte's bit 7 selects behaviour: ctrl & 0x80 == 0 is fade mode (linear interpolation start_rgb → end_rgb), ctrl & 0x80 == 1 is tint mode (mode = ctrl & 0x7F picks the blend formula, then linear interpolation to the computed target).** — `[S·R] 2/3`
  - S: mode selection at 0x80090258, per `research/wiki_articles/screen_effect_gradient_system.md`
  - R: `godot-learning/src/effects/ScreenSubsystem.gd` (bit-7 FADE/TINT gate, blend modes 0–10 match) + `godot-learning/tests/ScreenSubsystemTest.gd`
  - src: `research/wiki_articles/screen_effect_gradient_system.md`
  - src: `research/working_documents/E_BIN_FIELD_EDITABILITY_INVENTORY.md`
- **Screen-background interpolation is linear per-frame delta accumulation — current += (target − start)/(duration × 8) in 16.16 fixed point with 8-bit colors extracted via (current >> 16) & 0xFF — and does NOT use the dither-curve table at 0x800956e4, which belongs to the palette color tracks (unit tinting during spells).** — `[S] 1/3`
  - S: 0x800956e4 (dither table, palette tracks only) and gradient block 0x800a1b18, per `research/wiki_articles/screen_effect_gradient_system.md`
  - src: `research/wiki_articles/screen_effect_gradient_system.md`
- **During a combat ability effect the background gradient is armed by `FUN_80090258 @0x80090258` — the combat sibling of the scenario `{2E}` setter `screen_gradient_color_setter @0x80090048`, both arming the same gradient block (TOP `0x800a1b18` / BOTTOM `0x800a1b30`, 16.16 DDA at +0xc, active `DAT_800a1b10`, ramp over `Time×8` frames, byte baselines `0x800a1b48–4a`) — NOT by the full-screen subtractive quad `screen_tint_apply @0x80090840`, which E005 never calls: the subtractive quad is the scenario/`{1A}`/`{3E}` mechanism, the gradient is a first-class combat channel.** — `[S·D·R] 3/3`
  - S: `FUN_80090258 @0x80090258` (callers `0x800912A4`/`0x8009349C`; TOP/BOTTOM stride 0x18; byte baseline `0x800a1b48/49/4a`; `Time<<3` DDA; active `DAT_800a1b10`), `screen_gradient_set_backup @0x800901F8` (swr `0x800a1b48/4c`), `screen_tint_apply @0x80090840` (callers `0x800912A4`/`0x800934D0`; byte baseline `0x800a1b70`) (`battle_decompilation.c`)
  - D: Raise E005 exec-BP counts (2026-07-12, effect-editor session `raise`): `0x8008F710` 75×, `0x80090840` 0×, `0x80090048` 0× while the gradient block moved TOP `(32,64,124)`→`(16,32,62)`→recovering and BOTTOM `(108,120,112)`→`(54,60,56)` — armed by `FUN_80090258`, re-armed by the stepper
  - R: `godot-learning/src/effects/ScreenSubsystem.gd` tints the `ScreenBackground` gradient quad (the sky behind the map, `render_priority −100`), not a post-everything subtractive quad (branch `import-godot-game`)
  - src: `research/working_documents/MAP_COLOR_SUBSYSTEM_PARITY_RAISE_E005.md`
- **The spell-effect background gradient traces a smooth dim→recover→re-dim V on the PSX — it is authored, not a rendering bug (E005: baseline B≈124 → dim floor 61 at frame 9 → recover peak 133 at frame 15 → slow re-dim) — and Godot reproduces the shape continuously with no seam pop because the for_each transition starts from the persistent `background_color` rather than a fresh identity fold.** — `[D·R] 2/3`
  - D: Raise E005 framebuffer burst (2026-07-12, `takeScreenShot` while running, CLUT b3i8 `954E`→`8023`→`90E8` sampled alongside to confirm playback): B-channel of a pure-background patch (`x0–24, y0–24`, sky with no map geometry); beat frames `reference-assets/raise_e005_screen_dim.png` (frame 9) and `…_screen_recover.png` (frame 15)
  - R: `godot-learning/src/effects/ScreenSubsystem.gd` — persistent `background_color` + `transition_from` (pre-migration guard `ScreenSubsystemTest._test_screen_no_pop_across_phase_seam`, 19/0); post-migration the continuous `build_stream` fold is guarded by `ScreenSubsystemTest._test_folded_top_no_pop_across_seam` (branch `import-godot-game`)
  - src: `research/working_documents/MAP_COLOR_SUBSYSTEM_PARITY_RAISE_E005.md`
- **`ScreenSubsystem` now folds the map's TOP+BOTTOM gradient baselines through one continuous `ColorStack`/`ColorRecipe` op stream (`build_stream`, 8-bit, `quantize=false`; `ColorRecipe` is bit-depth-agnostic via `param_max` 31/255) and `ScreenEffectOverlay.compose_corners` composites top→tl/tr, bottom→bl/br — the imperative stepper is deleted; the migration is a unification, not a rewrite, because the PSX screen and palette channels are literally the same 11-mode engine (`FUN_80090258` ≡ the `color_tint_blend_apply` mode switch) applied to different targets.** — `[S·D·R] 3/3`
  - S: `FUN_80090258 @0x80090258` ≡ `color_tint_blend_apply @0x8008F710` mode switch (`battle_decompilation.c`)
  - D: Raise E005 live readout (2026-07-12): TOP baseline `(32,64,124)` and mode-5 dim floor `(16,32,62)` match Godot's `default_bg` and phase-1 mode-5 floor byte-exact
  - R: `godot-learning/src/effects/ScreenSubsystem.gd` `build_stream` + `godot-learning/src/effects/ScreenEffectOverlay.gd` `compose_corners` (commit `c0f965144`, branch `import-godot-game`), validated by `ScreenSubsystemTest.gd` (9 tests) alongside `ColorRecipeTest` (79) and `ColorStackTest` (144/0)
  - src: `research/working_documents/MAP_COLOR_SUBSYSTEM_PARITY_RAISE_E005.md`

## Notes

(empty — user territory)

## Related

- [[Color Track Interpolation]]
- [[Map Tint]]
- [[Effect Execution Model]]
- [[Effect File Format]]
- [[PSX GPU Primitives]]
