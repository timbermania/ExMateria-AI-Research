# WAVESET Instrument Bank

WAVESET.WD is FFT's shared instrument waveform bank (magic "dwds") holding ~181 named ADPCM instruments, referenced by the `AC` (Instrument) opcode in both SMD music and feds effect sound. Effect audio resolves to this bank through sample-resource matching (the WAVESET resource entry's ID 0x0000 matches the effect's lookup ID), and instrument 1 (AC 01, the placeholder used by E001 Cure's channels 0/2) is a silent entry — Cure's actual audio comes from instrument 0x43 (Tubular Bells) in channel 1. Instrument names are community observations (middle-C played through the Chorium FFT soundfont), not official. The 2026-04 synth-accuracy capture work added: the per-instrument ADSR bytes are individual rate/level fields (not packed registers), SMD `Instrument(N)` addresses WAVESET entry N+1 via `FUN_80016fb4`'s +0x30 data-pointer offset, the sample bank loads into SPU RAM at 0x1000 preserving file offsets (gaps read as silence), and instruments without ADPCM loop flags play through once. The 2026-05-12 audio-parity work verified both sides load the WAVESET ADPCM section into the same contiguous 512 KB SPU RAM bank (same bytes at the same addresses by construction) and that FFT's sustained tones can loop into the prior instrument's ADPCM data (voice 21's repeat_addr points 1824 bytes before its start_addr).

## Points

- **WAVESET.WD is a "dwds" waveform bank: 0x30-byte header, instrument waveform entries from 0x30 (instrument 1's entry at 0x40 holds offset=0x90, size=0x20), and VAG/ADPCM sample data from 0xB30; consecutive same-type instruments at different octaves (C-2..C-7) exist to allow pitch shifting without changing the octave opcode.** — `[S·R] 2/3`
  - S: WAVESET.WD file offsets (header 0x00–0x30, instrument entries 0x30–0xB30, sample data 0xB30+), per `research/working_documents/INSTRUMENT_MAPPING.md`
  - R: `smd-player/addons/exmateria_sound/runtime/waveset_parser.gd` (parse: "dwds" magic, 16-byte instrument entries, ADPCM pre-decode to PCM) + `godot-learning/tests/RenderInGameAudio.gd` (in-game audio capture harness driven by the shared WavesetParser)
  - src: `research/working_documents/INSTRUMENT_MAPPING.md`
- **Effect sound pulls its audio from WAVESET.WD via sample-resource matching: the resource table (DAT_80032A44) holds a registered WAVESET.WD entry whose ID 0x0000 (at +0x20) matches the effect's search ID, and the per-instrument base for SPU addressing comes from the entry's +0x06 (FUN_80016FB4); E001's feds carries resource_id = 2 but the search ID is 0x0000.** — `[S·D·R] 3/3`
  - S: FUN_80016FB4 (per-instrument base from the sample resource table), DAT_80032A44 resource table, per `research/working_documents/INSTRUMENT_MAPPING.md`
  - D: sample resource table memory dump entry 0x80037CB0 (magic 'dwds', ID 0x0000 at +0x20) (2026-04-16)
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
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **`FUN_80016fb4` maps SMD `Instrument(N)` to WAVESET entry N+1 (a +0x30 data-pointer offset in the entry walk), so e.g. Instrument(39) reads entry 40's fine_tune (−2404, not entry 39's) — applying the +1 shift raised voice 1 spectral correlation 0.952 → 0.993.** — `[S·D·R] 3/3`
  - S: `FUN_80016fb4` decompilation recorded in `research/working_documents/SYNTH_ACCURACY.md` (2026-04-16)
  - D: Lua capture — Instrument(39) fine_tune −2404 (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/instrument.gd` ("FFT's +1 indexing rule", `idx = param + 1`) + smd-player music parity Gate A
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **The SPU sample bank lives in SPU RAM with the WAVESET.WD layout preserved: ADPCM sample data sits at SPU sample base 0x1000 at its original file offsets, and gaps between instruments are zeroed — overflow/gap reads produce silence, matching the real PSX.** — `[D·R] 2/3`
  - D: MUSIC_41.SMD reference match under full-bank load (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/spu.gd:13` (`RAM_INSTRUMENT_BASE := 0x1000`) + `smd-player/addons/exmateria_sound/runtime/waveset_parser.gd` (per-instrument ADPCM pre-decode)
  - src: `research/working_documents/SYNTH_ACCURACY.md`
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
- **All 169 FFT WAVESET instruments set ADPCM loop flag 0x02 (instruments sustain via the SPU repeat mechanism); whether the loop is hardware-driven (set-and-forget flag in the ADPCM block header) or sequencer re-triggered is unconfirmed.** — `[ ] 0/3`
  - R: none — no test validates the all-169 flag-0x02 claim (probed `fft-sound-driver/src/parsers/fft_waveset_parser.cpp`, which models 0x02 as `kFlagLoopRepeat`, and godot-learning)
  - src: `research/effect_sound/working_documents/OPEN_QUESTIONS.md`

## Notes

(empty — user territory)

## Related

- [[FEDS Sound Definition Format]]
- [[Effect Sound Timing]]
- [[SPU Voice Engine]]
- [[E001.BIN Memory Mapping]]
- [[PSX Pitch Conversion]]
- [[Effect Sound Audio Divergence]]
