# Isometric Coordinate System

FFT battle-effect particles live in a 3D isometric coordinate space: Y is the vertical elevation axis (negative = up, positive = down) and X/Z are the two diagonal axes forming the ground plane (X: negative = right, positive = left; Z: negative = left, positive = right, perpendicular to X in the isometric view). Positions are 16-bit signed little-endian FFT units at 28 units per tile. The axis conventions were verified on 2025-11-20 by writing start/end positions into E001.BIN emitter 2 at RAM 0x801C2984 and observing the spawn-point sweeps; godot-learning's parser implements the Y-flip and 28-units/tile scale when converting to Godot units, but not the X/Z diagonal orientations.

## Points

- **FFT battle-effect Y axis is vertical with negative = up and positive = down; X and Z form the ground plane (the two diagonal axes), so start/end Y alone produces pure vertical spawn motion.** — `[D·R] 2/3`
  - D: E001 emitter 2 runtime write test (2025-11-20): start_y=200 (down) at 0x801C299A, end_y=−200 (up) at 0x801C29A0 — spawn point started low, swept smoothly upward, ended high
  - R: `godot-learning/tools/parse_effect.py:66-76` (`convert_position`: "FFT: -Y is up, 28 units/tile" — negates Y when converting to Godot +Y-up units) + `godot-learning/tests/EffectSoundCaptureTest.gd` (drives E001 through the parsed pipeline)
  - src: `research/working_documents/VERIFIED_spawn_position_curve.md`
- **In the isometric view X is diagonal axis 1 (negative = RIGHT, positive = LEFT) and Z is diagonal axis 2 (negative = LEFT, positive = RIGHT), perpendicular to X in the isometric view.** — `[D] 1/3`
  - D: E001 emitter 2 runtime write tests (2025-11-20): start_x=−100/end_x=+100 at 0x801C2998/0x801C299E swept the spawn point right→left; start_z=−100/end_z=+100 at 0x801C299C/0x801C29A2 swept left→right
  - R: none — X/Z diagonal orientations not asserted in godot-learning (probed godot-learning/src/effects/ and tools/parse_effect.py; `convert_position` only flips Y)
  - src: `research/working_documents/VERIFIED_spawn_position_curve.md`

## Notes

(empty — user territory)

## Related

- [[Particle Curve Indices]]
- [[Particle Emitter Format]]
- [[Emitter Anchor Modes]]
