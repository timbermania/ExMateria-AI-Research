# Combat Color Appliers

FFT has two distinct colour appliers that share only the 11-mode classification (0/4/9 additive · 1/5 halve+add · 2/3/6/7 luma · 8/10 reset): the palette/CLUT tint engine `color_tint_blend_apply @0x8008F710` (5-bit BGR555, committed base palette, ×1 params, absolute write-back to the CLUT staging strip) and the screen tint engine `screen_tint_apply @0x80090840` (16.16 fixed-point framebuffer, baseline `screen_tint_baseline_rgb`, absolute write-back to `screen_tint_current`). Neither applier doubles its mode param, and the screen render path is `>>16` throughout with a byte-verbatim quad copy. A live-Ghidra xref pass (2026-07-11) proves these are the only two tint sinks — the map is tinted via CLUT contents or the screen quad, never terrain vertex RGB — which refutes the ×8 map vertex-colour applier, and xref-proves that combat effects drive the palette applier, so Godot's `ColorRecipe` reproduces the exact engine (byte-exact 227/227). A 2026-08 E005/Raise pass split the palette path into three layers (timeline interpreter / per-CLUT-index applier / per-frame stepper — the applier carries no keyframe loop) and showed the combat background-gradient setter `FUN_80090258` runs the identical 11-mode engine on the gradient block — same engine, different targets.

## Points

- **FFT has two distinct colour appliers that share only the 11-mode classification (0/4/9 additive · 1/5 halve+add · 2/3/6/7 luma · 8/10 reset), differing by domain, base source, param scale, and write-back — the palette/CLUT tint is 5-bit BGR555 over the committed base palette with ×1 params, while the screen tint is 16.16 fixed-point over a baseline with absolute write-back.** — `[S] 1/3`
  - S: `color_tint_blend_apply @0x8008F710` (5-bit BGR555 CLUT; base = committed palette `DAT_80099D76`/`DAT_80099D78`; write-back absolute → `dynamic_clut_view_strip @0x800E4EA4`) and `screen_tint_apply @0x80090840` (16.16 fixed-point; baseline `screen_tint_baseline_rgb @0x800A1B70`; write-back absolute → `screen_tint_current @0x800A1B58`), per `battle_decompilation.c` / `battle_disassembly.txt`
  - src: `research/working_documents/COMBAT_COLOR_APPLIER_RECONCILIATION.md`
- **The screen tint applier operates in 16.16 fixed point: mode 0 promotes the byte param with `sll …,0x10` (fixed-point promotion, not doubling) and reads the accumulator back with `sra …,0x10`; results are written absolute to `screen_tint_current @0x800A1B58` from baseline `screen_tint_baseline_rgb @0x800A1B70`.** — `[S] 1/3`
  - S: `screen_tint_apply @0x80090840` and `caseD_0 @0x80090880`; `screen_tint_current @0x800A1B58`, `screen_tint_baseline_rgb @0x800A1B70`, per `battle_decompilation.c` / `battle_disassembly.txt`
  - src: `research/working_documents/COMBAT_COLOR_APPLIER_RECONCILIATION.md`
- **Neither applier doubles its mode param — palette mode 0 is `lbu`→`addu` with no shift, and the only `sll …,1`-style scaling in either applier is the luma weight math (2·R, 3·G) — so Godot `ScreenSubsystem`'s `param = signed × 2` (commented "PSX: <<1") and the effect-editor "SIGNED and DOUBLED" help text have no ROM counterpart.** — `[S] 1/3`
  - S: `caseD_0 @0x8008F7D0` (palette) and `caseD_0 @0x80090880` (screen); luma weights in `color_tint_blend_apply @0x8008F710`, per `battle_decompilation.c` / `battle_disassembly.txt`
  - src: `research/working_documents/COMBAT_COLOR_APPLIER_RECONCILIATION.md`
- **Q2 settled: palette modes 0–3 read the running working colour `DAT_8009967D`, while modes 4–9 read the committed base palette `DAT_80099D76` (word) / `DAT_80099D78` (colour entry) as absolute packed BGR555 — not a delta, not zero — with 4/9 = base + param, 5 = (base>>1) + param, 8 = base (reset), 10 = zero the descriptor, and write-back the final absolute RGB clamped to [0,0x1F].** — `[S] 1/3`
  - S: mode switch inside `color_tint_blend_apply @0x8008F710`; `DAT_8009967D`, `DAT_80099D76`/`DAT_80099D78`, per `battle_decompilation.c` / `battle_disassembly.txt`
  - src: `research/working_documents/COMBAT_COLOR_APPLIER_RECONCILIATION.md`
- **The screen tint render/consumer path is `>>16` throughout and byte-copied to the quad: reader `@0x800917B0` (channel shifts `sra …,0x10` at `0x80091808`/`0x80091828`/`0x80091848`) → dispatcher `FUN_800E8190` case `0x5A` (`caseD_5A @0x800E82B8`, verbatim `lbu` copy into `DAT_800F5B40`/`44`/`48`) → quad builder `SUB_8001D168` — no `>>15`, no `<<1`, no ×2 anywhere.** — `[S] 1/3`
  - S: reader `@0x800917B0`; `FUN_800E8190` case `caseD_5A @0x800E82B8`; `SUB_8001D168`; `DAT_800F5B40`/`44`/`48`, per `battle_decompilation.c` / `battle_disassembly.txt`
  - src: `research/working_documents/COMBAT_COLOR_APPLIER_RECONCILIATION.md`
- **A live-Ghidra xref pass (2026-07-11) proves exactly two tint sinks — callers of `color_tint_blend_apply @0x8008F710` are `color_unit_set_per_unit` ({32}), `color_field_apply` ({33}), `color_unit_frame_tick`, and the effect dispatcher `FUN_800E840C` (combat effects); callers of `screen_tint_apply @0x80090840` are `color_unit_frame_tick` and `map_darkness_shim` ({1A}) — so the map is tinted via CLUT contents (`dynamic_clut_view_strip @0x800E4EA4`) or the screen quad, never terrain vertex RGB.** — `[S] 1/3`
  - S: xrefs of `color_tint_blend_apply @0x8008F710` and `screen_tint_apply @0x80090840` (analyzeHeadless pass, 2026-07-11); callers `color_field_apply @0x80093170`, `color_unit_set_per_unit @0x800931C4`, `map_darkness_worker @0x801466B4`, per `battle_decompilation.c` / `battle_disassembly.txt`
  - src: `research/working_documents/COMBAT_COLOR_APPLIER_RECONCILIATION.md`
- **Combat effects reaching `color_tint_blend_apply` via the effect dispatcher `FUN_800E840C` xref-prove the palette applier is the exact engine Godot's `ColorRecipe` already reproduces byte-exact (227/227) — the import is proven, not inferred.** — `[S·R] 2/3`
  - S: xref `FUN_800E840C` → `color_tint_blend_apply @0x8008F710` (analyzeHeadless pass, 2026-07-11), per `battle_decompilation.c`
  - R: `godot-learning/src/core/ColorRecipe.gd` (byte-exact 227/227 against the palette applier's base-strip output), validated by `godot-learning/tests/ColorRecipeTest.gd`
  - src: `research/working_documents/COMBAT_COLOR_APPLIER_RECONCILIATION.md`
- **The palette/CLUT path is three layers, not one applier: an effect-timeline interpreter (decodes the file ctrl byte — `bit7` = enable, low bits = blend mode — applies the `max_keyframe` window, and calls the applier with the decoded mode), the per-CLUT-index applier `color_tint_blend_apply @0x8008F710` (no keyframe loop, no `0x80` gate, no `max_keyframe` — it iterates CLUT indices, runs the mode-0..10 switch, and either writes the CLUT immediately when `Time==0` or arms a per-index DDA delta when `Time!=0`), and the per-frame stepper `color_unit_frame_tick @0x800912A4` (owns runtime control block `DAT_800995f6`, the enable byte, the curve tables, the ramp cadence, and self re-arms with mode 8/9).** — `[S·D·R] 3/3`
  - S: `color_tint_blend_apply @0x8008F710` (per-index loop, mode switch, `param_2==0` immediate write vs DDA arm), `color_unit_frame_tick @0x800912A4`, `DAT_800995f6` (`battle_decompilation.c`)
  - D: Raise E005 exec-BP counts (2026-07-12, effect-editor session `raise`): the applier ran 75× during one E005 playback — the active map-dim driver — while the subtractive screen quad ran 0×
  - R: `godot-learning/src/effects/PaletteSubsystem.gd` `build_stack` mirrors the interpreter layer (ctrl decode + `0..max_keyframe−2` window) and `ColorStack`/`ColorRecipe` mirror the ramp, validated by `godot-learning/tests/PaletteSubsystemTest.gd` (56/0) and `ColorStackTest.gd` (144/0) (branch `import-godot-game`)
  - src: `research/working_documents/MAP_COLOR_SUBSYSTEM_PARITY_RAISE_E005.md`

## Notes

(empty — user territory)

## Related

- [[Color Tint Luma Modes]]
- [[Map Tint]]
- [[Screen Effect Gradient System]]
- [[Color Screen Opcode]]
