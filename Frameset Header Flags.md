# Frameset Header Flags

The 2-byte `header_flags` field at frameset offset 0x00 in the FRAMES section of E###.BIN is confirmed dead data: no BATTLE.BIN code reads frameset+0x00. The only runtime path into FRAMES data is `sprite_def_table_ptr` (0x801BBF78), with exactly three references (two init stores and one load in `render_particle_to_sprite`), and the only frameset field ever loaded at runtime is `frame_count` at +0x02. Framesets are resolved through a per-emitter group-offset table, each tick's frame pointer is `frameset_ptr + 4 + i×24`, and `submit_sprite_to_ordering_table` reads flags only from each 24-byte Frame at frame+0x00 (semi_trans_on 0x200, palette_id bits 0–3). A December 2024 corpus scan found header_flags == frame[0].flags for 91.5% of 2,594 framesets, and the godot-learning reimplementation bakes the field to JSON but never consumes it.

## Points

- **The frameset `header_flags` (offset 0x00) is dead data — BATTLE.BIN never reads frameset+0x00: `sprite_def_table_ptr` (0x801BBF78) has exactly 3 references (stores at 0x801a0f7c and 0x801a0fcc, one load at 0x801aa3c4) and the only frameset field loaded at runtime is `frame_count` at +0x02 (0x801aa3ec).** — `[S] 1/3`
  - S: sprite_def_table_ptr reference set (0x801a0f7c, 0x801a0fcc, 0x801aa3c4) and frameset+0x02 load at 0x801aa3ec, BATTLE.BIN disassembly, per `research/working_documents/FRAMESET_HEADER_FLAGS_ANALYSIS.md`
  - R: none — header_flags not consumed in godot-learning (tools/parse_effect.py bakes it into frames.json; no reader in src/ or tests/; probed godot-learning/src + godot-learning/tests)
  - src: `research/working_documents/FRAMESET_HEADER_FLAGS_ANALYSIS.md`
- **`render_particle_to_sprite` (0x801aa1f8) is the sole reader of `sprite_def_table_ptr`: it resolves a frameset through the per-emitter group-offset table (anim_state +0x1e `sprite_frameset_group_offset`, +0x1f `sprite_frame_index`) and builds the per-tick frame-pointer array as `frameset_ptr + 4 + (i × 24)` (0x801aa434–0x801aa448), skipping the 4-byte frameset header.** — `[S·R] 2/3`
  - S: frameset lookup 0x801aa3bc–0x801aa3ec and frame-pointer loop 0x801aa428–0x801aa464, BATTLE.BIN disassembly, per `research/working_documents/FRAMESET_HEADER_FLAGS_ANALYSIS.md`
  - R: godot-learning/src/effects/ActiveEmitter.gd (`frameset_group_offset` from `EffectData.frameset_group_offsets`) + godot-learning/src/effects/EffectParticleRenderer.gd (`effect_data.framesets[frameset_idx]`); no named validating test
  - src: `research/working_documents/FRAMESET_HEADER_FLAGS_ANALYSIS.md`
- **`submit_sprite_to_ordering_table` (0x801a5394) reads flags from each 24-byte Frame at frame+0x00 (0x801a55f4, 0x801a565c), never from the frameset header: 0x801a55fc masks 0x200 = the semi_trans_on bit (flag byte 1 bit 1) and 0x801a5664 extracts palette_id = flag byte 0 bits 0–3.** — `[S·R] 2/3`
  - S: 0x801a55f4, 0x801a55fc (andi 0x200), 0x801a565c, 0x801a5664 (andi 0xf), BATTLE.BIN disassembly, per `research/working_documents/FRAMESET_HEADER_FLAGS_ANALYSIS.md`
  - R: godot-learning/tools/parse_effect.py (`palette_id = flags_byte0 & 0x0F`, `semi_trans_on = flags_byte1 & 0x02`, FRAME_SIZE = 24) + godot-learning/src/effects/EffectParticleRenderer.gd (per-frame `semi_trans_on` selects the blend pass); no named validating test
  - src: `research/working_documents/FRAMESET_HEADER_FLAGS_ANALYSIS.md`
- **In a December 2024 scan of 64 DATA-format effect files, 91.5% of 2,594 framesets have `header_flags` == frame[0].flags; in the 8.5% of mismatches, `header_flags` often matches a later frame's flags (e.g. E009.BIN frameset 1: header 0x6e0, frame[0] 0x2a0, frame[1]/frame[2] 0x6e0) — the authoring tool populated the field inconsistently.** — `[D] 1/3`
  - D: frameset data scan across 64 DATA-format E###.BIN files (December 2024), per `research/working_documents/FRAMESET_HEADER_FLAGS_ANALYSIS.md`
  - R: none — mismatch statistics not present in godot-learning (probed godot-learning/src + godot-learning/tests)
  - src: `research/working_documents/FRAMESET_HEADER_FLAGS_ANALYSIS.md`

## Notes

(empty — user territory)

## Related

- [[Effect Texture Upload]]
- [[Effect File Format]]
- [[Effect Execution Model]]
- [[Particle Runtime State]]
