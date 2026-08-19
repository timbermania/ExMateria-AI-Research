# Map Darkness Opcode

Event opcode `{1A}` "Map Darkness" (Blend:1, Red:1, Green:1, Blue:1, Time:1) ramps a per-channel "oxide" accumulator toward `rest_baseline + signed(R,G,B)` over `Time*8` frames, and the PSX applier (`FUN_80090840`) emits it as a screen-blend quad over the map. Scenario-4 savestate proof (2026-07-05) settled the Orbonne "persistent map darkening after the lightning" question: the `{1A}` subtractive quad is **inert** in PSX battle beats — the +30/ch accumulator delta changes zero rendered pixels and the map palette in VRAM is byte-identical before/after the strike, so the visible map darken is the view-ticker `{33}`/`{32}` CLUT palette-content fade, not `{1A}`. Godot's front-of-world subtractive `OxideRect` was therefore a phantom with no PSX counterpart and was removed, opt-in legacy gate included (ADR-0051): the accumulator still integrates to mirror PSX RAM state, but nothing is drawn. Godot's oxide baseline remains scenario-1-hardcoded `(20,4,0)` vs scenario 4's PSX rest oxide `(73,61,46)`.

## Points

- **The `{1A}` Map Darkness subtractive quad is effectively NOT drawn in PSX battle beats: a +30/ch oxide-accumulator ramp changes zero rendered pixels (VRAM X≥256 byte-identical across the strike; patching `cur` back to base and re-running the same deterministic 0.2 s cutscene yields a byte-identical frame), and the visible map darken in those beats is the view-ticker `{33}`/`{32}` CLUT palette-content fade — `{1A}` oxide is the wrong channel for the map darken.** — `[S·D·R] 3/3`
  - S: oxide accumulator `0x800a1b58/5c/60` + `{2E}` bg-gradient context `0x800a1b18` (`scenario4_captures/read_oxide_grad.lua`, `research/working_documents/scenario_1_captures/map_darkness_oxide_decode.md` §7); real-darken channel: view ticker `0x800917b0` gated by each view's `thr` byte (`prayer_screen_tint_quad_decode.md`)
  - D: scenario-4 pre/post-strike savestates `reference-assets/scenario4_prestrike_before.sstate` / `scenario4_postflash_after.sstate` (2026-07-05): VRAM X≥256 (all texture pages + CLUTs) 0/~524k 16-bit words differ; `cur` (103,91,76)→(73,61,46) patch + deterministic re-run = 0 px changed, patch held; independently reproduces the scenario-6 poke (`map_darkness_oxide_decode.md` §7); the two frames differ mainly by camera (zoom onto shadowed stone + ongoing `{76}` Dark Screen progression, luma 44→39 over 0.5 s), not palette
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_map_darkness` → `ScenarioApply.map_darkness` → `ScenarioWorld.set_map_darkness` — the `_oxide_current_byte` accumulator still ramps but the `{1A}` subtractive render surface is gone (opt-in `SCN_OXIDE_RENDER=1` legacy gate removed, ADR-0051), validated by `godot-learning/tests/ScenarioMapDarknessTest.gd` `_test_darken_untint_ramps_accumulator` (ramps to (40,35,31) with no render surface)
  - src: `research/working_documents/MAP_FLASH_SCENARIO4_PERSISTENT_DARK.md`
- **Godot's oxide baseline is scenario-1-specific: `_OXIDE_BASELINE` is hardcoded `(20,4,0)` (scenario-1 prayer map) while scenario 4's PSX rest oxide is `(73,61,46)` — so the Blend-4 target math (`baseline + signed(R,G,B)`) is scenario-1-biased in magnitude for other scenarios.** — `[D·R] 2/3`
  - D: scenario-4 sstate captures (2026-07-05): rest baseline `(73,61,46)` identical in both the pre-strike and post-flash states
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_OXIDE_BASELINE := Vector3(20, 4, 0)` (fed to `ScenarioDecode.map_darkness`), validated by `godot-learning/tests/ScenarioMapDarknessTest.gd` `_test_blend4_target_math` (baseline + signed RGB = (40,35,31))
  - src: `research/working_documents/MAP_FLASH_SCENARIO4_PERSISTENT_DARK.md`

## Notes

(empty — user territory)

## Related

- [[Color Tint Luma Modes]]
- [[Background Opcode]]
- [[Color Screen Opcode]]
- [[DarkScreen Opcode]]
- [[Map Tint]]
