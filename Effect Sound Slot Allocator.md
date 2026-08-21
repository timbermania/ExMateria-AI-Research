# Effect Sound Slot Allocator

FFT's effect-SMD pair-slot allocator `play_sound_callee_12d40` (`FUN_80012D40`), which hands out 2-voice pairs out of `g_sound_resource_list` slots 0–5 (`DAT_80032A60`, 8 entities × `0x160` stride; pair 6/7 reserved for the music sequencer). Two-pass: pass 1 kills entities already playing the incoming sound_id; pass 2 sweeps free pairs high-to-low from slot `6 − pair_size`, and when the pool is exhausted preempts the busy slot with the smallest combined `(chan[0x12] << 16) | chan[0x10]` metric among slots whose `chan+0xd` gate byte is below `0x21`. The 2026-05-19 `disillusionment_3_no_music` capture pinned the preempt rule on live hardware (cadence-238 slot-2 eviction; pre-fix Godot held PCSX v18/v19 content in voices 20/21 via a hardcoded slot-4 fallback), and smd-player's LRU `bind_tick` proxy (Phase 1, commit `3ffd0b10`) now matches PCSX `probe_play_sound_alloc` row-for-row on that session's sid=5/sid=6 slot decisions. The deficit doc's "u32 priority" framing for `chan+0x10` was corrected: it is a u16 status bitfield, and pass 2b structurally prefers slots with its bit 0x8000 (SPU-servicing) cleared — the mechanism the LRU proxy correlates with. True `chan+0x10`/`chan+0xd` modeling is Phase 2, not yet ported. The two-pass structure and cure_4 occupancy behaviour live in [[Cure 4 Audio Parity]].

## Points

- **When no free pair exists, FFT's pair-slot allocator (`play_sound_callee_12d40` pass 2b) preempts the busy pair slot with the smallest `entity+0x10` (u32 priority/lifetime metric; smaller = more preemptible) among slots whose `entity+0xd` (release-state byte) is below `0x21` — the candidate sweep runs high-to-low in pair strides (slots 4, 2, 0 for stereo pairs; pair 6 is outside the search range, reserved for the music sequencer).** — `[S·D·R] 3/3`
  - S: preempt-candidate loop PCs `0x80012E08..0x80012E70` — candidate load `lw v1, 0x10(a2)` @ `0x80012E20`, release-state gate `sltiu v0, v0, 0x21` @ `0x80012E34`, candidate update `move t3, v1; move t5, a0` @ `0x80012E48`, preempt-path return @ `0x80012E78` (`scus_disassembly.txt` / `scus_decompilation.c` `play_sound_callee_12d40`)
  - D: `probe_play_sound_alloc.jsonl` (breakpoints at `0x80012D40`/`0x80012E78`), `disillusionment_3_no_music` capture (2026-05-19) — sid=5 allocation (call_index 4, cadence 238, all three pair slots busy) returns `slot:2, v0:1` (preempt path taken)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/pool.gd` pass 2b (`find_free_pair_slot` LRU preempt search, `bind_tick` proxy for `entity+0x10`; `entity+0xd` gate dropped, Phase 2) + `runtime/shared/slot_state.gd` `bind_tick`, consumed by `godot-learning/src/audio/EffectSfxEngine.gd` — validated by `godot-learning/tests/EffectSoundCaptureTest.gd` `_run_faithful_mode` (FAITHFUL 3-pair budget preempts under load, no SPU stacking)
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_DEFICIT.md`
  - ⚠ SUPERSEDED (2026-08-20) by: FFT's pass-2b preempt metric reads `chan+0x10` as a u16 status bitfield, not a designed u32 priority — the upper 16 bits the `lw v1, 0x10(a2)` at `0x80012E20` picks up come from the separate u16 field at `chan+0x12`, so the combined `(chan[0x12] << 16) | chan[0x10]` value is an incidental ordering metric

- **The effect-SMD pool is `g_sound_resource_list` at `DAT_80032A60` — 8 entities × `0x160` stride: `entity+0xb8` = `chan_word_0` (bit 0 = alive; zeroed by pass 1 on same-sid kill), `entity+0xc0` = sound_id (pass 1 kill match), `entity+0xec` = voice_mask (OR'd into `killed_kon_mask`), `entity+0x10` = preempt priority u32, `entity+0xd` = release-state u8; the global active bitmap feeding the pass 2 busy mask lives at `base+0x58`.** — `[S·D·R] 3/3`
  - S: entity-field loads in `0x80012D40..0x80012E78` (sound_id @ `0x80012D74`, voice_mask @ `0x80012D84`, priority @ `0x80012E20`, release byte @ `0x80012E34`, active bitmap @ `0x80012DF4`), `scus_disassembly.txt`
  - D: `diag_slot_allocator.jsonl` bind-time `data_loader` writes touching `entity+0xb8` et al., `disillusionment_3_no_music` capture (2026-05-19)
  - R: partial — `smd-player/addons/exmateria_sound/runtime/shared/slot_state.gd` models the `CHAN0_*` `chan_word_0` flags + `bind_tick` (LRU proxy for `entity+0x10`); `entity+0xd` release-state not present in smd-player (Phase 2)
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_DEFICIT.md`
  - ⚠ SUPERSEDED (2026-08-20) by: `entity+0x10` is a u16 status bitfield, not a u32 preempt priority — every use in `scus_decompilation.c` is ushort-typed, and the upper half of the pass-2b `lw` at `0x80012E20` belongs to the separate u16 field at `chan+0x12`

- **On `disillusionment_3_no_music`, sid=5 arrives at cadence 238 with all three pair slots (0, 2, 4) busy; PCSX preempts slot 2 (sid=2's pair, bound 160 cadences earlier) while the pre-fix Godot build preempts slot 4 via the hardcoded `6 − 2` fallback, and the sid=6 allocation cascades (PCSX slot 4 vs Godot slot 2); the KON-signature multiset match proves the stream-to-voice misassignment — PCSX voice 18 ≡ Godot voice 20 (57/58 KON signatures identical: recurring sa=4864 percussive trigger, raw_pitch 681/1362 alternating, `adsr1=19/27/30`, `adsr2=24518`) and voice 19 ≡ Godot voice 21 (14/14).** — `[S·D·R] 3/3`
  - S: FFT slot-2 selection per the pass 2b preempt rule, PCs `0x80012E08..0x80012E70` (`scus_disassembly.txt`)
  - D: `probe_play_sound_alloc.jsonl` (call_index 4 → `slot:2, v0:1`; call_index 5 → `slot:4`) vs `godot.log` `[timeline]` (sid=5 → slot 4, sid=6 → slot 2) + `spu_voice_events.jsonl` multiset signatures + `audio_score.json` (voice_18/voice_20 cos_dist 0.66/0.59), `disillusionment_3_no_music` capture (2026-05-19)
  - R: `smd-player/render_effect_sound.gd:275` / `:412` hardcoded slot-4 fallback (the doc's defect sites; paths since relocated into the addon) — replaced by `smd-player/addons/exmateria_sound/runtime/effect_sound/pool.gd` pass 2b LRU preempt, which picks the oldest-bound slot (slot 2 on this session's sid=5)
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_DEFICIT.md`

- **`play_sound_callee_12d40` has exactly two `jal` call sites — `0x800125C8` (in `FUN_800125A8`, the play_sound entry) and `0x80012694` (variant call path) — and its returned slot_idx is consumed by `FUN_80013B20` (`feds_channel_resolver`), which binds the SMD bytecode pointer into `entity+0xb8..` and writes the `entity+0x10` priority init (store within `0x80013BB0..0x80013C50`).** — `[S·R] 2/3`
  - S: `jal` sites `0x800125C8` / `0x80012694`, consumer `FUN_80013B20` (`scus_disassembly.txt`)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd` (port of `FUN_800125A8` + `FUN_80013B20`: `feds_channel_resolver` bridge, `play_feds_pair` post-allocation bind) + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_DEFICIT.md`

- **FFT's pass-2b preempt metric reads `chan+0x10` as a u16 status bitfield, not a designed u32 priority — every use in `scus_decompilation.c` is ushort-typed, and the upper 16 bits the `lw v1, 0x10(a2)` at `0x80012E20` picks up come from the separate u16 field at `chan+0x12`, so the combined `(chan[0x12] << 16) | chan[0x10]` 32-bit value is an incidental ordering metric.** — `[S] 1/3`
  - S: ushort access sites `FUN_8001216C`, `FUN_800121DC`, `FUN_80012284`, `FUN_80012338`, `FUN_80012380`, `FUN_8001255C`, `FUN_80012594`, `FUN_8001268C`, `FUN_800126A8`, `FUN_80012E7C`, `FUN_80012F08` + candidate load `lw v1, 0x10(a2)` @ `0x80012E20` (`scus_decompilation.c` / `scus_disassembly.txt`)
  - R: none — no `chan_x10`/`chan_x12` model in smd-player or godot-learning (probed smd-player/addons/exmateria_sound, godot-learning/src, godot-learning/tests)
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_FOLLOWUPS.md`

- **Provisional `chan+0x10` status-bit layout: bit 0x8000 is the SPU-update gate — gated per tick in `FUN_8001216C`, set on bind/pitch settle (`FUN_800121DC`, `FUN_80012F08` / `FUN_80012284` `& 0xfeff | 0x8000`), cleared on release-decay entry/consume (`FUN_80012338`); bit 0x100 is pitch-write pending — set by `FUN_80012380` (`& 0x7fff | 0x100`), consumed+cleared by `FUN_80012F08`/`FUN_80012284`; bit 0x10 is SPU-rebind pending — set by `FUN_8001255C`, consumed+cleared by `FUN_80012594`; bit 15 is read by `FUN_80012E7C` (`>> 0xf`); bit 0 is set at init by `FUN_8001268C`.** — `[S] 1/3`
  - S: bit set/clear sites listed in the claim (`scus_decompilation.c`); labels provisional pending Phase 2 `diag_pool_slot_state` instrumentation
  - R: none — no `STATUS_BIT_*` status-bit model in smd-player or godot-learning (probed smd-player/addons/exmateria_sound, godot-learning/src, godot-learning/tests)
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_FOLLOWUPS.md`

- **Pass 2b's `sltu` compares the combined `(chan[0x12] << 16) | chan[0x10]` metric against `0xFFFFFFFF`, so it structurally prefers slots whose `chan+0x10` bit 0x8000 is cleared — i.e. slots not currently servicing the SPU (in release-decay or dormant); an LRU on bind age correlates with this because the oldest-bound slot is the one most likely to have transitioned to release-decay by the time a fresh allocation arrives.** — `[S·D] 2/3`
  - S: pass-2b candidate loop `0x80012E08..0x80012E70` + `sltu` against `0xFFFFFFFF` (`scus_disassembly.txt`)
  - D: `probe_play_sound_alloc` pairing, `disillusionment_3_no_music` capture (2026-05-19) — LRU proxy picks the same sid=5/sid=6 slots as PCSX
  - R: none — smd-player models a `bind_tick` LRU only, no `chan+0x10` bit model (probed smd-player, godot-learning)
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_FOLLOWUPS.md`

- **The Phase 1 LRU `bind_tick` proxy (commit `3ffd0b10`) reproduces FFT's pass-2b slot decisions row-for-row against PCSX `probe_play_sound_alloc` on `disillusionment_3_no_music` (sid=5/sid=6), with the four target voices v18–v21 collapsing from cos_dist > 0.58 to < 0.014.** — `[D·R] 2/3`
  - D: `probe_play_sound_alloc.jsonl` + timeline-log pairing, `disillusionment_3_no_music` capture (2026-05-19) — sid=5/sid=6 slot-decision match
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/pool.gd` `find_free_pair_slot` LRU pass 2b (commit `3ffd0b10`) + orchestrator `research/effect_alignment/orchestrator/run_effect_iteration.py --session disillusionment_3_no_music` probe-pair check
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_FOLLOWUPS.md`

- **FFT's pass-1 same-sid kill writes its kill bitmap to `DAT_80032A10` and the KON-mask of killed slots to `DAT_80032A14`, feeding the pass-2 busy mask (`~killed_mask & global_active & pair_bitmask` at PC `0x80012E04`); smd-player has no pass-1 equivalent today, so Godot-side kill masks are always 0 and surface as probe-pair divergence only on sessions that exercise pass 1.** — `[S·D] 2/3`
  - S: `probe_play_sound_alloc.lua` exit-side `read32(DAT_80032A10)` / `read32(DAT_80032A14)`; busy-mask term PC `0x80012E04` (`scus_disassembly.txt`)
  - D: `probe_play_sound_alloc.jsonl` `killed_mask` / `killed_kon_mask` fields, `disillusionment_3_no_music` capture (2026-05-19)
  - R: none — pass-1 same-sid-kill not present in smd-player (probed smd-player/addons/exmateria_sound, godot-learning)
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_FOLLOWUPS.md`

- **PCSX bakes the bank-category bit into the high word of the sound id (`a0`) at the `FUN_800125A8` entry, while the Godot timeline driver emits `sound_id` without it — so `play_sound_alloc` `param_1` may diverge in the high 16 bits between sides and the probe-pair diff should mask the high word when comparing.** — `[S·R] 2/3`
  - S: `FUN_800125A8` entry (`scus_disassembly.txt`)
  - R: `smd-player/workspace/harness/render_effect_sound.gd:501-506` (documents the divergence; timeline calls use the per-effect bank, BATTLE.BIN catalog replay uses raw sound_id carrying the category high word)
  - src: `research/effect_sound/working_documents/DISILLUSIONMENT_PAIR_SLOT_PREEMPT_FOLLOWUPS.md`

## Notes

(empty — user territory)

## Related

- [[Cure 4 Audio Parity]]
- [[SPU Voice Engine]]
- [[Effect Sound Audio Divergence]]
- [[SFX Index]]
