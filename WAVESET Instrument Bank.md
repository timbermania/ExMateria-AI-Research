# WAVESET Instrument Bank

WAVESET.WD is FFT's shared instrument waveform bank (magic "dwds") holding ~181 named ADPCM instruments, referenced by the `AC` (Instrument) opcode in both SMD music and feds effect sound. Effect audio resolves to this bank through sample-resource matching (the WAVESET resource entry's ID 0x0000 matches the effect's lookup ID), and instrument 1 (AC 01, the placeholder used by E001 Cure's channels 0/2) is a silent entry — Cure's actual audio comes from instrument 0x43 (Tubular Bells) in channel 1. Instrument names are community observations (middle-C played through the Chorium FFT soundfont), not official. The 2026-04 synth-accuracy capture work added: the per-instrument ADSR bytes are individual rate/level fields (not packed registers), SMD `Instrument(N)` addresses WAVESET entry N+1 via `FUN_80016fb4`'s +0x30 data-pointer offset, the sample bank loads into SPU RAM at 0x1010 preserving file offsets (gaps read as silence), and instruments without ADPCM loop flags play through once. The 2026-05-12 audio-parity work verified both sides load the WAVESET ADPCM section into the same contiguous 512 KB SPU RAM bank (same bytes at the same addresses by construction) and that FFT's sustained tones can loop into the prior instrument's ADPCM data (voice 21's repeat_addr points 1824 bytes before its start_addr).

## Points

- **WAVESET.WD is a "dwds" waveform bank: 0x30-byte header, instrument waveform entries from 0x30 (instrument 1's entry at 0x40 holds offset=0x90, size=0x20), and VAG/ADPCM sample data from 0xB30; consecutive same-type instruments at different octaves (C-2..C-7) exist to allow pitch shifting without changing the octave opcode.** — `[S·R] 2/3`
  - S: WAVESET.WD file offsets (header 0x00–0x30, instrument entries 0x30–0xB30, sample data 0xB30+), per `research/working_documents/INSTRUMENT_MAPPING.md`
  - R: `smd-player/addons/exmateria_sound/runtime/waveset_parser.gd` (parse: "dwds" magic, 16-byte instrument entries, ADPCM pre-decode to PCM) + `godot-learning/tests/RenderInGameAudio.gd` (in-game audio capture harness driven by the shared WavesetParser)
  - src: `research/working_documents/INSTRUMENT_MAPPING.md`
- **Effect sound pulls its audio from WAVESET.WD via sample-resource matching: the resource table (DAT_80032A44) holds a registered WAVESET.WD entry whose ID 0x0000 (at +0x20) matches the effect's search ID, and the per-instrument base for SPU addressing comes from the entry's +0x06 (FUN_80016FB4); E001's feds carries resource_id = 2 but the search ID is 0x0000.** — `[S·D·R] 3/3`
  - S: FUN_80016FB4 (per-instrument base from the sample resource table), DAT_80032A44 resource table, per `research/working_documents/INSTRUMENT_MAPPING.md`
  - D: sample resource table memory dump entry 0x80037CB0 (magic 'dwds', ID 0x0000 at +0x20) (2026-04-16)
  - D: `diag_inst_bank_lookup` (BP @ `0x80017058` inside the 0xAC loader), `cure_4_no_music` (2026-05-14) — bank_base = 0x80037CB0 at every 0xAC dispatch of voices 18–21 (single shared bank), bank header `64776473` = "dwds" = WAVESET.WD loaded into main RAM; no per-effect instrument bank
  - R: `smd-player/addons/exmateria_sound/runtime/feds_bank.gd` (`assoc_wds_id = 0` — effect feds shares WAVESET.WD with music) + `godot-learning/src/audio/EffectSfxEngine.gd` (feds pairs play through the shared waveset) + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/working_documents/INSTRUMENT_MAPPING.md`
- **WAVESET.WD instrument 1 (0x01) is silent: its entry (0x40) points at a 0x20-byte sample at 0xB30+0x90 = 0xBC0 that is all zeros except the VAG header, so AC 01 channels produce no audio — E001 Cure's audible sound comes from instrument 0x43 (Tubular Bells) in channel 1, with channels 2/3 byte-copies of 0/1.** — `[S·R] 2/3`
  - S: WAVESET.WD instrument-1 sample at 0xBC0 (all zeros except VAG header) and E001 feds channel instrumentation (ch0/ch2 = AC 01, ch1/ch3 = AC 43), per `research/working_documents/INSTRUMENT_MAPPING.md`
  - R: `smd-player/addons/exmateria_sound/runtime/waveset_parser.gd` (per-instrument `is_null` flag for empty entries) + `godot-learning/tests/EffectSoundCaptureTest.gd` (E001 "Cure" case)
  - src: `research/working_documents/INSTRUMENT_MAPPING.md`
- **WAVESET.WD instruments 0–180 (0x00–0xB4) map to named waveforms (e.g. 0x08 Glockenspiel, 0x12/0x43 Tubular Bells, 0x3B Vibraphone, 0x83 High-Pitched Bell) with 0x01/0x2D/0x36–0x38/0x45/0x67–0x68 empty and 0xB5–0xFF mostly empty/clips; the names are community observations of middle C (C4) played through the Chorium FFT soundfont, not official.** — `[S] 1/3`
  - S: WAVESET.WD instrument table + ffhacktics "Instruments" wiki + Chorium soundfont C4 observations, per `research/working_documents/INSTRUMENT_MAPPING.md`
  - R: none — no WAVESET instrument name table present in godot-learning / smd-player (grepped "Tubular|Glockenspiel|Vibraphone" across both: no hits)
  - src: `research/working_documents/INSTRUMENT_MAPPING.md`

- **WAVESET.WD's 8 "ADSR bytes" per instrument are individual fields, not packed ADSR1/ADSR2 register values: byte 0 = Ar (0–127), 1 = Dr (0–15), 2 = Sr (0–127), 3 = Rr (0–31), 4 = Sl (0–15), 5–7 = mode flags; registers are constructed as `ADSR1 = (Am<<15)|(Ar<<8)|(Dr<<4)|Sl`, `ADSR2 = (Sd<<14)|(Sm<<13)|(Sr<<6)|(Rm<<5)|Rr`. Instrument 39's captured bytes yield ADSR1=0x00E3, ADSR2=0x6FAC (exact hardware match), and per-instrument ADSR raised first-note correlation 0.994 → 0.996 (SNR 17.6 → 18.8 dB).** — `[D·R] 2/3`
  - D: `PCSX.SPU.getVoiceInfo()` capture, inst 39 (Ar=0 Dr=14 Sl=3 Sr=62 Sm=1 Sd=1 Rm=1 Rr=12) + MUSIC_41.SMD first-note comparison (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/waveset_parser.gd:103-104` (per-instrument field parse + register construction — note: smd-player places Sm at bit 15 with Sd hardcoded 1, a bit layout that diverges from this doc's bit-13 Sm formula) + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - ⚠ SUPERSEDED (2026-08-21) by: the `ADSR2` formula's sustain-mode bit is wrong and **smd-player's code is the correct side of the divergence this R-line already flags**. On PSX hardware the ADSR upper halfword is: bit 15 sustain **mode/curve**, bit 14 sustain **direction**, bit 13 **unused**, bits 12–8 sustain shift, bits 7–6 sustain step, bit 5 release mode, bits 4–0 release shift. So `Sm` belongs at bit 15, not 13 — `ADSR2 = (Sm<<15)|(Sd<<14)|(Sr<<6)|(Rm<<5)|Rr`. The `Sd<<14` in this note is right, and [[SPU Voice Engine]] already documents the correct layout, so the vault contradicted itself here. For instrument 39 the two readings are the same bits in different places: `0x6FAC` under this note's formula, **`0xCFAC`** under the hardware layout. Falsifier needing no reference data: break at a key-on for any program with sustain mode set — a value with bit 13 set and bit 15 clear is not something the driver can emit. web-psx reports attack mode / sustain direction / sustain curve agreeing 24 of 24 at the key-on register under the hardware layout (ExMateria-AI-Research#2)
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **`FUN_80016fb4` maps SMD `Instrument(N)` to WAVESET entry N+1 (a +0x30 data-pointer offset in the entry walk), so e.g. Instrument(39) reads entry 40's fine_tune (−2404, not entry 39's) — applying the +1 shift raised voice 1 spectral correlation 0.952 → 0.993.** — `[S·D·R] 3/3`
  - S: `FUN_80016fb4` decompilation recorded in `research/working_documents/SYNTH_ACCURACY.md` (2026-04-16)
  - D: Lua capture — Instrument(39) fine_tune −2404 (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/instrument.gd` ("FFT's +1 indexing rule", `idx = param + 1`) + smd-player music parity Gate A
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **The SPU sample bank lives in SPU RAM with the WAVESET.WD layout preserved: ADPCM sample data sits at SPU sample base 0x1000 at its original file offsets, and gaps between instruments are zeroed — overflow/gap reads produce silence, matching the real PSX.** — `[D·R] 2/3`
  - D: MUSIC_41.SMD reference match under full-bank load (doc 2026-04-16)
  - D: original `PCSX.SPU.readSpuRam(0x1000, 0x100)` live SPU RAM dump: SPU `0x1000`–`0x100F` hold `0x07` pad bytes and SPU `0x1020` holds WAVESET `adpcm[0x10]`, pinning the bank base at `0x1010` (2026-05-17)
  - R: `smd-player/addons/exmateria_sound/runtime/spu.gd:13` (`RAM_INSTRUMENT_BASE := 0x1000`) + `smd-player/addons/exmateria_sound/runtime/waveset_parser.gd` (per-instrument ADPCM pre-decode)
  - R: commit `6a197ed4` set `kRamInstrumentBase = 0x1010` in `smd-player/src/shared/fft_spu_core_runtime.h:32` (mirrored in `fft-sound-driver/`); post-fix `reraise_no_music` full_mix cos_dist 0.1690 → 0.1442, full_mix cos_dist_phase 0.4747 → 0.2870, voice_18 cos_dist 0.0061 → 0.0033
  - ⚠ SUPERSEDED (2026-08-21) by: the SPU bank base is **`0x1010`**, not `0x1000` — `0x1000` is where the SPU capture buffers end, and the first ADPCM block sits one block later. Our own C++ already moved: `smd-player/src/shared/fft_spu_core_runtime.h:32` (mirrored in `fft-sound-driver/`) has `kRamInstrumentBase = 0x1010` behind a 9-line comment recording the dynamic proof — PCSX-Redux `readSpuRam` shows SPU `0x1020` holding `WAVESET adpcm[0x10]`, and `0x1000` shifts voice 21's loop region and yields a wrong-pitch second note (~4180 Hz vs PCSX's ~3800 Hz). **Live divergence flagged 2026-08-21:** the GDScript constant at `spu.gd:13` is still `0x1000` and feeds `channel.sample_start_addr` at `sequencer.gd:124`, `shared/opcodes/instrument.gd:50`, `instrument_reload.gd:37` and `effect_sound/play_sound.gd:885`, while the native mixer loads the bank at `kRamInstrumentBase` — so every GDScript-computed start address is one ADPCM block early relative to the bank it indexes. Not changed here; audio-parity change pending validation (ExMateria-AI-Research#2)
  - src: `research/working_documents/SYNTH_ACCURACY.md`
  - src: `research/effect_sound/working_documents/WAVESET_BANK_BASE_MISMATCH_FIX.md`
- **Instruments without ADPCM LOOP_START/LOOP_END flags are not looped: the sample plays through once and the ADSR decay takes it down — the reference capture shows 0 envelope rises after the attack (perfectly monotonic decay), so no audible loop restarts exist.** — `[D·R] 2/3`
  - D: MUSIC_41.SMD reference capture `envelope_rises` = 0 (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/waveset_parser.gd` (`loop_start = -1` unless the ADPCM loop flags are set) + `fft-sound-driver/src/parsers/fft_waveset_parser.cpp:91` (`kFlagLoopRepeat` handling)
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **Both sides place WAVESET.WD's ADPCM section (file `data_offset` 0xB30) into the same contiguous 512 KB SPU RAM bank — PCSX via BIOS DMA, Godot via `WavesetParser` slicing `data[data_offset:]` plus the native `fft_load_spu_adpcm_bank` — so sample bytes land at the same SPU addresses by construction; byte sanity at voice 21's start_addr (283280) confirmed real ADPCM data (not zeros/corruption).** — `[D·R] 2/3`
  - D: `dump_sample_bytes.py` capture (commit ee643e17; doc 2026-05-12)
  - R: `smd-player/addons/exmateria_sound/runtime/waveset_parser.gd` (data_offset read at +0x10, ADPCM slice) + `smd-player/src/shared/fft_spu_core_state_tools.cpp` (`fft_load_spu_adpcm_bank` → `spu_ram_` at `kRamInstrumentBase`, now 0x1010 after bank-base alignment to the PCSX capture)
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **FFT's sustained tones loop into shared cross-instrument data: voice 21's repeat_addr (0x44B30 = 281456) points 1824 bytes BEFORE its start_addr (0x45290 = 283280) into the prior instrument's ADPCM, and on LOOP_END the SPU wraps back there for the sustain-loop content; voice 21's own sample has NO LOOP_START flag, and its LOOP_END sits 2560 bytes past the start (~232 ms of audio).** — `[D·R] 2/3`
  - D: `dump_sample_bytes.py` LOOP_START/END scan (2026-05-12)
  - R: Godot's native mixer `spu_ram_` is one contiguous 512 KB buffer supporting the cross-instrument wrap (`smd-player/src/shared/fft_spu_core_runtime.cpp`) + `probe_sample_repeat_addr_register` pairs bit-exact (commit 47c5294b formula fix)
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **All 169 FFT WAVESET instruments set ADPCM loop flag 0x02 (instruments sustain via the SPU repeat mechanism); whether the loop is hardware-driven (set-and-forget flag in the ADPCM block header) or sequencer re-triggered is unconfirmed.** — `[S] 1/3`
  - S: WAVESET.WD flag scan across all 169 instruments — every instrument carries flag 0x02 with no 0x04/0x01 (verified against the ROM WAVESET image for the `reraise_no_music` instrument set)
  - R: none — no test validates the all-169 flag-0x02 claim (probed `fft-sound-driver/src/parsers/fft_waveset_parser.cpp`, which models 0x02 as `kFlagLoopRepeat`, and godot-learning)
  - src: `research/effect_sound/working_documents/OPEN_QUESTIONS.md`
  - src: `research/effect_sound/working_documents/RERAISE_VOICE_21_RENDERED_AUDIO_DIVERGENCE_WITH_BIT_IDENTICAL_REGISTER_WRITES.md`
  - S: SPU_VOICE_LIFECYCLE.md (2026-06-11) resolves the open question — sustain is hardware-level: the SPU ADPCM repeat flag (0x02) in the block header is a set-and-forget loop, not sequencer re-triggered
  - src: `research/effect_sound/working_documents/SPU_VOICE_LIFECYCLE.md`
- **Reraise voice 21's WAVESET instrument layout: `adpcm[0x00]` is a zero-pad block, `adpcm[0x30]` carries flag `0x06` (LOOP_START+REPEAT), and `adpcm[0x80]` carries flag `0x03` (END+REPEAT), so under bank base `0x1010` the six-block sustain loop sits at SPU `0x1040`..`0x1090`; both KEYONs of the voice write raw `sample_start_addr = 0x1010` (4112) and `sample_repeat_addr = 48`.** — `[D·R] 2/3`
  - D: live SPU RAM dump `PCSX.SPU.readSpuRam(0x1000, 0x100)` — SPU `0x1040` = `47063333...` (flag 0x06), SPU `0x1090` = `470313F1...` (flag 0x03), byte-identical to the WAVESET adpcm bank at file offset 0xB30 — plus the dense voice-21 per-output-sample trace, reraise_no_music (2026-05-17)
  - R: `smd-player/src/shared/fft_spu_voice_tools.cpp` (block-flag decode: `flags & 0x04` arms loop_addr at the block's SPU address; `flags == 3 && loop_addr > 0` wraps curr_addr back to it) with the bank loaded at `kRamInstrumentBase` (`smd-player/src/shared/fft_spu_core_runtime.h:32`) — validated by the post-`6a197ed4` `reraise_no_music` run, Godot's loop region now SPU 0x1040..0x1090 (matches PCSX)
  - src: `research/effect_sound/working_documents/WAVESET_BANK_BASE_MISMATCH_FIX.md`

- **The 0xAC instrument loader (`Hyp_instrument_data_loader` / FUN_80016FB4, called by the 0xAC handler at PC `0x80015E48`) fills 13 channel fields from one 16-byte bank entry at `chan+0x30 (inst_table_base) + 0x30 + inst_byte×16`: chan+0x2c, +0x2e, +0x50, +0x54, +0x58, +0x5c, +0x60, +0x64, +0x66, +0x68, +0x6a, +0x6c, +0x84 — the +0x84 finetune halfword via `lhu v0, 0x6(a0)` @ `0x80017058` + `sh v0, 0x84(a1)` @ `0x80017060`, and the bank is the shared WAVESET.WD, not a per-effect bank.** — `[S·D·R] 3/3`
  - S: loader body `0x80016FB4`–`0x80017090` (16-byte entry walk from `lw v1, 0x30(a1)`) + 0xAC call site `0x80015E48` — `project-assets/fft-rom/scus_disassembly.txt` (doc §5–§6.2)
  - D: `diag_inst_bank_lookup` (BP @ `0x80017058`, one row per 0xAC dispatch), `cure_4_no_music` (2026-05-14) — entry bytes + finetune captured; voice 20 byte 131 → finetune +412 = WAVESET[132] as Godot parses; `diag_chan_84_writers` write-BP captured 0 writes to chan+0x84 across the capture (finetune stable post-AC)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/instrument.gd` (0xAC handler mirroring the Godot-modeled fields: `fine_tune`, `adsr1`/`adsr2`, `release_rate_byte`, `sample_start_addr`, `sample_loop_addr` from the WAVESET entry) + `runtime/waveset_parser.gd` — validated by the `probe_opcode_instrument` / SPU register-probe pairs in `smd-player/workspace/orchestrator/probe_validation_manifest.py`; mirrored in `fft-sound-driver/src/driver/opcodes_instrument.cpp`
  - src: `research/effect_sound/working_documents/VOICE_20_PITCH_BASE_PLUS_1792.md`

## Notes

(empty — user territory)

## Related

- [[FEDS Sound Definition Format]]
- [[Effect Sound Timing]]
- [[SPU Voice Engine]]
- [[E001.BIN Memory Mapping]]
- [[PSX Pitch Conversion]]
- [[Effect Sound Audio Divergence]]
