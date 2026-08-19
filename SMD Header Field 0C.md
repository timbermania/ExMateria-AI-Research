# SMD Header Field 0C

The SMD header bytes at 0x0C–0x0F (between the size u16 @0x08 and the flags u16 @0x10) carried a "mystery_0C" TBD in the master byte report until 2026-05-27, when the investigation closed them as build-pipeline bookkeeping with no runtime consumer: `FUN_800136C0` reads only SMD[0x10..0x1D], the disc loader sizes its load from a hardcoded SCUS table instead of the header, and a zero-hit 30-second read-breakpoint probe on MUSIC_25 confirmed no play-phase read. Across all 100 vanilla files the bytes have shape `00 01 XX YY`, with 30 distinct u16 values at 0x0E–0x0F in a categorical (clustered, not size/track-count-correlated) distribution. The reimplementation SMD parsers skip these bytes, so zeroing and passthrough are parity-equivalent.

## Points

- **SMD bytes 0x0C..0x0F are not consumed by any FFT runtime code — no init-path reader, no play-path reader; the field is build-pipeline bookkeeping, reclassified `unused (build_pipeline_0C)` in the master parser.** — `[S·D·R] 3/3`
  - S: `FUN_800136C0` (ram:800136c0–ram:80013764) reads only SMD[0x10..0x1d]; pointer-flow scan of `scus_disassembly.txt` + `battle_disassembly.txt` found 0 SMD +0xC..+0xF hits (sole BATTLE.BIN hit 0x801a5d54 is a timeline-channel cursor struct, not SMD); reclassification in `research/key_documents/smd_master_parser.py`
  - D: Read-BP probe `smd-player/workspace/probes/probe_smd_header_0c_reads.lua` on RAM 0x80051324..0x80051327, 30s MUSIC_25 playback, 0 hits with sister callstack probes armed and firing (2026-05-27)
  - R: `smd-player/addons/exmateria_sound/runtime/smd_parser.gd` and `fft-sound-driver/src/parsers/fft_smd_parser.cpp` both skip 0x0C–0x0F (parse 0x08, then 0x14 onward); validated by the smd-player music parity baseline and the `fft-sound-driver` music-hash gate
  - src: `research/working_documents/SMD_HEADER_FIELD_0C_INVESTIGATION.md`

- **Structural shape across all 100 vanilla MUSIC_NN.SMD files: 0x0C always 0x00, 0x0D always 0x01; 0x0E–0x0F is a u16 LE with 30 distinct values (0x06D8..0xFD18) in a categorical distribution — 0x18A3 in 28 files, 0x68D4 exclusively M83–M90, 0x0810 exclusively M92–M96, 0xBF28 exactly the three shortest files M20/M21/M49 — with no correlation to file size or track count.** — `[S] 1/3`
  - S: 100-file vanilla survey rendered as the gold TBD band in `research/key_documents/smd_master_parser.py`; parent survey in `research/key_documents/SMD_HEADER_GROUND_TRUTH.md`
  - R: none — 0x0E–0x0F value distribution not present in godot-learning, smd-player, or fft-sound-driver (both SMD parsers skip these bytes)
  - src: `research/working_documents/SMD_HEADER_FIELD_0C_INVESTIGATION.md`

- **The SMD disc-load chain never inspects header bytes: high-level song-slot init (FUN_80043628 / FUN_800436C0 / FUN_8001391C / FUN_80043E58) calls disc loader `FUN_800446D8(category=0x01, size_bytes)` with size and category taken from a hardcoded SCUS table at `0x80047080[id*8]`; it 0x800-aligns the pool alloc (`FUN_800442BC`, slot in `&DAT_8004EB18[16]`), sector-reads `size>>11` sectors via `FUN_80011BD0`, and `FUN_800120F4` parks the loaded buffer pointer in entity[+0x8].** — `[S] 1/3`
  - S: call graph above, per `scus_disassembly.txt` (caller chain of `FUN_800136C0`)
  - R: none — loader call chain (FUN_800446D8, size table 0x80047080) not present in godot-learning, smd-player, or fft-sound-driver
  - src: `research/working_documents/SMD_HEADER_FIELD_0C_INVESTIGATION.md`

- **`FUN_800136C0` (canonical SMD header reader) copies only SMD[0x10..0x1D] into the entity: flags u16 @0x10 → entity[+0x12], format-version bytes @0x12/0x13 (0x02, 0x01), track count @0x14 (drives the track_count×0x160+0x0B8 entity alloc in `FUN_80014278`), @0x15 → entity[+0x17], assoc_wds_id u16 @0x16 → entity[+0x18], u16 @0x18 → entity[+0x1A], @0x1A → entity[+0x44], tempo byte @0x1B → entity[+0x48] via `lbu/sll 8/sh` at ram:80013740, @0x1C → entity[+0x4C], @0x1D → entity[+0x50].** — `[S·R] 2/3`
  - S: instruction table ram:800136e0..ram:8001375c, `scus_disassembly.txt` + `scus_decompilation.c`
  - R: `smd-player/addons/exmateria_sound/runtime/smd_parser.gd` (track_count @0x14, assoc_wds_id @0x16, initial_volume @0x18) and `fft-sound-driver/src/parsers/fft_smd_parser.cpp` (same, plus initial_tempo @0x1B — its comment notes 0x1B matches the FFT disasm and the GDScript 0x1A read predates that fix); validated by the smd-player music parity baseline
  - src: `research/working_documents/SMD_HEADER_FIELD_0C_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[FEDS Sound Definition Format]]
- [[WAVESET Instrument Bank]]
- [[SPU Voice Engine]]
- [[MIDI Import Mapping]]
- [[SMD Header Layout]]
