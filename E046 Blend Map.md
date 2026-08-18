# E046 Blend Map

E046 (DEMI2, the two-part battle effect) per-frameset blend-mode and emitter-binding layout, read authoritatively from the ABR bits of each frame's `tpage_blend` field (0=normal/avg, 1=ADD, 2=SUB, 3=add¼). As of the 2026-07-31 fold audit the map is: seq0/seq1 subtractive (emitters e0/e1 render the big black blob), seq2/seq3 additive (e2/e3 magenta blob, e4 ~1px), and seq4 a big white glow with no emitter bound — so the subtractive particles are e0/e1 and e2/e3/e4 are all additive. The audit's congruent additive target was emitter-2/seq-2 held on frameset 26 (a 36×36 magenta blob) over a flat mid-gray (128,128,128) screen, where `with − without` reduces to `WITH − 128` per channel. The purple halo of the additive particles is baked into the source texels (a magenta-cores, blue-edged radial gradient in the shared CLUT — the visible white points are additive saturation of overlapping magenta cores), and the white additive (emitter 3, spawns f36–49) is newer than the black subtractive (emitter 1, f22–38), which is what a newest-on-top tie-break puts on top at the overlap tail.

## Points

- **E046 (DEMI2)'s frameset blend modes come from each frame's `tpage_blend` ABR bits (0=normal/avg, 1=ADD, 2=SUB, 3=add¼) — SEQ0 framesets 0–12 SUBTRACTIVE (emitters e0/e1, the big black blob), SEQ1 framesets 13–25 SUBTRACTIVE (no emitters bound), SEQ2 framesets 26–32 ADDITIVE (e2/e3, the magenta blob), SEQ3 framesets 47–50 ADDITIVE (e4, ~1px), SEQ4 framesets 36–46 ADDITIVE with no emitter bound (the big white glow) — so the subtractive particles are e0/e1 and e2/e3/e4 are all additive, correcting the prior "e2 = subtractive cloud / e4 = the gold additive target" notes.** — `[D] 1/3`
  - D: oracle held-particle capture `captures/oracle_emitter2_seq2_fs26_held.{raw,png}` (2026-07-31): emitter-2/seq-2 held on frameset 26 renders as a 36×36 additive magenta blob (white-clamped core + magenta halo falloff)
  - src: `research/working_documents/demi2_fold_audit/README.md`- **DEMI2's additive 'white' particle is NOT white — its texture is a magenta-cores, blue-edged radial gradient baked into the shared 8bpp CLUT/texels (edge (0,0,24) → mid (40,16,88)→(144,48,128) magenta → near-white (240,232,240) core; anim 2, framesets 26–32), so the purple halo is in the source texels, not a compositing artifact; the 'black' subtractive particle samples the SAME CLUT's bright grey/white ramp (up to (248,248,248)) used subtractively — the black/white distinction is purely the blend mode, and the visible white points are additive saturation of overlapping magenta cores.** — `[D] 1/3`
  - D: `demi2` PSX oracle box pixel grid, savestate `SCUS94221.sstate8` (2026-07-20) — magenta ring (200,80,184) R≈B≫G, white core (248,248,248), dark-blue deep-cloud skirt (8,0,48); consistent with the extracted `godot-learning/assets/effects/E046/` CLUT/texel analysis (E046.BIN 45660 bytes, 9 sections)
  - src: `research/working_documents/DEMI2_E046_ADDITIVE_SUBTRACTIVE_ORDERING.md`
- **DEMI2's emitter/timeline map: emitter 1 = the black subtractive particle (spawns f22–38), emitter 3 = the white additive particle (spawns f36–49) — the white additive is newer than the black subtractive, so a newest-on-top tie-break only visibly puts the white over the black at the overlap tail (f~40–48); a separate emitter draws the ~1px confounding yellow dots.** — `[D] 1/3`
  - D: `probe_demi_pair.gd` isolation re-render (emitters idx1 + idx3 isolated, f36→45, 2026-07-20) + user-confirmed timeline data
  - src: `research/working_documents/DEMI2_E046_ADDITIVE_SUBTRACTIVE_ORDERING.md`

## Notes

(empty — user territory)

## Related

- [[Display Space Blend Fold]]
- [[PSX GPU Primitives]]
