# Color Track Interpolation

FFT color-track keyframe transitions are not parameterized in the effect data: the ROM contains a hardcoded 64-entry dither-curve table at 0x800956e4 (BATTLE.BIN), and the engine selects a curve from the magnitude of the color delta at keyframe time, then adds one sign-extended curve byte per frame for 32 frames. This gives smooth linear fades — or rapid pulses under mode 9 — with no division and no per-effect curve storage. The tracks modify unit palettes (caster/target/affected sprites), not the effect's own textures, which use the separate static BGR555 texture palette. Stored keyframe durations use a different unit: actual duration = time_value << 3 (×8) game frames.

## Points

- **Color track keyframe transitions use a hardcoded dither-based interpolation table at 0x800956e4 (BATTLE.BIN): 2048 bytes = 64 curves × 32 bytes of signed per-frame byte deltas.** — `[S] 1/3`
  - S: ROM address 0x800956e4 (BATTLE.BIN), per `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - src: `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
- **There is no curve parameter in the effect data; the game selects the interpolation curve from the color delta at keyframe time: curve_index = target_RGB - current_RGB + 31.** — `[S] 1/3`
  - S: delta calculation at 0x8008faa0, per `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - src: `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
- **Each frame the engine adds the sign-extended curve byte at curve_table[curve_index*32 + frame_counter] to the current color, so every transition completes over 32 frames.** — `[S] 1/3`
  - S: per-frame update code at 0x80091d38, per `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - src: `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
- **Curve index 31 is the neutral no-change curve; indices 0–30 are fade-out (decreasing) curves and 32–63 are fade-in (increasing) curves, each dither pattern distributing exactly N delta units across the 32 frames (linear interpolation without division).** — `[S] 1/3`
  - S: table layout at 0x800956e4 (curves 0–63), per `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - src: `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
- **Mode 9 pulsing is not a special code path; the palette state is simply set to use curve indices with rapid 0xFF/0x00 patterns, producing the strobe-like effect.** — `[S] 1/3`
  - S: curve table patterns at 0x800956e4, per `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - src: `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
- **The effect's texture palette (texture_ptr section; static 256-entry BGR555 CLUT) colors the effect's own sprites, while the timeline color tracks (RGB delta + mode, keyframed) modify unit palettes (caster/target/affected units) — the two are separate mechanisms with different scope and format.** — `[S] 1/3`
  - S: palette vs color-track distinction table, per `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - src: `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
- **Color-track keyframe durations use a different stored unit: actual duration = time_value << 3 (×8) game frames; the result still erodes once per game loop, so its wall-clock duration stretches with time scale exactly like other durations.** — `[ ] 0/3`
  - R: none — no <<3/×8 color-duration scaling found in godot-learning (src/effects/ColorSubsystem.gd probed)
  - src: `research/working_documents/TIME_SCALE_IMPLEMENTATION_GUIDE.md`

## Notes

(empty — user territory)

## Related

- [[Map Tint]]
- [[Effect Texture Upload]]
- [[Effect Frame Pacing]]
