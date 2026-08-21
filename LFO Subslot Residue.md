# LFO Subslot Residue

LFO modulation state that FFT holds in main-RAM chan-blocks (not SPU registers) and that can survive savestates: when a savestate is captured mid-effect, active effect-pool slots carry non-zero LFO subslot accumulators that were armed by D7/D8/D9-family opcodes *before* the capture window, so Godot cannot reconstruct them from SMD bytecode alone — and SPU-side snapshots (`spu_initial_state.json`) cannot capture them in principle. On `haste_no_music` (2026-05-23) all four active slots (2–5, voices 18–21) carried such residue; its subslots validated as dormant (`active_dir=0x0000`), and the doc's LFO-residue-as-root-cause theory for the voice-21 divergence was displaced by the +48-cad timeline-driver lag (see [[Effect Sound Timing]]). smd-player now primes Godot's `lfo_blocks` from the `chan_lfo_residue.json` snapshot via `set_voice_lfo_subslot` (gated on the `active_dir` bit 0), and the paired `probe_lfo_subslot{0,1,2,3}_state` probes track subslot state per cadence.

## Points

- **FFT's LFO modulation state is a CPU-side simulation living in main-RAM chan-blocks, not in any SPU register: each 0x160-byte effect-pool chan-block holds four 32-byte LFO subslots at chan+0xE0 (pitch-LFO, D7 family) / +0x100 (volume-LFO, D8) / +0x120 (pan/pitch mode-2, D9 / 0xEC / 0xED) / +0x140 (reserved / mode-specific), and each subslot lays out sub+0x00 callback ptr, sub+0x04 accumulator (s32 phase position), sub+0x08 step_current (s32 per-tick increment), sub+0x0C step_source (reload value), sub+0x10 countdown (u16 samples until next inner tick), sub+0x12 inner_reload (u16), sub+0x14/0x16 delay counter/reload (u16), sub+0x18/0x1A depth/reload (u16, modulation amplitude), sub+0x1C mode byte (0=pitch, 1=vol, 2=pan/pitch), sub+0x1D jumptable idx, sub+0x1E active+dir flags (u16) — so SPU-side savestate snapshots (`diag_spu_initial_state.lua`'s `getVoiceInfo`-based 5-field routing rows and its `residue_snapshot.voices` SPU-register rows) carry no LFO subslot data anywhere, in principle.** — `[S·D·R] 3/3`
  - S: 0x160-byte chan-block stride + chan+0xE0/0x100/0x120/0x140 subslot offsets — `research/key_documents/E001_STRUCTURE.md` memory map (cited in doc §3.1); per-subslot field layout per `probe_lfo_subslot2_state.lua` source comments
  - D: `probe_lfo_subslot{0,1,2}_state.lua` per-cadence probes + `diag_chan_lfo_residue_snapshot.lua` → `pcsx/chan_lfo_residue.json` (all 32 subslot blocks, 4 × 8 slots; `haste_no_music`, snapshot_cadence_index=3, 2026-05-23)
  - R: `smd-player/src/native/fft_spu_mixer_native.cpp:337` (`set_voice_lfo_subslot` → `FFTSpuCoreRuntime::set_voice_lfo_subslot`, writing `voices_[idx].lfo_blocks[subslot]`; `std::array<FFTPitchLfoBlock, 4> lfo_blocks` at `smd-player/src/shared/fft_spu_voice_runtime.h:99`) + `smd-player/workspace/harness/render_effect_sound.gd:381` (primes subslots from the LFO-residue JSON, skipping subslots with `active_dir & 0x1 == 0`) + `fft-sound-driver/src/driver/channel_state.h:158-176` (per-channel arrays for 4 LFO subslots: countdown/depth/depth_delta/mode/active/step_source/step_current/accumulator, `LFO_SUB_SLOT_COUNT = 4`) — validated by the `probe_lfo_subslot{0,1,2,3}_state` paired entries in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/HASTE_VOICE_21_FMOD_LFO_RESIDUE_FIX.md`
- **At the `haste_no_music` savestate anchor (snapshot_cadence_index=3), all four active effect-pool slots carry non-zero LFO subslot residue: slot 2 / voice 18 (chan_base=0x80037458) subslot 0 mode 0 depth 256 accum 15859712 step 3964928 and subslot 1 mode 1 accum −671088624; slot 3 / voice 19 (0x800375B8) adds mode-2 subslot 2 (accum −914358272, step_source 57147392, countdown 49); slot 4 / voice 20, the FM modulator (0x80037718), carries mode-2 subslot 2 with accum −939524094, step_source −52195783, countdown 1; slot 5 / voice 21, the FM carrier (0x80037878), subslot 2 accum −375809637 step −53687091 countdown 24 — none of it replayable from the SMD bytecode, because no D7/D8/D9 re-arms it inside the capture window (`probe_opcode_d9_lfo` fires exactly once, at cadence 322, with depth=40 / subslot=2 — completely different parameters from the depth=256 residue), and the doc's 2026-05-23 status update records that these active LFO subslots validated with `active_dir=0x0000` (dormant), so they do not modulate voice 20's per-sample output.** — `[D·R] 2/3`
  - D: `diag_chan_lfo_residue_snapshot.lua` → `pcsx/chan_lfo_residue.json` (haste_no_music, snapshot_cadence_index=3, 2026-05-23) + `probe_opcode_d9_lfo` 1/1 PAIR @ cadence 322 (depth=40)
  - R: `smd-player/workspace/harness/render_effect_sound.gd:370-391` (reads `subslot_0..3` per slot and primes `mixer.set_voice_lfo_subslot(accumulator, step_current, step_source, countdown, inner_reload, depth, depth_reload, mode, active_dir)`, gated on `active_dir & 0x1`) + `smd-player/addons/exmateria_sound/runtime/effect_sound/probes/probe_emit.gd` (`emit_lfo_handler_inactive` emits the seeded residue for undriven slots) — validated by the `probe_lfo_subslot*_state` / `probe_chan_pitch_state` paired entries in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/HASTE_VOICE_21_FMOD_LFO_RESIDUE_FIX.md`

## Notes

(empty — user territory)

## Related

- [[Dormant Slot Residue]]
- [[LFO Sub-Slot Period Reset]]
- [[Savestate Residue Voice]]
- [[Effect Sound Audio Divergence]]
- [[Effect Sound Timing]]
- [[SPU Voice Engine]]
- [[Noise LFO PRNG]]
