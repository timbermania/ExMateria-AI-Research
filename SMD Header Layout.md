# SMD Header Layout

Ground truth for the SMD (`MUSIC_NN.SMD`) header, established 2026-05-27 by disassembly-verification of the in-game init path `FUN_800136C0` plus a 100-file vanilla survey: only `SMD[0x10..0x1D]` is read at song init; `0x1A` is a mode discriminator (constant `0x04`) for the `FUN_80018140` parameter dispatch, and `0x1B` is the real tempo byte passed tempo×256 (Q8.8). `0x04–0x07` carries a per-file unique value, closed 2026-05-27 as build-pipeline-only (zero-hit read-watchpoint + init-path decompilation; sibling conclusion for `0x0C–0x0F` in [[SMD Header Field 0C]]); the flags u16 @`0x10..0x11` is read at runtime but never non-zero in vanilla, and the 2 bytes between the track-ptr table and `song_title_ptr` are alignment padding, not read at runtime. Every vanilla conductor track re-asserts tempo via a `Tempo` (0xA0) opcode within the first 1–2 events, so audible tempo comes from the conductor, not the header; the reimplementation mirrors this. `fft-sound-driver`'s parser reads the correct `0x1B`; the smd-player GDScript parser still reads `0x1A` for its diagnostic field (deferred cosmetic fix).

## Points

- **`SMD[0x1A]` (constant `0x04` in all vanilla files) is the mode-discriminator argument to the `FUN_80018140` parameter dispatch — mode 4 = "SMD song-init tempo set" — and `SMD[0x1B]` is the real tempo byte, read unsigned, shifted ×256 (Q8.8), and passed as a signed short in `a1`.** — `[S·D·R] 3/3`
  - S: `FUN_800136C0` at `ram:80013734`–`ram:80013764` in `scus_disassembly.txt` (lbu SMD[0x1A] → sign-extend → `a0`; lbu SMD[0x1B] → sll 8 → sh entity+0x48 → `a1`; jal FUN_80018140 at `ram:80013764`); call sites 0x800122E4, 0x80012FC4, 0x80013764, 0x800160C4, 0x80017AE0, 0x80017BD4; `FUN_80018140` signature in `scus_decompilation.c`
  - D: 100-file vanilla MUSIC_NN.SMD survey (2026-05-27): 0x1A constant `0x04`, 0x1B has 21 distinct values decoding to 54–160 BPM
  - R: `fft-sound-driver/src/parsers/fft_smd_parser.cpp:150` reads `data[0x1B]` as initial_tempo (comment notes 0x1B matches the FFT disasm; the GDScript `smd-player/addons/exmateria_sound/runtime/smd_parser.gd:56` 0x1A read is the known-cosmetic leftover, deferred); validated by the smd-player music parity baseline (`smd-player/workspace/regression/music_parity_inventory.json`)
  - src: `research/working_documents/SMD_HEADER_GROUND_TRUTH.md`

- **`SMD[0x04–0x07]` carries a per-file-unique value (100 distinct across 100 vanilla files) and is never read by the song-init path `FUN_800136C0` — presumed build-pipeline hash/ID, not validated at runtime.** — `[S·D] 2/3`
  - S: `FUN_800136C0` (`ram:800136c0`–`ram:80013780`, `scus_disassembly.txt`) references no SMD offset below 0x10
  - D: 100-file survey (2026-05-27): 100 distinct values, every file unique
  - D: Read-watchpoint over RAM `0x8005131C..0x8005131F` during a full MUSIC_25 capture (`smd-player/workspace/probes/probe_smd_header_04_reads.lua`, `--callstack-probes`, 6s post-anchor) — 0 JSONL rows (2026-05-27); field closed as build-pipeline-only, parser re-tagged `tbd` → `dead`
  - R: none — 0x04–0x07 build ID not present in godot-learning, smd-player, or fft-sound-driver (both SMD parsers skip it: magic 0x00–0x03, file_size 0x08–0x09, then 0x14 onward)
  - src: `research/working_documents/SMD_HEADER_GROUND_TRUTH.md`

- **All 100 vanilla `MUSIC_NN.SMD` files carry a conductor track that emits a `Tempo` (0xA0) opcode within its first 1–2 events (right after ReverbOn), so the audible tempo is always the conductor opcode's value — the header-init tempo window is sub-millisecond.** — `[S·D·R] 3/3`
  - S: per-tick sequencer (`FUN_800153B0` family) executes the conductor track and the `smd_tempo` handler at `0x80015CB0` updates entity tempo state before any sample is emitted (`scus_disassembly.txt`)
  - D: 100-file survey (2026-05-27): 100/100 conductor tracks have Tempo at pos+1 or pos+2; e.g. MUSIC_00 conductor Tempo(0x55)=99.8 BPM vs header 0x1B=0x3C=70.5 BPM
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/tempo.gd` (0xA0 → `sequencer.tempo_bpm = fft_tempo_to_bpm(byte)` plus smd_tempo's entity+0x78/+0x7c writes); validated by the smd-player music parity baseline (`smd-player/workspace/regression/music_parity_inventory.json`)
  - src: `research/working_documents/SMD_HEADER_GROUND_TRUTH.md`

- **SMD tempo bytes convert to BPM as `bpm = val*256/218`, with `val == 0` mapping to a 120.0 default; the engine carries the header tempo byte ×256 in Q8.8 fixed point.** — `[S·D·R] 3/3`
  - S: `sll v0,v0,0x8` at `ram:80013748` in `scus_disassembly.txt` (×256 scaling in the init path); the 0→120 default per the `tempo_val == 0` branch in `fft-iso-patcher/fft_iso_patcher/smd/opcodes.py:354`
  - D: 100-file survey decodes (2026-05-27): 21 distinct 0x1B values → 54–160 BPM (0x36=63.4, 0x3C=70.5, 0x40=75)
  - R: `smd-player/addons/exmateria_sound/runtime/smd_opcodes.gd:174-177` `fft_tempo_to_bpm` (`val*256.0/218.0`, `0 → 120.0`) + the no-param 120.0 fallback in `tempo.gd`; validated by the smd-player music parity baseline
  - src: `research/working_documents/SMD_HEADER_GROUND_TRUTH.md`

- **The SMD flags u16 @ `0x10..0x11` is read at runtime — `FUN_800136C0` copies it to `entity[+0x12]` at `ram:800136e0` — but is `0x0000` in all 100 vanilla files, so no bit semantics has ever been exercised in the vanilla corpus; custom SMDs emitting `00 00` match vanilla behavior.** — `[S·D] 2/3`
  - S: copy site `ram:800136e0` in `FUN_800136C0`, `scus_disassembly.txt` + `scus_decompilation.c`
  - D: 100-file vanilla survey (2026-05-27): `0x0000` in 100/100 files
  - R: none — 0x10..0x11 flags not present in godot-learning, smd-player (`smd_parser.gd` parses 0x08, then jumps to 0x14), or fft-sound-driver (`fft_smd_parser.cpp`); bit mapping remains an open multi-hour disassembly task (sweep of `entity[+0x12]` readers)
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **The 2 bytes between the end of the track-pointer table (`0x22 + tracks*2`) and `song_title_ptr` are `00 00` in all 100 vanilla files (`song_title_ptr - tt_end == 2` invariant) — alignment padding for the 4-byte-aligned `song_title_ptr`, not read at runtime (init reads only SMD[0x10..0x1D]); reclassified `tbd` → `unused (align_pad)` in the master parser.** — `[S·D] 2/3`
  - S: `FUN_800136C0` decompilation reads only SMD[0x10..0x1D] (`scus_decompilation.c`); reclassification in `research/key_documents/smd_master_parser.py`
  - D: 100-file vanilla survey (2026-05-27): `00 00` in all 100 files, invariant `song_title_ptr - tt_end == 2`
  - R: none — align_pad region not present in godot-learning, smd-player, or fft-sound-driver (both SMD parsers jump past it to `song_title_ptr` @ 0x1E)
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

## Notes

(empty — user territory)

## Related

- [[SMD Header Field 0C]]
- [[SPU Voice Engine]]
- [[WAVESET Instrument Bank]]
- [[SMD Opcodes]]
