# PCSX-Redux Capture Rig

Gotchas verified live when driving PCSX-Redux programmatically with savestates, Exec BPs, and live-memory polling (E173 Night-Sword session, 2026-07-14; Orbonne tile-grid session, 2026-07-06): how effect-editor vs GUI-slot savestates must be loaded, how to time frames after a `loadSaveState`, which addresses are safe to breakpoint versus poll, how `takeScreenShot()` display-window geometry and pause/capture interplay behave, and which slice of a raw VRAM dump holds the 256×240 display frame.

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
  - R: `effect-editor/commands/savestate.lua` — `is_gzip_file` magic check (`0x1f 0x8b`) branches to `Support.File.zReader` + `loadSaveState` for gzip (no automated test)
  - src: `research/working_documents/scenario_1_captures/cinematic_seq_source_decode.md`
- **Scenario VRAM-dump frame captures are 16bpp555 with the 256×240 display buffer as the `[0:240]` rows × `[0:256]` cols slice of the dump bin — the Orbonne prayer settled frame (`psx_settled_buf0.png` from `last_run/vram_dump_orbonne_prayer_mid_dialog.bin`) measures Agrias (blue sprite) at native (107,158), frame centre (128,120).** — `[D] 1/3`
  - D: `last_run/vram_dump_orbonne_prayer_mid_dialog.bin` VRAM dump at `orbonne_prayer_mid_dialog.sstate` (2026-06-29)
  - R: none — VRAM-dump display-buffer slicing not present in godot-learning (probed `godot-learning/` for `vram_dump`/`16bpp555`; the slice lives in the research reproject tooling)
  - src: `research/working_documents/scenario_1_captures/handoff_camera_framing_v3.md`

## Notes

(empty — user territory)

## Related

- [[Scenario Beat Capture]]
- [[Screen Effect Gradient System]]
- [[Scenario Camera Framing]]
- [[Lua Effect Editor]]
- [[GTE World-to-Screen Transform]]
