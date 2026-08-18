# Effect Animation Sequences

The animation section of an E###.BIN (header pointer 0x04) holds the sprite sequences that emitter anim_index values reference: a u32 sequence count, a u16 offset table, then one bytecode stream per sequence — 3-byte FRAME entries (frameset_index, duration, depth_mode) plus control opcodes 0x81 LOOP, 0x82 SET_OFFSET, 0x83 ADD_OFFSET. The opcode numbering below is verified against the master parser and both reimplementations; the 2026-08-17 editability inventory table mis-numbered them by one and was not recorded.

## Points

- **The animation sequence section (header[0x04]) is a u32 count followed by a u16 offset table, then per-sequence bytecode: opcodes 0x00–0x7F are FRAME entries (3 bytes: frameset_index, duration, depth_mode); 0x81 = LOOP (1 byte, restarts the sequence); 0x82 = SET_OFFSET (5 bytes: s16 offset_x, offset_y); 0x83 = ADD_OFFSET (3 bytes: s8 delta_x, delta_y); a sequence runs until its next offset-table entry.** — `[S·R] 2/3`
  - S: sequence bytecode dispatch (0x81 LOOP / 0x82 SET_OFFSET / 0x83 ADD_OFFSET / <0x80 FRAME, u32 count + u16 offset-table header), per `research/key_documents/master_parser.py`
  - R: `effect-editor/core/parser.lua` (M.SEQUENCE_OPCODES + parse_sequence_instruction) + `godot-learning/src/effects/ParticleAnimator.gd` (SET_OFFSET/ADD_OFFSET/FRAME/LOOP processing; no automated test)
  - src: `research/working_documents/E_BIN_FIELD_EDITABILITY_INVENTORY.md`

## Notes

(empty — user territory)

## Related

- [[Effect File Format]]
- [[Particle Emitter Format]]
- [[Effect Execution Model]]
- [[Lua Effect Editor]]
