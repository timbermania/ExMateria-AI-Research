# SPU Voice Engine

The shared SCUS SPU voice engine behind all FFT sound playback: a small set of driver primitives (trigger voices, per-handle key-off, per-voice set-volume) that every SFX path — system SFX, ambient `{6B}` bg sounds, and the feds/SMD effect sounds — funnels into, gated by a single audio-enable flag. The `{6B}` BG Sound investigation (2026-07) established the full static + dynamic picture of these primitives on live hardware, and the Godot reimplementation mirrors the volume path exactly (`vol << 8`). The 2026-04 MUSIC_41 synth-accuracy work (software SPU synth vs PCSX-Redux capture) pinned down the voice-level details: static sequential track-to-voice allocation, the `FUN_80017118` volume chain, ADSR sustain-direction bit and floor behaviour, and the SPU's Gaussian/ADPCM sample pipeline.

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
## Notes

(empty — user territory)

## Related

- [[Event Sound OpCodes]]
- [[FEDS Sound Definition Format]]
- [[Effect Sound Timing]]
- [[PSX Pitch Conversion]]
- [[PSX SPU Reverb]]
