# SPU Voice Engine

The shared SCUS SPU voice engine behind all FFT sound playback: a small set of driver primitives (trigger voices, per-handle key-off, per-voice set-volume) that every SFX path — system SFX, ambient `{6B}` bg sounds, and the feds/SMD effect sounds — funnels into, gated by a single audio-enable flag. The `{6B}` BG Sound investigation (2026-07) established the full static + dynamic picture of these primitives on live hardware, and the Godot reimplementation mirrors the volume path exactly (`vol << 8`).

## Points

- **All FFT sound playback funnels through the shared SCUS SPU voice engine: `FUN_80013b20(mode, handle, 0x6000, 0x4000)` triggers voices (play), `FUN_80012990(handle)` keys off every active voice whose stored handle matches (per-handle stop, not global), and `FUN_80012b6c(handle, vol)` sets each matching voice's volume to `vol << 8` with `vol == 0` taking the same key-off path — all gated by the audio-enable flag `DAT_80032a54 & 0x1000`.** — `[S·D·R] 3/3`
  - S: `FUN_80013b20`, `FUN_80012990` (voice slots at `DAT_80032a60 + 0xb8 + n*0x160`), `FUN_80012b6c`, gate `DAT_80032a54` (`scus_decompilation.c`)
  - D: Orbonne battle live run — play trigger at `FUN_80013b20` with handles 0x10001/0x10012, `FUN_8004408c` set-volume firing once per frame, and the enable flag reading `0x9101` (gate set) (2026-07-01)
  - R: `godot-learning/src/audio/EffectSfxEngine.gd` `set_bg_gain` = `set_voice_volume_lr(voice, vol<<8, vol<<8)` (exact mirror of `FUN_80012b6c`) + `godot-learning/tests/ScenarioBgSoundTest.gd`
  - src: `research/working_documents/BGSOUND_OPCODE_6B_INVESTIGATION.md`
- **PSX SPU hardware ground truth behind FFT's sound driver: the 24 voice channels are all capability-identical (no hardware even/odd distinction), pitch modulation (PMON) is sequential — voice N−1 modulates voice N — not even/odd, PMON bit 0 is unused (voice 0 has no predecessor), and any channel can act as a modulator by zeroing its left/right volume while still generating its waveform.** — `[R] 1/3`
  - R: `smd-player/addons/exmateria_sound/runtime/spu.gd` (NUM_VOICES = 24; fmod mode = "this voice modulated by previous voice's") + `smd-player/addons/exmateria_sound/runtime/shared/opcodes/fmod_enable.gd` (0xB2: previous voice as frequency-channel modulation source) + `godot-learning/tests/EffectSoundCaptureTest.gd` (regression over the SPU engine incl. fmod)
  - src: `research/working_documents/INSTRUMENT_MAPPING.md`

## Notes

(empty — user territory)

## Related

- [[Event Sound OpCodes]]
- [[FEDS Sound Definition Format]]
- [[Effect Sound Timing]]
