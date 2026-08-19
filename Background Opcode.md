# Background Opcode

Event opcode `{2E}` "Background" (0x2E, 8-byte operand `Rt,Gt,Bt,Rb,Gb,Bb,Time,Unk`) is FFT's scenario-level screen backdrop: it sets the top/bottom colours of a full-screen Gouraud gradient quad driven by the 16.16 RAM gradient block at `0x800a1b18`/`0x800a1b30`, snapping when `Time==0` and otherwise ramping linearly over `Time*8` frames. Orbonne scenario-4's lightning is exactly this mechanism — a scripted `{2E}` sequence (rest → snap bright → ramp back, twice) paired with `{33}` Color Field map blue-hue, both channels converged frame-for-frame by the shared per-frame color ticker while weather `{3C}` stays idle. The handler, Unk-byte setter routing, and byte-exact ramp were grounded live against the Orbonne reference savestate, and the port ships in `godot-learning` (`ScenarioBackground` ramp model + `ScenarioVM._op_background` → `ScreenBackground` quad), unit-tested byte-exact and headful-verified on scenario 4 (2026-07-05).

## Points

- **`{2E}` "Background" (0x2E, 8 operand bytes `Rt,Gt,Bt,Rb,Gb,Bb,Time,Unk`) sets the top/bottom colours of the screen background gradient: `Time==0` snaps instantly, `Time>0` ramps linearly over exactly `Time*8` frames (per-frame delta `(target−current)/(Time·8)`), landing on target and then holding.** — `[S·D·R] 3/3`
  - S: dispatch `0x8014506c` bne chain, handler `0x80145074`, ramp setter `FUN_80090048` (`project-assets/fft-rom/battle_disassembly.txt`)
  - D: write-BP trace on `0x800a1b24` free-run through both flashes from reference savestate `orbonne_prescenario_load_lightning_pan.sstate` (2026-07-05): flash 48→184 in 8 frames (+17/frame), decay 184→48 in 32 frames (−4.25/frame)
  - R: `godot-learning/src/scenarios/ScenarioBackground.gd` (`apply`/`tick`, `ramp_frames_for_time`) + `ScenarioVM._op_background`, validated by `godot-learning/tests/ScenarioBackgroundTest.gd` (50/0)
  - src: `research/working_documents/LIGHTNING_FLASH_OPCODE_2E_BACKGROUND.md`
- **The `{2E}` Unk byte selects which setter runs — Unk!=0 → bare `FUN_800935f4`, Unk==0 → `FUN_800901f8` plus the secondary resting copy `0x800a1b48/4c` — not snap-vs-ramp; snap-vs-ramp is controlled entirely by Time, and both paths render the primary gradient identically.** — `[S·D] 2/3`
  - S: live BP at both handler paths `0x801450ac`/`0x801450c0` (`battle_disassembly.txt`)
  - D: live breakpoint at both setter paths, Orbonne reference savestate (2026-07-05)
  - R: none — Unk-gated setter routing not present in godot-learning (probed `godot-learning/src/` + `godot-learning/tests/`; the Unk byte is decoded but unused in `ScenarioDecode.background`, and `ScenarioBackground` ignores it)
  - src: `research/working_documents/LIGHTNING_FLASH_OPCODE_2E_BACKGROUND.md`
- **Orbonne scenario-4's lightning is authored as a `{2E}` sequence in the event chunk — rest (48,56,48)/(48,56,96) Time=0 → flash 1 (184,188,119)/(16,29,61) Time=1 → decay Time=4 → flash 2 (113,124,106)/(16,16,24) Time=1 → decay Time=4 — and the committed chunk bytes match live PSX RAM byte-exact.** — `[S·D·R] 3/3`
  - S: opcode-0x2E entries at chunk offsets 61/184/202/229/247 in `godot-learning/assets/scenarios/chunks/scenario_004_chunk.json`
  - D: live trace from the Orbonne reference savestate (2026-07-05): flash-1 arm grad top=(184,188,119) bot=(16,29,61) — byte-exact match to chunk offset 184
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_background` → `ScreenBackground` quad; headful-verified 2026-07-05 (scenario 4 fires all 5 `{2E}` + 5 `{33}` with byte-exact operands)
  - src: `research/working_documents/LIGHTNING_FLASH_OPCODE_2E_BACKGROUND.md`
- **The `{2E}` background is NOT a VRAM CLUT — it is a Gouraud-shaded full-screen quad driven by the 16.16 RAM gradient block `0x800a1b18` (top R/G/B `0x800a1b18/1c/20`, top per-frame deltas `0x800a1b24/28/2c`, bottom R/G/B `0x800a1b30/34/38`, bottom deltas `0x800a1b3c/40/44`, secondary resting copy `0x800a1b48/4c`, animation flag `0x800a1b14` stayed 0 during the flashes), with the top colour at the top two vertices and the bottom colour at the bottom two.** — `[S·D·R] 3/3`
  - S: gradient-block addresses (`battle_disassembly.txt`); Gouraud quad + per-vertex colour mapping per `research/wiki_articles/screen_effect_gradient_system.md`
  - D: full-color state frozen at the flash-1 arm + top→both-top-verts / bottom→both-bottom-verts trace, Orbonne reference savestate (2026-07-05)
  - R: `godot-learning/assets/scenes/PlayerCamera.tscn` `ScreenBackground` quad + `assets/shaders/screen_background.gdshader` corner colours, pushed via `ScreenEffectOverlay` / `ScenarioVM._apply_screen_background`; validated by `godot-learning/tests/ScenarioBackgroundTest.gd`
  - src: `research/working_documents/LIGHTNING_FLASH_OPCODE_2E_BACKGROUND.md`
- **Both flash channels (gradient + view-0 map blue) are converged frame-for-frame by the shared per-frame color ticker (`0x800917b0` family), which is why the snap looks coordinated — and weather `{3C}` stays idle throughout (`0x80173f68 == 0xFFFF`), so the lightning is scripted, not weather-generated.** — `[S·D] 2/3`
  - S: per-frame ticker `0x800917b0` family; weather state `0x80173f68` (`battle_disassembly.txt`)
  - D: full-color state + correlation trace, Orbonne reference savestate (2026-07-05)
  - R: none — the `0x800917b0` ticker not present in godot-learning (probed `godot-learning/src/` + `godot-learning/tests/`; Godot ticks the scenario background and field-tint ramps in `ScenarioVM._tick_once` instead)
  - src: `research/working_documents/LIGHTNING_FLASH_OPCODE_2E_BACKGROUND.md`

## Notes

(empty — user territory)

## Related

- [[Screen Effect Gradient System]]
- [[Color Tint Luma Modes]]
- [[Event VM Index]]
