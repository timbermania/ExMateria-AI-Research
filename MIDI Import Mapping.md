# MIDI Import Mapping

Rules and curated findings for mapping General MIDI material into FFT WAVESET instruments. Base family selection (from the GM source program via a curated GM→FFT mapping) happens before octave refinement; in-family variant selection minimizes distance to the sample's actual played root note; and one shared octave shift must cover the whole part inside the family's root ± 1 octave window rather than hopping per-note. Rendered comparisons (FluidSynth GM reference vs FFT candidates) anchor the curation: the Aerith flute lead maps to bassoon (clarinet produced repeated-tail artifacts), the horn lead stays in the softer `french_horn` lower subset, the high synth-string lead stays in `synth_str` on variants 86/87 (88/89 render ~1 octave high), and Inst 94 is timbre-rejected for sustained high-register use.

## Points

- **FFT WAVESET `synth_str` top variants 88/89 render about one octave high (~+1188 cents) at MIDI 78 (F#) and still at MIDI 90, and are the worst brightness/long-hold candidates in the family; lower variants 86/87 keep rendered pitch correct with materially better long-hold/timbre scores, so the high-register synth-string lead (GM 49/51) keeps family `synth_str` with one shared whole-part octave shift landing on 86/87.** — `[D] 1/3`
  - D: rendered variant comparison (FluidSynth GM reference vs FFT candidate instruments, measured cents/band-energy values as recorded in the doc) (2026-04-25)
  - R: none — GM→FFT variant curation not present in godot-learning / smd-player / fft-sound-driver (grepped `synth_str|bassoon|GM_TO_FFT|GM 73|flute`)
  - src: `research/working_documents/MIDI_IMPORT_MAPPING_RULES.md`
- **FFT WAVESET Inst 94 (`synth_str1`, the GM 46 Harp mapping) is a timbre reject for sustained high-register material despite a correct fundamental: its sustain centroid rises to roughly 3.1–3.9 kHz and its first active phrase carries ~20.9% of its energy in 2–4 kHz plus ~7.2% in 4–8 kHz, making it glassy and violin-shrill, so the family is kept but Inst 94 is rejected and the part shifts down onto the lower valid variants.** — `[D] 1/3`
  - D: rendered variant comparison (FluidSynth GM reference vs FFT candidate instruments, measured centroid/band-energy values as recorded in the doc) (2026-04-25)
  - R: none — Inst 94 timbre curation not present in godot-learning / smd-player / fft-sound-driver (grepped `bassoon|synth_str|GM_TO_FFT`)
  - src: `research/working_documents/MIDI_IMPORT_MAPPING_RULES.md`
- **GM 73 (Flute) material — the Aerith flute-like sustained high-note lead — is curated to the FFT `bassoon` family because the original `clarinet` family produced repeated-tail / harsh-sustain artifacts on the high sustained notes.** — `[D] 1/3`
  - D: rendered variant comparison (FluidSynth GM reference vs FFT candidate instruments, sustain artifacts observed in candidate renders) (2026-04-25)
  - R: none — GM→FFT family curation not present in godot-learning / smd-player / fft-sound-driver (grepped `bassoon|GM_TO_FFT|GM 73`)
  - src: `research/working_documents/MIDI_IMPORT_MAPPING_RULES.md`
- **GM 60 (French Horn) stays in the `french_horn` family but is constrained to the softer lower subset (samples 60/61/62) rather than the brighter upper samples (63 and 74–78) for the Aerith horn case, so whole-part octave fitting lands on the softer region.** — `[D] 1/3`
  - D: rendered variant comparison (FluidSynth GM reference vs FFT candidate instruments, brightness comparison of horn subset) (2026-04-25)
  - R: none — GM→FFT family curation not present in godot-learning / smd-player / fft-sound-driver (grepped `french_horn|GM 60|GM_TO_FFT`)
  - src: `research/working_documents/MIDI_IMPORT_MAPPING_RULES.md`

## Notes

(empty — user territory)

## Related

- [[WAVESET Instrument Bank]]
