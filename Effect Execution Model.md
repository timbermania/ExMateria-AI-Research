# Effect Execution Model

FFT's runtime execution architecture for E###.BIN visual effects: a per-frame main loop drives a 16-bit opcode script interpreter whose words pack a 9-bit opcode (bits 0–8) and a 7-bit flags field (bits 9–15) and which dispatches through the jump table at 0x801B67C8, which wires 46 of the nominal 512 nine-bit opcode slots, with two dominant execution patterns — a two-phase timeline-driven animation (opcode 41, op_process_timeline_frame; 64-byte two-entry script, root at 0x00 plus per-target child at the offset passed to opcode 41, e.g. 0x24) and a simple single-phase tick (opcode 40, op_animate_tick; single 36-byte entry-0x00 script) — all state held in 248-byte EffectState instances in a fixed array. All Pattern 1 phase logic lives inside opcode 41 (0x801A4838): it runs the phase-1 channel region (0x82A) while frame_counter < phase1_end (timeline[0x04]), spawns one per-target child spaced spawn_delay (timeline[0x06]) apart, and switches to the phase-2 region (0xAAA) at phase2_start = phase1_end + (target_count − 1) × spawn_delay + phase2_delay, with target_count = effect_context_value (0x801BAD0C). The main loop (effect_system_main_loop 0x801A18D8) is itself a 7-state state machine whose state 2 performs the one-time texture/CLUT VRAM upload (FUN_801a0e80) before state 3 initializes globals and state 4 runs the per-frame execution. Emitters are fired by timeline-channel keyframes in Pattern 1 (not by script opcodes directly), feeding the shared particle pipeline (emitter_control_routine / update_all_particles); when a keyframe's action_flags bits 0–2 name a callback slot (1–7) instead, the channel invokes the MIPS callback registered into EffectState (callback_state at 0x22, callback_ptrs at 0xD4) via JALR at 0x801A4134. A 2026-04-16 working document adds a raw timer/camera-section model (12-byte global header + 5×0x80 emitter-timing blocks) and a partial byte-oriented script-opcode table, recorded below as low-evidence points.

## Points

- **The per-frame effect main loop (effect_system_main_loop at 0x801A18D8) iterates the active-effect linked list, executes each effect's script via effect_script_dispatcher (0x801A4CF0) until the script yields (return 0), continues (return 1), or aborts (return 2), then increments frame_counter, which wraps at 160.** — `[S] 1/3`
  - S: effect_system_main_loop 0x801A18D8, effect_script_dispatcher 0x801A4CF0, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Effect scripts are 16-bit opcodes dispatched through a jump table at 0x801B67C8: the interpreter reads the opcode at script_data_ptr[script_position], masks it with 0x1FF to select a handler index 0–511, and the handler advances script_position by 2, 4, or 6 bytes.** — `[S] 1/3 CONTESTED`
  - S: jump table 0x801B67C8 and 0x1FF opcode mask, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - S: same 0x1FF opcode mask, per `research/key_documents/SCRIPT_EDITOR_LESSONS.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **The effect-script dispatch table at 0x801B67C8 wires only 46 of the nominal 512 nine-bit opcode slots (handler indices 46–511 are out-of-bounds and would read garbage — a malformed-file path, not a real opcode); all 46 dispatcher opcodes are documented, and the standard vanilla scripts use only 13 of the 46.** — `[S·R] 2/3`
  - S: dispatcher 0x801A4CF0, 0x1FF mask, 46-entry jumptable at 0x801B67C8, per `research/key_documents/EFFECT_EXECUTION_MODEL.md` (lines 13–23, cited by MASTER_PARSER_GAPS.md); 13-of-46 usage, per `research/wiki_articles/effect_script_bytecode.txt` lines 498–499; 6-effect sample sweep produced no `unknown_{opcode}` output
  - R: `effect-editor/core/parser.lua` `M.SCRIPT_OPCODES` (46 entries, same 0x1FF opcode / 7-bit-flags word split; no automated test)
  - src: `research/working_documents/MASTER_PARSER_GAPS.md`
- **Pattern 1 (complex two-phase animation) is handled by op_process_timeline_frame (opcode 41, 0x801A4838) and is used by 152 E###.BIN files, including E001 Cure and E019 Fire 4.** — `[S] 1/3`
  - S: op_process_timeline_frame 0x801A4838, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Pattern 1's ROOT effect progresses through four stages: phase 1 (timeline triggers emitters via keyframes), child spawning (per-target children with configurable delay), phase 2 (resolution animations), and winddown (waits for active particles to die).** — `[S] 1/3`
  - S: per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Pattern 1 child effects are spawned per target during the phase transition, start execution at script offset 0x24, and run op_animate_tick (opcode 40) autonomously rather than op_process_timeline_frame.** — `[S] 1/3`
  - S: child script start offset 0x24, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Pattern 1's timeline carries 5 particle, 4 color, and 3 sound channels plus a camera track.** — `[S] 1/3`
  - S: per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Pattern 2 (simple single-phase animation) is handled by op_animate_tick (opcode 40, 0x801A3408), is used by 136 files, tracks 5 particle + 3 sound + 4 color channels, and tracks progress via the anim_progress counter.** — `[S] 1/3`
  - S: op_animate_tick 0x801A3408, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **EffectState's common fields sit at fixed offsets — next_effect_index 0x00, script_position 0x06, script_data_ptr 0x08, child_effect_indices[4] 0x0C, active_particle_count 0x1C, frame_counter 0x20, particle_list_head 0xD0 — with Pattern 1 fields occupying 0x28–0xCF and Pattern 2 fields 0x28–0x67.** — `[S] 1/3`
  - S: per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Tracked child effects are spawned by opcode 2 (op_spawn_child_effect) into up to 4 child_effect_indices slots and referenced by opcodes 3, 26, and 27, whereas fire-and-forget children spawned by process_timeline_frame during the phase transition are unlimited and allocated from the global EffectState pool.** — `[S] 1/3`
  - S: opcodes 2/3/26/27 and child_effect_indices, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **In Pattern 1, emitters are not started directly by script opcodes: op_process_timeline_frame reads timeline channel data each frame, the channels specify when each emitter spawns particles via keyframes, and emitters fire via emitter_control_routine (0x801A634C).** — `[S] 1/3`
  - S: emitter_control_routine 0x801A634C, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **The timeline section (at header offset 0x1C) holds a header (phase1_duration, spawn_delay, phase2_offset), 5 particle channels per phase of 128 bytes each, and color/sound/camera tracks.** — `[S] 1/3`
  - S: timeline section at header[0x1C], per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - S: channel composition (5 particle + 4 color + 3 sound per phase; camera as 3 tables × 3 tracks), per `research/key_documents/EFFECT_FILE_FORMAT.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Opcode 38 (op_spawn_emitter) calls emitter_control_routine (0x801A634C) to read an emitter template from the Particle System section, interpolate its parameters using animation curves, and spawn particles into particle_list_head.** — `[S] 1/3`
  - S: emitter_control_routine 0x801A634C, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Opcode 37 (update_all_particles, 0x801A2EB4) advances all particles each frame: it calls integrate_particle_motion (0x801A9BB0) for physics, processes lifetime countdown / animation-driven death, and spawns child emitters on death or mid-life.** — `[S] 1/3`
  - S: update_all_particles 0x801A2EB4, integrate_particle_motion 0x801A9BB0, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **The effect subsystem's global pointers are 0x801BBF78 (sprite_def_table_ptr), 0x801BBF7C (effect_anim_tbl_ptr), 0x801BBF84 (timeline_channel_base), 0x801BBF88 (effect_data_ptr), 0x801BBF8C (animation_table_ptr), 0x801BC0C8 (timeline_section_ptr), 0x801BACC8 (effect_flags_ptr), and 0x801B9258 (time_scale_ptr).** — `[S] 1/3 CONTESTED`
  - S: global pointer addresses 0x801BBF78–0x801B9258, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - S: 0x801BC0C8 (timeline_section_ptr), per `research/key_documents/LUA_DEBUGGING.md`
  - S: same pointer set with header-field provenance, per `research/key_documents/EFFECT_FILE_FORMAT.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **EffectState slot 1's timeline_frame_counter sits at 0x801BF14C (offset 0x28 into the slot, whose base is 0x801BF124).** — `[S] 1/3`
  - S: 0x801BF14C (slot 1 timeline_frame_counter), per `research/key_documents/LUA_DEBUGGING.md`
  - src: `research/key_documents/LUA_DEBUGGING.md`
- **Four further effect-subsystem globals are mapped: 0x801BBF74 (g_sound_section_ptr, feds), 0x801BC0DC (g_sound_data_base), 0x801BAD0C (g_effect_context), and 0x801B48D0 (effect_base_lookup_table).** — `[S] 1/3`
  - S: addresses 0x801BBF74/0x801BC0DC/0x801BAD0C/0x801B48D0, per `research/key_documents/LUA_DEBUGGING.md`
  - src: `research/key_documents/LUA_DEBUGGING.md`
- **The upper 7 bits (bits 9–15) of an effect-script 16-bit word form a flags field: flags = (word >> 9) & 0x7F, complementing the 9-bit opcode in bits 0–8.** — `[S] 1/3`
  - S: word layout [FLAGS:7 bits][OPCODE:9 bits], per `research/key_documents/SCRIPT_EDITOR_LESSONS.md`
  - src: `research/key_documents/SCRIPT_EDITOR_LESSONS.md`
- **Effect-script instruction sizes are fixed per opcode — 2 bytes (no args): opcodes 3,4,5,9,10,12,13,15,32,33,36,37,38,39,40,42,43,44,45; 4 bytes (1 arg): opcodes 0,1,2,6,7,16,26,27,29,30,31,34,35,41; 6 bytes (2 args): opcodes 17,18,19,20,21,22,23,24,25,28; 8 bytes (3 args): opcodes 8,11,14 — each instruction being a 16-bit word followed by (size−2)/2 16-bit args.** — `[S] 1/3 CONTESTED`
  - S: per-opcode instruction size table, per `research/key_documents/SCRIPT_EDITOR_LESSONS.md`
  - src: `research/key_documents/SCRIPT_EDITOR_LESSONS.md`
- **EffectState's timeline header (0x22–0x29, common to Patterns 1 and 2) holds callback_state[4] at 0x22 (set to 3 when the callback is 2 frames from ending), effect_target_index at 0x26, and the timeline frame counter — timeline_frame_counter in Pattern 1, anim_progress in Pattern 2 — at 0x28.** — `[S] 1/3`
  - S: EffectState timeline header offsets 0x22–0x29, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **EffectState's timeline footer (0xD0–0xF7, common to all variants) holds particle_list_head at 0xD0, the four timeline callback function pointers (callback_ptrs[4]) at 0xD4, sprite_ptrs[4] at 0xE4, and current_callback_ptr at 0xF4.** — `[S] 1/3`
  - S: EffectState timeline footer offsets 0xD0–0xF7, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **EffectState_P2 (Pattern 2, opcode 40) keeps its channel state at 0x2A–0x5E: particle_keyframe[5] 0x2A, sound_keyframe[3] 0x34, color_keyframe[4] 0x3A, particle_duration[5] 0x44, sound_duration[3] 0x4E, color_duration[4] 0x54, and particle_spawn_counter[5] 0x5E.** — `[S] 1/3`
  - S: EffectState_P2 layout offsets 0x2A–0x5E, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **EffectState_P1 (Pattern 1, opcode 41) keeps its channel state at 0x2A–0xA0: child_spawn_delay 0x2A, spawned_target_count 0x2C (fire-and-forget children are not limited to 4), phase1/phase2 particle keyframes at 0x2E/0x38, color track keyframes (palette, caster, target, screen, tracks 4–7) at 0x42–0x60, phase1/phase2 particle durations at 0x62/0x6C, color track durations at 0x76–0x92, and phase1/phase2 particle spawn counters at 0x96/0xA0.** — `[S] 1/3`
  - S: EffectState_P1 layout offsets 0x2A–0xA0, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **EffectState_P1's six int16 sound fields (phase1/phase2 sound4, sound5, sound6) at 0xBB–0xC5 are read and passed to the particle-spawn routine but ignored there — dead code.** — `[S] 1/3`
  - S: 0xBB–0xC5 (s0+0x95–0x9F with s0 = effect_state+0x26), per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **effect_cleanup lives at 0x801A1D9C with signature `void effect_cleanup(int16_t effect_index)`.** — `[S] 1/3`
  - S: effect_cleanup 0x801A1D9C, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Three more effect-system globals are mapped: active_effect_list_head 0x801BBF90 (head of the active-effect linked list), free_effect_list_head 0x801B9158 (head of the free effect pool), and effect_system_state 0x801B63E8 (current state-machine state).** — `[S] 1/3`
  - S: global addresses 0x801BBF90/0x801B9158/0x801B63E8, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Pattern 1's timeline channels each advance through a dedicated processor function: advance_affected_units_palette_track 0x801A41A0, advance_caster_palette_track 0x801A436C (takes an extra unit_id), advance_target_palette_track 0x801A444C, advance_screen_color_track 0x801A45C8, and advance_p1_sound_track 0x801A478C, each taking (track_data, keyframe_state, duration_state).** — `[S] 1/3`
  - S: track-processor addresses 0x801A41A0–0x801A478C, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - S: 0x801A45C8 (advance_screen_color_track, screen track processing), per `research/wiki_articles/screen_effect_gradient_system.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **effect_system_main_loop (0x801a18d8) is a 7-state state machine (effect_system_state): 0 idle/cleanup (0x801a1bb8), 1 waiting for file load (0x801a195c), 2 texture upload (0x801a1920, calls FUN_801a0e80 at 0x801a1938), 3 initialize globals from header (0x801a1964), 4 main effect execution loop (0x801a1ab0), 5 exit/return (0x801a1c20), 6 cleanup (0x801a1bc0).** — `[S] 1/3`
  - S: state-machine case addresses 0x801a1bb8–0x801a1bc0 and texture upload call at 0x801a1938, per `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - src: `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
- **Pattern 1's phase boundaries are computed from timeline header words: phase1_end = timeline[0x04], spawn_delay = timeline[0x06], phase2_delay = timeline[0x0A]; with target_count = effect_context_value (0x801BAD0C), phase2_start = phase1_end + (target_count − 1) × spawn_delay + phase2_delay (e.g. 96 + (4−1)×10 + 8 = 134 for Fire 4 on 4 enemies).** — `[S·D·R] 3/3`
  - S: timeline header words 0x04/0x06/0x0A and effect_context_value 0x801BAD0C, per `research/working_documents/EFFECT_PATTERNS_AND_PHASES.md`
  - D: trace_phases.lua probe, sample output in doc (EFFECT START phase1_end=96, phase2_start=134, targets=4; doc last updated 2024-12-01)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound_controller.gd` `_compute_phase2_start()` (identical formula) + `godot-learning/tests/EffectSoundCaptureTest.gd`; also `effect-editor/commands/workflow.lua` (reads the header at +4/+6/+10, live audio-capture workflow, no automated test)
  - src: `research/working_documents/EFFECT_PATTERNS_AND_PHASES.md`
- **In Pattern 1 all phase logic lives inside op_process_timeline_frame (0x801A4838), not in the script bytecode: the root script merely loops opcode 41, and the opcode runs the phase-1 channel region while frame_counter < phase1_end, spawns one child per target during the spawn window (children spaced spawn_delay frames apart), and switches to the phase-2 channel region once frame_counter ≥ phase2_start.** — `[S·D·R] 3/3`
  - S: op_process_timeline_frame 0x801A4838, per `research/working_documents/EFFECT_PATTERNS_AND_PHASES.md`
  - D: trace_phases.lua probe, sample output in doc (PHASE 1 ENDED frame 96, CHILD SPAWNED 1/4–4/4, PHASE 2 STARTED frame 134; doc last updated 2024-12-01)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound_controller.gd` (`_tick` gates the phase-1/phase-2 channel blocks on the frame counter; `_try_spawn_for_each` schedules child n at `phase1_duration + n × spawn_delay`) + `godot-learning/tests/EffectSoundCaptureTest.gd`; `godot-learning/src/effects/EffectPhase.gd` `open_phases()` + `godot-learning/tests/EffectTimelineTest.gd`
  - src: `research/working_documents/EFFECT_PATTERNS_AND_PHASES.md`
- **Pattern 1's timeline keeps parallel copies of every track for the two phases at fixed offsets from the timeline section base: 5 particle channels 0x82A/0xAAA, unit palette 0xDDE/0x1162, caster palette 0xEA6/0x122A, target palette 0xF6E/0x12F2, screen effect 0x1036/0x13BA, and 3 sound tracks 0xD2A+/0xD84+.** — `[S·R] 2/3`
  - S: phase-1/phase-2 channel offsets 0x82A/0xAAA/0xDDE/0x1162/0xEA6/0x122A/0xF6E/0x12F2/0x1036/0x13BA/0xD2A/0xD84, per `research/working_documents/EFFECT_PATTERNS_AND_PHASES.md`
  - R: `effect-editor/core/parser.lua` `M.COLOR_TRACK_OFFSETS` (phase-1 0xDDE/0xEA6/0xF6E/0x1036, phase-2 0x1162/0x122A/0x12F2/0x13BA; no automated test)
  - src: `research/working_documents/EFFECT_PATTERNS_AND_PHASES.md`
- **Pattern 1's effect script is a 64-byte region with two entries — 0x00 the root (INIT: set_texture_page, init_physics_params; MAIN LOOP: process_timeline_frame, update_all_particles, yield) and 0x24 the per-target child (animate_tick loop) — with the child entry offset passed as opcode 41's argument (0x24 in E019.BIN Fire 4); Pattern 2's script is a single 36-byte entry-0x00 animate_tick loop.** — `[S·R] 2/3`
  - S: E019.BIN (Fire 4) script bytecode, entries 0x00/0x24, per `research/working_documents/EFFECT_PATTERNS_AND_PHASES.md`
  - R: `effect-editor/ui/script_tab.lua` (entry 0x00 root / 0x24 per-target, opcode 41 child-entry param) + `effect-editor/core/parser.lua` (opcode 41 param, opcode 2 for-each entry-offset param; no automated test)
  - src: `research/working_documents/EFFECT_PATTERNS_AND_PHASES.md`
- **Tracked child effects are choreographed with opcodes 26/27: opcode 26 (branch_child_active) and 27 (branch_child_inactive) branch on whether the slot's child effect is still active, so the parent script can wait on a staged child before proceeding (e.g. a summon waits on its magic-circle child before spawning the creature child).** — `[S·R] 2/3`
  - S: opcodes 26/27 (branch_child_active/branch_child_inactive), per `research/working_documents/EFFECT_PATTERNS_AND_PHASES.md`
  - R: `effect-editor/core/parser.lua` opcode table (26 branch_for_each_active, 27 branch_for_each_inactive with branch-target offset params; 2 spawn_for_each_instance, 3 terminate_for_each; no automated test)
  - src: `research/working_documents/EFFECT_PATTERNS_AND_PHASES.md`

- **Each particle timeline channel is 128 bytes holding 25 keyframes; a keyframe is (time, emitter_id, action_flags) where action_flags packs callback_slot, use_global_target, trigger_hit_reaction, refresh_tile_state, apply_ability_reaction and an action_param, and each channel ends with a max_keyframe count.** — `[S·R] 2/3`
  - S: particle-channel keyframe composition (15 channels × 128 B for 3-phase effects), per `research/key_documents/master_parser.py` (byte-reconciliation source of the inventory)
  - S: keyframe action_flags bit ranges — bits 0–2 = callback_slot (0 = direct spawn, 1–7 = callback slot), bits 8–15 = action_param, action_flags stored at keyframe offset 0x4A, per `research/working_documents/MIPS_CALLBACK_SYSTEM.md`
  - R: `godot-learning/src/effects/TimelineData.gd` (Keyframe time/emitter_id/action_flags, Channel.max_keyframe) + `godot-learning/src/effects/PhaseBlock.gd` (emitter_id 0 = skip / N = spawn emitter N−1; action_flags_triggered signal; no automated test)
  - src: `research/working_documents/E_BIN_FIELD_EDITABILITY_INVENTORY.md`

- **The 2026-04-16 working document defines the raw layout of the timer/camera data section (header[0x1C]): a 12-byte global header (effect_start_time at 0x04, target_switching_delay at 0x06, effect_max_duration_per_target at 0x0A), then 5 × 0x80-byte emitter-timing blocks at 0x0C — each holding uint16[16] spawn frames at +0x00, unknown uint16[9] at +0x20, uint8[16] 1-indexed emitter IDs (0 = none) at +0x32, and unknown timing control at +0x42 — with spawn times 0x257–0x259 having special meaning and a spawn valid when `emitter_id >= 0 and spawn_time < 0x200`.** — `[ ] 0/3`
  - R: none — godot-learning `TimelineData`/`PhaseBlock` model the timeline as 128-byte particle channels (25 keyframes) plus color/sound/camera tracks, not raw 5×0x80 emitter-timing blocks
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
- **The working document gives this effect-script opcode table: 0x00 = unconditional jump (4 bytes, relative uint16 offset at +2), 0x04 = end effect (2 bytes), 0x1D = conditional branch on a timing condition, 0x1E = multi-target branch, 0x1F = hit-counter branch (hit/miss/evade), 0x29 = graphics multi-target branch; it notes script execution is not fully documented and requires PS1 RAM tracing.** — `[ ] 0/3`
  - R: none — godot-learning/effect-editor use the 16-bit 9-bit-opcode + 7-bit-flags word model with a 512-entry jump table and named handlers (opcodes 40/41 dominant), not this byte-oriented opcode list
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
- **Timeline callback routing: dispatch_unit_triggers_and_register_callback (0x801A3D30) reads action_flags from the keyframe at offset 0x4C and, when bits 0–2 ≠ 0, sets effect_state->callback_state[slot] = 1 (offset 0x22) and copies callback_slots[slot] into effect_state->callback_ptrs[slot] (offset 0xD4); process_particle_channel_and_unit_triggers (0x801A4000) then reads the channel keyframe's action_flags at offset 0x4A and, when bits 0–2 ≠ 0, JALRs callback(effect_target, slot, emitter_index − 1, spawn_counter) at 0x801A4134 instead of the normal emitter spawn.** — `[S·R] 2/3`
  - S: registration disassembly 0x801A3FA4–0x801A3FD4, invocation 0x801A412C–0x801A4134 (keyframe offsets 0x4C/0x4A, JALR at 0x801A4134), per `research/working_documents/MIPS_CALLBACK_SYSTEM.md`
  - R: `godot-learning/src/effects/ParticleSubsystem.gd:195-201` (action_flags & 0x7; nonzero → callback_manager.invoke(callback_slot − 1, emitter_idx, spawn_counter, …)) + `godot-learning/tests/CB91TimingTest.gd` (E317 callback timing monitor; playback test, no hard asserts)
  - src: `research/working_documents/MIPS_CALLBACK_SYSTEM.md`

## Notes

(empty — user territory)

## Related

- [[E001.BIN Memory Mapping]]
- [[Color Track Interpolation]]
- [[E317 Choco Ball Callback System]]
- [[Effect File Format]]
- [[Embedded MIPS Effect Code]]
- [[Effect Texture Upload]]
