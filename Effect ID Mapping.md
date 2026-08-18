# Effect ID Mapping

FFT's E###.BIN battle effect ID space maps IDs to ability names across E000–E510, with the full mapping documented from ffhacktics.com/effects.php. Gaps in the numbering are unused slots, and E000/E509/E510 are crash slots. The godot-learning reimplementation re-encodes the mapping per ability in `AbilityDatabase.gd` (`effect_id` + `effect_file` fields).

## Points

- **The E### effect IDs map to ability names across E000–E510 (E001 Cure, E016 Fire, E019 Fire 4, E051 Drain, E259 Omnislash); the complete ID→name table is recorded in the source doc, sourced from ffhacktics.com/effects.php.** — `[R] 1/3`
  - R: `godot-learning/src/data/AbilityDatabase.gd` (every ability entry carries `effect_id` + `effect_file`, e.g. Cure2 → `E002.BIN`) + `godot-learning/tests/AbilityViewTest.gd` (validates `effect_id`/`effect_file` survive the ability-view pipeline)
  - src: `research/working_documents/EFFECT_ID_MAPPING.md`
- **Gaps in effect ID numbering (E037–E038, E042, E048, …) indicate unused/missing effect slots in the ROM's effect space.** — `[ ] 0/3`
  - R: none — "unused effect slot" not present in godot-learning (probed `godot-learning/src/data/AbilityDatabase.gd`; it simply has no entries in the gap ID ranges)
  - src: `research/working_documents/EFFECT_ID_MAPPING.md`
- **Effect IDs E000, E509, and E510 are "crashes game" entries in the effect ID space.** — `[ ] 0/3`
  - R: none — crash-game semantics not present in godot-learning (`AbilityDatabase.gd` references `E510.BIN` only as a placeholder `effect_file` for several abilities, no crash behaviour)
  - src: `research/working_documents/EFFECT_ID_MAPPING.md`

## Notes

(empty — user territory)

## Related

- [[Effect File Format]]
- [[Effect Execution Model]]
