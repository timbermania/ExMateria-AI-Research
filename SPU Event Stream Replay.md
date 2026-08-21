# SPU Event Stream Replay

Offline replay rig that makes the stream of inputs hitting the SPU the contract: a capture probe inside pcsx-redux records every SPU input (register writes/reads, DMA in/out, CD-XA injection, IRQ acks) plus an initial-state snapshot into `spu_events.bin`, and a library-agnostic replay driver dispatches the stream through independent SPU implementations (Mednafen-PSX first; pcsx-redux-as-library and Godot's native mixer the other planned arms) and diffs the audio, localizing divergence to the SPU implementation rather than the input stream. Phases 1/2/4/5 implemented 2026-05-15: capture probe + Lua triggers, the `mn_spu_*` Mednafen C library, the `spu_replay` driver with `--diff`, and redacted committed fixtures with a CI-style smoke test. Phase 3 (pcsx-redux SPU as a standalone library) is deferred — `PCSX::SPU::impl` is structurally fused with the emulator (settings, MainThread, MiniAudio, SaveState, nlohmann::json) — so capture losslessness remains unverified. First measured result (protect_no_music, 2026-05-15): both the Mednafen replay and Godot's mixer diverge from live PCSX by similar margins, corroborating that PCSX's SPU render is the outlier.

## Points

- **The Mednafen-PSX SPU is extracted as a standalone C library driving an offline replay pipeline: `tools/spu_replay/libspu_mednafen` exposes the `mn_spu_*` API (`reset(regs, ram)`, `write/read_reg`, `dma_in/out`, `xa_push`, `tick(cycles) -> samples`); `spu_replay.cpp` drives it with a `--diff` mode reporting first_divergence_sample / divergence_count / rms+peak error / cos_dist; `compare_pipeline.py` diffs the Mednafen replay against the live `spu_mix.wav`; the orchestrator triggers a lazy first build of `tools/spu_replay/` on the first run that finds `spu_events.bin` — no manual cmake/make step.** — `[D] 1/3`
  - D: `protect_no_music` orchestrator run (2026-05-15): replay produced `spu_mednafen.wav` (104,926 frames); sample-level diff vs the post-trim live mix — first divergence sample 823 (~18.6 ms in), 230,165/231,902 samples (99.2%) divergent, peak |error| 16,829, rms 2,228
  - D: `run_fixture_smoke.sh` fixture smoke test (2026-05-15): the committed redacted `protect_no_music.spu_events.bin` fixture (SPU RAM section zeroed by `redact_spu_ram.py`, format-preserving) parses and replays through Mednafen with asserted-silent output
  - R: none — replay rig lives in `tools/spu_replay/` + `vendor/pcsx-redux/`, not in godot-learning or smd-player (probed `godot-learning/src`, `godot-learning/tests`, `smd-player/addons/exmateria_sound`, `smd-player/src`, `fft-sound-driver`; `smd-player/workspace/orchestrator` only drives the capture/scoring harness)
  - src: `research/effect_sound/working_documents/SPU_EVENT_STREAM_REPLAY_PLAN.md`
- **The event stream carries no explicit TICK event: the replay driver advances the SPU by CPU-cycle delta between events (saves millions of redundant records), in PSX CPU clocks at 33.8688 MHz, which Mednafen converts internally to 44.1 kHz SPU sample clocks (~768 CPU clocks per SPU sample) — the only clock-domain translation the rig needs.** — `[D] 1/3`
  - D: `protect_no_music` `spu_events.bin` (2026-05-15): 3,441 events over a ~2.1 s span replayed to `spu_mednafen.wav` purely via cycle-delta ticks
  - R: none — no cycle-delta SPU tick present in godot-learning or smd-player (probed `godot-learning/src`, `godot-learning/tests`, `smd-player/addons/exmateria_sound`, `smd-player/src`, `fft-sound-driver`; smd-player renders on its own fixed cadence, not event-stream cycles)
  - src: `research/effect_sound/working_documents/SPU_EVENT_STREAM_REPLAY_PLAN.md`

## Notes

(empty — user territory)

## Related

- [[PCSX-Redux Capture Rig]]
- [[Effect Sound Audio Divergence]]
- [[SPU Voice Engine]]
