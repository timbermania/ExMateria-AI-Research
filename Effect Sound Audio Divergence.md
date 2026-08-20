# Effect Sound Audio Divergence

State of the Godot effect-sound synth vs the PCSX-Redux capture for `cure_no_music` after the 2026-05-12 audio-parity final state: every probe-able boundary is aligned (per-voice SPU register writes bit-exact across 10 probe families, Layer B global SPU registers empty at runtime, Layer C same WAVESET bank at the same SPU addresses), so the residual `voice_21` audio divergence (cos_dist ≈ 0.45, the two signals not scaled/shifted variants of each other) lives in per-sample dynamic processes below the probe-able layer. Leading candidate: a 1-cadence (~4.17 ms) timing offset in the pitch-LFO walker pipeline — observed directly in the pitch-register write cadences (Godot fires one cadence early from the 2nd write onward, values bit-exact), which puts the two sides in different LFO phases on the cure tone's aggressively oscillating pitch register.

## Points

- **`cure_no_music` `voice_21` audio still diverges from the PCSX capture at cos_dist ≈ 0.45 and is not a scaled-or-shifted variant of it (best cross-correlation −91 at +792 samples vs +52 at zero shift; least-squares gain −0.0225): the 2026-05-12 run shows matching peaks (PCSX 0.5511 vs Godot 0.5517) with the dominant frequency off by ~40 Hz / ~37 cents (PCSX 1832 Hz vs Godot 1872.5 Hz) — a per-sample pitch-process difference, not a gain or alignment bug.** — `[D·R] 2/3`
  - D: `audio_diff_report.py` + cross-correlation run on the `voice_21` WAV pair (commit ba7acba0; 2026-05-12)
  - R: `smd-player/src/shared/` native SPU mixer (`fft_spu_core_runtime.cpp`, `fft_spu_pitch_runtime.cpp` et al.)
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **Godot fires the pitch-register writes one cadence (~4.17 ms / 184 samples) EARLIER than PCSX starting from the 2nd write, bit-exact on value (voice_21: 165/165 writes value-paired; row 0 matches exactly, rows 1..164 all `diff_cad = −1` — cadence sequences PCSX `1,3,6,8,…` vs Godot `1,2,5,7,…`); with the cure tone's pitch register oscillating aggressively between ~6800 and ~3050 under LFO/vibrato, the 1-cadence shift puts the two sides in different LFO phases at every transition.** — `[D·R] 2/3`
  - D: `probe_pitch_register` voice_21 capture (2026-05-12)
  - R: pitch staging/walker path `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` + `smd-player/src/shared/fft_spu_voice_runtime.h` (`FFTPitchUpdateScheduled` pending queue); validated by the `probe_pitch_register` pair
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **Godot's effect-sound LFO runs entirely in the GDScript engine (`channel.lfo_*` → `advance_lfo` → `pitch_bend` → pitch staging → SPU pitch register); the C++ `FFTPitchLfoBlock` machinery (`fft_spu_lfo_tools.cpp`, `mixer.init_voice_pitch_lfo`) is only reachable from the DAW/VST3 build, so `lfo_blocks[0].enabled == false` for all effect voices and `fft_effective_voice_sinc()` is a no-op on that path.** — `[R] 1/3`
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd` (GDScript LFO engine) + `smd-player/addons/exmateria_sound/runtime/sequencer/opcodes/pitch_lfo_init.gd` (Pass 7.D.d — Godot no longer calls `mixer.init_voice_pitch_lfo`; the C++ engine stays in `smd-player/src/shared/fft_spu_lfo_tools.cpp` for the VST3 build)
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **The `cure_no_music` playback timeline (cadence = SPU IRQ tick): cadence 0 dispatches both the silent driver Note (voice 20, `delta_time = 4`) and the audible Note (voice 21, `delta_time = 144` — the long first cure tone); the first KON v20+v21 lands at cadence 1 (registers seeded via the 0x1FF walker mask); KOFF v20 at cadence 349 (silent driver pre-Rest release, fired by the idle-timeout drain) and again at 351 when the Rest opcode dispatches; KOFF v21 at 359; the next note's KON is off-by-one at 361 (Godot) / 362 (PCSX).** — `[D·R] 2/3`
  - D: probe-derived timeline (2026-05-11/12; commit fbfe2830 — idle_timeout drain fires KOFF, resolving the cad 349/359 directional inversions)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` + note-handler idle-timeout drain (KOFF on drain); validated by the `probe_kon_koff_mask` / `probe_pitch_register` pairs
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`

## Notes

(empty — user territory)

## Related

- [[Effect Sound Parity Ladder]]
- [[SPU Voice Engine]]
- [[WAVESET Instrument Bank]]
- [[KON KOFF Mask Dispatch]]
- [[Effect Sound Timing]]
- [[SFX Index]]
