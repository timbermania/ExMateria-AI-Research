# Poison Visual Recolor

How the FFT PSX renders the green tint on a poisoned scenario unit ({92} SS=2, issue #154, doc §12). RESOLVED 2026-07-05: poison green is a **static** in-place VRAM CLUT recolor of the unit sprite — not a pulse, not an SPR-row swap, and not the {32}/{33} palette tint engine (`color_tint_blend_apply @ 0x8008F710` fires 0× across the SS=2 window). It is the {92}-scheduled sprite/tile rebuild driven by the shared status-recolor dispatch (writer store `0x80092CA0`), and the exact transform is a computed per-channel shift: `out = base/2 + (R:0, G:8, B:0)` in 5-bit BGR555, pinned from the dispatch's live registers. `godot-learning` ports it as a static tint (`scale 0.5, bias (0, 8/31, 0)` in `ScenarioColorTint`/`ScenarioWorld`) and `ScenarioInflictStatusTest` validates it. The green is baked once at the apply transition and held in VRAM (clearing the poison bit afterwards yields 0 pixel change over ~60 frames); the SS=2 rebuild drives two passes through the dispatch (base-load offsets 1/0/−1, then green offsets 0/8/0 from the poison branch `LAB_80082248` @ ~`0x80082180`); and the green keys on the poison bit, not the critical pose.

## Points

- **Poison green is a static in-place palette recolor of the unit sprite, not a pulse: the |ss00 − ss02| framebuffer diff isolates to exactly the three poisoned units (dialog box = 0 diff), and a same-unit same-pose before/after capture (silhouette IoU 0.98) shows blonde/purple → wholly green with no frame-to-frame flicker.** — `[D·R] 2/3`
  - D: op92_status_probe poison_visual captures `recolor_diff_ss00_vs_ss02.png` and `unit_recolor_ss00_base.png` / `unit_recolor_ss02_green.png`, scenario 6 {92} SS=2 poke, source frames `../op92_ss_00/frame.png` / `../op92_ss_02/frame.png` (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioColorTint.gd` `POISON_SCALE`/`POISON_GREEN_BIAS` applied as a static tint via `src/scenarios/ScenarioWorld.gd:225` `tint.set_static(...)` — validated by `tests/ScenarioInflictStatusTest.gd` `_test_status2_poisons_and_kneels`
  - src: `research/working_documents/op92_status_probe/captures/poison_visual/README.md`
- **Poison green does NOT route through the {32}/{33} palette tint engine: a live breakpoint on `color_tint_blend_apply @ 0x8008F710` during the SS=2 poke fired 0 times; the recolor is the {92}-scheduled sprite/tile rebuild that overwrites the unit's VRAM CLUT in place via the shared status-recolor dispatch (writer store `0x80092CA0`).** — `[S·D·R] 3/3`
  - S: `color_tint_blend_apply @ 0x8008F710` (excluded sink) and status-recolor dispatch writer store `0x80092CA0`, per op92 doc §12
  - D: live BP probe `probe_color_tint_blend_apply.py`, 0 firings across the SS=2 window (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioColorTint.gd` models the poison recolor as the static CLUT tint (not the `ColorRecipe` {32}/{33} tint path), applied at `src/scenarios/ScenarioWorld.gd:225` — validated by `tests/ScenarioInflictStatusTest.gd`
  - src: `research/working_documents/op92_status_probe/captures/poison_visual/README.md`
- **The poison recolor is a computed per-channel shift, not an SPR-row swap: `out = base/2 + (R:0, G:8, B:0)` in 5-bit BGR555 (halve every channel, then add green 8/31), with the green offset triple read straight from the status-recolor dispatch's registers (writer store `0x80092CA0`), deterministically re-derived by `probe_poison_clut_delta.py --regs`.** — `[S·D·R] 3/3`
  - S: status-recolor dispatch writer store `0x80092CA0` (offset triple in the dispatch's stack args), per op92 doc §12.6
  - D: deterministic register re-derivation `probe_poison_clut_delta.py --regs` vs the live poisoned CLUT (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioColorTint.gd` `POISON_SCALE = Vector3(0.5, 0.5, 0.5)` / `POISON_GREEN_BIAS = Vector3(0.0, 8.0/31.0, 0.0)` (lines 106–107), applied via `src/scenarios/ScenarioWorld.gd:225` `tint.set_static(...)` — validated by `tests/ScenarioInflictStatusTest.gd`
  - src: `research/working_documents/op92_status_probe/captures/poison_visual/README.md`
- **The poison green is baked once at the apply transition and HELD in VRAM — it is not re-derived per frame from the status bit: clearing the raw poison bit (`+0x5b & 0x80`) after the recolor settled and waiting ~60 frames produced 0 pixel change; the CLUT holds across walk/idle/kneel until the next palette transition (e.g. a cure).** — `[D·R] 2/3`
  - D: op92 rig post-clear dump — poison bit cleared, ~60-frame wait, 0 pixel change (scenario 6 {92} SS=2, 2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioColorTint.gd` `set_static` (a held, no-tick tint) applied by `src/scenarios/ScenarioWorld.gd` `inflict_poison_critical` — validated by `tests/ScenarioInflictStatusTest.gd` `_test_status2_poisons_and_kneels`
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`
- **The SS=2 rebuild drives two passes through the shared status-recolor dispatch `FUN_800927bc`: pass 1 (per-channel offsets 1/0/−1) loads the base palette unchanged, pass 2 (offsets 0/8/0) applies the poison green; the per-status offset triple is picked by a selector @ ~`0x80082180` (poison = the table-indexed `LAB_80082248` branch), staging into the strip at `0x800E54A4` — the same palette-content machinery as prayer darkening.** — `[S·D] 2/3`
  - S: `FUN_800927bc`, selector `LAB_8008218x`/`LAB_80082248` @ ~`0x80082180`, staging strip `0x800E54A4` (doc §12.6)
  - D: live BP on the writer store `sh v0, 0(a0) @ 0x80092CA0` observed both passes per rebuild (2026-07-05)
  - R: none — two-pass rebuild not present in godot-learning (a single static tint is applied instead)
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`
- **The green keys on the poison bit, not the critical pose: green appears only with SS=2; SS=1 Crystal shows the crystal look, not green; a normally-critical kneeling unit is not green (full dead→critical-no-poison isolation not run — game behaviour + the SS-branch evidence make poison the source).** — `[D·R] 2/3`
  - D: op92 `ss_01`/`ss_02` captures (2026-07-05)
  - R: `godot-learning` applies the green only in the SS=2 branch (`ScenarioWorld.inflict_poison_critical`); the critical-only pose (`AnimationStateController.to_critical_idle`) carries no tint — validated by `tests/ScenarioInflictStatusTest.gd`
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`

## Notes

(empty — user territory)

## Related

- [[Inflict Status Opcode]]
- [[Combat Color Appliers]]
- [[Crystal Status Visual]]
