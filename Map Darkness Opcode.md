# Map Darkness Opcode

Event opcode `{1A}` "Map Darkness" (Blend:1, Red:1, Green:1, Blue:1, Time:1) ramps a per-channel "oxide" accumulator toward `rest_baseline + signed(R,G,B)` over `Time*8` frames, and the PSX applier (`FUN_80090840`) emits it as a screen-blend quad over the map. Scenario-4 savestate proof (2026-07-05) settled the Orbonne "persistent map darkening after the lightning" question: the `{1A}` subtractive quad is **inert** in PSX battle beats — the +30/ch accumulator delta changes zero rendered pixels and the map palette in VRAM is byte-identical before/after the strike, so the visible map darken is the view-ticker `{33}`/`{32}` CLUT palette-content fade, not `{1A}`. Godot's front-of-world subtractive `OxideRect` was therefore a phantom with no PSX counterpart and was removed, opt-in legacy gate included (ADR-0051): the accumulator still integrates to mirror PSX RAM state, but nothing is drawn. Godot's oxide baseline remains scenario-1-hardcoded `(20,4,0)` vs scenario 4's PSX rest oxide `(73,61,46)`. A 2026-07-05 cross-source pass closed the baseline-origin question: the PSX byte baseline is the map mesh's parsed 3-byte ambient-light color (mesh header pointer `0x64`, `+36` past the 3 directional lights) — map-loaded data, byte-identical live-vs-parsed for MAP056 — and external sources (ffhacktics, FFTPatcher `EventCommands.xml`) confirm the opcode's purpose and element-for-element parameter layout.

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
- **The `{1A}` applier's `Blend==4` ("byte-register add") computes `target_c = byte_baseline[c] + signed(param_c)` (clamped to `[0,0xFF]`); `Time==0` snaps the current color to the target, otherwise `duration = Time×8` frames with per-frame delta `(target−current)/(Time×8)` integrated by the per-frame ticker — scenario 1's darken arm faded fully to (40,35,31) before the untint arm fired, and the untint target = baseline (20,4,0).** — `[S·D·R] 3/3`
  - S: `FUN_80090840` caseD_4 @0x80090950, clamp tail @0x80090ab4, snap / `Time<<3` @0x80090b14, delta store @0x80090b2c → `0x800a1b64/68/6c`, per-frame ticker ~0x800917b0; 11-way jump table `switchD_80090878` (BATTLE.BIN load base `0x80067000`, doc §2)
  - D: `probe_map_darkness.py` capture (2026-06-30, `last_run/probe_map_darkness.jsonl`): arms hit applier `0x80090840` filtered `ra==0x800934f4`; at the untint arm `cur` already (0x28,0x23,0x1f)=(40,35,31), `FINAL cur`=(20,4,0)=baseline
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_map_darkness` (Blend==4 target math, `Time*8` duration, Time==0 snap, persistent `_oxide_current_byte`), validated by `godot-learning/tests/ScenarioMapDarknessTest.gd` `_test_blend4_target_math` / `_test_duration_is_time_times_eight` / `_test_time_zero_snaps`
  - src: `research/working_documents/scenario_1_captures/map_darkness_oxide_decode.md`
- **Scenario 1 fires `{1A}` twice to bracket the Orbonne prayer `Display Message` (`Dialog=0x09`): chunk offset 250 `1a 04 14 1f 1f 04` (Blend 4, signed +(20,31,31), Time 4 = DARKEN) just before the overlay, and offset 274 `1a 04 00 00 00 04` (all-zero signed params = UNTINT to baseline) just after the `Wait 0x56` (86 vsyncs) that follows it.** — `[S·D·R] 3/3`
  - S: `godot-learning/assets/scenarios/scenario_1_chunk.json` offsets 250/274 + param layout `Blend:1 R:1 G:1 B:1 Time:1` (doc §1); element-for-element corroborated by FFTPatcher `EventCommands.xml` (`hex=1A "Map Darkness"`) and ffhacktics `MapDarkness(xBM, +RED, +GRN, +BLU, TIM)` — tooltip "Modifies the color of the map's lighting. Battle-friendly." (doc §8)
  - D: `probe_map_darkness.py` capture (2026-06-30): both arms live-captured with byte-exact params `blend=4 time=4 R=20 G=31 B=31` then `R=0 G=0 B=0`
  - R: `godot-learning/tests/ScenarioMapDarknessTest.gd` `_test_real_chunk_arms_do_not_halt` + `_test_prayer_overlay_untint_fires_during_typewriter` (headful-verified: darken target (40,35,31), untint (20,4,0))
  - src: `research/working_documents/scenario_1_captures/map_darkness_oxide_decode.md`
- **The `{1A}` oxide state lives in fixed RAM: current R/G/B as 16.16 fixed point at `0x800a1b58/5c/60` (>>16 = the applied 8-bit color), per-frame deltas at `0x800a1b64/68/6c`, the byte baseline (the map's rest darkness color) at `0x800a1b70/71/72`, and an animation/mode flag at `0x800a1b54` (mode 10 = STOP clears it).** — `[S·D] 2/3`
  - S: doc §2 register table (BATTLE.BIN load base `0x80067000`); the consumer ticker `0x800917b0` reads current + delta and writes back
  - D: `probe_map_darkness.py` live reads at applier frames (2026-06-30: `base=14,04,00` stable across both arms + final) and the scenario-6 abduct beat (2026-07-05: cur=(103,91,76), base=(73,61,46))
  - R: none — 16.16 fixed-point register file not present in godot-learning (probed `godot-learning/src/` + `godot-learning/tests/`; `ScenarioVM.gd` mirrors the semantics with 8-bit `_oxide_current_byte` / `_OXIDE_BASELINE` only)
  - src: `research/working_documents/scenario_1_captures/map_darkness_oxide_decode.md`
- **The applier emits the oxide current color on the shared screen-tint path — a Gouraud full-screen quad (GP0 `0x2C`/`0x38`; GPU setup `0x800e8190`, `advance_screen_color_track` `0x801A45C8`) — and the designed blend is SUBTRACTIVE: raising the applied color subtracts from the scene, so the prayer's intended effect is an extra subtraction of (20,31,31)/255 ≈ (0.078, 0.122, 0.122), a gentle warm/oxide cast; the exact PSX ABR semi-transparency bits remain unconfirmed (doc §6).** — `[S·D] 2/3`
  - S: doc §2 "Consumer" — GP0 0x2C/0x38, GPU setup 0x800e8190, `advance_screen_color_track 0x801A45C8`
  - D: `probe_map_darkness.py` capture (2026-06-30): rest baseline a small (20,4,0) raised to (40,35,31) is consistent with the subtractive reading (doc §3 interpretation table)
  - R: none — `BLEND_MODE_SUB` / `OxideRect` render surface not present in godot-learning (probed `godot-learning/src/` + `godot-learning/tests/`; removed as a phantom, ADR-0051 — `ScenarioVM.gd` L266 comment documents the removal)
  - src: `research/working_documents/scenario_1_captures/map_darkness_oxide_decode.md`
- **The oxide byte baseline (`0x800a1b70/71/72`) IS the map mesh's parsed ambient-light color — a 3-byte RGB at mesh header pointer `0x64`, `+36` past the 3 directional lights — so the baseline is map-loaded data, not an opcode-set value: live GTE ambient @ `0x800f6aa2` read at the scenario-6 abduct beat (73,61,46) is byte-identical to the parsed MAP056 state-2 ambient (73,61,46) (other states differ: state 0 = (64,48,32), state 3 = (87,88,86)); ffhacktics `Maps/Mesh` and GaneshaDx `ProcessAmbientLight` (same `+36` offset) corroborate.** — `[S·D·R] 3/3`
  - S: live GTE ambient register `0x800f6aa2` (doc §7 "Bonus ground truth"); mesh lighting-chunk layout (pointer `0x64`, ambient `+36` after 3 directional lights) per GaneshaDx `MeshResourceData.cs::ProcessAmbientLight` + ffhacktics `Maps/Mesh` (doc §8)
  - D: parsed-vs-live byte diff closed 2026-07-05 (doc §8 Q4) against `reference-assets/scenario6_abduct_ovelia_letgo.sstate`
  - R: `godot-learning/tools/fft_exporter/parsers/lighting.py` (`LIGHTING_POINTER = 100`, ambient read at `offset += 36`) → `assets/maps/MAP056/scene_manifest.json` state 2, consumed by `godot-learning/src/map/MapLightingConfig.gd` `ambient_light` uniform (state-selected via `MapStateSelector.gd` / `MapComposer.gd`)
  - src: `research/working_documents/scenario_1_captures/map_darkness_oxide_decode.md`
- **The `{1A}` darkness state persists across event-member boundaries within a battle group: at the scenario-6 abduct beat RAM `current − baseline` = (103,91,76) − (73,61,46) = (30,30,30) — exactly scenario 4's `{1A}` Blend-4 (30,30,30) Time=4 param, with scenarios 5/6 issuing no `{1A}` of their own — so Godot's `ScenarioVM.start()` not resetting the oxide accumulator on member advance is faithful (the `{76} Dark Screen` sibling, by contrast, IS reset on fresh boot).** — `[D·R] 2/3`
  - D: `reference-assets/scenario6_abduct_ovelia_letgo.sstate` loaded in pcsx-redux, darkness globals read from main RAM (2026-07-05, doc §7); chunk census: scn3 root issues 12, scn4 issues 1, netting baseline+(30,30,30) by the abduct beat
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `start()` — persistent scene overlays carry over on member advance (`fresh=false`), only `{76} Dark Screen` is freed on fresh boot; the oxide accumulator is left untouched either way
  - src: `research/working_documents/scenario_1_captures/map_darkness_oxide_decode.md`

## Notes

(empty — user territory)

## Related

- [[Color Tint Luma Modes]]
- [[Background Opcode]]
- [[Color Screen Opcode]]
- [[DarkScreen Opcode]]
- [[Map Tint]]
- [[Screen Effect Gradient System]]
- [[Display Message Opcode]]
