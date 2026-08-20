# Reveal Opcode

Event opcode `{4D}` Reveal fades the screen in from black via the BATTLE.BIN screen-fade subsystem: the opcode handler stores the `Time` param into `DAT_8016d9bc`, a per-frame consumer arms applier `FUN_8008c364` (mode `0x3a`, step `0x100/Time`), so the fade lasts exactly `Time` frames — no scale factor. The 2026-06-30 Orbonne-prayer trace (disasm + reader-independent live polls) pinned the prayer's ~17.5 s black: ~16 s of it is pre-event-script (chunk `0x8004a6bc` silent until t≈21 s — banner teardown / scenario-load latency, not authored bytecode), and the remainder is the 128-frame Reveal that arms 0.7 s into the chunk, mid camera-fusion descent. Godot's 128-tick `fade_rect` reveal reproduces the within-chunk part faithfully (duration and start phase); the pre-chunk black phase has no Godot counterpart.

## Points

- **The `{4D}` Reveal opcode handler performs no inline fading — it only stores the Time param into global `DAT_8016d9bc`: the `s4==0x4D` dispatch branch is gated at `0x80143db0`, the handler body at `0x80143db4` is a single `sh s2,-0x2644(at)` at `0x80143dbc` (s2 = param & 0xffff) followed by a jump to the shared per-instruction tail `LAB_80144b40`.** — `[S·D·R] 3/3`
  - S: `0x80143db0` / `0x80143db4` / `0x80143dbc`, `battle_disassembly.txt` (2026-06-26 export, post BATTLE.BIN base-fix)
  - D: `probe_prayer_fade_arm_timeline.py` Exec BP on the writer PC `0x80143dbc` fired exactly once, at t=21.79 s (2026-06-30, `last_run/probe_prayer_fade_arm_timeline.jsonl`)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_reveal` → `ScenarioApply.reveal` → `ScenarioWorld.arm_reveal` (arms `_reveal_duration_ticks = Time`, floored at 1), validated by `godot-learning/tests/ScenarioApplyTest.gd` `_test_reveal_floors_time`
  - src: `research/working_documents/scenario_1_captures/map_darkness_prayer_hold_decode.md`
- **The Reveal fade duration is exactly `Time` frames, unscaled: the per-frame consumer at `0x80143468` reads `DAT_8016d9bc` and, when nonzero, calls applier `FUN_8008c364` (@`0x8008c364`), which sets fade mode `0x3a` (previous mode saved to `DAT_800960e8`) and the per-frame step `0x100/Time` into `DAT_8009610c` (store @`0x8008c374`) — a 256-unit alpha ramp therefore completes in `Time` frames (Time=128 ⇒ step 2 ⇒ 128 frames ≈ 2.1 s); a `Time<<2`/`Time×8` scale factor does not exist.** — `[S·D·R] 3/3`
  - S: consumer `0x80143468`, applier `FUN_8008c364` @`0x8008c364` (step store @`0x8008c374`), `battle_disassembly.txt` (2026-06-26 export)
  - D: `probe_prayer_opcode_frames.py` + `probe_prayer_fade_state_poll.py` (2026-06-30, doc §10/§9.2): the prayer's 256-unit level ramped 255→0 in 128 frames (t=21.80→23.88 s), "confirmed in disasm AND live"
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_reveal` uses the raw `Time` as tick count (`_reveal_duration_ticks`), matching the PSX 1:1 — headful ScenarioPlayer run confirmed duration + start phase faithful (doc §9.3), unit-validated by `godot-learning/tests/ScenarioApplyTest.gd` `_test_reveal_floors_time`
  - src: `research/working_documents/scenario_1_captures/map_darkness_prayer_hold_decode.md`
- **The BATTLE.BIN screen-fade subsystem holds its state in fixed globals: `DAT_800960e4` = fade mode (`0x3a` = fade-in/reveal running, `0x3b` = fade-out running, `0x34` = steady/hold, `0x36` = done), `DAT_800961b0` = fade level u16 0..0x100 with HIGH = black and LOW = clear (level 255 ⟺ luma 0, level 0 ⟺ map visible), `DAT_8009610c` = per-frame step, and `DAT_80098d8c`+[0..2] = applied grayscale (R=G=B=clamp(level,0xFF)) that the display compositor subtracts/multiplies onto the frame.** — `[S·D] 2/3`
  - S: globals decoded from disasm, `battle_disassembly.txt` (doc §4 + §9.1, 2026-06-30)
  - D: `probe_prayer_fade_state_poll.py` reader-independent per-frame poll (2026-06-30, `last_run/probe_prayer_fade_state_poll.jsonl`) confirmed all four live and cross-checked against framebuffer mean luminance
  - R: none — the screen-fade state machine (mode/level/step globals) not present in godot-learning (probed `godot-learning/src/` + `tests/`; Godot approximates the fade with a black `fade_rect` ColorRect alpha ramp in `ScenarioPlayerScene.gd` / `ScenarioVM._apply_fade_alpha`)
  - src: `research/working_documents/scenario_1_captures/map_darkness_prayer_hold_decode.md`
- **The opposite-direction (to-black) applier `FUN_8008c3a0` @`0x8008c3a0` (fade mode `0x3b`, plus `DAT_800a778c = 1`) has exactly one call site — opcode handler `0x80144330`, which passes a hardcoded `Time = 0x20` (32 frames ≈ 0.5 s) inside a scene-transition block — and it fired zero times during the 33 s prayer capture, so the t≈5 s banner→black edge is not produced by this scripted to-black fade.** — `[S·D] 2/3`
  - S: `FUN_8008c3a0` @`0x8008c3a0` (mode write @`0x8008c3b4`), sole caller `0x80144330`, `battle_disassembly.txt` (2026-06-26 export)
  - D: `probe_prayer_fade_arm_timeline.py` Exec BP on `0x8008c3b4`: 0 fires in the whole 33 s window (2026-06-30)
  - R: none — no to-black reveal path in godot-learning (probed `godot-learning/src/` + `tests/`; only the fade-in `_op_reveal` exists)
  - src: `research/working_documents/scenario_1_captures/map_darkness_prayer_hold_decode.md`
- **The bulk of the Orbonne prayer black hold is pre-event-script: the prayer event chunk (RAM slot `0x8004a6bc`) dispatches its first opcode only at t≈21 s and the event dispatcher read (`0x80143d34`) is silent from t≈5 s to t≈21 s, so the ~16 s of black (t≈5→21 s) is banner teardown / scenario-load latency, not authored event bytecode — the `{4D}` reveal accounts for only the last ~2 s.** — `[S·D] 2/3`
  - S: dispatcher read `0x80143d34`, chunk slot `0x8004a6bc` (`battle_disassembly.txt`; live-re-verified anchors, doc header)
  - D: `probe_prayer_opcode_timeline.py` Exec BP on the dispatch read (2026-06-30, `last_run/probe_prayer_opcode_timeline.jsonl`) + luma ground truth `probe_prayer_darkness_timeline.jsonl` (screen black t=5.08→22.55 s)
  - R: none — pre-chunk black hold not present in godot-learning (probed `godot-learning/src/` + `tests/`; ScenarioPlayer plays the chunk immediately on entry, so the ~16 s load-black never happens)
  - src: `research/working_documents/scenario_1_captures/map_darkness_prayer_hold_decode.md`
- **The prayer's darkness during the camera move is the `{4D}` Reveal fade-from-black: the camera-fusion descent begins at t=21.63 s with the screen still full black, the Reveal arms 0.7 s into the chunk (t=21.80 s, simultaneously with `{1D}` CamFusionStart) and ramps the level 255→0 over exactly 128 frames (~2.1 s) through the first ~22 % of the ~9 s descent; after the overlay clears, the continued brightening (luma 3→22, t=24→28 s) is camera geometry — the high-angle frame filling with lit map as it drops — not the fade.** — `[S·D·R] 3/3`
  - S: Reveal write `0x80143dbc`, chunk slot `0x8004a6bc`, `battle_disassembly.txt`
  - D: `probe_prayer_fade_state_poll.py` (2026-06-30): mode `0x34`→`0x3a` at t=21.80 s, level 255→0 complete at t=23.88 s, camera still descending at window end; `probe_prayer_opcode_timeline.py`: `{4D}` at t=21.82 s co-incident with `{1D}`; luma `probe_prayer_darkness_timeline.jsonl`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_reveal` → `ScenarioWorld.arm_reveal` + `ScenarioPlayerScene.gd` `fade_rect` (black ColorRect) reproduce the 128-tick reveal-from-black at chunk entry — headful ScenarioPlayer run confirmed duration + start phase faithful (doc §9.3), unit-validated by `godot-learning/tests/ScenarioApplyTest.gd` `_test_reveal_floors_time`
  - src: `research/working_documents/scenario_1_captures/map_darkness_prayer_hold_decode.md`

## Notes

(empty — user territory)

## Related

- [[Map Darkness Opcode]]
- [[Camera Fusion Chain]]
- [[DarkScreen Opcode]]
- [[Event Opcode Catalog]]
- [[Effect Frame Pacing]]
- [[Screen Effect Gradient System]]
