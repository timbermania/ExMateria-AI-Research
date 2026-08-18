# Effect Sound Timing

Effect sound playback in E###.BIN is scheduled by frame-based sound tracks in the timeline section, not by the SMD data itself: opcodes 40/41 tick the tracks each frame with a per-keyframe duration counter (`time_values[N]` = exact frames between sound N and sound N+1, the sound fires the instant its keyframe is entered), each keyframe's `sound_id` (0/1 = none, ≥2 = config index − 2) is resolved through the effect-flags sound-channel configs (five selection modes in `lookup_sound_effect`), and the resolved sound triggers an even/odd channel pair into the feds/SMD engine, at NTSC 30 FPS (1 frame = 33.33 ms). The 2026-04-16 working document reads the sound section as repeating uint16 (frame, sound ID) trigger pairs, an alternative model to the opcode-driven timing above, recorded as a low-evidence point.

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
  - src: `research/wiki_articles/sound_timing_godot.md`
- **E010.BIN outer sound track 2 (raw data at file offset +0x2F8) decodes to time_values [6, 14, 580] and sound_ids [0, 4, 0] (max_keyframe 3), so its single sound (config 2) fires at frame 6 — about 200 ms after the effect starts.** — `[S] 1/3`
  - S: raw track bytes at E010.BIN +0x2F8 (time_values) and +0x31E (sound_ids), per `research/wiki_articles/sound_timing_godot.md`
  - src: `research/wiki_articles/sound_timing_godot.md`
- **Every sound trigger always processes exactly two channels: FUN_80013B20 selects the pair with a ×4 stride (sll at 0x80013C38, skipping the 0x14 header) and the playback loop count is hardcoded to 2 (ori v0, zero, 0x2 at 0x80012534) — the even channel is often a silent placeholder (AC 01) or primary voice while the odd channel carries the actual audio (E001 Cure: ch0 = AC 01 placeholder, ch1 = AC 43 tubular bells).** — `[S] 1/3`
  - S: pair stride at 0x80013C38 and 2-channel loop count at 0x80012534, per `research/wiki_articles/sound_timing_godot.md`
  - src: `research/wiki_articles/sound_timing_godot.md`
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

## Notes

(empty — user territory)

## Related

- [[Effect Execution Model]]
- [[Effect File Format]]
- [[Effect Frame Pacing]]
- [[FEDS Sound Definition Format]]
