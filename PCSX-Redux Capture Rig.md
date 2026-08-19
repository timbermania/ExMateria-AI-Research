# PCSX-Redux Capture Rig

Gotchas verified live (E173 Night-Sword session, 2026-07-14) when driving PCSX-Redux programmatically with savestates, Exec BPs, and live-memory polling: how effect-editor vs GUI-slot savestates must be loaded, how to time frames after a `loadSaveState`, and which addresses are safe to breakpoint versus poll.

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

## Notes

(empty — user territory)

## Related

- [[Scenario Beat Capture]]
- [[Screen Effect Gradient System]]
- [[Lua Effect Editor]]
