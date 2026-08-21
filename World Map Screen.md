# World Map Screen

The FFT world map (overworld) state, target of every `post_scenario_step=0x80` scenario exit. RE maturity ~40% as of 2026-08-21: `WLDCORE.BIN`'s load base (flat at RAM `0x80067000`, confirmed by the file's baked absolute pointers) and the whole render pipeline — a flat 1:1 scrolled 8 bpp bitmap in 240×240-texel slabs, a two-layer CLUT-ramp aperture over a distance field, pinned HUD tpage/CLUT/uv — are solved offline from captures, and a savestate pair brackets the entire entry load. Still open: the load sequence, the node/tile index tables, a `WLDCORE.BIN` parser, and the `0x80` arrival fork at selector `0x801C3740`. No Godot counterpart yet.

## Points

- **A savestate pair brackets the entire world-map entry load: `world_map_ss0_scenario_end_prequicksave.sstate` is parked at a scenario end with the screen black and `WLDCORE.BIN` not resident anywhere in RAM, and `world_map_ss1_settled_dialog_closed.sstate` is 27.9 s later with it resident — the uncaptured `0x80` fork at selector `0x801C3740` lies strictly between the two states.** — `[D] 1/3`
  - D: PCSX-Redux savestate pair `reference-assets/world_map_ss0_scenario_end_prequicksave.sstate` + `world_map_ss1_settled_dialog_closed.sstate` (2026-08-21)
  - R: none — world map / `WLDCORE` not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`

- **`WLDCORE.BIN` loads flat at RAM `0x80067000` (confirmed by the file's own baked absolute pointers), and the world-map render is solved end to end: the map is a flat 1:1 scrolled 8 bpp bitmap stored in 240×240-texel slabs, the view aperture is a two-layer CLUT ramp over a distance field applied once per frame, and every HUD element's tpage/CLUT/uv is pinned.** — `[S·D] 2/3`
  - S: static check of the absolute pointers baked in the ROM file `WLDCORE.BIN`, pinning the `0x80067000` load base (reported in doc §4.3; byte-level detail in `research/working_documents/WORLD_MAP_SCREEN.md`)
  - D: dynamic measurement + offline RAM analysis on the world-map savestate pair (2026-08-21)
  - R: none — `WLDCORE` / world-map render not present in godot-learning (probed godot-learning/src, godot-learning/tests; only an unrelated BATTLE.BIN-base constant in `tests/CinematicPoseLUTTest.gd`)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`

## Notes

(empty — user territory)

## Related

- [[Scenario Transition Graph]]
- [[Scenario Table]]
- [[Inter Scene Orchestration]]
- [[CDROM DMA Load Pipeline]]
