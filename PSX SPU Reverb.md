# PSX SPU Reverb

FFT's music reverb is the standard PSX SPU "Neill" reverb unit: an IIR filter feeding an accumulator and an allpass feedback network, run at 22.050 kHz with interpolation. Matching that algorithm against a MUSIC_41.SMD PCSX-Redux capture lifted full-mix spectral correlation from 0.818 to 0.967. The exact reverb preset per SMD is not yet pinned down — the battle.bin reverb table's "Room" preset (type 0x007D) is the working assumption, with reverb start address 0x3E000 and volume 0x3FFF as defaults; the centroid still sits ~650 Hz below the reference.

## Points

- **FFT reverb is the PSX SPU "Neill" IIR reverb: IIR filter → accumulator → allpass feedback at 22.050 kHz with interpolation; replacing a fake delay-line with this algorithm took MUSIC_41 full-mix spectral correlation 0.818 → 0.967 (centroid 2473 → 3085 Hz vs 3735 reference). The battle.bin reverb table "Room" preset (type 0x007D) is the working preset; MUSIC_41's actual preset is unconfirmed (start address 0x3E000 and volume 0x3FFF are defaults).** — `[D·R] 2/3`
  - D: MUSIC_41.SMD reference-capture mix comparison (doc 2026-04-16)
  - R: `smd-player/src/shared/fft_spu_reverb.cpp` (IIR/ACC/FB model, `FFTSpuReverbModel`; mirrored in `fft-sound-driver/src/shared/fft_spu_reverb.cpp`) + smd-player music parity Gates A/D (`smd-player/workspace/regression/verify_all.sh`)
  - src: `research/working_documents/SYNTH_ACCURACY.md`

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[SFX Index]]
