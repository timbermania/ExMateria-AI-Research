# Effect File Format

The static on-disk specification for FFT's E###.BIN battle effect files. Each file is one of two variants — a pure DATA blob (first word a small offset) or a MIPS CODE executable with an embedded DATA section (first word the prologue `0x27BDXXXX`) — with the corpus split 291 DATA / 107 CODE. A 40-byte header of ten section pointers defines every section boundary, the particle-system and effect-flags sections have fixed small headers, and the runtime effect globals are initialised directly from those header fields. The file encodes two independent bytecode languages: the visual effect-script opcodes 0–45 (run by BATTLE.BIN) and the SMD sound opcodes 0x80–0xFF inside the FEDS sub-section.

## Points

- **E###.BIN files come in two on-disk formats, distinguished by the first 32-bit word: DATA when it is a small value (< 0x1000) and CODE when it matches 0x27BDXXXX (the MIPS prologue `addiu sp, sp, -X`); the corpus contains 291 DATA and 107 CODE files (398 total).** — `[S] 1/3`
  - S: format-detection table and word test, per `research/key_documents/EFFECT_FILE_FORMAT.md`
  - src: `research/key_documents/EFFECT_FILE_FORMAT.md`
- **The E###.BIN header is 40 bytes holding ten section pointers at offsets 0x00–0x24 — frames, animation, script bytecode, particle system (effect_data), animation curves, time scales (0 = unused), effect flags, timeline, sound definition, and texture data.** — `[S] 1/3`
  - S: header offset table, per `research/key_documents/EFFECT_FILE_FORMAT.md`
  - S: same pointer offsets 0x08–0x24 (script through texture), per `research/key_documents/SCRIPT_EDITOR_LESSONS.md`
  - src: `research/key_documents/EFFECT_FILE_FORMAT.md`
- **E###.BIN section boundaries are defined by adjacent header pointers (e.g. frames = animation_ptr − frames_ptr, script = effect_data_ptr − script_data_ptr, timeline = sound_def_ptr − timeline_section_ptr), and the time-scale section is a fixed 600 bytes (0x258) when its pointer is non-zero.** — `[S] 1/3`
  - S: section boundary table, per `research/key_documents/EFFECT_FILE_FORMAT.md`
  - src: `research/key_documents/EFFECT_FILE_FORMAT.md`
- **The Particle System section starts with a 20-byte header (constant 0x0002, emitter_count at +0x02, gravity at +0x04–0x0F, inertia threshold at +0x10) followed by 196-byte (0xC4) emitters at +0x14, with emitter_count = (anim_table_ptr − effect_data_ptr − 0x14) / 0xC4.** — `[S] 1/3`
  - S: particle-system header layout and emitter-count formula, per `research/key_documents/EFFECT_FILE_FORMAT.md`
  - src: `research/key_documents/EFFECT_FILE_FORMAT.md`
- **The Effect Flags section is 24 bytes: a flag byte (bit 3 TERRAIN_HEIGHT_ADJUST, bit 4 AUDIO_FADE, bit 5 TIME_SCALE_3PHASE, bit 6 TIME_SCALE_1PHASE) plus four sound channel configs (mode, sound_id_a, sound_id_b, reserved) at offsets 0x08, 0x0C, 0x10, 0x14.** — `[S·R] 2/3`
  - S: flag bit table and sound channel offsets, per `research/key_documents/EFFECT_FILE_FORMAT.md`
  - S: bits 3–6 are the only engine-read bits of the flag byte (bits 0–2/7 loaded but AND-masked away), proven exhaustively at read sites 0x801A1530/0x801A61E0/0x801A3BF8/0x801A4A5C, per `godot-learning/docs/adr/0092-effect-flags-author-on-the-effect-settings-surface-sound-channels-stay-in-their-container-view.md` (feature/effect-studio-authoring)
  - R: godot-learning/src/effects/studio/EffectFlagsSaver.gd → tools/write_effect_flags.py, validated by tests/EffectFlagsSaverTest.gd (tests/run_all_tests.sh) + tools/test_write_effect_flags.py (feature/effect-studio-authoring)
  - src: `research/key_documents/EFFECT_FILE_FORMAT.md`
  - src: `research/working_documents/EFFECT_STUDIO_COVERAGE.md`
- **CODE-format E###.BIN files begin with MIPS executable code followed by an embedded DATA section; the embedded header is located by signature (frames_ptr typically 0x28, pointers in ascending order) and its pointers are relative to the embedded header location.** — `[S] 1/3`
  - S: CODE detection and embedded-header rules, per `research/key_documents/EFFECT_FILE_FORMAT.md`
  - src: `research/key_documents/EFFECT_FILE_FORMAT.md`
- **The runtime effect globals are initialised from the loaded file header: sprite_def_table_ptr (0x801BBF78) ← frames_ptr + 4, effect_anim_tbl_ptr (0x801BBF7C) ← anim_table_ptr, timeline_channel_base (0x801BBF84) ← timeline_section_ptr + 8, effect_data_ptr (0x801BBF88) ← header[0x0C], animation_table_ptr (0x801BBF8C) ← animation_ptr + 4, timeline_section_ptr (0x801BC0C8) ← header[0x1C], effect_flags_ptr (0x801BACC8) ← header[0x18], time_scale_ptr (0x801B9258) ← header[0x14].** — `[S] 1/3 CONTESTED`
  - S: runtime global pointer table, per `research/key_documents/EFFECT_FILE_FORMAT.md`
  - src: `research/key_documents/EFFECT_FILE_FORMAT.md`

- **E019.BIN (Fire 4) full section map (52,100 bytes, DATA format, all bytes accounted for): header 0x000–0x028, frames 0x028–0x13C8 (5,024 B), animation 0x13C8–0x16D0 (776 B, 9 sequences), script 0x16D0–0x1710 (64 B), particle system 0x1710–0x21DC (2,764 B = 20 B header + 14 emitters × 196 B), animation curves 0x21DC–0x2B40 (2,404 B = 15 curves × 160 B), time scale 0x2B40–0x2D98 (600 B), effect flags 0x2D98–0x2DB0 (24 B), timeline + camera 0x2DB0–0x4638 (6,280 B), sound def/FEDS 0x4638–0x4780 (328 B), texture 0x4780–0xCB84 (33,796 B).** — `[S] 1/3`
  - S: E019.BIN byte reconciliation (every byte assigned to a section), per `research/key_documents/master_parser.py`
  - R: none — E019 section map not present in godot-learning (probed godot-learning/src, godot-learning/tests; effect-editor parses E###.BIN generically, no E019-specific map)
  - src: `research/working_documents/E_BIN_FIELD_EDITABILITY_INVENTORY.md`
- **E###.BIN encodes two independent bytecode languages: the visual effect-script opcodes 0–45 (particle spawning, animation timelines, camera, register arithmetic — executed by BATTLE.BIN's effect dispatcher, correctly absent from the sound runtime) and the SMD opcodes 0x80–0xFF inside the FEDS sub-section (sound dispatcher; the only E###.BIN opcodes relevant to smd-player).** — `[S·R] 2/3`
  - S: scope note and P2 analysis, per `research/working_documents/MASTER_PARSER_GAPS.md` (2026-05-27); visual-side jumptable 0x801B67C8 (BATTLE.BIN), SMD-side jumptable 0x80028B0C
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer/opcodes/_table.gd` (implements the SMD 0x80–0xFF set, header cites "FFT's smd_opcode_jumptable @ 0x80028B0C") + `godot-learning/tests/EffectSoundCaptureTest.gd` (drives the effect-sound path through it); visual-side parsing in `effect-editor/core/parser.lua` `M.SCRIPT_OPCODES` (no automated test)
  - src: `research/working_documents/MASTER_PARSER_GAPS.md`

## Notes

(empty — user territory)

## Related

- [[Effect Execution Model]]
- [[E001.BIN Memory Mapping]]
