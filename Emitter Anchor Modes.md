# Emitter Anchor Modes

FFT battle-effect emitters anchor particle spawn position (emitter_anchor_mode, byte 0x03 bits 1–3) and homing destination (target_anchor_mode, byte 0x02 bits 5–7) to one of 7 modes. The two unit-relative modes look similar but resolve differently at runtime: TARGET (0x06) snaps to the target unit's tile center on the map grid via get_anchor_position() @ 0x801a90d0, while TRACKED (0x0C) follows the target's actually-rendered 3D model position via get_unit_position() @ 0x8008cc48, so they diverge only when the model is off its tile (Lucavi/Zodiac boss transformations, jumps, knockback). In the 291-file / 1945-emitter corpus TARGET dominates spawn anchoring (64.2%); TRACKED is spawn-only (0% as a homing target) and appears in exactly 3 boss-transformation effect files. godot-learning implements the six non-TRACKED modes (target anchor = target unit's global position) but has no TRACKED enum value — a TRACKED flag falls through to WORLD.

## Points

- **TARGET anchor (emitter_anchor_mode value 3 / raw 0x06) resolves to the target unit's tile center on the map grid — x = tile_x × 0x1C + 0x0E, z = tile_z × 0x1C + 0x0E, y = terrain tile height — computed in get_anchor_position() at 0x801a90d0, and equals the model position for units at rest (most spells).** — `[S·R] 2/3`
  - S: 0x801a90d0 get_anchor_position and 0x801a6468 anchor dispatch switch (code-locations table + simplified C), plus anchor enum values per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - R: `godot-learning/src/effects/EffectManager.gd:69-101` (target anchor = target.global_position − caster, `attach_anchors_to_units` tracks the unit) + `godot-learning/src/effects/ActiveEmitter.gd:207-214` / `ParticleSubsystem.gd:541-547` (TARGET branches); validated by `godot-learning/tests/EffectSoundCaptureTest.gd` (E001 E2 is TARGET-anchored, byte 0x03 default 0x06)
  - src: `research/working_documents/TRACKED_VS_TARGET_ANCHOR.md`

- **TRACKED anchor (emitter_anchor_mode value 6 / raw 0x0C) resolves to the target unit's actually-rendered 3D model position, not its tile: the spawn routine fetches it into tracked_entity_ptr via get_unit_position() at 0x8008cc48 (call at 0x801a62d0) and adds it per axis in the TRACKED case at LAB_801a65e4 inside emitter_control_routine at 0x801a634c, so particles follow the model through transformation/jump/knockback animations that move it off the tile.** — `[S] 1/3`
  - S: 0x801a634c emitter_control_routine, 0x801a62d0 tracked_entity_ptr setup, 0x801a6468 anchor mode switch, 0x801a65e4 TRACKED case, 0x8008cc48 get_unit_position (disassembly excerpts in doc)
  - R: none — TRACKED not present in godot-learning: `godot-learning/src/effects/EffectEmitter.gd:5` `AnchorMode` enum is WORLD–CAMERA only and "TRACKED" falls through to the WORLD default in `get_emitter_anchor_mode()`; also probed smd-player/addons/exmateria_sound and fft-sound-driver (no TRACKED anchor); effect-editor/ui/particles_tab.lua exposes TRACKED only as an editable combo item
  - src: `research/working_documents/TRACKED_VS_TARGET_ANCHOR.md`

- **TRACKED is spawn-only: it is never used as a homing (target) anchor (0% in the 1945-emitter / 291-file corpus), and only 15 emitters in 3 effect files use it — E470 (Virgo's Glow), E479 (Stone Glow), E484 (Stone Glow, high altitude) — all Lucavi/Zodiac boss transformation effects where the model animates away from its tile position.** — `[ ] 0/3`
  - R: none — E470/E479/E484 and TRACKED not present in godot-learning (smd-player and fft-sound-driver also probed)
  - src: `research/working_documents/TRACKED_VS_TARGET_ANCHOR.md`

## Notes

(empty — user territory)

## Related

- [[Particle Emitter Format]]
- [[Particle Runtime State]]
- [[Effect System Index]]
