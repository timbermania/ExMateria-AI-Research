# Caster Terrain Element Multiplier

Caster terrain modifies spell damage per element in BATTLE.BIN: a fixed ±0.25 scaling keyed on the caster's standing-tile terrain ID and the ability element, run inside the common spell-damage post-processing chain. Geometry-driven, not status-driven — but it sits in the same `FUN_8018877C` chain as the status overlay, so any encoder mirroring that chain needs terrain context.

## Points

- **`FUN_80186ED0` at `0x80186ED0` scales `Target_CurActData+0x4` (damage) by ±0.25 for specific caster-terrain × element pairs, reading the caster's standing-tile terrain ID via `FUN_8018E660`: on terrain `3` or `4`, a Fire ability (`0x80`) takes ×0.75 and a Lightning ability (`0x40`) takes ×1.25; on terrain `6` or `7`, an Ice ability (`0x20`) takes ×1.25; it runs in the `FUN_8018877C` chain between the damage finalizer and the status overlay and is not a status interaction.** — `[S] 1/3`
  - S: `FUN_80186ED0` at `0x80186ED0` (terrain read via `FUN_8018E660`; branch `a1 - 3 < 2` / `a1 - 6 < 2` in `battle_disassembly.txt`)
  - R: none — caster terrain × element damage scaling not present in godot-learning (no terrain element branch in `src/gpu/shaders/` or `tests/`)
  - src: `research/working_documents/status_element_interplay.md`

## Notes

(empty — user territory)

## Related

- [[Status Element Defense Interplay]]
