# Color Screen Opcode

Event instruction `0x3E` "Color Screen" (Mode:1, startRGB:3, endRGB:3, Time:2 — 9-byte body) is a timed full-screen colour ramp in the scenario event VM: the dispatcher (`FUN_8013e904`) spawns a cooperative fiber worker (`FUN_801467dc`, fiber kind `0xC`) that lerps a screen-wide colour from start to end RGB over `Time` frames, pushing a new intermediate target every 2 fiber yields via `FUN_8008efb4`, which stages Q8.8 per-frame deltas in a fixed state block and builds a Gouraud full-screen quad rendered per-frame by `FUN_8008f208` (skipped while the current RGB is 0). The `Mode` operand goes verbatim into the GP0 `E1h` semi-transparency ABR field (0=mix, 1=add, 2=B−F subtractive, 3=add-¼ — confirmed live by draw-mode word `0xE1000040`), and a following `{E5} Wait For Instruction Task=12` blocks the main cursor until the fiber completes. Unlike the palette-domain `0x33` Color Field / `0x32` Color Unit, `0x3E` composites a blended quad on top of the already-drawn framebuffer — in scenario 8's scene-out it follows a `{33}` darken and drives the scene to pure black (not white). All static citations were grounded live on 2026-07-11 (scenario 8, `scenario_8_PC_275.sstate`), and the Godot mirror (`ScenarioColorScreen.gd` + `screen_color_mode0..3` shader family, `tests/ScenarioColorScreenTest.gd`) has landed. A 2026-07-21 universal-routing scoping added the display-space perspective: the tints are CPU-folded and clamped before the shader (`ScreenSubsystem._fold_color`), so the hardware blend receives a pre-saturated [0,1] value — no Mobile UNORM-clamp dependence, already Forward+-safe, and the universal routing rule must explicitly exempt this category.

## Points

- **Opcode `0x3E` "Color Screen" has a 9-byte operand body — `Mode, R1, G1, B1, R2, G2, B2, Time(u16 LE)` — as confirmed by the live opcode size-table entry `DAT_8014d170[0x3E]` (`0x8014D1AE`) reading `0x09`.** — `[S·D] 2/3`
  - S: size table `DAT_8014d170[0x3E]` @ `0x8014D1AE` = 0x09 (`battle_disassembly.txt`); param names `Mode, Red (1)…Time` in the `event_instructions.json` catalog
  - D: live RAM reads — `sizetbl[0x3E] @0x8014D1AE = 0x09`, event bytes @ `0x8004AC61` = `3E 02 00 00 00 FF FF FF 0A 00` matching scenario 8 chunk instr 275 (savestate `scenario_8_PC_275.sstate`, 2026-07-11)
  - src: `research/working_documents/COLOR_SCREEN_OPCODE_3E.md`
- **`0x3E` is a fiber-spawn opcode, not an inline one: dispatcher case `uVar25 == 0x3e` in `FUN_8013e904` allocates a fiber slot via `FUN_80149bec(0x10)` and spawns worker `FUN_801467dc`, handing it the operand pointer through the event fiber slot array `DAT_8016986c` (stride 0x400, 16 slots) — the same spawn shape as `0x1A` Map Darkness, `0x41`, and `0x1B`.** — `[S·D] 2/3`
  - S: `FUN_8013e904` case 0x3e, `FUN_80149bec`, `FUN_801467dc`, fiber slot array `DAT_8016986c` (`battle_disassembly.txt`)
  - D: live prologue grounding — dispatcher `0x8013E904` = `27BDFFA8`, worker `0x801467DC` = `27BDFFA0`, primitive `0x8008EFB4` = `27BDFFE0` (savestate `scenario_8_PC_275.sstate`, 2026-07-11)
  - src: `research/working_documents/COLOR_SCREEN_OPCODE_3E.md`
- **The Color Screen worker `FUN_801467dc` (@ `0x801467DC`) reads `Mode` and the six RGB bytes as unsigned 0..255 (contrast `0x32`/`0x33`, which sign-extend) and `Time` as a u16 LE halfword via `event_bytecode_reader_c` (`0x80146078`), then lerps start→end RGB over `Time` frames, pushing one new intermediate target via `FUN_8008efb4` every 2 fiber yields (step `k = 0,2,4,…<Time`), then snaps to the end colour and calls `event_fiber_mark_complete`.** — `[S·D] 2/3`
  - S: `FUN_801467dc` @ 0x801467DC, `event_bytecode_reader_c` @ 0x80146078 (`battle_disassembly.txt`)
  - D: non-pausing logging BP at `0x8008EFB4` captured all six pushes `0→51→102→153→204→255` ~2 frames (≈1.13M cycles) apart (savestate `scenario_8_PC_275.sstate`, 2026-07-11)
  - src: `research/working_documents/COLOR_SCREEN_OPCODE_3E.md`
- **The worker registers fiber kind `0xC` via `event_task_set_kind(0xc)`, so the scenario's `{E5} Wait For Instruction Task=12` (`E5 0C 00`, instr 276) blocks the main scenario cursor until `event_fiber_mark_complete` clears the kind — the VM halts until the ramp finishes.** — `[S·D] 2/3`
  - S: `event_task_set_kind(0xc)` in `FUN_801467dc` (`battle_disassembly.txt`); instr 276 `E5 0C 00` (`scenario_008_chunk.json`)
  - D: scenario 8 free-run from the worker-entry park — the main context resumes only after the ramp, landing on the post-scenario save prompt (savestate `scenario_8_PC_275.sstate`, 2026-07-11)
  - src: `research/working_documents/COLOR_SCREEN_OPCODE_3E.md`
- **The screen primitive `FUN_8008efb4` (@ `0x8008EFB4`) stages the target RGB + per-frame Q8.8 deltas in the fixed state block `DAT_800961c4..e0` and builds a Gouraud full-screen quad at `DAT_800e6aa8` via `FUN_800254cc` (SCUS `0x800254CC`), replicating the vertex colour to all 4 vertices; the inner sub-ramp length is `p5 / _DAT_80045980`, and with live-read `_DAT_80045980 = 1` each push is a 2-frame sub-ramp.** — `[S·D] 2/3`
  - S: `FUN_8008efb4` @ 0x8008EFB4, `FUN_800254cc` @ 0x800254CC, state block `DAT_800961c4..e0`, quad `DAT_800e6aa8`, `_DAT_80045980` (`battle_disassembly.txt`)
  - D: live reads — `_DAT_80045980 = 1`; built primitive at `DAT_800e6aa8`: OT tag word+0 = `0x020C7C70` (vertex buffer `0x800C7C70`), GP0 E1h word+4 = `0xE1000040` (savestate `scenario_8_PC_275.sstate`, 2026-07-11)
  - src: `research/working_documents/COLOR_SCREEN_OPCODE_3E.md`
- **`0x3E`'s `Mode` operand is fed verbatim into the GP0 `E1h` Set-Draw-Mode packet as `mode<<5` — i.e. straight into the PSX semi-transparency ABR field — so Mode 0 = 0.5·B+0.5·F mix, 1 = B+F additive, 2 = B−F subtractive, 3 = B+¼F soft-additive; Mode=2 was confirmed live by the built primitive's draw-mode word `0xE1000040`.** — `[S·D] 2/3`
  - S: `FUN_8008efb4` → `FUN_800257c8` (mode<<5 into the primitive attribute word) (`battle_disassembly.txt`)
  - D: GP0 E1h word read live from the built primitive at `DAT_800e6aa8` = `0xE1000040` (ABR bits [6:5] = 2) (savestate `scenario_8_PC_275.sstate`, 2026-07-11)
  - src: `research/working_documents/COLOR_SCREEN_OPCODE_3E.md`
- **The per-frame tick `FUN_8008f130` (@ `0x8008F130`) advances the current RGB toward the target in Q8.8 (snapping to the target on the final frame), and the per-frame render `FUN_8008f208` (@ `0x8008F208`) writes the current RGB into the vertex buffer at `DAT_800c7c74` and submits the quad — but only when the current RGB ≠ (0,0,0), so a black overlay draws nothing.** — `[S·D] 2/3`
  - S: `FUN_8008f130` @ 0x8008F130, `FUN_8008f208` @ 0x8008F208, vertex buffer `DAT_800c7c74` (`battle_disassembly.txt`)
  - D: scenario 8 live capture — framebuffer shows the quad absent at the black ramp start, appearing and darkening the scene as the ramp climbs (savestate `scenario_8_PC_275.sstate`, 2026-07-11)
  - src: `research/working_documents/COLOR_SCREEN_OPCODE_3E.md`
- **In scenario 8 (MAP024/entd392) the chunk's only `0x3E` is instr 275 — raw `3E 02 00 00 00 FF FF FF 0A 00` (Mode=2 subtractive, start black → end white, Time=10) — following a `{33}` Color Field palette darken, and the net visual is a fade to BLACK at scene-out (framebuffer brightness 8181 → 0), after which the game shows the post-scenario save prompt; not a fade to white.** — `[S·D] 2/3`
  - S: instrs 273–276 (`godot-learning/assets/scenarios/chunks/scenario_008_chunk.json`)
  - D: live event bytes @ `0x8004AC61` match the chunk; framebuffer brightness sampling 8181 (post-`{33}`) → 0 (ramp climbing); free-run lands on the save prompt (savestate `scenario_8_PC_275.sstate`, 2026-07-11)
  - src: `research/working_documents/COLOR_SCREEN_OPCODE_3E.md`
- **`0x3E` Color Screen is a screen-space composited quad over the framebuffer, not a palette tint: unlike `0x33` Color Field / `0x32` Color Unit, which modify the CLUT view-palettes, `0x3E` draws a blended quad on top of the already-drawn (and possibly palette-darkened) scene.** — `[S] 1/3`
  - S: `0x3E` spawns `FUN_801467dc` → `FUN_8008efb4` (quad path) while `0x33` calls `color_field_apply` inline (`battle_disassembly.txt`)
  - src: `research/working_documents/COLOR_SCREEN_OPCODE_3E.md`
- **The Godot reimplementation handles `0x3E`: `ScenarioVM._op_color_screen` parses the 9-byte body and drives a full-screen NDC quad node (`ScenarioColorScreen.gd`, mirroring `ScenarioDarkScreen`) from the unified park-gated 1/60 render clock, reproducing the PSX 2-tick discrete stepping (0,51,102,153,204,255 for Time=10), hides the quad at (0,0,0), and registers kind-12 task liveness so a following `{E5} Task=12` blocks until the ramp ends — the real-chunk scenario 8 `{3E}` now dispatches without halting.** — `[S·R] 2/3`
  - S: PSX behaviour mirrored — 2-tick stepping + end snap in `FUN_801467dc`, draw-skip at zero in `FUN_8008f208` (`battle_disassembly.txt`)
  - R: `godot-learning/src/scenarios/ScenarioColorScreen.gd`, `ScenarioVM.gd` (`_op_color_screen`, `_tick_time_driven_effects`, `TASK_COLORSCREEN := 12`), `ScenarioDecode.color_screen` / `ScenarioApply.color_screen` — test `godot-learning/tests/ScenarioColorScreenTest.gd` (33 assertions, registered in `run_all_tests.sh`; 2026-07-11)
  - src: `research/working_documents/COLOR_SCREEN_OPCODE_3E.md`
- **The four ABR blend equations are implemented in Godot as a per-mode shader family — `screen_color_mode0..3.gdshader` (0=mix ½, 1=add, 2=sub, 3=add ¼) — consolidated around a shared `psx_screen_blend.gdshaderinc` include (NDC vertex + `screen_color` uniform only, since render_mode is compile-time per file), following the existing `TileOverlayConfig` `MODE_SHADERS` swap pattern.** — `[R] 1/3`
  - R: `godot-learning/assets/shaders/psx_screen_blend.gdshaderinc` + `screen_color_mode0..3.gdshader`; `godot-learning/src/map/TileOverlayConfig.gd` (MODE_SHADERS precedent) — guarded by the existing tile/particle shader tests + `tests/ScenarioColorScreenTest.gd` (2026-07-11)
  - src: `research/working_documents/COLOR_SCREEN_OPCODE_3E.md`
- **The screen tints `screen_color_mode0..3` / `psx_screen_blend` are CPU-folded and clamped before the shader (`ScreenSubsystem._fold_color`, `ScreenEffectOverlay.compose_corners`, `ScenarioColorScreen._push`) and applied as a full-screen quad — the hardware blend receives a pre-saturated [0,1] value, so the family does not depend on the Mobile UNORM clamp and is already Forward+-safe; routing them through the compositor would be redundant (a single whole-canvas fold, not bounded prims), so the universal routing rule must explicitly exempt this Category D.** — `[R] 1/3`
  - R: `godot-learning/src/effects/ScreenSubsystem.gd:158-160` (`_fold_color`), `godot-learning/src/effects/ScreenEffectOverlay.gd:184-198` (`compose_corners`), `godot-learning/src/scenarios/ScenarioColorScreen.gd:118-123` (`_push`) (doc-cited 2026-07-21)
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Reset Palette Opcode]]
- [[Screen Effect Gradient System]]
- [[PSX GPU Primitives]]
