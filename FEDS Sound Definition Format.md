# FEDS Sound Definition Format

The "feds" sound definition section (header[0x20]) of E###.BIN files does not contain raw audio: it holds SMD-like sequenced instructions interpreted at runtime by the sound engine, with a header (magic, size, pair count, resource ID, data offset, linked-list pointer), a channel offset table readable pair-major by FFT or channel-major by tools, a per-sound-ID env-multiplier table (chan_92_init), and a verified opcode set including the newly identified D0/D1 pitch-bend opcodes used by Cure's "rising shimmer".

## Points

- **The sound definition section (header[0x20], magic "feds") contains SMD-like sequenced instructions, NOT raw VAG/ADPCM samples; it is interpreted at runtime by the sound engine, with a jump table at 0x80028B0C and handlers (0x80015874–0x80016680) that return input_ptr + params consumed.** — `[S] 1/3`
  - S: jump table 0x80028B0C and handler return convention, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **FEDS header fields: magic "feds" at 0x00, data_size at 0x04, pair_count_plus1 at 0x08 (num_channels = (field−1)×2 — E001 field=3 → 4 channels, E019 field=4 → 6), resource_id at 0x0A, data_offset at 0x0C (FFT reads `lhu v0, 0xc(s5)` at PC 0x80013BEC; upper 16 bits zero across all 512 E*.BIN), list_next_ptr at 0x10 (zeroed per-effect by FUN_801A18EC at PC 0x801A1944; non-zero only for the SCUS-static blocks DAT_80047610/DAT_8004D9B8).** — `[S] 1/3`
  - S: FEDS header table 0x00–0x13, PC 0x80013BEC, FUN_801A18EC PC 0x801A1944, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - S: E001 raw header values (resource_id = 2, data_offset = 0x20, 0x10–0x17 zero), per `research/working_documents/E001_FEDS_HEXDUMP.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The channel offset table is one memory with two views: pair_record[0] sentinel [0,0] at 0x14 (sid=0; FFT skips emit when sound_id < 2), pair_record[1..N] at 0x18 = [ch_a_offset, ch_b_offset], read pair-major at +0x14 + sid×4 by FFT (FUN_80013B20, PC 0x80013C34–0x80013C40) or channel-major at +0x18 + ch×2 by tools; there is no end marker — channel N's size is offset[N+1] − offset[N] and the last channel runs to data_size.** — `[S] 1/3`
  - S: channel offset table dual view, FUN_80013B20 PC 0x80013C34–0x80013C40, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - S: E001 raw channel offsets 0x23/0x59/0x82/0xB8, per `research/working_documents/E001_FEDS_HEXDUMP.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **chan_92_init[] sits at data_offset (pair_count_plus1 bytes, one instr_byte per sound ID 0..N) and FFT computes `chan_92 = min((instr_byte × 0x6000) >> 7, 0x7FFF)` (PC 0x80013C04) with a saturation clamp when instr_byte > 0xAA (PC 0x80013C2C); two layout patterns exist — a dedicated block (channel_offsets[0] > data_offset + num_sids, e.g. cure_4) and an overlapping block (e.g. original E001, where the ASCII "PTT" bytes 0x50/0x54/0x54 ARE chan_92_init[0..2] → chan_92 15360/16128/16128).** — `[S] 1/3`
  - S: chan_92_init layout, PC 0x80013C04/0x80013C2C, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The verified FEDS opcode set (param counts from handler returns): Note 00–7F (1 param), Rest 80 (1, 0x80015874), Fermata 81 (1, 0x8001589C), NOP 82 (0, 0x8001586C), EndBar 90 (0, 0x800158F8), Loop 91 (0, 0x800159DC), Octave 94 (1, 0x80015A10), RaiseOctave 95 (0, 0x80015A28), LowerOctave 96 (0, 0x80015A40), Repeat 98 (1, 0x80015AB8), Coda 99 (0, 0x80015B00), Tempo A0 (1, 0x80015CB0), Instrument AC (1, 0x80015DD0), Flag_0x800 B0 (0, 0x80015EA8), ReverbOn BA (0, 0x800160E4), ReverbOff BB (0, 0x80016110), Release C4 (1, 0x800161E0), Dynamics E0 (1, 0x80016614), Expression E2 (1, 0x80016680).** — `[S] 1/3`
  - S: SMD-like opcode table with handler addresses, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Pitch bend opcodes D0 SetPitchBend (0x800162B4) and D1 AddPitchBend (0x800162D8) take a 1-byte param that is sign-extended and scaled ×32 (sll 24 / sra 19) and stored to channel +0x86 — D0 sets the value, D1 adds to the current one; Cure uses them for the ascending "rising shimmer" sparkle.** — `[S] 1/3`
  - S: D0/D1 handler disassembly at 0x800162B4/0x800162D8, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Resource lookup: resource_id (0x0A) << 16 forms the lookup key in g_sound_data_base (0x801BC0DC, verified at 0x801A1158), and play_sound(address) matches the address's upper 16 bits against registered resource_id fields while walking g_sound_resource_list (0x80032A00) in FUN_80013B20; init_sound_section (0x801A10EC), register_sound_resource (0x80017E7C), unregister_sound_resource (0x80017EB8), g_sound_section_ptr (0x801BBF74).** — `[S] 1/3`
  - S: resource lookup at 0x801A1158, g_sound_resource_list 0x80032A00, runtime globals 0x801BBF74/0x801BC0DC, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **In E001's feds section, the channel SMD data regions are 0x23–0x58 (channel 0, 54 bytes), 0x59–0x81 (channel 1, 41 bytes), and 0x82–0xB7 (channel 2, 54 bytes), with channel 2's data being a byte-for-byte copy of channel 0's.** — `[S·R] 2/3`
  - S: raw hex dump of E001.BIN file offsets 0x2A5C–0x2B3C (feds section, 0xE1 bytes), per `research/working_documents/E001_FEDS_HEXDUMP.md`
  - R: `smd-player/addons/exmateria_sound/runtime/feds_bank.gd` (FedsBank.parse: pair_count_plus1 @ 0x08, data_offset @ 0x0C, track offsets @ 0x18+i×2, track size bounded by next offset) + `godot-learning/tests/EffectSoundCaptureTest.gd` (E001 "Cure" case)
  - src: `research/working_documents/E001_FEDS_HEXDUMP.md`
- **The `AC` (0xAC) Instrument opcode handler (LAB_80015DD0) stores its 1-byte parameter to channel struct +0x7A and consumes exactly one byte (returns a0+1); the sample lookup happens later during playback.** — `[S·R] 2/3`
  - S: LAB_80015DD0 disassembly (`lbu v0, 0(a0); sh v0, 0x7a(a2); jr ra; addiu v0, a0, 0x1`), per `research/working_documents/INSTRUMENT_MAPPING.md`
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/instrument.gd` + `shared/opcodes/_table.gd` (0xAC dispatch sets channel.instrument_idx and resolves the WAVESET entry) + `godot-learning/tests/EffectSoundCaptureTest.gd` (drives the real effect path incl. AC)
  - src: `research/working_documents/INSTRUMENT_MAPPING.md`
- **SPU sample start address = ((instrument × 256) + base at channel+0x84 + offset at channel+0x82) << 16, computed at 0x80015778 and stored to channel+0x7E; the instrument × 256 term gives each instrument slot 256 bytes (16 VAG blocks) and the base is supplied from the sample resource table.** — `[S·R] 2/3`
  - S: 0x80015778–0x80015790 disassembly (`lbu v0, 0x7a(s0); lh v1, 0x84(s0); sll v0, v0, 0x8; sll v0, v0, 0x10; sw v0, 0x7e(s0)`), base via FUN_80016FB4, per `research/working_documents/INSTRUMENT_MAPPING.md`
  - R: `smd-player/addons/exmateria_sound/runtime/shared/channel_state.gd` + `shared/slot_state.gd` (model the FFT slot+0x82/+0x84 addends and the +0x7E word; ×256 baseline in `runtime/sequencer/note_handler/note_handler.gd`) + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/working_documents/INSTRUMENT_MAPPING.md`

## Notes

(empty — user territory)

## Related

- [[Effect File Format]]
- [[E001.BIN Memory Mapping]]
- [[WAVESET Instrument Bank]]
