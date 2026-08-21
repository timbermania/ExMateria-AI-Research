# Effect Sound Audio Divergence

State of the Godot effect-sound synth vs the PCSX-Redux capture for `cure_no_music` as of the 2026-05-16 post-walker-flag-fix doc (`AUDIO_PARITY_NEXT_STEPS_V2`): the walker flag word entry shortfall is closed — the end-of-note ADSR2_LOW write (commit 9bf658cc) was the single biggest mover (full_mix cos_dist 0.38→0.31, voice_21 0.47→0.37; `probe_walker_flag_word_entry` pairs 320/320 within ±1 drift, 318/320 bit-exact) — and the pitch chain re-pairs with its write cadence now FFT-faithful. Residual divergences at that date: 2 init-seed walker entries (Godot seeds 0xC5 vs PCSX's 0x1FF, missing the ADSR1/SAMPLE_ADDR/VOL_LR_SWEEP arms), 2 extra PCSX walker exits via `chan_word_0 & 0x500` (`probe_slur_propagation_pre` 81/79), a 2-row `_fan_vol_lr_raw` excess at voice-21 ramp start (`probe_vol_register` 122/124, values aligned), plus still-unprobed ADSR1-register and KON/KOFF-mask paths. The 2026-05-12 accounts below are historical: the 0.45 residual was the pre-fix baseline, and the 1-cadence pitch-offset account is superseded. The 2026-05-13 CAUSE_A_PRESTAGE_TIMING doc re-attributes the "init-seed" 0x1FF entries to bind-time AC dispatch (PC `0x80017088`), and the follow-up Godot fix (deferred `walker_seed_pending` arm, synthetic first_keyon arm removed) landed since. The 2026-05-18 death (E030) V21 deficit closed a second divergence class: Godot's 1.24 s truncation of voice 21 was the missing 3rd `play_sound` — a BATTLE.BIN-driven global-bank trigger (`0x8006B97C`, sound_id 0x45) structurally outside the effect-only renderer, resolved by smd-player's catalog replay (see [[Battle Action SFX]]).

## Points

- **`cure_no_music` `voice_21` audio still diverges from the PCSX capture at cos_dist ≈ 0.45 and is not a scaled-or-shifted variant of it (best cross-correlation −91 at +792 samples vs +52 at zero shift; least-squares gain −0.0225): the 2026-05-12 run shows matching peaks (PCSX 0.5511 vs Godot 0.5517) with the dominant frequency off by ~40 Hz / ~37 cents (PCSX 1832 Hz vs Godot 1872.5 Hz) — a per-sample pitch-process difference, not a gain or alignment bug.** — `[D·R] 2/3`
  - D: `audio_diff_report.py` + cross-correlation run on the `voice_21` WAV pair (commit ba7acba0; 2026-05-12)
  - R: `smd-player/src/shared/` native SPU mixer (`fft_spu_core_runtime.cpp`, `fft_spu_pitch_runtime.cpp` et al.)
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **Godot fires the pitch-register writes one cadence (~4.17 ms / 184 samples) EARLIER than PCSX starting from the 2nd write, bit-exact on value (voice_21: 165/165 writes value-paired; row 0 matches exactly, rows 1..164 all `diff_cad = −1` — cadence sequences PCSX `1,3,6,8,…` vs Godot `1,2,5,7,…`); with the cure tone's pitch register oscillating aggressively between ~6800 and ~3050 under LFO/vibrato, the 1-cadence shift puts the two sides in different LFO phases at every transition.** — `[D·R] 2/3`
  - D: `probe_pitch_register` voice_21 capture (2026-05-12)
  - R: pitch staging/walker path `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` + `smd-player/src/shared/fft_spu_voice_runtime.h` (`FFTPitchUpdateScheduled` pending queue); validated by the `probe_pitch_register` pair
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
  - ⚠ SUPERSEDED (2026-08-20) by: SPU register write cadence (ADSR2_LOW, PITCH, VOL_LR_RAW, ADSR2_HIGH) is FFT-faithful post-walker-flag fix — probe_pitch_inputs/_staging/_register all pair with no row-count or value diff
- **Godot's effect-sound LFO runs entirely in the GDScript engine (`channel.lfo_*` → `advance_lfo` → `pitch_bend` → pitch staging → SPU pitch register); the C++ `FFTPitchLfoBlock` machinery (`fft_spu_lfo_tools.cpp`, `mixer.init_voice_pitch_lfo`) is only reachable from the DAW/VST3 build, so `lfo_blocks[0].enabled == false` for all effect voices and `fft_effective_voice_sinc()` is a no-op on that path.** — `[R] 1/3`
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd` (GDScript LFO engine) + `smd-player/addons/exmateria_sound/runtime/sequencer/opcodes/pitch_lfo_init.gd` (Pass 7.D.d — Godot no longer calls `mixer.init_voice_pitch_lfo`; the C++ engine stays in `smd-player/src/shared/fft_spu_lfo_tools.cpp` for the VST3 build)
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **The `cure_no_music` playback timeline (cadence = SPU IRQ tick): cadence 0 dispatches both the silent driver Note (voice 20, `delta_time = 4`) and the audible Note (voice 21, `delta_time = 144` — the long first cure tone); the first KON v20+v21 lands at cadence 1 (registers seeded via the 0x1FF walker mask); KOFF v20 at cadence 349 (silent driver pre-Rest release, fired by the idle-timeout drain) and again at 351 when the Rest opcode dispatches; KOFF v21 at 359; the next note's KON is off-by-one at 361 (Godot) / 362 (PCSX).** — `[D·R] 2/3`
  - D: probe-derived timeline (2026-05-11/12; commit fbfe2830 — idle_timeout drain fires KOFF, resolving the cad 349/359 directional inversions)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` + note-handler idle-timeout drain (KOFF on drain); validated by the `probe_kon_koff_mask` / `probe_pitch_register` pairs
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **As of the 2026-05-12 probe state the main remaining structural divergence on `cure_no_music` is the walker flag word entry count: PCSX fires 320 per-slot fan-out entries (`s1 != 0`) vs Godot's 270 — Godot never enters the walker on the 50 ticks where FFT enters with a pure ADSR2_LOW (0x080) flag word, and fires `_fan_vol_lr_raw` alone (0x001, 48 entries) where FFT fires the ADSR2_LOW companion (0x081, 24 entries); the ~70 missing ADSR2-low re-commits leave the voice-21 release-envelope shape divergent (full_mix cos_dist ~0.38, voice_21 ~0.47 vs the 0.09 parity threshold).** — `[D·R] 2/3`
  - D: `probe_walker_flag_word_entry` 320/270 bucket distribution + 7-second audio-score window (2026-05-12)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/spu_irq_walker.gd` + `runtime/shared/dispatcher.gd` (walker-entry and `_fan_vol_lr_raw` cadence) — the gap is tracked as the unpaired `probe_walker_flag_word_entry` in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_NEXT_STEPS.md`
  - ⚠ SUPERSEDED (2026-08-20) by: post end-of-note ADSR2_LOW fix (commit 9bf658cc) the walker flag word entry gap is closed — probe_walker_flag_word_entry pairs 320/320 within ±1 drift (318/320 bit-exact, 2 init-seed row diffs remain)
- **Pitch is end-to-end FFT-faithful on `cure_no_music`: `probe_pitch_inputs`, `probe_pitch_staging`, and `probe_pitch_register` all pair 222/222 with no row-count or value diff — SPU pitch register writes (voice + pitch) match end-to-end, ruling out pitch encoding/staging as the source of the remaining audio divergence.** — `[D·R] 2/3`
  - D: cure_no_music Layer 5 pitch probe pairs, 222/222 each (2026-05-12)
  - D: post-walker-flag-fix re-pair — `probe_pitch_inputs`/`_staging`/`_register` all pair with no row-count or value diff, write cadence FFT-faithful ("full pitch pipeline FFT-faithful", 2026-05-16)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/pitch_staging.gd` + `runtime/shared/spu_irq_walker.gd` (`_fan_pitch`) — validated by the `probe_pitch_register` pair
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_NEXT_STEPS.md`
- **Post end-of-note ADSR2_LOW write fix (commit 9bf658cc — 74 added walker passes committing `_fan_adsr2_low`), `probe_walker_flag_word_entry` pairs 320/320 within ±1 drift on `cure_no_music` (318/320 bit-exact, 2 init-seed row diffs) and the audio baseline improved ~0.07 on full_mix (0.38 → 0.31) and ~0.10 on voice_21 (0.47 → 0.37) — the end-of-note ADSR2_LOW write was the single biggest mover.** — `[D·R] 2/3`
  - D: `probe_walker_flag_word_entry` pair + `run_effect_iteration.py --session cure_no_music` audio-score baseline, post-9bf658cc (2026-05-16)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` (KOFF-prep gate sourcing the 74 walker passes) + `runtime/shared/spu_irq_walker.gd::_fan_adsr2_low`
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_NEXT_STEPS_V2.md`
- **The play_sound init seed sets the walker flag word to 0x1FF on PCSX — arming every arm, including ADSR1 (0x10/0x20/0x100), SAMPLE_ADDR (0x8), and VOL_LR_SWEEP (0x2) — while Godot's init seeds only 0xC5, missing those five arms; the two surviving `probe_walker_flag_word_entry` row diffs are exactly these init-seed entries.** — `[D·R] 2/3`
  - ⚠ SUPERSEDED (2026-08-20) by: on PCSX the 0x1FF walker arm is written only by the 0xAC AC opcode's data loader (`Hyp_instrument_data_loader`, store at PC `0x80017088`) — the cad=0/1 `cure_no_music` 0x1FF entries are bind-time AC dispatches at savestate load, not an unconditional play_sound init seed
  - D: `probe_walker_flag_word_entry` 2 init-seed row diffs, PCSX 0x1FF vs Godot 0xC5 (2026-05-16)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd` init-seed path — the doc names Godot's init as skipping the missing arms (observed 0xC5 seed in the Godot capture)
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_NEXT_STEPS_V2.md`
- **`probe_slur_propagation_pre` shows PCSX takes 2 more per-channel walker exits than Godot on `cure_no_music` (81 vs 79) via the `chan_word_0 & 0x500 != 0` exit condition — a pre-existing structural diff.** — `[D·R] 2/3`
  - D: `probe_slur_propagation_pre` capture (2026-05-16)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` (walker-exit loop condition mirror on `channel_word_0 & 0x500`)
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_NEXT_STEPS_V2.md`
- **Godot fires `_fan_vol_lr_raw` 2× more than FFT at voice 21's ramp start on `cure_no_music` (`probe_vol_register` 122/124), with values bit-exact when aligned — the drift is in walker-flag timing, not vol logic.** — `[D·R] 2/3`
  - D: `probe_vol_register` capture (2026-05-16)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/spu_irq_walker.gd::_fan_vol_lr_raw`
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_NEXT_STEPS_V2.md`
- **Death's (E030) 3rd `play_sound` (cad 254) re-allocates pair slot 4 with no preemption — `killed_kon_mask=0`/`killed_mask=0` because the V20/V21 pair already released at the cad-191 EndBar — and re-binds it with global-bank bytecode dispatching from cad 256 and re-KEYONing at cad 257, producing the second audible burst of `spu_voice_21.wav` in [1.24 s, 2.40 s] (PCSX 105,835 frames vs Godot 54,725; the first 0.91 s matched within ±7% RMS) — Godot truncated V21 at 1.24 s until the catalog replay fired the equivalent trigger.** — `[D·R] 2/3`
  - D: `probe_play_sound_alloc` (call 3: slot=4, killed_kon_mask=0, killed_mask=0), `probe_kon_koff_mask` (PCSX-only rows at cad 255/257/404/407), and the `spu_voice_21.wav` pair (RMS ratio 0.00 after 1,133 ms on Godot), death_no_music captures (2026-05-18)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/pool.gd` `find_free_pair_slot` (pass 1 free-slot scan, pass 2b preempt scan — LRU proxy for FFT's entity+0x10 preempt priority, PC 0x80012E08–0x80012E70) + `runtime/effect_sound/play_sound.gd` `play_feds_pair` slot re-bind; validated by the post-fix death_no_music orchestrator re-run (probe_play_sound_call 2/2, KON pair_rate 0.7241 → 1.0000)
  - src: `research/effect_sound/working_documents/DEATH_V21_MISSING_THIRD_PLAY_SOUND_DEFICIT.md`

## Notes

(empty — user territory)

## Related

- [[Effect Sound Parity Ladder]]
- [[Battle Action SFX]]
- [[SPU Voice Engine]]
- [[WAVESET Instrument Bank]]
- [[KON KOFF Mask Dispatch]]
- [[Effect Sound Timing]]
- [[SFX Index]]
