# Effect Sound Timing

Effect sound playback in E###.BIN is scheduled by frame-based sound tracks in the timeline section, not by the SMD data itself: opcodes 40/41 tick the tracks each frame with a per-keyframe duration counter (`time_values[N]` = exact frames between sound N and sound N+1, the sound fires the instant its keyframe is entered), each keyframe's `sound_id` (0/1 = none, ≥2 = config index − 2) is resolved through the effect-flags sound-channel configs (five selection modes in `lookup_sound_effect`), and the resolved sound triggers an even/odd channel pair into the feds/SMD engine, at NTSC 30 FPS (1 frame = 33.33 ms). The 2026-04-16 working document reads the sound section as repeating uint16 (frame, sound ID) trigger pairs, an alternative model to the opcode-driven timing above, recorded as a low-evidence point. A 2026-05-13 probe bisection adds pre-trace behaviour: on PCSX the parent sound-track ticker runs 11–12 effect-ticks (and binds 2 FEDS channel pairs) before the SMD interpreter's first opcode dispatch, a pre-tick Godot's tick-0 start does not reproduce, driving the 24-cadence first-keyframe drift (see [[Cure 4 Audio Parity]]).

## Points

- **Effect sound playback is triggered from the opcode 40/41 timeline tick handlers: the handler advances the sound track (advance_p1_sound_track 0x801A478C for parent tracks), lookup_sound_effect (0x801A32E8) resolves the mode-selected sound ID, play_sound (0x80012520) triggers the SPU, and FUN_80013B20 starts the SMD interpreter over the channel pair.** — `[S] 1/3`
  - S: advance_p1_sound_track 0x801A478C, lookup_sound_effect 0x801A32E8, play_sound 0x80012520, FUN_80013B20, per `research/wiki_articles/sound_timing_godot.md`
  - src: `research/wiki_articles/sound_timing_godot.md`
- **Opcode 41 (op_process_timeline_frame) processes the six parent sound tracks (30 bytes each, 3 per phase) at timeline_section_ptr + 0xD2A, and opcode 40 (op_animate_tick) processes the three child sound channels (54 bytes each) at timeline_channel_base + 0x284.** — `[S] 1/3`
  - S: sound-track locations timeline_section_ptr (0x801BC0C8) + 0xD2A and timeline_channel_base + 0x284, per `research/wiki_articles/sound_timing_godot.md`
  - src: `research/wiki_articles/sound_timing_godot.md`
- **Sound tracks are driven by a per-frame duration counter: when the counter reaches 0 the keyframe's sound plays immediately (if sound_id ≥ 2), the counter is reloaded from time_values[keyframe], and the keyframe advances — so time_values[N] is the exact number of frames between sound N and sound N+1, not a delay before the sound.** — `[S] 1/3`
  - S: duration-counter logic in the timeline tick handlers (advance_p1_sound_track 0x801A478C), per `research/wiki_articles/sound_timing_godot.md`
  - src: `research/wiki_articles/sound_timing_godot.md`
- **Child (for-each) sound channels are 54 bytes — time_values[17] int16 @ 0x00, sound_ids[17] uint8 @ 0x22, padding @ 0x33, max_keyframe int16 @ 0x34 — and parent (outer-phase) sound tracks are 30 bytes — time_values[9] int16 @ 0x00, sound_ids[9] uint8 @ 0x12, padding @ 0x1B, max_keyframe int16 @ 0x1C.** — `[S] 1/3`
  - S: track layouts at timeline_channel_base + 0x284 (54 B) and timeline_section_ptr + 0xD2A (30 B), per `research/wiki_articles/sound_timing_godot.md`
  - src: `research/wiki_articles/sound_timing_godot.md`
- **A keyframe's sound_id of 0 or 1 means no sound (skip), and sound_id ≥ 2 selects sound config channel (sound_id − 2) — the −2 offset lets 0/1 be no-ops without negative values.** — `[S] 1/3`
  - S: sound_id − 2 config mapping in the trigger path (lookup_sound_effect 0x801A32E8), per `research/wiki_articles/sound_timing_godot.md`
  - S: same −2 mapping restated (values 0, 1 = no sound; 2+ = config channel value − 2), per `research/working_documents/SOUND_CHANNEL_ARCHITECTURE.md`
  - src: `research/wiki_articles/sound_timing_godot.md`
  - src: `research/working_documents/SOUND_CHANNEL_ARCHITECTURE.md`
- **The sound-channel config section (effect_flags_ptr 0x801BACC8 + 0x08–0x17: 4 channels × 4 bytes — mode, id_a, id_b, id_c) selects which feds sound each timeline keyframe plays: lookup_sound_effect (0x801A32E8) returns id_a for mode 0 (DIRECT_A), alternates id_a/id_b for mode 1 (PARITY_AB), returns id_a on the first call and id_b on all subsequent calls for mode 2 (FIRST_A_THEN_B), returns id_a then alternates id_b/id_c for mode 3 (FIRST_A_THEN_BC), and cycles id_a→id_b→id_c for mode 4 (CYCLE_ABC), driven by the per-channel call counter sound_call_count[4] at 0x801B9250.** — `[S·R] 2/3`
  - S: lookup_sound_effect 0x801A32E8, config offsets 0x08–0x17, effect_flags_ptr 0x801BACC8, counter 0x801B9250; E001 example (config channel 1 at +0x0C: mode 0, id_a 2 → play_sound(0x20002)), per `research/working_documents/SOUND_CHANNEL_ARCHITECTURE.md`
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound_resolver.gd` (EffectSoundResolver — port of lookup_sound_effect with the same 5 modes and the per-container counter annotated DAT_801b9250) + `godot-learning/tests/EffectSoundCaptureTest.gd` (drives the effect-sound path through the resolver)
  - src: `research/working_documents/SOUND_CHANNEL_ARCHITECTURE.md`
- **E010.BIN outer sound track 2 (raw data at file offset +0x2F8) decodes to time_values [6, 14, 580] and sound_ids [0, 4, 0] (max_keyframe 3), so its single sound (config 2) fires at frame 6 — about 200 ms after the effect starts.** — `[S] 1/3`
  - S: raw track bytes at E010.BIN +0x2F8 (time_values) and +0x31E (sound_ids), per `research/wiki_articles/sound_timing_godot.md`
  - src: `research/wiki_articles/sound_timing_godot.md`
- **Every sound trigger always processes exactly two channels: FUN_80013B20 selects the pair with a ×4 stride (sll at 0x80013C38, skipping the 0x14 header) and the playback loop count is hardcoded to 2 (ori v0, zero, 0x2 at 0x80012534) — the even channel is often a silent placeholder (AC 01) or primary voice while the odd channel carries the actual audio (E001 Cure: ch0 = AC 01 placeholder, ch1 = AC 43 tubular bells).** — `[S·D·R] 3/3`
  - S: pair stride at 0x80013C38 and 2-channel loop count at 0x80012534, per `research/wiki_articles/sound_timing_godot.md`
  - S: loop-counter global DAT_800329F0 written 2 at 0x80012534, read at 0x80013C74, decremented at 0x80013DC4–0x80013DC8; per-channel +2 advance of the offset-table cursor at 0x80013D90/0x80013DB8, per `research/working_documents/INSTRUMENT_MAPPING.md`
  - D: WRITE breakpoint at 0x80013D54 (`sw v1, -0x1c(s0)` storing the SMD pointer to the channel struct): v1 = feds_ptr + 0x59 (file channel 1's SMD address), s6 = feds_ptr + 0x1A (offset-table position read) (2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd` (`play_feds_pair` reads `feds + 0x14 + config*4`) + `godot-learning/src/audio/EffectSfxEngine.gd` (`play_pair(token, feds_bank, pair_idx, sound_id)`) + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/wiki_articles/sound_timing_godot.md`
  - src: `research/working_documents/INSTRUMENT_MAPPING.md`
- **The effect sound-setup call to FUN_80013B20 packs a1 = (resource_id << 16) | config_channel (e.g. 0x00020001 = resource 2, config channel 1): the low byte selects the channel pair via the ×4 stride, and the high word is the resource bank (sound_data_base = resource_id << 16, stored to g_sound_data_base at 0x801BC0DC).** — `[S] 1/3`
  - S: stride read of a1 at 0x80013C38 (`sll v0, a1, 0x2`), bank store at 0x801A1160, FUN_80013B20, per `research/working_documents/INSTRUMENT_MAPPING.md`
  - R: none — the packed a1 word not present in godot-learning / smd-player (components modeled separately: `resource_id << 16` bank base in `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd`, `feds + 0x14 + config*4` pair selection in the same file and `feds_bank.gd`)
  - src: `research/working_documents/INSTRUMENT_MAPPING.md`
- **Effect sound timing is in game frames at NTSC 30 FPS: 1 frame = 33.33 ms, 30 frames = 1 s, 600 frames = 20 s (a common "end of track" duration value).** — `[ ] 0/3`
  - src: `research/wiki_articles/sound_timing_godot.md`
- **The runtime global timeline_channel_base sits at 0x801BBF80 (derived as timeline_section_ptr + 8).** — `[S] 1/3 CONTESTED`
  - S: 0x801BBF80 (timeline_channel_base) in the "Global Pointers at Runtime" table, per `research/wiki_articles/sound_timing_godot.md`
  - src: `research/wiki_articles/sound_timing_godot.md`
- **SMD Rest (opcode 0x80) inside feds channel data is a different pause from timeline duration: Rest takes a tick count and pauses between notes within one sound's playback (e.g. `80 04` = 4-tick rest), while the track's time_values[] spaces separate sound triggers in frames.** — `[S] 1/3`
  - S: Rest opcode 0x80 with tick-count param in feds channel data, per `research/wiki_articles/sound_timing_godot.md`
  - src: `research/wiki_articles/sound_timing_godot.md`

- **The 2026-04-16 working document defines the sound effects section (at header offset 0x20 in its section numbering) as a variable-length list of repeating uint16 pairs: sound_start_time (frame to trigger the sound) at +0x00 and sound_id at +0x02.** — `[ ] 0/3`
  - R: none — godot-learning sound timing is the opcode 40/41 track above (time_values/sound_ids arrays in the timeline section); [[FEDS Sound Definition Format]] treats the sound data as feds SMD-like sequenced instructions, not raw pairs.
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
- **The child-track layout is confirmed — 54 bytes a track, `time_values` at `+0`, `sound_ids` at `+0x22` — and so is `timeline_channel_base = timeline + 8`; both labels on the E010 example are wrong, though, and the numbers in it are right.** — `[S] 1/3`
  - S: found by searching for the bytes rather than by computing them. `E010`'s timeline section is at file `0x1D94`, and the bytes `06 00 0e 00 44 02` appear at file `0x208C` — which is `timeline + 0x2F8`, and `0x2F8` is `0x28C + 2 × 54`, i.e. **child track 2** by the note's own `0x284 + 54k` rule with `channel_base = timeline + 8`. The ids read `0, 4, 0` at `+0x22` (the child layout) and `0, 0, 0` at `+0x12` (the parent layout). So `0x2F8` is **section-relative, not a file offset**, and the track is a **child, not a parent** — the two labels, not the data (web-psx `docs/effect-format.md` [effect.xref.timeline.sound]) (2026-08-19)
  - src: external contribution — web-psx `docs/effect-format.md` [effect.xref.timeline.sound] (see [[Web-psx Cross-Validation]])
- **On PCSX the effect-sound timeline runs pre-trace: the phase-1 sound-track ticker (`advance_p1_sound_track` / FUN_801A478C @ 0x801A478C, BATTLE.BIN) fires 11–12 effect-ticks before the SMD interpreter's first opcode dispatch (cure_4_no_music: 33 pre-trace fires = 11 ticks × 3 phase1 tracks, first fire at cadence 17; cure_no_music: 36 fires = 12 ticks × 3 tracks, all pre-trace, ticker stops once phase1 completes), fires stepping 7–8 cadences apart (a 30 Hz effect-tick rate at 240 Hz cadences), and `resolve_feds_channel` (0x80013B20) binds 2 FEDS channel pairs pre-trace on cure_4.** — `[S·D] 2/3`
  - S: `advance_p1_sound_track` / FUN_801A478C @ 0x801A478C in BATTLE.BIN (ticker symbol per the doc's bisection trail; same address recorded in this note from `research/wiki_articles/sound_timing_godot.md`)
  - D: `probe_timeline_track_tick` @ 0x801A478C (no FIRST_OPCODE_FIRED gate; `smd-player/workspace/probes/probe_timeline_track_tick.lua` wired into `smd-player/workspace/orchestrator/effect_capture_orchestrator.lua`), cure_4_no_music + cure_no_music sessions (2026-05-13); `probe_resolve_feds_channel_pre_trace`: 2 pre-first-opcode fires on cure_4
  - R: none — no pre-trace ticker warm-up in godot-learning or smd-player (probed; the per-frame ticker itself is mirrored in `smd-player/addons/exmateria_sound/runtime/effect_sound_controller.gd`, verified against `advance_p1_sound_track` 0x801A478C, and starts at tick 0)
  - src: `research/effect_sound/working_documents/CADENCE_DRIFT_SPAWN_DELAY.md`

## Notes

(empty — user territory)

## Related

- [[Effect Execution Model]]
- [[Effect File Format]]
- [[Effect Frame Pacing]]
- [[FEDS Sound Definition Format]]
- [[WAVESET Instrument Bank]]
- [[Cure 4 Audio Parity]]
