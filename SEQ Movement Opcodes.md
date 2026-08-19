# SEQ Movement Opcodes

FFT's SEQ animation system moves unit sprites by two mechanisms: small per-frame pixel nudges via the MoveUnit opcodes (no-parameter ±1/±2 increments 0xFFCB–0xFFD2, and signed-byte parameterized 0xFFEE/0xFFEF/0xFFF0/0xFFFA) and larger cross-tile translations via the distort system (0xFFC1 QueueDistortAnim / 0xFFC0 WaitForDistort, used exclusively by Dash's Rush Front/Back). The godot-learning port implements both — `AnimationPlayback.gd` accumulates pixel offsets (emitting `move_offset_changed`) and `DistortMovementController.gd` lerps a visual offset over the frame count — validated by `tests/GPUSEQMovementTest.gd` and `tests/GPUDashMovementTest.gd`. A full TYPE1.SEQ scan maps which of the 252 animation slots carry movement opcodes: shield blocks, melee wind-ups, shield recoils, evade dodges, float bobs, and the death bounce.

## Points

- **The fixed-increment movement opcodes (no parameters) shift the sprite by 1–2 px along a fixed axis: 0xFFD2 MoveForward1 / 0xFFD1 MoveForward2 (+1/+2 px along facing), 0xFFCE MoveBackward1 / 0xFFCD MoveBackward2 (−1/−2 px along facing), 0xFFCC MoveUp1 / 0xFFCB MoveUp2 (+1/+2 px up), 0xFFD0 MoveDown1 / 0xFFCF MoveDown2 (−1/−2 px down).** — `[S·R] 2/3`
  - S: opcode-table codes 0xFFCB–0xFFD2 (`godot-learning/tools/opcodeParameters.txt`: MoveUp2,0,ffcb … MoveForward1,0,ffd2), per `research/working_documents/SEQ_MOVEMENT_OPCODES.md` §1
  - R: `godot-learning/src/animation/AnimationPlayback.gd` (`_FIXED_MOVE_OFFSETS` dict, applied in `_process_side_effects`) + `godot-learning/src/animation/AnimationOpcodes.gd` (name → op map) — validated by `godot-learning/tests/GPUSEQMovementTest.gd`
  - src: `research/working_documents/SEQ_MOVEMENT_OPCODES.md`
- **The parameterized movement opcodes take signed-byte (s8) parameters: 0xFFEE MoveUnitFB (1 s8, N px forward/back), 0xFFEF MoveUnitDU (1 s8, N px down/up), 0xFFF0 MoveUnitRL (1 s8, N px right/left), 0xFFFA MoveUnitRLDUFB (3 s8, axis order RL, DU, FB); 0xFFC0 WaitForDistort also decodes its parameter as s8.** — `[S·R] 2/3`
  - S: opcode-table entries MoveUnitFB,1,ffee / MoveUnitDU,1,ffef / MoveUnitRL,1,fff0 / MoveUnitRLDUFB,3,fffa / WaitForDistort,1,ffc0 (`godot-learning/tools/opcodeParameters.txt`), per `research/working_documents/SEQ_MOVEMENT_OPCODES.md` §1–2
  - R: `godot-learning/src/animation/AnimationPlayback.gd` (MOVE_UNIT_FB/DU/RL/RLDU_FB branches decode parameters via `_to_signed_byte`) — validated by `godot-learning/tests/GPUSEQMovementTest.gd` (move_offset_changed fires during melee attack animation, slots 130/131)
  - src: `research/working_documents/SEQ_MOVEMENT_OPCODES.md`
- **0xFFC0 WaitForDistort(N) waits to execute the next command when no distort animation was queued by 0xFFC1 QueueDistortAnim, and loops the last N animation bytes (N signed) while a distort animation is underway; QueueDistortAnim(id, var) queues a secondary cross-tile translation with id ∈ {2 = slide/charge to target, 3 = slide back to home, 13 = wind-up step backward} and var = duration in frames.** — `[S·R] 2/3`
  - S: opcodes 0xFFC0/0xFFC1, per `research/working_documents/SEQ_MOVEMENT_OPCODES.md` §2 (ref: ffhacktics Animate_Unit_Distorts)
  - R: `godot-learning/src/animation/AnimationPlayback.gd` (QUEUE_DISTORT_ANIM → `distort_requested(type, frame_count)`; WAIT_FOR_DISTORT stops processing while the controller is active) + `godot-learning/src/animation/DistortMovementController.gd` (type 2/3/13 target offsets, lerp over the frame count; type meanings per FFT wiki) — validated by `godot-learning/tests/GPUDashMovementTest.gd`
  - src: `research/working_documents/SEQ_MOVEMENT_OPCODES.md`
- **The Dash ability's Rush Front (TYPE1 slot 212) and Rush Back (slot 213) animations use the distort system exclusively — no MoveUnit opcodes — in three phases each: QueueDistortAnim(13,10) + WaitForDistort(-2) + LoadFrameAndWait(frame 92 front / 141 back, delay 2); QueueDistortAnim(2,6) + WaitForDistort(-2) + LoadFrameAndWait(100/149, 2); QueueDistortAnim(3,8) + WaitForDistort(-2) + LoadFrameAndWait(100/149, 2); ending in PauseAnimation.** — `[S·R] 2/3`
  - S: TYPE1.SEQ slots 212/213 opcode sequences, per `research/working_documents/SEQ_MOVEMENT_OPCODES.md` §2–4
  - R: `godot-learning/src/animation/DistortMovementController.gd` (step-back 0.3× / charge 0.9× distance factors) + `godot-learning/src/units/Unit.gd` (`_on_distort_requested` wiring) — validated by `godot-learning/tests/GPUDashMovementTest.gd` (Dash ability 147: step back, charge to target, return home; target takes damage)
  - src: `research/working_documents/SEQ_MOVEMENT_OPCODES.md`
- **The full TYPE1.SEQ movement scan (252 slots): Shield Block slots 176–181 use a single MoveBackward2 (pushed back with no return); melee wind-up is MoveBackward2 → MoveForward2, with high attacks adding a second MoveForward2 + MoveBackward1 (net +1 px forward), low attacks adding MoveForward1 (net +1 px), and mid attacks net 0; Evade Defend slots 48/49 side-step with 4× MoveUnitRL(-2) then 4× MoveUnitRL(2); Float Stop slots 18–23 hover-bob with ±1 px vertical ops; DeathBounce slots 104/105 open with MoveUnitDU(-8, -4), fall +4/+6, and settle on fixed MoveDown2/MoveUp2 pairs; Dance Execute 222/223 is a 20-op wiggle where only the front version adds a MoveUp1 hop; Confused (74/75) and Dancing (82/83) idle share the same ±2 px stumble pattern.** — `[S] 1/3`
  - S: TYPE1.SEQ slot-scan tables (slots 18–251), per `research/working_documents/SEQ_MOVEMENT_OPCODES.md` §3–4
  - R: none — per-slot movement data is ROM SEQ data, not asserted in godot-learning code (probed `godot-learning/src/` and `godot-learning/tests/`; the opcodes are executed generically by `AnimationPlayback.gd` from parsed SEQ data)
  - src: `research/working_documents/SEQ_MOVEMENT_OPCODES.md`

## Notes

(empty — user territory)

## Related

- [[Unit Sprite SEQ Opcodes]]
- [[Unit Sprite Render Pipeline]]
- [[Weapon Animation System]]
- [[Unit Sprite Index]]
