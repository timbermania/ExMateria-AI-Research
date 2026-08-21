# Effect Sound Slot Allocator

FFT's effect-SMD pair-slot allocator `play_sound_callee_12d40` (`FUN_80012D40`), which hands out 2-voice pairs out of `g_sound_resource_list` slots 0–5 (`DAT_80032A60`, 8 entities × `0x160` stride; pair 6/7 reserved for the music sequencer). Two-pass: pass 1 kills entities already playing the incoming sound_id; pass 2 sweeps free pairs high-to-low from slot `6 − pair_size`, and when the pool is exhausted preempts the busy slot with the smallest `entity+0x10` priority among slots whose `entity+0xd` release-state byte is below `0x21`. The 2026-05-19 `disillusionment_3_no_music` capture pinned the preempt rule on live hardware (cadence-238 slot-2 eviction; pre-fix Godot held PCSX v18/v19 content in voices 20/21 via a hardcoded slot-4 fallback), and smd-player now mirrors pass 2b with an LRU `bind_tick` proxy — the doc's Phase 1; true `entity+0x10`/`entity+0xd` instrumentation is Phase 2. The two-pass structure and cure_4 occupancy behaviour live in [[Cure 4 Audio Parity]].

## Points

- **When no free pair exists, FFT's pair-slot allocator (`play_sound_callee_12d40` pass 2b) preempts the busy pair slot with the smallest `entity+0x10` (u32 priority/lifetime metric; smaller = more preemptible) among slots whose `entity+0xd` (release-state byte) is below `0x21` — the candidate sweep runs high-to-low in pair strides (slots 4, 2, 0 for stereo pairs; pair 6 is outside the search range, reserved for the music sequencer).** — `[S·D·R] 3/3`
  - S: preempt-candidate loop PCs `0x80012E08..0x80012E70` — candidate load `lw v1, 0x10(a2)` @ `0x80012E20`, release-state gate `sltiu v0, v0, 0x21` @ `0x80012E34`, candidate update `move t3, v1; move t5, a0` @ `0x80012E48`, preempt-path return @ `0x80012E78` (`scus_disassembly.txt` / `scus_decompilation.c` `play_sound_callee_12d40`)
  - D: `probe_play_sound_alloc.jsonl` (breakpoints at `0x80012D40`/`0x80012E78`), `disillusionment_3_no_music` capture (2026-05-19) — sid=5 allocation (call_index 4, cadence 238, all three pair slots busy) returns `slot:2, v0:1` (preempt path taken)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/pool.gd` pass 2b (`find_free_pair_slot` LRU preempt search, `bind_tick` proxy for `entity+0x10`; `entity+0xd` gate dropped, Phase 2) + `runtime/shared/slot_state.gd` `bind_tick`, consumed by `godot-learning/src/audio/EffectSfxEngine.gd` — validated by `godot-learning/tests/EffectSoundCaptureTest.gd` `_run_faithful_mode` (FAITHFUL 3-pair budget preempts under load, no SPU stacking)
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_DEFICIT.md`

- **The effect-SMD pool is `g_sound_resource_list` at `DAT_80032A60` — 8 entities × `0x160` stride: `entity+0xb8` = `chan_word_0` (bit 0 = alive; zeroed by pass 1 on same-sid kill), `entity+0xc0` = sound_id (pass 1 kill match), `entity+0xec` = voice_mask (OR'd into `killed_kon_mask`), `entity+0x10` = preempt priority u32, `entity+0xd` = release-state u8; the global active bitmap feeding the pass 2 busy mask lives at `base+0x58`.** — `[S·D·R] 3/3`
  - S: entity-field loads in `0x80012D40..0x80012E78` (sound_id @ `0x80012D74`, voice_mask @ `0x80012D84`, priority @ `0x80012E20`, release byte @ `0x80012E34`, active bitmap @ `0x80012DF4`), `scus_disassembly.txt`
  - D: `diag_slot_allocator.jsonl` bind-time `data_loader` writes touching `entity+0xb8` et al., `disillusionment_3_no_music` capture (2026-05-19)
  - R: partial — `smd-player/addons/exmateria_sound/runtime/shared/slot_state.gd` models the `CHAN0_*` `chan_word_0` flags + `bind_tick` (LRU proxy for `entity+0x10`); `entity+0xd` release-state not present in smd-player (Phase 2)
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_DEFICIT.md`

- **On `disillusionment_3_no_music`, sid=5 arrives at cadence 238 with all three pair slots (0, 2, 4) busy; PCSX preempts slot 2 (sid=2's pair, bound 160 cadences earlier) while the pre-fix Godot build preempts slot 4 via the hardcoded `6 − 2` fallback, and the sid=6 allocation cascades (PCSX slot 4 vs Godot slot 2); the KON-signature multiset match proves the stream-to-voice misassignment — PCSX voice 18 ≡ Godot voice 20 (57/58 KON signatures identical: recurring sa=4864 percussive trigger, raw_pitch 681/1362 alternating, `adsr1=19/27/30`, `adsr2=24518`) and voice 19 ≡ Godot voice 21 (14/14).** — `[S·D·R] 3/3`
  - S: FFT slot-2 selection per the pass 2b preempt rule, PCs `0x80012E08..0x80012E70` (`scus_disassembly.txt`)
  - D: `probe_play_sound_alloc.jsonl` (call_index 4 → `slot:2, v0:1`; call_index 5 → `slot:4`) vs `godot.log` `[timeline]` (sid=5 → slot 4, sid=6 → slot 2) + `spu_voice_events.jsonl` multiset signatures + `audio_score.json` (voice_18/voice_20 cos_dist 0.66/0.59), `disillusionment_3_no_music` capture (2026-05-19)
  - R: `smd-player/render_effect_sound.gd:275` / `:412` hardcoded slot-4 fallback (the doc's defect sites; paths since relocated into the addon) — replaced by `smd-player/addons/exmateria_sound/runtime/effect_sound/pool.gd` pass 2b LRU preempt, which picks the oldest-bound slot (slot 2 on this session's sid=5)
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_DEFICIT.md`

- **`play_sound_callee_12d40` has exactly two `jal` call sites — `0x800125C8` (in `FUN_800125A8`, the play_sound entry) and `0x80012694` (variant call path) — and its returned slot_idx is consumed by `FUN_80013B20` (`feds_channel_resolver`), which binds the SMD bytecode pointer into `entity+0xb8..` and writes the `entity+0x10` priority init (store within `0x80013BB0..0x80013C50`).** — `[S·R] 2/3`
  - S: `jal` sites `0x800125C8` / `0x80012694`, consumer `FUN_80013B20` (`scus_disassembly.txt`)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd` (port of `FUN_800125A8` + `FUN_80013B20`: `feds_channel_resolver` bridge, `play_feds_pair` post-allocation bind) + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_DEFICIT.md`

## Notes

(empty — user territory)

## Related

- [[Cure 4 Audio Parity]]
- [[SPU Voice Engine]]
- [[Effect Sound Audio Divergence]]
- [[SFX Index]]
