# Map Tint

The "Affected Units" palette track (Track 0) is the map's only colour channel: a live Raise/E005 capture shows only field CLUT banks 0/3/4 of the dynamic strip moving, the stored base palette is never committed, and `caster`/`target` route to unit palettes, never the field strip. Its map path is not a terrain vertex-colour applier: a live-Ghidra xref pass (2026-07-11) found no ×8 vertex-colour scaling and no third tint sink — map tint routes through the two colour appliers (palette CLUT contents or the screen tint quad), never terrain vertex RGB, and the CLUT rewrite lands before per-face Gouraud lighting so terrain shadows survive a dim. Godot delivers the channel as one continuous `ColorStack` fold across all started phases (`PaletteSubsystem.build_stream` → `MapTintOverlay`), with no phase-boundary pop. Script opcodes 0x75/0x76 can disable/re-enable the map tint via bit 2 of effect_render_flags.

## Points

- **Track 0 (Affected Units) tints both units and map/terrain from the same keyframe data; the map/terrain RGB is scaled ×8 (R<<3, G<<3, B<<3) relative to the unit values.** — `[S] 1/3`
  - S: `advance_affected_units_palette_track` at 0x801A45C8, per `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - src: `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - ⚠ SUPERSEDED (2026-08-14) by: the affected-units palette track advances through advance_affected_units_palette_track at 0x801A41A0; 0x801A45C8 is advance_screen_color_track
  - ⚠ SUPERSEDED (2026-08-16) by: no ×8 map vertex-colour scaling exists — a live-Ghidra xref pass (2026-07-11) found exactly two tint sinks (palette CLUT `color_tint_blend_apply @0x8008F710` and screen `screen_tint_apply @0x80090840`); the map is tinted via CLUT contents or the screen quad, never terrain vertex RGB
- **The scaled map tint RGB is written to register DAT_800f5b58 via FUN_80090dec() → FUN_800e8190(99, &rgb) and applied to terrain polygon vertex colors during map rendering in FUN_8012d2b4.** — `[S] 1/3`
  - S: symbols FUN_80090dec, FUN_800e8190, FUN_8012d2b4 and register DAT_800f5b58, per `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - src: `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - ⚠ SUPERSEDED (2026-08-16) by: no map vertex-colour applier exists — map tint routes through the two colour appliers (palette CLUT staging `dynamic_clut_view_strip @0x800E4EA4` or the screen tint quad), per the live-Ghidra xref pass (2026-07-11)
- **Map tint applies only while (effect_render_flags & 2) == 0; script opcode 0x75 disables map tint (flag |= 2) and opcode 0x76 re-enables it (flag &= ~2).** — `[S] 1/3`
  - S: opcodes 0x75/0x76 and effect_render_flags bit 2, per `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - src: `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
- **The map tint is a single palette channel — `affected_units`: during Raise/E005 only field CLUT banks 0, 3, 4 of `dynamic_clut_view_strip @0x800e4ea4` change (banks 1, 2, 5–13 constant), the stored base `DAT_80099d76` is never committed (bank-0 idx-1 word `0x80099d78` constant through the whole playback), and `caster`/`target` tint unit palettes via `color_unit_set_per_unit @0x800931C4`, never the field strip.** — `[S·D·R] 3/3`
  - S: `dynamic_clut_view_strip @0x800e4ea4` (bank stride 0x200), `DAT_80099d78`, `color_unit_set_per_unit @0x800931C4` (`battle_decompilation.c`)
  - D: Raise E005 CLUT capture (2026-07-12, effect-editor session `raise` fired via `ee_test`; raw `/tmp/raise_clut_clean.csv`): per-bank distinct-value count over the whole capture — banks 0/3/4 vary, all others constant; base word constant
  - R: `godot-learning/src/effects/PaletteSubsystem.gd` routes `AFFECTED_UNITS`→`MapTintOverlay.update_stack`, `CASTER`/`TARGET`→`UnitTintOverlay.update_stack`, guarded by the end-to-end `advance()` tests in `godot-learning/tests/PaletteSubsystemTest.gd` (branch `import-godot-game`)
  - src: `research/working_documents/MAP_COLOR_SUBSYSTEM_PARITY_RAISE_E005.md`
- **The map colour pipeline order is CLUT rewrite → per-face Gouraud lighting → framebuffer, so a dimmed/tinted CLUT is still modulated by per-vertex lighting — terrain shadows and lit relief survive the dim (the darkened field in the capture keeps its shading, it does not flatten to one tint); Godot mirrors the order with the palette fold pre-lighting.** — `[D·R] 2/3`
  - D: Raise E005 framebuffer stills (2026-07-12): `reference-assets/raise_e005_map_darkest.png` (plateau dim, terrain shading still visible) vs `reference-assets/raise_e005_map_rest.png`
  - R: `godot-learning/assets/shaders/indexed_color.gdshader` — `:225` `clamp(psx_color_apply(final_color.rgb, 0))` (the CLUT fold) before `:246` `lit_color = final_color.rgb * (ambient_term + diffuse_term)`; `map_light_debug` uniform (`:40`) isolates ambient/diffuse/normals/albedo to A/B lighting vs tint
  - src: `research/working_documents/MAP_COLOR_SUBSYSTEM_PARITY_RAISE_E005.md`
- **The map and caster/target palette channels fold ONE continuous `ColorStack` spanning all started phases — `PaletteSubsystem.build_stream` concatenates each phase's keyframes at absolute offsets (`_phase_first_frame`) and evaluates at the absolute effect frame — so the ~1-frame pop back to full-bright base at each phase boundary (frame 8 phase1→for_each, frame 124 for_each→phase2) is gone; the PSX shows no such pop because its DDA stepper tweens the CLUT from its current value, making a re-issued identical target a no-op.** — `[S·D·R] 3/3`
  - S: `color_unit_frame_tick @0x800912A4` — the per-frame stepper tweens the DDA from the running CLUT value, not from base (`battle_decompilation.c`)
  - D: Raise E005 CLUT capture (2026-07-12): b3i8 dims smoothly across capture-frames 6→18 with no return to base `(14,10,5)` at the phase seam
  - R: `godot-learning/src/effects/PaletteSubsystem.gd` `build_stream`/`_phase_first_frame` + `godot-learning/src/core/ColorStack.gd` `fold_packed` oracle (branch `import-godot-game`, commits `bc9f21223`, `5ad96e7e3`), validated by `PaletteSubsystemTest._test_map_stream_is_continuous_across_phase_seam` (seam holds `(3,1,0)`, not base), `…_test_caster_unit_tint_continuous_across_phase_seam`, `ColorStackTest._test_fold_packed_reproduces_fold` (56/0, 144/0)
  - src: `research/working_documents/MAP_COLOR_SUBSYSTEM_PARITY_RAISE_E005.md`
- **`build_stack` applies keyframe indices `0..max_keyframe−2` — kf0 is applied, not skipped, and a disabled kf advances timing only — which for E005's `affected_units` gives the authored arc: phase1 idx0 mode-5 δ(−4,−4,−4) Time-8 pre-cast dim; for_each idx0..7 all `ctrl=0x85` mode-5 (dim/hold/breathe/re-dim) with idx8 a `ctrl=0` terminator excluded by both the window bound and the enabled gate; phase2 idx0 `ctrl=0x88` mode-8 absolute snap back to base.** — `[D·R] 2/3`
  - D: Raise E005 CLUT capture (2026-07-12): b3i8 beat trace matches the fold — rest `(14,10,5)` → dim plateau `(3,1,0)` (capture-frames 20–52) → recover/breathe → post-effect re-read exactly `(14,10,5)` (mode-8 snap landed)
  - R: `godot-learning/src/effects/PaletteSubsystem.gd` `build_stack` (`last_idx = min(size, max(0, max_keyframe−1))`), validated by `PaletteSubsystemTest._test_respects_max_keyframe_window` + `…_test_disabled_keyframe_contributes_nothing` (branch `import-godot-game`); E005 authored data in `project-assets/fft-extract/EFFECT/E005.BIN` (editor: `assets/effects/E005/palette.json`)
  - src: `research/working_documents/MAP_COLOR_SUBSYSTEM_PARITY_RAISE_E005.md`

## Notes

(empty — user territory)

## Related

- [[Color Track Interpolation]]
- [[Combat Color Appliers]]
- [[Screen Effect Gradient System]]
