# Scenario Beat Capture

Cross-engine method for capturing a "scenario beat" — the frame frozen at the moment the event VM is about to execute event instruction N ("beat PC N") — with the same pre-execution park semantics on both engines: the PSX parks via a Read-BP on instruction N's opcode byte in the in-RAM event chunk (scenario-event-debugger) and grabs the frozen VRAM, while Godot parks via the debug session's rewind target (F3 panel double-click or the headless capture tool), so the two frames are directly comparable. Endorsed by the user 2026-07-07 as *the* way to get a beat, after the scn6 carry investigation showed that settled-anim savestates and anim-id-labeled vsync raws are not beat PC N. On the Godot side a park is a true global freeze — every time-driven visual effect ticks behind one `if not paused:` gate in `ScenarioVM._tick_once` — so a parked frame stops fading instead of running its ramps.

## Points

- **A "beat PC N" frame is captured on the PSX by parking a Read-BP on instruction N's opcode byte (`0x8004A6BC + offset`) *before* it executes — scenario-event-debugger's `scn_jump(N)` reloads a mid-scene base savestate, arms the Read-BP, and fast-forwards with Circle auto-advance through dialogue gates until the VM reaches N — then grabbing the paused-safe frozen framebuffer via `GET /api/v1/gpu/vram/raw` (1 MB = VRAM 1024×512×2; display window (0,0) 256×240, RGB555 little-endian, `v = lo | hi<<8; R=(v&31)<<3; G=((v>>5)&31)<<3; B=((v>>10)&31)<<3`), which decodes to exactly what was on screen at beat N.** — `[S·D] 2/3`
  - S: opcode-byte base `0x8004A6BC` (`SCENARIO_LOADING.md` §3.2.4); VRAM raw endpoint/format per `research/scenario-event-debugger/README.md`
  - D: PSX frozen-VRAM capture at scn6 PC210 (`/tmp/sxs2/psx_pc210_fb_00.png`) (2026-07-07)
  - src: `research/working_documents/CAPTURE_SCENARIO_BEAT_HOWTO.md`
- **The same beat is captured in Godot by setting the debug session's rewind target to N and letting the VM fast-play there: double-clicking PC N in the F3 debug panel (`DebugOverlay` KEY_F3 → `ScenarioVMDebugPanel`) or the headless `PC=N RES=256 BRIGHT=1 SHOT=… godot --path . -s res://tools/capture_carry_ab.gd` both set `ScenarioDebugSession.rewind_target_pc = N` and park pre-execution, matching the PSX read-BP park semantics so the frames are comparable.** — `[D·R] 2/3`
  - D: Godot beat capture at scn6 PC210, native 256 (`/tmp/sxs2/godot_pc210_bright.png`) (2026-07-07)
  - R: `godot-learning/src/debug/ScenarioDebugSession.gd` (`rewind_target_pc`), `godot-learning/src/debug/ScenarioVMDebugPanel.gd` (double-click rewind), `godot-learning/tools/capture_carry_ab.gd` (no named test)
  - src: `research/working_documents/CAPTURE_SCENARIO_BEAT_HOWTO.md`
- **`CAM_SIZE=9.147` in the Godot beat capture (native 256) makes one map tile render as 21.68 px — the same as the PSX — so a Godot beat frame overlays the PSX beat frame with no per-axis rescale; camera framing at the same PC still differs (scenario director), so sprite/pose A/B compares compare unit silhouettes, not absolute screen placement.** — `[D·R] 2/3`
  - D: scn6 PC210 side-by-side compare captures (`/tmp/sxs2/CMP_pc210_*.png`) (2026-07-07)
  - D: Jacobian cross-check at native 256×256 (`content_scale_size`, NOT a 1280×960 downscale) with `CAM_SIZE=9.15`: per-tile screen Δ Godot (−19.78, 8.84) |21.67| vs PSX (−19.80, 8.84) |21.68|, foreshorten ratio sy/sx 0.447 on both — scale-matches to <0.1px/tile; the handoff's K≈3.6 "Ovelia 1.45× too far" alarm was purely a ÷5/÷4 anisotropic-squash artifact of the 1280×960→256×240 downscale (2026-07-07, `tools/capture_carry_ab.gd` Jacobian dump)
  - R: `godot-learning/tools/capture_carry_ab.gd` (`CAM_SIZE`/`RES`/`BRIGHT` env vars) (no named test)
  - src: `research/working_documents/CAPTURE_SCENARIO_BEAT_HOWTO.md`
- **The scn6 abduct-carry scenario event list is 459 instructions; the mid-scene savestate `scenario6_abduct_punch_pickup_start` parks at ≈PC193, and the `pre_events` state holds only a 73-instr loader, not the real event — so a beat's base savestate must be mid-scene, after the scenario started and before PC N.** — `[D] 1/3`
  - D: `scn_base("punch_pickup")` parsing the real 459-instr list via scenario-event-debugger (2026-07-07)
  - src: `research/working_documents/CAPTURE_SCENARIO_BEAT_HOWTO.md`
- **In the Godot reimplementation a debug park is a true global freeze-frame: `ScenarioVM._tick_time_driven_effects()` runs behind one `if not paused:` gate in `_tick_once`, so time-driven visual effects (fades, tints, ramps) no longer advance while a PC is parked.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_tick_once` / `_tick_time_driven_effects` (commit `fb2cafc8`); no named test
  - src: `research/working_documents/HANDOFF_scenario8_display_message_pc35.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Unit Anim Opcode]]
- [[Scenario Camera Opcodes]]
- [[Display Message Opcode]]
