# PSX SPU Reverb

FFT's music reverb is the standard PSX SPU "Neill" reverb unit: an IIR filter feeding an accumulator and an allpass feedback network, run at 22.050 kHz with interpolation. Matching that algorithm against a MUSIC_41.SMD PCSX-Redux capture lifted full-mix spectral correlation from 0.818 to 0.967. As of 2026-08-21 the preset **is** pinned: libspu reverb mode **4 = Studio Large**, carried as `4` in all 100 SMD headers at `[0x1A]`, with the per-song `[0x1B]` byte supplying reverb depth (`<< 8` into `vLOUT`/`vROUT`) rather than a second room. The earlier `0x007D` "Room" working assumption is superseded, and the residual ~650 Hz centroid deficit under it is read as the cost of fitting a filter to an unidentified room.

## Points

- **FFT reverb is the PSX SPU "Neill" IIR reverb: IIR filter → accumulator → allpass feedback at 22.050 kHz with interpolation; replacing a fake delay-line with this algorithm took MUSIC_41 full-mix spectral correlation 0.818 → 0.967 (centroid 2473 → 3085 Hz vs 3735 reference). The battle.bin reverb table "Room" preset (type 0x007D) is the working preset; MUSIC_41's actual preset is unconfirmed (start address 0x3E000 and volume 0x3FFF are defaults).** — `[D·R] 2/3`
  - D: MUSIC_41.SMD reference-capture mix comparison (doc 2026-04-16)
  - R: `smd-player/src/shared/fft_spu_reverb.cpp` (IIR/ACC/FB model, `FFTSpuReverbModel`; mirrored in `fft-sound-driver/src/shared/fft_spu_reverb.cpp`) + smd-player music parity Gates A/D (`smd-player/workspace/regression/verify_all.sh`)
  - ⚠ SUPERSEDED (2026-08-21) by: the preset is pinned — it is **Studio Large, libspu reverb mode 4** (psx-spx "SPU Reverb Examples"), not the `0x007D` "Room" working assumption. Confirmed statically off our own disc image in two independent ways: (a) all **100 of 100** `SOUND/MUSIC_*.SMD` headers carry `4` at `[0x1A]`, which `FUN_800136C0` passes as argument 1 of libspu's `SpuSetReverbModeParam` (`FUN_80018140`, mode bound-checked `< 10` at `ram:800181A0`); (b) the image carries libspu's 10-entry reverb work-area size table at `ram:80028F9C` — `0x80, 0x26C0, 0x1F40, 0x4840, 0x6FE0, 0xADE0, 0xF6C0, 0x18040, 0x18040, 0x3C00` — whose index 4 is `0x6FE0`, Studio Large's published work-area size. web-psx's live `mBASE` observations close on the same number from the dynamic side: `0x79020` = `0x80000 − 0x6FE0`, with a measured 948-sample pre-delay matching the impulse response rendered from Studio Large's published halfwords alone. Per-song variation is the **reverb depth** at `[0x1B]`, not a second room (see [[SMD Header Layout]]). **Why it matters:** with the mode pinned, the mode→halfwords table is published, so a player with no emulated SPU needs to lift nothing out of the binary to reverberate correctly — and the 0.818 → 0.967 correlation climb with a centroid ~650 Hz low is the signature of fitting a filter to an unidentified room; web-psx reports 0.9990/0.9982/0.9996 **at lag 0** after pinning (ExMateria-AI-Research#2)
  - src: `research/working_documents/SYNTH_ACCURACY.md`

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[SFX Index]]
