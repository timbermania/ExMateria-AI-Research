# SPU Voice Engine

The shared SCUS SPU voice engine behind all FFT sound playback: a small set of driver primitives (trigger voices, per-handle key-off, per-voice set-volume) that every SFX path — system SFX, ambient `{6B}` bg sounds, and the feds/SMD effect sounds — funnels into, gated by a single audio-enable flag. The `{6B}` BG Sound investigation (2026-07) established the full static + dynamic picture of these primitives on live hardware, and the Godot reimplementation mirrors the volume path exactly (`vol << 8`). The 2026-04 MUSIC_41 synth-accuracy work (software SPU synth vs PCSX-Redux capture) pinned down the voice-level details: static sequential track-to-voice allocation, the `FUN_80017118` volume chain, ADSR sustain-direction bit and floor behaviour, and the SPU's Gaussian/ADPCM sample pipeline. The 2026-06 effect_sound folder README records the authoritative voice-space split: music owns SPU voices 0–15, effect SFX own voices 16–23. The 2026-05-12 audio-parity final-state doc (cure_no_music) closed the per-voice register layer: all of FFT's per-voice SPU register writes are bit-exact paired against Godot across 10 probe families, and a side-by-side C++ read confirmed Godot's native SPU mixer matches PCSX-Redux's implementation for ADPCM init/coefficients/loop flags, the ADSR attack/decay curves, the mix constants (/1023, /0x4000), and pitch→sinc (<< 4). The residual audio divergence lives in per-sample pitch modulation (see [[Effect Sound Audio Divergence]]).

## Points

- **All FFT sound playback funnels through the shared SCUS SPU voice engine: `FUN_80013b20(mode, handle, 0x6000, 0x4000)` triggers voices (play), `FUN_80012990(handle)` keys off every active voice whose stored handle matches (per-handle stop, not global), and `FUN_80012b6c(handle, vol)` sets each matching voice's volume to `vol << 8` with `vol == 0` taking the same key-off path — all gated by the audio-enable flag `DAT_80032a54 & 0x1000`.** — `[S·D·R] 3/3`
  - S: `FUN_80013b20`, `FUN_80012990` (voice slots at `DAT_80032a60 + 0xb8 + n*0x160`), `FUN_80012b6c`, gate `DAT_80032a54` (`scus_decompilation.c`)
  - D: Orbonne battle live run — play trigger at `FUN_80013b20` with handles 0x10001/0x10012, `FUN_8004408c` set-volume firing once per frame, and the enable flag reading `0x9101` (gate set) (2026-07-01)
  - R: `godot-learning/src/audio/EffectSfxEngine.gd` `set_bg_gain` = `set_voice_volume_lr(voice, vol<<8, vol<<8)` (exact mirror of `FUN_80012b6c`) + `godot-learning/tests/ScenarioBgSoundTest.gd`
  - src: `research/working_documents/BGSOUND_OPCODE_6B_INVESTIGATION.md`
- **PSX SPU hardware ground truth behind FFT's sound driver: the 24 voice channels are all capability-identical (no hardware even/odd distinction), pitch modulation (PMON) is sequential — voice N−1 modulates voice N — not even/odd, PMON bit 0 is unused (voice 0 has no predecessor), and any channel can act as a modulator by zeroing its left/right volume while still generating its waveform.** — `[R] 1/3`
  - R: `smd-player/addons/exmateria_sound/runtime/spu.gd` (NUM_VOICES = 24; fmod mode = "this voice modulated by previous voice's") + `smd-player/addons/exmateria_sound/runtime/shared/opcodes/fmod_enable.gd` (0xB2: previous voice as frequency-channel modulation source) + `godot-learning/tests/EffectSoundCaptureTest.gd` (regression over the SPU engine incl. fmod)
  - src: `research/working_documents/INSTRUMENT_MAPPING.md`

- **FFT's SPU voice allocation is static and sequential, not dynamic: `FUN_800138ac` assigns track N to SPU voice N−1 (the conductor track gets 0xFF = invalid) and performs no voice reallocation during playback — the apparent "voice sharing" in MUSIC_41 (tracks 1 and 3 alternating on voice 0) was a misattribution: track 1 itself alternates between instruments 38 and 39.** — `[S·D·R] 3/3`
  - S: `FUN_800138ac` decompilation recorded in `research/working_documents/SYNTH_ACCURACY.md` (2026-04-16)
  - D: MUSIC_41.SMD reference capture (`vsprojects/x64/Release/spu_capture/`, 6.76 s, 9 of 24 voices active; doc 2026-04-16) — first-note correlation 0.994 on voice 0 / track 3 under the static assignment
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer.gd:258-262` (`load_trackset`: track i ≥ 1 → `voice_idx = i − 1`, conductor unassigned) + smd-player music parity Gate A (`smd-player/workspace/regression/verify_all.sh`)
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **FFT's per-voice volume chain (`FUN_80017118`, PC `0x800171C4`–`0x80017228`) computes dynamics_14bit × velocity_14bit >> 15, then × master_vol >> 16, then applies the PSX pan curve with 0x5A00/0x2500 coefficients — dynamics=127, velocity=64, center pan gives ~5713 per channel (the old formula gave 16256).** — `[S·D·R] 3/3`
  - S: `FUN_80017118` decompilation recorded in `research/working_documents/SYNTH_ACCURACY.md` (2026-04-16); PC range per the smd-player port header
  - D: MUSIC_41.SMD reference-capture comparison (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/note_handler/compute_vol_lr.gd` (port of the `>> 15` / `>> 16` chain + pan polynomials ≈ 0x5A00/0x2500) + `smd-player/workspace/probes/probe_vol_formula_stages.lua` + music parity Gate D
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **PSX SPU ADSR2 bit 14 is the sustain direction: set = sustain decreases, clear = sustain increases (the initial reimplementation had this inverted).** — `[D·R] 2/3`
  - D: ADSR register capture via the `PCSX.SPU.getVoiceInfo()` Lua binding, instrument 39 (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/adsr.gd` `set_from_regs` — `sustain_increase = 1 - ((adsr2 >> 14) & 1)`
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **The PSX ADSR decay step truncates toward zero rather than flooring: the exponential decrement is effectively `int((dec * vol) / 32768)`, so the envelope holds at its mathematical floor instead of decaying past it — observed as a stable envVol of 2385–5461 on the reference capture.** — `[D] 1/3`
  - D: stable envVol 2385–5461 via `PCSX.SPU.getVoiceInfo()` envVol trace (doc 2026-04-16)
  - R: none — truncation-toward-zero ADSR decay not present in godot-learning, smd-player, or fft-sound-driver (smd-player's `runtime/adsr.gd` uses `floori(product / 32768.0)` and fft-sound-driver's `src/shared/fft_adsr_envelope.cpp` uses arithmetic `>> 15`; both floor toward −inf)
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **PSX SPU sample interpolation is a 4-sample circular-buffer Gaussian with fixed coefficients (PCSX-Redux `gauss.h`), not linear interpolation.** — `[D·R] 2/3`
  - D: MUSIC_41.SMD first-note comparison — 0.994 waveform / 0.999 spectral correlation after switching to the real coefficients (doc 2026-04-16)
  - R: `smd-player/src/shared/fft_gauss_table.inc` + `smd-player/src/shared/fft_spu_sample_runtime.cpp` (per-voice `gauss_buf` / `gauss_pos`) + `smd-player/workspace/regression/native_core_smoke_suite.json`
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **The PSX SPU ADPCM decoder uses separate prediction-filter shifts with no rounding bias, and decoder state is preserved across loop boundaries.** — `[D·R] 2/3`
  - D: MUSIC_41.SMD reference-capture match (doc 2026-04-16)
  - R: `smd-player/src/shared/fft_spu_sample_runtime.cpp` (ADPCM voice sample runtime) + `smd-player/workspace/regression/native_core_smoke_suite.json`
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **FFT splits the SPU's 24-voice space between music and effect SFX: music uses voices 0–15 and effect SFX use voices 16–23.** — `[S·R] 2/3`
  - S: the music/SFX voice split (music 0–15 / SFX 16–23) recorded as AUTHORITATIVE in `research/effect_sound/working_documents/SPU_VOICE_ALLOCATION.md` (indexed by `research/effect_sound/README.md`)
  - R: `godot-learning/src/audio/EffectSfxEngine.gd` `_make_pool` — FAITHFUL mode `_Pool.new(8, 16, 6)` ("exact FFT pool: voices 16-23, slots 6-7 reserved") + `godot-learning/tests/EffectSoundCaptureTest.gd` (sets `VoiceMode.FAITHFUL`, drives the effect-sound path)
  - src: `research/effect_sound/README.md`
- **Every per-voice SPU register write FFT performs is bit-exact paired between PCSX-Redux and Godot on `cure_no_music` — 10 probe families cover Vol_L no-sweep (PAIR 122/122), Vol_L sweep (4/4), pitch (222/222), sample start address (4/4), ADSR1 mid/low/high (4/4 each), ADSR2 (11/11), sample loop address (4/4), and ADSR2 low (values bit-exact, 154 PCSX vs 152 Godot rows — init-seed count only) — 100% per-voice register coverage.** — `[S·D·R] 3/3`
  - S: SPU-offset ↔ FFT-helper mapping: `+0x00`/`+0x02` Vol_L/R ← `FUN_8001B428`/`FUN_8001B4B0`, `+0x04` pitch ← `FUN_8001B628`, `+0x06` start ← `FUN_8001B6A4`, `+0x08` ADSR1 mid/low/high ← `FUN_8001B79C`/`FUN_8001B8B0`/`FUN_8001B938`, `+0x0A` ADSR2/low ← `FUN_8001B9D4`/`FUN_8001BAB8`, `+0x0E` loop ← `FUN_8001B720` (per `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md` §2)
  - D: cure_no_music paired probe captures (2026-05-11/12): `probe_vol_register`, `probe_vol_register_sweep`, `probe_pitch_register`, `probe_sample_start_addr_register`, `probe_adsr1_{mid,low,high}_register`, `probe_adsr2_register`, `probe_adsr2_low_register`, `probe_sample_repeat_addr_register` (`smd-player/workspace/probes/`)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` (per-voice register write path) + GOLD probe set `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **PSX SPU ADPCM voice state at KON initializes `s1 = 0`, `s2 = 0`, block position (SBPos) = 28, and interpolation position `spos = 0x30000` — Godot's native core initializes all four identically.** — `[R] 1/3`
  - R: `smd-player/src/shared/fft_spu_voice_runtime.cpp` (`buf_pos = adpcm_samples_per_block` = 28, `adpcm_s1/s2 = 0`, `spos = 0x30000`) side-by-side with `vendor/pcsx-redux/src/spu/spu.cc` KON path (doc §7.1)
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **PSX SPU ADPCM predictor-filter coefficients per 4-bit predict flag: 0 {0,0}, 1 {60,0}, 2 {115,-52}, 3 {98,-55}, 4 {122,-60} — Godot's native core carries the identical table.** — `[R] 1/3`
  - R: `smd-player/src/shared/fft_spu_voice_tools.cpp` (`kAdpcmFilter`) + SPU hardware spec; side-by-side per doc §7.2
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **PSX SPU ADSR attack step: with attack-mode-exp set and the envelope ≥ 0x6000 the attack rate bumps +8; the envelope advances via the rate-indexed denominator/numerator tables, saturates at 32767 into DECAY, and the output is `envelope_vol >> 5` — Godot additionally clamps `rate + 8` to 127 where PCSX-Redux does not (never engages for cure, `attack_rate = 0`).** — `[R] 1/3`
  - R: `smd-player/src/shared/fft_adsr_envelope.cpp` (`kAdsrExponentialThreshold = 0x6000`, `min(rate + 8, 127)`) vs `vendor/pcsx-redux/src/spu/adsr.cc` (doc §7.3)
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **The PSX SPU ADSR decay step's exp/lin choice is gated on the release-mode-exp bit, not a separate decay-mode flag: exp → `env += (numDec[rate] * env) >> 15`, lin → `env += numDec[rate]`, with `rate = decay_rate * 4` — a PCSX-Redux quirk Godot ported faithfully.** — `[R] 1/3`
  - R: `smd-player/src/shared/fft_adsr_envelope.cpp` (`release_mode_exp` decay gate) vs `vendor/pcsx-redux/src/spu/adsr.cc` (doc §7.4)
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **PSX SPU sample-mix pipeline constants: `mixed = sample * env_vol / 1023`, then per-channel `out = mixed * vol_LR / 0x4000` (floor division both); the SPU sinc (interpolation shift) is `raw_pitch << 4`, with Godot clamping pitch to ≥ 1 before the shift.** — `[R] 1/3`
  - R: `smd-player/src/shared/fft_spu_mix_tools.cpp` (`floor_div_i64(..., 1023)`, `/ 0x4000`) + `smd-player/src/shared/fft_spu_voice_runtime.cpp` (`sinc = clamp(raw_pitch, 1, volume_max) << 4`) vs `vendor/pcsx-redux/src/spu/spu.cc` (doc §7.5–7.7)
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **PSX SPU ADPCM block-flag semantics: `flags & 0x01` terminates the block — loop iff `flags == 3` (LOOP_START|LOOP_END) with a loop address set, else the voice stops after the block; `flags & 0x04` (LOOP_START) sets the loop address to that block's start (PCSX-Redux stores `start - 16` because start has already advanced past the block).** — `[R] 1/3`
  - R: `smd-player/src/shared/fft_spu_voice_tools.cpp` (`flags == 3 && loop_addr > 0` loop test; LOOP_START clamp) vs `vendor/pcsx-redux/src/spu/spu.cc` (inverted stop/loop check, same result; doc §7.8–7.9)
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **The 1024-value Gaussian interpolation table is byte-identical between PCSX-Redux (`vendor/pcsx-redux/src/spu/gauss.h`) and Godot's native core (`smd-player/src/shared/detail/fft_gauss_table.inc`) — all values match.** — `[R] 1/3`
  - R: one-shot byte diff of the two table files (2026-05-12, doc 2026-05-12 update); no dedicated named test
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
## Notes

(empty — user territory)

## Related

- [[Event Sound OpCodes]]
- [[FEDS Sound Definition Format]]
- [[Effect Sound Timing]]
- [[PSX Pitch Conversion]]
- [[PSX SPU Reverb]]
- [[KON KOFF Mask Dispatch]]
- [[Effect Sound Audio Divergence]]
