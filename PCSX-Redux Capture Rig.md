# PCSX-Redux Capture Rig

Gotchas verified live when driving PCSX-Redux programmatically with savestates, Exec BPs, and live-memory polling (E173 Night-Sword session, 2026-07-14; Orbonne tile-grid session, 2026-07-06): how effect-editor vs GUI-slot savestates must be loaded, how to time frames after a `loadSaveState`, which addresses are safe to breakpoint versus poll, how `takeScreenShot()` display-window geometry and pause/capture interplay behave, and which slice of a raw VRAM dump holds the 256×240 display frame. The 2026-05-24 HASTE_VOICE_21 doc adds the `silenceAllVoices` scope fact: it wipes SPU registers only, so CPU-side effect state — including the entity timeline-VM cursor — survives the orchestrator's start-of-run silencing after savestate load and is replayable. The 2026-05-21 ICE V21 phase-drift doc adds the audio-side gotcha: PCSX-Redux's SPU audio throttle is one-sided (`psxcounters.cc` `Counters::update` blocks via `waitForGoal` only when the CPU is ahead of audio), so the per-cadence sample rate varies with load (mean 233.571 samples/cad in the ice capture) and stretches every capture WAV ~4–7% relative to Godot's constant-rate render. A 2026-05-17 reraise first-light adds the R3000 load-delay-slot gotcha: a load's result is not in the destination register one instruction later (only from PC N+8 onwards), so probes must capture loaded values from memory, not the destination register. The 2026-05-15 protect ear-A/B adds a fidelity gotcha: against a real-PSX recording, PCSX-Redux renders `protect_no_music` ~30–50% slower than real hardware and attenuates voice 20's mix contribution, so Godot's render — not the PCSX capture — is the closer oracle for that effect.

## Points

- **The effect-editor's `nightsword.sstate` (its sstate triplet under `~/.config/pcsx-effect-editor/{savestates,bins,meta}/`, `meta = {effect_index:173, name:"nightsword"}`) is a RAW protobuf `PCSX-Redux SaveState v4` (magic `0a 1b 0a 17 …`), NOT gzip — load it with `PCSX.loadSaveState(f)` directly; wrapping it in `Support.File.zReader` crashes the load. GUI-slot savestates are gzip (magic `1f 8b`) and DO need the zReader path (the `linux-verified-methods.md` §2 pattern). Check the magic before loading any sstate.** — `[D·R] 2/3`
  - D: E173 nightsword capture rig (2026-07-14): raw sstate loaded directly over armed BPs; a zReader wrap crashes the load — matches the repeated "scenario sstate is RAW not gzip" memory warnings
  - R: `effect-editor/commands/savestate.lua` — `is_gzip_file` magic check (`:42`, `0x1f 0x8b`) branches to `zReader` + `loadSaveState(decompressed)` for gzip and `loadSaveState(file)` direct for raw (no automated test)
  - src: `research/working_documents/SCREEN_KEYFRAME_BLEND_GRADIENT_E173_NIGHTSWORD.md`
- **PCSX-Redux's heartbeat `vsync` counter freezes after `loadSaveState` (stuck at 441 across the whole E173 run while the CPU advanced) — time frames with `getCPUCycles()` and the target block's own counters (e.g. the `DAT_800a1b10` byte-2 ramp counter), never `vsync`, and do not conclude "dormant" from a frozen vsync.** — `[D] 1/3`
  - D: E173 nightsword capture (2026-07-14): vsync stuck at 441 while cycles climbed and the block's ramp counter incremented — the documented §6.5 freeze (`pcsx-agent/docs/linux-verified-methods.md`)
  - R: none — vsync post-load timing workaround not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`; pcsx-agent's heartbeat still reads `vsync`)
  - src: `research/working_documents/SCREEN_KEYFRAME_BLEND_GRADIENT_E173_NIGHTSWORD.md`
- **The screen-gradient setters are cool BPs — `screen_gradient_color_setter @0x80090048` and `FUN_8009349c @0x8009349C` fire once per keyframe advance, safe for non-pausing logging BPs (`return true`, no `pauseEmulator`) — while `color_unit_frame_tick @0x800912A4` is hot (fires every vblank): do not attach an Exec BP there, poll the block `DAT_800a1b18/30/1b10` each frame instead.** — `[D] 1/3`
  - D: E173 nightsword capture (2026-07-14): two non-pausing logging BPs on the setters logged the full 13-call setter sequence with no HOT-BP crash; the per-vblank ticker was sampled by free-run block polling
  - R: none — BP coolness is a property of the PSX call sites, not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/SCREEN_KEYFRAME_BLEND_GRADIENT_E173_NIGHTSWORD.md`

- **`takeScreenShot()` returns the VRAM display window (`DisplayPosition`..`DisplayEnd`, 256×240 for FFT), not the wider GPU drawing space where GTE coordinates live; the gte→screenshot-pixel mapping is `K = DrawOffset − DisplayPosition`, and because those GPU regs are not Lua-exposed, `K` is calibrated once by eye — for the Orbonne prayer camera pose `K = (−130, 0)` (Ovelia feet (159,161) ↔ gte (289,160); Agrias & Simon markers on-feet).** — `[D] 1/3`
  - D: Orbonne sstate marker calibration (2026-07-06): magenta unit markers must sit on unit feet; re-verify `K` if the camera moves
  - R: none — takeScreenShot display-window / `K` mapping not present in godot-learning or effect-editor (probed both; the capture path lives in the research/ reproject tool)
  - src: `research/working_documents/psx_tile_grid/reproject_tile_grid.md`
- **`takeScreenShot()` while the emulator is paused hard-deadlocks pcsx-redux (webserver + GUI hang; needs `kill -9` + relaunch), and a lone `PCSX.pauseEmulator()` exec wedges subsequent Lua execs — so the tool always resumes before capture, and pause+load+resume must happen in one exec (its `load_sstate`).** — `[D] 1/3`
  - D: Orbonne tile-grid session (2026-07-06): pause-capture deadlock hit; `load_sstate` does pause+load+resume in one exec
  - R: none — resume-before-capture guard not present in godot-learning or effect-editor (probed both; guard lives in the research/ reproject tool)
  - src: `research/working_documents/psx_tile_grid/reproject_tile_grid.md`
- **The repo's `reference-assets/` research savestates (e.g. `orbonne_prayer_mid_dialog.sstate`) are gzip — `PCSX.loadSaveState(Support.File.open(f))` silently no-ops (state not loaded, no error), so load via `PCSX.loadSaveState(Support.File.zReader(f))`.** — `[D·R] 2/3`
  - D: Orbonne chapel-prayer session (2026-06-25): raw `Support.File.open` tried first and silently no-opped the state; `zReader` load verified
  - R: `effect-editor/commands/savestate.lua` — `is_gzip_file` magic check (`0x1f 0x8b`) branches to `Support.File.zReader(f)` and `loadSaveState(decompressed)` for gzip
  - src: `research/working_documents/scenario_1_cinematic_seq_source_decode.md`
- **Scenario VRAM-dump frame captures are 16bpp555 with the 256×240 display buffer as the `[0:240]` rows × `[0:256]` cols slice of the dump bin — the Orbonne prayer settled frame (`psx_settled_buf0.png` from `last_run/vram_dump_orbonne_prayer_mid_dialog.bin`) measures Agrias (blue sprite) at native (107,158), frame centre (128,120).** — `[D] 1/3`
  - D: `last_run/vram_dump_orbonne_prayer_mid_dialog.bin` VRAM dump at `orbonne_prayer_mid_dialog.sstate` (2026-06-29)
  - R: none — VRAM-dump display-buffer slicing not present in godot-learning (probed `godot-learning/` for `vram_dump`/`16bpp555`; the slice lives in the research reproject tooling)
  - src: `research/working_documents/scenario_1_captures/handoff_camera_framing_v3.md`

- **PCSX-Redux's `silenceAllVoices` wipes SPU register state only, not CPU RAM — so the effect entity's animation-VM timeline cursor and any other CPU-side effect state survive the orchestrator's start-of-run silencing after savestate load, and are replayable (the layer-2 basis of the timeline-VM snapshot approach).** — `[ ] 0/3`
  - R: none — `silenceAllVoices` lives in `vendor/pcsx-redux/src/spu/spu.cc:1394-1410`, not in a reimplementation repo (probed smd-player and godot-learning for the term)
  - src: `research/effect_sound/working_documents/HASTE_VOICE_21_FAITHFUL_TIMELINE_VM_REPLAY.md`
- **PCSX-Redux's SPU audio throttle is one-sided: `Counters::update` calls `m_spu->waitForGoal(target)` only when `framesDiff > 0` (CPU ahead of audio) and never blocks when audio runs ahead of CPU, so the per-cadence sample rate is non-constant and jitters with instantaneous CPU/SPU load (mean 233.571 samples/cad in the ice capture, individual cadences ~220–250) — the source of the per-run capture-rate variance that the effect-sound orchestrator averages into a single integer `--samples-per-sub` for Godot.** — `[S·D] 2/3`
  - S: `vendor/pcsx-redux/src/core/psxcounters.cc` `Counters::update` — `waitForGoal(target)` guarded on `framesDiff > 0`, no inverse-direction block
  - D: `probe_cadence_wallclock` / `cadence_calibration.json` — 234 samples/cad from Δsample=353160 / Δcad=1512, five `ice_no_music` runs (2026-05-21)
  - R: none — the one-sided throttle is emulator-side, not present in godot-learning or smd-player runtime (Godot renders at a constant pinned rate; probed godot-learning/src, godot-learning/tests, smd-player runtime)
  - src: `research/effect_sound/working_documents/ICE_V21_COS_DIST_PHASE_DRIFT_INVESTIGATION.md`
- **PCSX-Redux strictly emulates the MIPS R3000 load-delay-slot: the result of an `lh`/`lhu` at PC N is not in the destination register at PC N+4, only at PC N+8 onwards — so an Exec BP placed one instruction after a load and reading the destination register returns stale residue from a previous channel iteration (the `probe_pitch_formula_stages` first-light on `reraise_no_music` produced a spurious divergence at BPs `0x80017344`/`0x80017348`/`0x8001734c` until the probe was re-anchored); loaded values must be captured from memory (`probe_read16`), while ALU results (post-`addu`/`andi`/`sra`) ARE safe to read from registers.** — `[S·D] 2/3`
  - S: pitch-formula load instructions at `0x80017340`/`0x80017344`/`0x80017348` (`project-assets/fft-rom/scus_disassembly.txt`) — the PCs whose destination registers read stale in the first-light
  - D: `probe_pitch_formula_stages` first-light capture vs post-fix capture — `reraise_no_music` (2026-05-17)
  - R: none — probe-side capture rule, not present in godot-learning (probed `godot-learning/src`, `godot-learning/tests`; the canonical workaround pattern is `smd-player/workspace/probes/probe_pitch_inputs.lua` reading `probe_read16(s0 + 0x80/0x86)` / `probe_read16(s2 + 0xa2)`)
  - src: `research/effect_sound/working_documents/PROBE_PITCH_FORMULA_STAGES_INTRODUCED.md`
- **PCSX-Redux is not a faithful audio oracle for `protect_no_music`: ear-A/B against a real-PSX recording (user-confirmed) shows PCSX renders the effect ~30–50% slower than real hardware and attenuates voice 20's mix contribution, while Godot's audio — which does not model the PCSX-side per-cadence `chan+0x92` zeroing at PC `0x80150AEC` — matches real hardware more closely; the voice-20 vol-probe FAILs on this session are emulator artifacts, not Godot defects (modeling the clear would silence the correct voice-20 sparkle at t≈1.85–2.10 s).** — `[D] 1/3`
  - D: ear-A/B against a real-PSX recording, user-confirmed (2026-05-15); durable note `project_protect_no_music_pcsx_borked.md`
  - R: none — real-hardware A/B oracle check not present in godot-learning or smd-player (probed godot-learning/src, godot-learning/tests, smd-player/addons/exmateria_sound)
  - src: `research/effect_sound/working_documents/PROTECT_CHAN_92_INIT_TABLE_PLAN.md`

## Notes

(empty — user territory)

## Related

- [[Scenario Beat Capture]]
- [[Screen Effect Gradient System]]
- [[Scenario Camera Framing]]
- [[Lua Effect Editor]]
- [[GTE World-to-Screen Transform]]
- [[Savestate Residue Voice]]
- [[Effect Sound Audio Divergence]]
- [[SPU Voice Engine]]
