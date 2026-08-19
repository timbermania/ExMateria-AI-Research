# Effect Animation Sequences

The animation section of an E###.BIN (header pointer 0x04) holds the sprite sequences that emitter anim_index values reference: a u32 sequence count, a u16 offset table, then one bytecode stream per sequence — 3-byte FRAME entries (frameset_index, duration, depth_mode) plus control opcodes 0x81 LOOP, 0x82 SET_OFFSET, 0x83 ADD_OFFSET. The opcode numbering below is verified against the master parser and both reimplementations; the 2026-08-17 editability inventory table mis-numbered them by one and was not recorded. The 2026-04-16 working document's start/end marker reading (0x82 = animation start, 0x81 = end) is recorded below as a low-evidence point; the 2/3 opcode model above remains current.

## Points

- **The animation sequence section (header[0x04]) is a u32 count followed by a u16 offset table, then per-sequence bytecode: opcodes 0x00–0x7F are FRAME entries (3 bytes: frameset_index, duration, depth_mode); 0x81 = LOOP (1 byte, restarts the sequence); 0x82 = SET_OFFSET (5 bytes: s16 offset_x, offset_y); 0x83 = ADD_OFFSET (3 bytes: s8 delta_x, delta_y); a sequence runs until its next offset-table entry.** — `[S·R] 2/3`
  - S: sequence bytecode dispatch (0x81 LOOP / 0x82 SET_OFFSET / 0x83 ADD_OFFSET / <0x80 FRAME, u32 count + u16 offset-table header), per `research/key_documents/master_parser.py`
  - S: the ADD_OFFSET (0x83) handler at LAB_801aa35c inside `render_particle_to_sprite` (0x801AA1F8) — sign-extends the s8 delta (lbu + sll 0x18 + sra 0x18), adds it to sprite_offset_x at struct +0x08 (Y at +0x0A), and advances frame_counter by 3 bytes — corroborates the 3-byte size and signed 8-bit deltas
  - R: `effect-editor/core/parser.lua` (M.SEQUENCE_OPCODES + parse_sequence_instruction) + `godot-learning/src/effects/ParticleAnimator.gd` (SET_OFFSET/ADD_OFFSET/FRAME/LOOP processing; no automated test)
  - src: `research/working_documents/E_BIN_FIELD_EDITABILITY_INVENTORY.md`
  - src: `research/working_documents/SPRITE_OFFSET_VS_VERTEX_POSITION.md`

- **The 2026-04-16 working document reads 0x82 as an animation START marker (bytes 1–4 = int16 initial screen-offset x/y, allowing multiple animations per set) and 0x81 as the sequence END marker that terminates the animation, with 0x83 as a move command (signed X/Y screen-space shift of the emitter's spawn point) — an alternative reading to this note's verified 0x81 LOOP / 0x82 SET_OFFSET / 0x83 ADD_OFFSET opcode set.** — `[ ] 0/3`
  - R: `godot-learning` `ParticleAnimator.gd` and `effect-editor` keep the 0x81 = LOOP, 0x82 = SET_OFFSET, 0x83 = ADD_OFFSET model; no start/end-marker parsing exists in the repo.
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
  - src: `research/working_documents/VFX_ADDITIONAL_FINDINGS.md`
  - src: `research/working_documents/VFX_PARTICLES_EMITTERS_DEEP_DIVE.md`

## Notes

(empty — user territory)

## Related

- [[Effect File Format]]
- [[Particle Emitter Format]]
- [[Effect Execution Model]]
- [[Lua Effect Editor]]
- [[Sprite Offset vs Vertex Position]]
