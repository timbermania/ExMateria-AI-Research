# WAVESET Instrument Bank

WAVESET.WD is FFT's shared instrument waveform bank (magic "dwds") holding ~181 named ADPCM instruments, referenced by the `AC` (Instrument) opcode in both SMD music and feds effect sound. Effect audio resolves to this bank through sample-resource matching (the WAVESET resource entry's ID 0x0000 matches the effect's lookup ID), and instrument 1 (AC 01, the placeholder used by E001 Cure's channels 0/2) is a silent entry — Cure's actual audio comes from instrument 0x43 (Tubular Bells) in channel 1. Instrument names are community observations (middle-C played through the Chorium FFT soundfont), not official.

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

## Notes

(empty — user territory)

## Related

- [[FEDS Sound Definition Format]]
- [[Effect Sound Timing]]
- [[SPU Voice Engine]]
- [[E001.BIN Memory Mapping]]
