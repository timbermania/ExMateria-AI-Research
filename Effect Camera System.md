# Effect Camera System

FFT's effect camera controls position, rotation (angles), and zoom during E###.BIN playback. Camera keyframes live in the Timeline Section and are processed each frame by advance_camera_tracks against three purpose-specific tables (main, per-target, cleanup); each keyframe carries a 16-bit command word selecting tracks, position source, and interpolation curve — including three shake curves that reinterpret keyframe data as per-axis amplitudes — with all interpolation state held in fixed global slots. The command word's param_index (bits 3–4) and flags (bits 13–15) fields are proven never-read on the engine's camera path (dead bits), and no authored keyframe in the vanilla ROM sets flags. Random shake is applied in the camera state machine (FUN_801AB724), which runs after the effect main loop in the per-frame update (FUN_801A1C40); damped states linearly decay amplitude toward zero, direct holds it constant, and Fire 4 (E019.BIN) chains all three states on the position track. Source modes apply per track with asymmetric behaviour: OFFSET is angles-only (pitch set absolute, yaw/roll added), MAP reads the live camera state at fire time, and ALL_TARGETS averages over the total target count; both the EFFECT_CTR source and the particle CAMERA anchor resolve to the map center via get_camera_position 0x8008DF48 (map dimensions ×14). The 2026-04-16 working document claims an additional 0x42C+ camera-control region inside the timer/camera data, recorded below as a low-evidence point.

## Points

- **Camera data lives in the Timeline Section (header[0x1C]; timeline_section_ptr at 0x801BC0C8) and is processed each frame by advance_camera_tracks (0x801AD198).** — `[S] 1/3`
  - S: timeline_section_ptr 0x801BC0C8 and advance_camera_tracks 0x801AD198, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The camera system has three tables: TABLE 0 MAIN (camera during the entire spell, absolute frame timing), TABLE 1 FOR-EACH-TARGET (per-target timing, active when effect_context_value at 0x801BAD0C > 0), and TABLE 2 CLEANUP (restores the saved pre-turn camera state).** — `[S] 1/3`
  - S: camera tables 0/1/2 and effect_context_value 0x801BAD0C, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Each camera keyframe carries a 16-bit command word: bits 0–2 track_enable (1 = angles, 2 = position, 4 = zoom), bits 3–4 parameter, bits 5–8 position_src (& 0x1E0), bits 9–12 interp_curve (& 0x1E00), bits 13–15 extra flags.** — `[S] 1/3`
  - S: CameraCommand struct and masks (CAMERA_CMD_TRACK_MASK 0x0007, CAMERA_CMD_SOURCE_MASK 0x01E0, CAMERA_CMD_CURVE_MASK 0x1E00), per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - S: keyframe selectors FUN_801ACA58 and find_active_camera_keyframe 0x801ACB08 test `track_selector & command_word` (selectors ∈ {1,2,4}), consuming only the low 3 bits, Ghidra decompile (2026-07-12 export), per `research/working_documents/CAMERA_COMMAND_PARAM_FLAGS_ARE_INERT.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The command word's `param_index` (bits 3–4, mask 0x0018) and `flags` (bits 13–15, mask 0xE000) are NOT-READ on the PSX camera path — dead bits; the dispatchers pass the word whole and unmasked, and the per-track handlers apply only the 0x1E0 (source) and 0x1E00 (interpolation) masks.** — `[S·R] 2/3`
  - S: `advance_camera_tracks` 0x801AD198 and FUN_801ACF90 0x801ACF90 (unmasked pass-through); `execute_camera_angle_command` 0x801AAD98 (andi 0x801AADB4 / 0x801AAF60), `execute_camera_position_command` 0x801AB1D4 (0x801AB1F0 / 0x801AB334 / 0x801AB550), `execute_camera_zoom_command` 0x801AB590 (0x801AB5AC / 0x801AB65C / 0x801AB6EC) apply only 0x1E0/0x1E00; no andi 0x18 / srl 3 / andi 0xE000 / srl 13 anywhere on the camera path; `camera_state_machine` 0x801AB724 reads only the stored state globals — Ghidra decompile (2026-07-12 export), per `research/working_documents/CAMERA_COMMAND_PARAM_FLAGS_ARE_INERT.md`
  - R: `godot-learning/src/effects/CameraSubsystem.gd` likewise never reads either field; bytes round-trip exactly through `godot-learning/tools/write_effect_camera.py`
  - src: `research/working_documents/CAMERA_COMMAND_PARAM_FLAGS_ARE_INERT.md`
- **No authored camera keyframe in the vanilla ROM sets `flags` (bits 13–15) — every non-zero flags value comes from a single misparsed table (E471 `phase2`, 0xFF-ish padding); of 23659 scanned keyframes the 75 legit non-zero hits are all param-only.** — `[S] 1/3`
  - S: command-word tables at timeline_section_ptr + 0x806 (MAIN), +0x168a (FOR-EACH), +0x185a (CLEANUP); full-ROM scan of 401 non-empty E###.BIN by `godot-learning/tools/survey_camera_param_flags.py` (2026-08-10): 23659 keyframes, flags histogram {0:23639, 2:1, 5:1, 6:1, 7:17}, all 20 suspect hits in E471 phase2 (max_keyframe=0, all end_frame=0), per `research/working_documents/CAMERA_COMMAND_PARAM_FLAGS_ARE_INERT.md`
  - src: `research/working_documents/CAMERA_COMMAND_PARAM_FLAGS_ARE_INERT.md`
- **Position sources (& 0x1E0): 0x000 TARGET, 0x020 OFFSET, 0x040 DIRECT (absolute), 0x060 ORIGIN, 0x080 EFFECT_CTR (camera pos ×14 scaling), 0x0C0 MAP, 0x100 SLOT_COPY (saved pre-turn slot), 0x140 CASTER, 0x180 ALL_TARGETS (average), 0x1C0 CURSOR.** — `[S] 1/3`
  - S: position-source command table (CAMERA_SRC_TARGET…CAMERA_SRC_CURSOR), per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Interpolation curves (& 0x1E00): 0x0200 IMMEDIATE, 0x0400 COSINE_A (<<11), 0x0600 COSINE_B (<<12), 0x0800 LINEAR, 0x0A00 COSINE_C (angle wrapping), 0x0C00 ADDITIVE, 0x0E00 ADDITIVE_B, 0x1000 SHAKE_DAMPED (amplitude decays to zero), 0x1200 SHAKE_DIRECT (constant amplitude), 0x1400 SHAKE_DAMPED_B (variant).** — `[S] 1/3`
  - S: interpolation-curve table (CAMERA_CURVE_IMMEDIATE…CAMERA_CURVE_SHAKE_DAMPED_B), per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Shake curves (0x1000–0x1400) reinterpret position keyframe data as per-axis amplitudes (int16 amp_x/amp_y/amp_z); each frame apply_random_position_shake (0x801A95E8) applies a random offset in [−amp, +amp] per axis via random_in_range (0x801A8690).** — `[S] 1/3`
  - S: shake-state description, apply_random_position_shake 0x801A95E8, random_in_range 0x801A8690, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Camera runtime state: track flags 0x801B69CC/0x801B69D0/0x801B69D4 (0 = idle, 0x200+ = interpolating); angle slots at 0x801B8A60–0x801B8A87 (frames_total 0x801B8A60, frame_counter 0x801B8A64, start/target/current/saved int16[3] angles 0x801B8A68–0x801B8A87); position slots at 0x801B8A88+ (frames_total, frame_counter, final_position int32[4], track2_output int32[4]); zoom at frames_total 0x801B8AD0, frame_counter 0x801B8AD4, track3_output 0x801B8AB0, zoom_value 0x801B8ABC — with zoom 4096 = 1.0× (less = out, more = in).** — `[S] 1/3`
  - S: CameraInterpolationState address tables 0x801B69CC–0x801B8AD4, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Camera function map: find_active_camera_keyframe (0x801ACB08), execute_camera_angle_command (0x801AAD98), execute_camera_position_command (0x801AB1D4), execute_camera_zoom_command (0x801AB590), camera_state_machine (0x801AB724), interp_cosine_a (0x801A894C), interp_cosine_b (0x801A89B0), interp_linear (0x801A8A14), lerp_component (0x801A8898).** — `[S] 1/3`
  - S: camera function table, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The per-frame effect update (FUN_801A1C40) runs the effect main loop (effects/particles/scripts), then FUN_801B47E0 (purpose unidentified), and finally the camera state machine (FUN_801AB724) — so camera interpolation, including shake, sees the final effect state before rendering.** — `[S] 1/3`
  - S: per-frame update call order in FUN_801A1C40 (effect_system_main_loop → FUN_801B47E0 → FUN_801AB724), per `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
  - src: `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
- **The camera state machine (FUN_801AB724) processes interpolation states 0x0200–0x1400 on all three camera tracks, with per-track handler code for each random shake state: 0x1000 RANDOM_SHAKE_DAMPED → LAB_801AC2D0 (position) / LAB_801AC940 (zoom), 0x1200 RANDOM_SHAKE_DIRECT → LAB_801AC300 / LAB_801AC970, 0x1400 RANDOM_SHAKE_DAMPED_B → LAB_801AC340 / LAB_801AC9A0 (position block at LAB_801ABD38+, zoom block at LAB_801AC3CC+).** — `[S] 1/3`
  - S: shake-state handler labels and per-track code regions in FUN_801AB724, per `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
  - src: `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
- **random_in_range (FUN_801A8690) returns min when min == max, else min + rand() % (max − min) for max ≥ min and max + rand() % (min − max) for max < min — half-open [min, max) sampling that tolerates swapped bounds.** — `[S] 1/3`
  - S: decompiled body of FUN_801A8690, per `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
  - src: `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
- **apply_random_position_shake (FUN_801A95E8) samples one random offset per axis in [−amp, +amp] using the per-axis amplitudes at keyframe-data indices 0/2/4 (X/Y/Z), and writes each output as (current_position + random_offset) << 12 fixed-point.** — `[S] 1/3`
  - S: decompiled body of FUN_801A95E8, per `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
  - src: `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
- **The damped shake states (0x1000/0x1400) linearly interpolate the stored amplitude toward zero each frame and call apply_random_position_shake with the damped amplitude; direct shake (0x1200) applies the full amplitude every frame; all three states clear their state back to 0 once frame_counter reaches frames_total.** — `[S] 1/3`
  - S: shake-state flow in the FUN_801AB724 handlers (LAB_801AC2D0/LAB_801AC300/LAB_801AC340), per `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
  - src: `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
- **Fire 4 (E019.BIN) arms random shake on the position track with three keyframes: kf3 (end_frame 36, command 0x1042 → 0x1000 RANDOM_SHAKE_DAMPED), kf6 (end_frame 94, 0x1442 → 0x1400 RANDOM_SHAKE_DAMPED_B), kf7 (end_frame 601, 0x1242 → 0x1200 RANDOM_SHAKE_DIRECT).** — `[S] 1/3`
  - S: Fire 4 shake keyframe table in E019.BIN, per `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
  - src: `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
- **Shake amplitudes are stored in the position keyframe tables as 6-byte entries (int16 amp_x, amp_y, amp_z) at offsets 0x158E (table 0), 0x073A (table 1), and 0x175E (table 2); each entry's three values define the ± random-offset range for X/Y/Z.** — `[S] 1/3`
  - S: shake-amplitude keyframe table layout, per `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
  - src: `research/working_documents/CAMERA_SHAKE_SYSTEM_WIP.md`
- **Each camera table in the Timeline Section is five parallel sub-tables — end frames, command, angle, position, zoom — with 2-byte end_frame/command entries and 6-byte data entries; section-relative offsets are MAIN 0x14e6/0x168a/0x1510/0x158e/0x160c, FOR-EACH 0x6b2/0x806/0x6d4/0x73a/0x7a0, CLEANUP 0x16b6/0x185a/0x16e0/0x175e/0x17dc (Fire 4 E019.BIN section starts at file 0x2db0).** — `[S] 1/3`
  - S: camera table offset table; ROM-verified in E019.BIN — MAIN command table at section-relative 0x168a (file 0x443a) holds the Fire 4 MAIN keyframes 0x1042/0x1442/0x1242 at end frames 36/94/601, FOR-EACH command at 0x806 (file 0x35b6), CLEANUP command at 0x185a (file 0x460a, 0x507 SLOT_COPY-all-tracks at frame 16), per `research/working_documents/CAMERA_SYSTEM.md`
  - src: `research/working_documents/CAMERA_SYSTEM.md`
- **OFFSET (0x020) is angle-track-only and asymmetric: it SETs pitch directly (replacing the current pitch) while ADDing to yaw and roll; the position and zoom tracks do not recognize OFFSET (no-op), and because the default battle camera already sits at ~26.5° pitch (raw 302), an OFFSET pitch of +26.5° is usually a no-op.** — `[S] 1/3`
  - S: assembly at LAB_801AB0FC in execute_camera_angle_command 0x801AAD98, per `research/working_documents/CAMERA_SYSTEM.md`
  - src: `research/working_documents/CAMERA_SYSTEM.md`
- **MAP (0x0C0) reads the live camera state at the time the command fires — position from work_position (>>12, 20.12 fixed-point to integer), angles via FUN_8008BA50, zoom via FUN_8008B824 — each plus the keyframe value, so `MAP zoom +0` keeps the current zoom rather than resetting to a stored default.** — `[S] 1/3`
  - S: MAP branches of execute_camera_position_command 0x801AB1D4 (FUN_8008B2FC), execute_camera_angle_command 0x801AAD98 (FUN_8008BA50), execute_camera_zoom_command 0x801AB590 (FUN_8008B824), per `research/working_documents/CAMERA_SYSTEM.md`
  - src: `research/working_documents/CAMERA_SYSTEM.md`
- **ALL_TARGETS (0x180) is position-only and computes the average by dividing the summed target positions by the total target count (effect_context_value at 0x801BAD0C), not the count of valid (flag==0) entries — tile-based targets (flag != 0) contribute zero to the sum but still dilute the average.** — `[S] 1/3`
  - S: ALL_TARGETS loop over effect_targets_table 0x801BAD10 with divisor = effect_context_value, per `research/working_documents/CAMERA_SYSTEM.md`
  - src: `research/working_documents/CAMERA_SYSTEM.md`
- **Ghidra labels two different functions `get_camera_position()`: 0x8008DF48 reads the map dimension bytes (DAT_800E4E9C width, DAT_800E4EA0 depth, single bytes via lbu) and is the map-center source (EFFECT_CTR, particle CAMERA anchor, ×14 scaling), while 0x8008C410 takes a unit ID and returns unit_struct+0x40 (TARGET/CASTER/ALL_TARGETS) — distinguish by address.** — `[S] 1/3`
  - S: 0x8008DF48 (JAL d237020c) and 0x8008C410 (JAL 0431020c); EFFECT_CTR path via FUN_801AAD3C, per `research/working_documents/CAMERA_SYSTEM.md`
  - src: `research/working_documents/CAMERA_SYSTEM.md`
- **SLOT_COPY (0x100) restores the saved pre-turn camera — angles from 0x801B8A80 (yaw shortest-path wrapped against current), position from 0x801B8AC0, zoom from 0x801B8B08 — plus the keyframe offset; Fire 4's CLEANUP table (TABLE 2) uses it at frame 16 (command 0x507, all three tracks).** — `[S] 1/3`
  - S: saved slots 0x801B8A80/0x801B8AC0/0x801B8B08 (copy destinations 0x801B8A70/0x801B8AA0/0x801B8AE8); CLEANUP 0x507 keyframe at section-relative 0x185a in E019.BIN, per `research/working_documents/CAMERA_SYSTEM.md`
  - src: `research/working_documents/CAMERA_SYSTEM.md`
- **TARGET (0x000)/CASTER (0x140)/CURSOR (0x1C0) position lookup is two-path per target entry (0x801BAD10/0x801BADB0/0x801BADCA): flag==0 → get_camera_position 0x8008C410 by unit ID, else tile coords → X=tile_x×28+14, Z=tile_z×28+14, Y=height×−12, then the keyframe is added; angles come from the facing calc FUN_801AAC28 (pitch/roll copied from the current state, yaw snapped to the nearest 90° quadrant from tile passability) plus the keyframe.** — `[S] 1/3`
  - S: target/caster/cursor entries 0x801BAD10/0x801BADB0/0x801BADCA, FUN_801AAB44/FUN_801AAB90/FUN_801AAC28, per `research/working_documents/CAMERA_SYSTEM.md`
  - src: `research/working_documents/CAMERA_SYSTEM.md`

- **The 2026-04-16 working document claims a camera control region at offset 0x42C+ of the timer/camera data section (0x178 bytes) holding int16 camera_vertical_tilt (+0x26), camera_face_tilt (+0x2A), camera_rotation_vert/horiz (+0x48/+0x4A), and camera_zoom_x/y/z (+0xF8/+0xFA/+0xFC), with observed 16-bit command patterns: 0xC704 stop rotation, 0x4404 zoom change, 0x2104 tilt change, 0x04xx huge zoom out, 0x0304/0x0204 focus on target, 0x0104 rotate battlefield.** — `[ ] 0/3`
  - R: none — godot-learning's camera model is the 16-bit command-word + three-table keyframe system above; no 0x42C region parser exists in the repo.
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
  - src: `research/working_documents/VFX_PARTICLES_EMITTERS_DEEP_DIVE.md`

## Notes

(empty — user territory)

## Related

- [[Effect Execution Model]]
- [[Particle Runtime State]]
