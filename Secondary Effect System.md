# Secondary Effect System

FFT renders a family of in-memory "secondary effects" — arrow and stone projectiles, the reflect barrier, spell-charge and summon-charge animations, golem block, splash, poof, and venom-trap effects — that are separate from the 196-byte emitter/particle system stored in E###.BIN's Particle System section. Charge-effect routing (0x801b84ac anim→type table, 0x801b8900 type→handler jump table, and the dispatch loop) is documented in [[Spell Charge Effect System]]; this note records the projectile/barrier model pointers and the full effect-type list from the 2026-04-16 VFX additional findings (FFT Hacktics wiki).

## Points

- **Secondary effect models are in-memory VFX distinct from the 196-byte emitter/particle system in the E###.BIN Particle System section: arrow projectile at 0x801b69dc, stone projectile at 0x801b6d00, reflect barrier at 0x801b6f00, alongside spell-charge, summon-charge, golem-block, splash, poof, and venom-trap effect types.** — `[S] 1/3`
  - S: model pointers 0x801b69dc / 0x801b6d00 / 0x801b6f00 and effect-type list, per FFT Hacktics wiki via `research/working_documents/VFX_ADDITIONAL_FINDINGS.md`
  - R: none — no 0x801b69dc / 0x801b6d00 / 0x801b6f00 model pointers in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/VFX_ADDITIONAL_FINDINGS.md`

## Notes

(empty — user territory)

## Related

- [[Spell Charge Effect System]]
- [[Particle Emitter Format]]
- [[Effect System Index]]
