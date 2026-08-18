# Scenario Table

The scenario table in `EVENT/ATTACK.OUT` is the join row that ties a map, a battle song, and a unit deployment into one playable battle: 24-byte records, 490 slots whose last 10 are all-zero padding (480 real scenarios in retail US), keyed by the meaningful and unique `scenario_id` rather than the table index. The full record layout is decoded, validated semantically against FFTPatcher EntryEdit's label tables (every record's name/location matches its `map_id`), and ported into the Godot parser that emits `scenarios.json`. In the bigger event model one Event ID bundles cutscene script + battle setup + battle conditionals, and the battle song lives in the scenario record — not in the event script, which drives cutscene music through its own Play/Switch/End Song opcodes.

## Points

- **`EVENT/ATTACK.OUT` (retail US, ~125,956 bytes) holds several tables: the scenario table starts at file offset `0x10938` (0x1EA = 490 records, stride 24 bytes) and the deployment-zone table (start tiles/facing, deferred) at `0xBBD4` (0x300 = 768 records, stride 12 bytes).** — `[S·R] 2/3`
  - S: file offsets in retail US `EVENT/ATTACK.OUT` — ffhacktics ATTACK.OUT wiki, cross-checked against TacticsTemplateG `attack_out_data.gd`
  - R: `godot-learning/tools/parse_scenarios.py` reads the scenario table at these offsets, emits `godot-learning/assets/scenarios/scenarios.json`
  - src: `research/wiki_articles/attack_out_scenario_table.md`
- **A scenario record is 24 bytes: `scenario_id` u16 @0x00, `map_id` @0x02 (→ `MAP{map_id:03d}` GNS file number, identity map), `weather` @0x03 (0–3 = None/Normal/Strong/VeryStrong; 0x04/0x10/0x20 also seen — unmapped, `weather_raw` kept), `is_nighttime` @0x04, `music_file_one_id` @0x05 (primary song → `MUSIC_{id:02d}.SMD`), `music_file_two_id` @0x06, `entd_idx` u16 @0x07 (→ ENTD unit-deployment table), `first_squad_deployment_idx`/`second_squad_deployment_idx` u16 @0x09/0x0B (→ deployment-zone table), `flags` @0x11 (bit 0 = Ramza mandatory during deployment), `next_scenario_id` u16 @0x12 (story-flow link by scenario_id), `post_scenario_step` @0x14 (0x80 world map · 0x81 next scenario · 0x82 reset), `battle_conditionals_id` u16 @0x16 (→ BattleConditionals set; TacticsG mislabels this "event_script_id"); bytes 0x0D–0x10 and 0x15 not yet mapped.** — `[S·R] 2/3`
  - S: FFTPatcher EntryEdit resource tables (`EntryEdit/EntryData/PSX/*.xml`, authoritative labels) + `PatchHelper.cs` (reads `ScenariosRAM + id*24 + 22` for the conditionals id)
  - R: `godot-learning/tools/parse_scenarios.py` implements this layout, emits `godot-learning/assets/scenarios/scenarios.json`
  - src: `research/wiki_articles/attack_out_scenario_table.md`
- **The last 10 of the 490 scenario records (indices 480–489) are all-zero padding, so the retail US disc has 480 real scenarios; `scenario_id` is a meaningful FFT id, not the table index (81 records have `scenario_id != index + 1`), so the parser also stores the table `index` for index-based cross-references.** — `[S·R] 2/3`
  - S: retail US `EVENT/ATTACK.OUT` extract — scenario table @`0x10938`: records 480–489 all-zero, `scenario_id` strictly increasing, unique, 1–490
  - R: `godot-learning/tools/parse_scenarios.py` drops the all-zero tail and keys `scenarios.json` by `scenario_id` while retaining `index`
  - src: `research/wiki_articles/attack_out_scenario_table.md`
- **The scenario parse is validated semantically by joining the artifact to FFTPatcher EntryEdit's label tables — `ScenarioNames.xml` (keyed by `scenario_id`), `MapNames.xml` (by `map_id`, 0x00–0x7D), `MusicChoices.xml` (by song id, 0x01–0x63): every record lands on a real (non-"Unusable"/"Empty") ScenarioName whose implied location matches the record's `map_id`; spot-check scenario 1 → map 0x3E "Chapel of Orbonne Monastery" + song 0x33 "Pray" → name "Orbonne Prayer (Setup)".** — `[S·R] 2/3`
  - S: FFTPatcher `EntryEdit/EntryData/PSX/ScenarioNames.xml` (500 entries), `MapNames.xml`, `MusicChoices.xml`
  - R: `godot-learning/tools/parse_scenarios.py` → `godot-learning/assets/scenarios/scenarios.json` (artifact keyed by `scenario_id` for the join)
  - src: `research/wiki_articles/attack_out_scenario_table.md`
- **In the bigger FFT model (per FFTPatcher), an Event ID is a parallel-array index that selects both the cutscene event script (instructions in `Events.bin`) and the 24-byte scenario (battle-setup) record; the scenario's `battle_conditionals_id` @0x16 then names a BattleConditionals set — so one "Event" bundles cutscene script + battle setup (map/music/units) + conditionals.** — `[S] 1/3`
  - S: FFTPatcher `EntryEdit/PatchHelper.cs` (event → event-script + scenario parallel-array model)
  - src: `research/wiki_articles/attack_out_scenario_table.md`
- **FFT has two distinct music mechanisms: battle music comes from the scenario record's `music_file_one_id` (→ `MUSIC_{id:02d}.SMD`), while cutscene music is driven by event-script opcodes — `{84}` Play Song, `{22}` Switch Track, `{5E}` End Song, `{60}` Fade Sound — plus the global SFX instructions `{21}` Sound Effect (system bank) and `{6B}` BG Sound / `{6A}` Edit BG Sound (env bank).** — `[S·R] 2/3`
  - S: FFTPatcher `EntryEdit/EntryData/PSX/EventCommands.xml` (event-script opcode labels)
  - S: FFTPatcher `EventCommands.xml` master catalog param layouts — {84} Play Song Song:1, {22} Switch Track 1:1+Volume:1+Time:1, {5E} End Song Unknown:1, {60} Fade Sound Shift:1+Time:1, {21} Sound Effect Sound:2, {6A} Edit BG Sound Sound:1+Echo:1+Volume:1+Unknown:2, {6B} BG Sound Sound:1+Echo:1+Volume?:1+Unknown:1+Time?:1
  - R: `godot-learning/src/audio/SfxCatalog.gd` (system/env bank slot names for the {21}/{6B} instructions), consumed by `godot-learning/src/scenarios/ScenarioVM.gd` (`_SfxCatalog.name_for("env", …)` / `is_loop("env", …)`)
  - src: `research/wiki_articles/attack_out_scenario_table.md`
- **The retail US scenario table spans 480 real scenarios over 105 distinct `map_id`s (range 1–115) and 48 distinct `music_file_one_id`s (range 0–98 — all within the 100 present `MUSIC_00..99.SMD`, so no missing-song gap); `map_id` 0 and 53 are the only referenced-but-absent maps (empty map 0 padding + MAP053).** — `[R] 1/3`
  - R: `godot-learning/tools/parse_scenarios.py` → `godot-learning/assets/scenarios/scenarios.json` (retail US extract statistics)
  - src: `research/wiki_articles/attack_out_scenario_table.md`

## Notes

(empty — user territory)

## Related

- [[ENTD Unit Deployment Table]]
- [[Event Opcode Catalog]]
