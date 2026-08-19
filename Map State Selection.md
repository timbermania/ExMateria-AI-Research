# Map State Selection

A FFT map (GNS) ships multiple (arrangement, time, weather) state rows, each with its own sky gradient, ambient/directional lights, and map palette; a scenario picks exactly one row at battle setup by matching the scenario's RAW weather int + night flag against the row's `weather_raw` — never by label, because the scenario weather enum and the GNS weather enum are offset by a NoneAlt slot, so a label match loads the wrong sky (ADR-0056). The selector at `0x800f3f94` writes the resolved-lighting struct at `0x800f6aa0`. For MAP056 (Orbonne) every shipped palette file is warm in every weather/night row — the map's blue hue is not a palette-selection matter at all, it is the colour-engine `{33}` tint + `{66}` commit baked over the warm base. Godot mirrors the raw-int selection in `MapStateSelector` and loads byte-identical warm bases from the extracted palettes.

## Points

- **PSX resolves a scenario's map state by matching the scenario's raw weather int against the GNS state table — selector `0x800f3f94` writes the resolved-lighting struct at `0x800f6aa0`; scenario 4 (Orbonne, MAP056, weather_raw=2, day) lands on the GNS NORMAL row (ambient (73,61,46), cool dir-lights (148,158,158)/(130,160,160), blue sky). The scenario weather enum (0 None, 1 Normal, 2 Strong…) and the GNS weather enum (0 None, 1 NoneAlt, 2 Normal…) are offset by the NoneAlt slot, so label-based matching would load the wrong state.** — `[S·R] 2/3`
  - S: selector `0x800f3f94`, resolved-lighting struct `0x800f6aa0` (BATTLE.BIN, `battle_decompilation.c`; mapping closed in `WEATHER_MAP_STATE_MAPPING_INVESTIGATION.md`, cross-ref'd in doc §7)
  - R: `godot-learning/src/map/MapStateSelector.gd` `select` (raw-int match on `weather_raw`+`night` within `arrangement_id`, asymmetric fallback mirroring the ROM selector), validated by `godot-learning/tests/MapStateSelectorTest.gd` (weather_raw=2 → wx2 row; day/night split; default-row fallback)
  - src: `research/working_documents/MAP_HUE_WEATHER_STATE_CLUT_BAKE.md`

- **All shipped MAP056 map palettes are warm — a full-row scan of `palettes.json`, `palettes_c92f8c16…`, and `palettes_f1944d…` (which differ only in rows 5–7, water/sky) finds no blue terrain palette, and day states differ by ≤1/256 colors — so the per-weather difference lives in the GNS lighting/ambient/background block, not the map palette file; the map's blue hue is entirely the colour-engine tint bake, not a palette-selection choice.** — `[S·R] 2/3`
  - S: GaneshaDx palette parse `vendor/GaneshaDx` (`MeshResourceData.cs:511-548`) over the MAP056 file bytes + full-row scan of the three shipped palettes JSON
  - R: `godot-learning/src/map/MapComposer.gd` `palette_texture` from `godot-learning/assets/maps/MAP056/palettes*.json` — the loaded base is byte-identical to the PSX un-tinted source (`0x800f6ab0`-equivalent warm), confirmed by the `tools/probe_map_palette_beat.gd` dump (SCN=4 TARGET_PC=49, 2026-07-09)
  - src: `research/working_documents/MAP_HUE_WEATHER_STATE_CLUT_BAKE.md`

## Notes

(empty — user territory)

## Related

- [[Color Tint Luma Modes]]
- [[Background Opcode]]
- [[Map Darkness Opcode]]
- [[Scenario Table]]
