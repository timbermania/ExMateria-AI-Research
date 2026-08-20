# Map State Selection

A FFT map (GNS) ships multiple (arrangement, time, weather) state rows, each with its own sky gradient, ambient/directional lights, and map palette; a scenario picks exactly one row at battle setup by matching the scenario's RAW weather int + night flag against the row's `weather_raw` — never by label, because the scenario weather enum and the GNS weather enum are offset by a NoneAlt slot, so a label match loads the wrong sky (ADR-0056). The selector at `0x800f3f94` writes the resolved-lighting struct at `0x800f6aa0`. For MAP056 (Orbonne) every shipped palette file is warm in every weather/night row — the map's blue hue is not a palette-selection matter at all, it is the colour-engine `{33}` tint + `{66}` commit baked over the warm base. Godot mirrors the raw-int selection in `MapStateSelector` and loads byte-identical warm bases from the extracted palettes. The ROM builds a 16-bit composite key `(dn<<15)|(weather<<12)|arrangement` from event-script vars 0x22–0x24 and exact-matches it via `FUN_800f2618` mode 1; the rule was verified dynamically against the live Orbonne rain state (2026-07-02) and headful-verified in Godot (2026-07-03).

## Points

- **PSX resolves a scenario's map state by matching the scenario's raw weather int against the GNS state table — selector `0x800f3f94` writes the resolved-lighting struct at `0x800f6aa0`; scenario 4 (Orbonne, MAP056, weather_raw=2, day) lands on the GNS NORMAL row (ambient (73,61,46), cool dir-lights (148,158,158)/(130,160,160), blue sky). The scenario weather enum (0 None, 1 Normal, 2 Strong…) and the GNS weather enum (0 None, 1 NoneAlt, 2 Normal…) are offset by the NoneAlt slot, so label-based matching would load the wrong state.** — `[S·D·R] 3/3`
  - S: selector `0x800f3f94`, resolved-lighting struct `0x800f6aa0` (BATTLE.BIN, `battle_decompilation.c`; mapping closed in `WEATHER_MAP_STATE_MAPPING_INVESTIGATION.md`, cross-ref'd in doc §7)
  - D: live rain state `orbonne_rain_battle_active.sstate` on PCSX :8087 (2026-07-02): script var 0x23=2, var 0x24=0, composite var 0x22=0x2000; resolved-lighting struct at `0x800f6aa0` byte-exact amb(73,61,46)/top(135,138,149)/bot(0,8,16); background gradient at `0x800f6854` (base(128,128,128), top `0x800f6858`=(135,138,149), bottom `0x800f685c`=(0,8,16)); full-RAM scan shows only the NORMAL top triple resident (`0x800f6858`, `0x800f6aa3`, `0x800f7a44`)
  - R: `godot-learning/src/map/MapStateSelector.gd` `select` (raw-int match on `weather_raw`+`night` within `arrangement_id`, asymmetric fallback mirroring the ROM selector), validated by `godot-learning/tests/MapStateSelectorTest.gd` (weather_raw=2 → wx2 row; day/night split; default-row fallback)
  - R: headful verify (2026-07-03): godot-learning ScenarioPlayer scenario 4 (MAP056, weather_raw=2) renders the grey overcast sky/palette/lighting + rain, not the default cyan
  - src: `research/working_documents/MAP_HUE_WEATHER_STATE_CLUT_BAKE.md`

- **All shipped MAP056 map palettes are warm — a full-row scan of `palettes.json`, `palettes_c92f8c16…`, and `palettes_f1944d…` (which differ only in rows 5–7, water/sky) finds no blue terrain palette, and day states differ by ≤1/256 colors — so the per-weather difference lives in the GNS lighting/ambient/background block, not the map palette file; the map's blue hue is entirely the colour-engine tint bake, not a palette-selection choice.** — `[S·R] 2/3`
  - S: GaneshaDx palette parse `vendor/GaneshaDx` (`MeshResourceData.cs:511-548`) over the MAP056 file bytes + full-row scan of the three shipped palettes JSON
  - R: `godot-learning/src/map/MapComposer.gd` `palette_texture` from `godot-learning/assets/maps/MAP056/palettes*.json` — the loaded base is byte-identical to the PSX un-tinted source (`0x800f6ab0`-equivalent warm), confirmed by the `tools/probe_map_palette_beat.gd` dump (SCN=4 TARGET_PC=49, 2026-07-09)
  - src: `research/working_documents/MAP_HUE_WEATHER_STATE_CLUT_BAKE.md`

- **The GNS row match is an exact-equality on a live 16-bit composite key `(dn<<15)|(weather<<12)|arrangement`, built in the map-refresh case at `LAB_800f3f94` from event-script var 0x24 (&0x1), var 0x23 (&0x7) and var 0x22 (&0xfff), and compared by `FUN_800f2618` mode 1 (`==`; modes 2/3 are `<`/`>` for numeric record types like the 0x70 deep-dungeon counter) — no translation table, no bit remap** — `[S·D] 2/3`
  - S: `LAB_800f3f94` @ `0x800f3f94` (record stride `+= 0x14` at `0x800f41d0`), key build `0x800f3ff4`–`0x800f401c`, `jal FUN_800f2618` at `0x800f4044`, comparator `FUN_800f2618` @ `0x800f2618` (BATTLE.BIN, `project-assets/fft-rom/battle_disassembly.txt`)
  - D: rain sstate `orbonne_rain_battle_active.sstate` (PCSX :8087, 2026-07-02): composite var 0x22 = `0x2000` exact-matches the GNS NORMAL/day/Primary row
  - R: none — the 16-bit composite key / `FUN_800f2618` comparator not present in godot-learning (raw-match semantics mirrored by `MapStateSelector.gd`, see sibling point)
  - src: `research/working_documents/WEATHER_MAP_STATE_MAPPING_INVESTIGATION.md`

- **NONE_ALT (raw weather 1) is a real, distinct engine weather level: the GNS weather field is a full 3-bit code — the selector masks it `&0x7` so 0..4 all stay distinguishable and nothing folds 1→0; the GNS wiki field-C decode lists 0=NoWeather, 1=NoWeatherAlternate, 2=LightWeather, 3=NormalWeather, 4=HeavyWeather (our parser's NORMAL/STRONG/VERY_STRONG labels at 2/3/4 are mis-named vs Light/Normal/Heavy)** — `[S·R] 2/3`
  - S: `andi &0x7` at `0x800f400c` in the selector loop at `LAB_800f3f94` (BATTLE.BIN, `project-assets/fft-rom/battle_disassembly.txt`); GNS wiki field-C decode; GaneshaDx `GlobalEnums.cs` corroborates (editor mirror, not authoritative)
  - R: `godot-learning/src/map/MapStateSelector.gd` `select` keys on `weather_raw` 0-4 as raw ints (1 stays a distinct level), validated by `godot-learning/tests/MapStateSelectorTest.gd`
  - src: `research/working_documents/WEATHER_MAP_STATE_MAPPING_INVESTIGATION.md`

- **MAP056 (Orbonne) GNS ships 10 state rows, all PRIMARY arrangement, with per-row gradient TOP: DAY/NONE (160,232,239) bright cyan = the shipped INITIAL default, DAY/NONE_ALT (168,180,199), DAY/NORMAL (135,138,149) with amb(73,61,46), DAY/STRONG (99,99,98), DAY/VERY_STRONG (76,80,80); NIGHT/NONE top (37,43,73)** — `[D·R] 2/3`
  - D: MAP056 GNS row dump via the fft_exporter GNS parser over `project-assets/fft-rom/MAP056.{GNS,0..}` (2026-07-02)
  - R: `godot-learning/tests/MapStateSelectorTest.gd` `_test_map056_normal_sky` pins the wx=2 NORMAL top (135,138,149) against `godot-learning/assets/maps/MAP056/scene_manifest.json` (asset-gated)
  - src: `research/working_documents/WEATHER_MAP_STATE_MAPPING_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[Color Tint Luma Modes]]
- [[Background Opcode]]
- [[Map Darkness Opcode]]
- [[Scenario Table]]
- [[Event Variable File]]
