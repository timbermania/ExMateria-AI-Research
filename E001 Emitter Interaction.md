# E001 Emitter Interaction

E001.BIN's (Cure effect's) seven-emitter particle system shows cross-emitter dependencies: in the 2026-04-16 PCSX-Redux isolation session (emitters soloed by zeroing `particle_count_base`, breakpoint at the 0x801a634c spawn function), E2 was the only emitter that renders particles when solo and the recommended one for isolated testing; E0+E1 together produce a trail that neither shows alone; E3 is a modifier that adds a tail/trail to E2's particle. `particle_count_base` changes *when* particles appear (base=1 → early single long-lived particle, base=20 → particles only at the end of the effect), base=target=20 on E2 destabilizes the game, byte 0x03 (`animation_target_flag`) was later found to move the emitter's anchor position (0x04 → caster, 0x06 → each target; the original 'no visible effect' observation is superseded), and E2's particles moved despite its byte 0x02 reading 0x00.

## Points

- **E001's E2 is the only emitter that renders visible particles when solo; all six other E001 emitters are invisible when isolated.** — `[D] 1/3`
  - D: PCSX-Redux emitter-isolation session at breakpoint 0x801a634c (particle spawn function), 2026-04-16: each emitter soloed with all other emitters' `particle_count_base` set to 0
  - src: `research/working_documents/E001_EMITTER_INTERACTION_FINDINGS.md`
- **When both E0 and E1 are active on E001, a trail of particles appears that neither emitter produces when soloed.** — `[D] 1/3`
  - D: PCSX-Redux emitter-isolation session at breakpoint 0x801a634c (particle spawn function), 2026-04-16: emitters 0+1 enabled together, all others disabled
  - src: `research/working_documents/E001_EMITTER_INTERACTION_FINDINGS.md`
- **E001's E3 is invisible when soloed but adds a visible tail/trail effect to E2's particles when both are active — a base/modifier emitter relationship.** — `[D] 1/3`
  - D: PCSX-Redux emitter-isolation session at breakpoint 0x801a634c (particle spawn function), 2026-04-16: emitters 2+3 enabled together, all others disabled
  - src: `research/working_documents/E001_EMITTER_INTERACTION_FINDINGS.md`
- **Raising E2's `particle_count_base` from 1 to 20 changes when E2's particles appear: base=1 spawns a single particle early with long duration, while base=20 spawns particles only near the end of the effect (after all spawn breakpoints complete).** — `[D] 1/3`
  - D: PCSX-Redux emitter-isolation session at breakpoint 0x801a634c (particle spawn function), 2026-04-16: E2 soloed with base=1 vs base=20
  - src: `research/working_documents/E001_EMITTER_INTERACTION_FINDINGS.md`
- **Changing E2's `animation_target_flag` (target values 0, 4, or 10) had no visible effect on E001 particle rendering.** — `[D] 1/3`
  - D: PCSX-Redux emitter-isolation session at breakpoint 0x801a634c (particle spawn function), 2026-04-16: E2 soloed with target=0/4/10
  - src: `research/working_documents/E001_EMITTER_INTERACTION_FINDINGS.md`
  - ⚠ SUPERSEDED (2026-08-17) by: changing byte 0x03 moves the emitter itself to a new anchor position (0x04 → caster unit, 0x06 → each target) and particles spawn relative to the emitter
- **Setting E2's `particle_count_base` and target both to 20 destabilizes the game on E001.** — `[D] 1/3`
  - D: PCSX-Redux emitter-isolation session at breakpoint 0x801a634c (particle spawn function), 2026-04-16: E2 soloed with base=20, target=20
  - src: `research/working_documents/E001_EMITTER_INTERACTION_FINDINGS.md`
- **E2's particles moved visibly during testing even though E2's byte 0x02 (motion_type) is 0x00 ("static"), calling that field's "static" interpretation into question.** — `[D] 1/3`
  - D: PCSX-Redux emitter-isolation session at breakpoint 0x801a634c (particle spawn function), 2026-04-16: E2 soloed, particle movement observed
  - src: `research/working_documents/E001_EMITTER_INTERACTION_FINDINGS.md`
- **With all E001 emitters disabled (`particle_count_base` = 0), the Cure's character still flashes blue, so the character tint is not part of the particle emitters.** — `[D] 1/3`
  - D: PCSX-Redux emitter-isolation session at breakpoint 0x801a634c (particle spawn function), 2026-04-16: all-emitters-disabled combination test
  - src: `research/working_documents/E001_EMITTER_INTERACTION_FINDINGS.md`

## Notes

(empty — user territory)

## Related

- [[E001.BIN Memory Mapping]]
- [[Particle Emitter Format]]
- [[Particle Runtime State]]
