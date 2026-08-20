# Summon Charge Lines System

Handler 18 (FUN_801b33d4) renders the summon charge line visual effect: gouraud-shaded LINE_G2 primitives that converge from a spawn radius inward toward the *target* unit (not the caster). It is routed from the charge VFX init for charging anim_type 0x0A (DAT_801b84dc[0x0A] = func_id 18) and runs on the standard process_charge_effects dispatch loop for initial_frame + 4 ticks (minimum 30). Unlike the particle-based charge handlers in [[TRAP Charge Particle System]], handler 18 draws the shared `render_charge_effect` LINE_G2 geometry (the same function as handler 4's spell lines) out of a 0x798-byte double-banked GPU buffer with a 12-halfword RGB5551+STP CLUT that fades through a 4-frame white flash. The effect centers on the target's torso (target.Y − height/2), oriented by a full 3D caster→target direction (yaw + pitch with a 90° pitch offset), and rotates around the target each frame via a frame×128 angle offset. Handler 18 is not reimplemented in godot-learning (only handler 4's spell lines, via TrapChargeLineEffect).

## Points

- **Abilities with charging anim_type 0x0A route to the summon charge lines: DAT_801b84dc[0x0A×4] (0x801b8504) holds func_id 0x12 (18), g_charge_effect_handlers[18] (0x801b8948) points to 0x801b33d4, and the process_charge_effects dispatch loop at 0x801b47e0 calls the handler once per game tick until its state 3 returns 0.** — `[S] 1/3`
  - S: 0x801b8504, 0x801b8948, 0x801b33d4, 0x801b47e0 (BATTLE.BIN disassembly), per `research/working_documents/handler_18_summon_charge_lines.md` §1
  - R: none — handler 18 / summon charge lines not present in godot-learning (`CHARGE_POSE_TO_HANDLER` in `EffectManager.gd` yields 8/15 plus 6 for Charge+X ability IDs and pose-1 lines only; probed godot-learning/src, godot-learning/tests, smd-player/addons/exmateria_sound, fft-sound-driver)
  - src: `research/working_documents/handler_18_summon_charge_lines.md`
- **Handler 18 uses the 0x54-byte charge slot with element_id at +0x02 (render param = (element−1)<<4+1), secondary_anim_id at +0x04 (indexes the duration table), initial_frame at +0x06 (clamped to a minimum of 0x1A), state at +0x08 (1=init, 2=render, 3=done), frame_counter at +0x0C, caster_unit_id at +0x12, target_unit_id at +0x1C, and the allocated GPU buffer pointer at +0x50.** — `[S] 1/3 CONTESTED`
  - S: slot usage table and state-machine addresses 0x801b33d4–0x801b381c (BATTLE.BIN disassembly), per `research/working_documents/handler_18_summon_charge_lines.md` §1–2
  - R: none — slot field usage not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_18_summon_charge_lines.md`
- **State 1 allocates a 0x798-byte (1944) GPU buffer via alloc_gpu_primitive(0x798) and stores it at slot+0x50: 24 bytes of CLUT at +0x00, then two banks of 24 primitives at stride 0x28 at +0x18 and +0x3D8, double-buffered per GPU frame via g_primitive_buffer_index; each primitive is initialized by FUN_80023cf4 and flagged with FUN_80023c68(1).** — `[S] 1/3`
  - S: 0x801b3464–0x801b34b8, FUN_80023cf4 / FUN_80023c68 (BATTLE.BIN disassembly), per `research/working_documents/handler_18_summon_charge_lines.md` §2/§6
  - R: none — LINE_G2 GPU-buffer charge lines not present in godot-learning (TrapChargeLineEffect renders with Godot's VisualServer, not a PSX GPU command buffer; probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_18_summon_charge_lines.md`
- **Handler 18 builds a 12-halfword CLUT from the RGB component tables at DAT_801b8550/801b8551/801b8552 (12 entries at 3-byte stride, 6 colors × 2 rows), packing each color as R | (G<<5) | (B<<10) | 0x8000 (PSX RGB5551 with the STP bit set), and storing row 0 at buffer+0x00 and row 1 at buffer+0x0C.** — `[S] 1/3`
  - S: packing loop 0x801b34bc–0x801b3538, tables DAT_801b8550–801b8561 (BATTLE.BIN disassembly + byte dump), per `research/working_documents/handler_18_summon_charge_lines.md` §5
  - R: none — RGB5551 CLUT packing for charge lines not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_18_summon_charge_lines.md`
- **During state 2 a frame-counter-driven fade window overwrites CLUT entries with 0xFFFF (white) while frame_counter is in [initial_frame+1, initial_frame+4) and with 0x0000 (black) after, sweeping a 4-frame white flash through the entries so the lines blink at full brightness before fading out.** — `[S] 1/3`
  - S: 0x801b3558–0x801b3600 (BATTLE.BIN disassembly), per `research/working_documents/handler_18_summon_charge_lines.md` §6
  - R: none — CLUT fade window not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_18_summon_charge_lines.md`
- **Handler 18 computes a full 3D caster→target direction: get_camera_position for caster (slot+0x12) and target (slot+0x1C), difference vectors dx/dy/dz = caster − target, yaw = ratan2(−dz, dx), pitch = ratan2(dy, sqrt(dx²+dz²)) with the sqrt from FUN_8001bf38, plus 0x400 added to pitch (90° offset in the engine's 4096-unit full circle) so the lines point upward from the target position rather than along the ground.** — `[S] 1/3`
  - S: 0x801b3658–0x801b3750, ratan2 / FUN_8001bf38 call sites (BATTLE.BIN disassembly), per `research/working_documents/handler_18_summon_charge_lines.md` §3
  - R: none — target-directed yaw/pitch charge lines not present in godot-learning (TrapChargeLineEffect is caster-anchored with no target angles; probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_18_summon_charge_lines.md`
- **Handler 18 centers the effect at the target's torso instead of the caster's head: FUN_8008dc74(target_unit_id) at 0x801b36e8 returns the sprite height, divided by 2 with the signed srl/addu/sra idiom, and the effect renders at target.Y − height/2, versus handler 4 (spell lines, 0x801b1c04) which renders at caster.Y − (height + 8) above the caster's head.** — `[S] 1/3 CONTESTED`
  - S: 0x801b36e4–0x801b3714, FUN_8008dc74 at 0x801b36e8, handler 4 `addiu s2, v0, 0x8` (BATTLE.BIN disassembly), per `research/working_documents/handler_18_summon_charge_lines.md` §4/§9
  - R: none — target.Y − height/2 not present in godot-learning (only handler 4's height+8 is mirrored by HEIGHT_OVERSHOOT = 8.0 in `TrapChargeLineEffect.gd`; probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_18_summon_charge_lines.md`
- **Handler 18 calls the shared render_charge_effect (the same function as handler 4) to draw the LINE_G2 charge lines, passing a fixed scale (320, 320, 320), roll = 0, the angle pair (0, frame_counter×128) so the lines rotate around the target each frame, and a duration parameter from g_duration_param_table (0x801b84de) indexed by secondary_anim_id × 4.** — `[S] 1/3`
  - S: 0x801b3758–0x801b37b0, g_duration_param_table at 0x801b84de (BATTLE.BIN disassembly), per `research/working_documents/handler_18_summon_charge_lines.md` §7
  - R: none — handler-18 render parameters (frame×128 rotation, scale 320, duration-table lookup) not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_18_summon_charge_lines.md`
- **The effect runs for initial_frame + 4 ticks total (minimum 30 frames, since initial_frame is clamped to ≥ 0x1A): state 2 increments frame_counter and moves to state 3 when it exceeds the threshold, and state 3 (0x801b37f4) returns 0 to remove the handler from the dispatch loop at 0x801b47e0.** — `[S] 1/3`
  - S: 0x801b37c0–0x801b37f4 (BATTLE.BIN disassembly), per `research/working_documents/handler_18_summon_charge_lines.md` §8
  - R: none — handler 18 termination not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_18_summon_charge_lines.md`

## Notes

(empty — user territory)

## Related

- [[Spell Charge Effect System]]
- [[Spell Charge Lines System]]
- [[TRAP Charge Particle System]]
- [[TRAP Sprite Effect System]]
- [[Unit Sprite Height Table]]
- [[PSX GPU Primitives]]
- [[Effect System Index]]
