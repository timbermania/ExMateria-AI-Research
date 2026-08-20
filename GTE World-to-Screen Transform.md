# GTE World-to-Screen Transform

The PSX GTE (Geometry Transform Engine) world→screen transform used by FFT's effect renderers: load_gte_matrix (0x8001D0A8) and init_matrix (0x8001D138) load the camera's view rotation matrix (32 bytes at 0x80098A24) into the GTE, then rotate_vector (0x8001D578) multiplies the input vector by that camera matrix to produce screen X/Y/Z and depth — because it is a full 3D rotation, position offsets must be applied to screen coordinates after the transform, exactly as the original render_spell_charge_lines (0x801B1C04) does. The same view matrix drives the battle map projection, which is a pure affine ortho (no perspective divide): live-validated to 0.27px RMS over 9 on-field units by the PSX-side tile-grid reproject (2026-07-06), with the translation datum calibrated live from the units' own GTE-stored screen anchors.

## Points

- **FFT's world→screen transform for effects: load_gte_matrix (0x8001D0A8) loads the camera's view rotation matrix stored at 0x80098A24 (32 bytes) into the GTE, init_matrix (0x8001D138) initializes the translation from the same matrix, and rotate_vector (0x8001D578; args: 3×int16 input, 3×int32 output, int32 depth output) computes output = rotation_matrix × input.** — `[S] 1/3`
  - S: view_rotation_matrix 0x80098A24, load_gte_matrix 0x8001D0A8, init_matrix 0x8001D138, rotate_vector 0x8001D578, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **rotate_vector is a full 3D camera-matrix rotation, not a simple coordinate transform: a pure world-space offset (e.g. on Y) emerges as a mix of screen X, Y, and Z, so world-space position offsets before the GTE have unpredictable screen effects; offsets must instead be added to screen coordinates after rotate_vector (e.g. addiu s1, s1, offset on screen Y).** — `[S] 1/3`
  - S: rotate_vector 0x8001D578, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **render_spell_charge_lines (0x801B1C04) shows how FFT combines effect positions: it builds a relative offset vector (0, arc_height, 0), rotates that vector through the GTE to get a screen-space offset, and only then adds the caster's screen position — i.e. FFT combines positions in screen space, not world space.** — `[S] 1/3`
  - S: render_spell_charge_lines 0x801B1C04, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **E317's bicone callback (CB92, FUN_801c44a0) projects each 3D vertex with the GTE's single-vertex perspective command RTPS (`copFunction 0x480012` — rotate + translate + perspective divide in one instruction) after `RotMatrix_gte`/`SetRotMatrix` (0x8001D0A8/0x8001D138) load the camera matrix at 0x80098A24, reading screen X/Y and depth back through the GTE's 3-deep output FIFO (SXY0/SXY1/SXY2) while projecting the next cross-section; the companion trail ribbon (CB91) uses the matrix-multiply path (ApplyMatrixLV 0x8001D578) instead.** — `[S] 1/3`
  - S: copFunction 0x480012, camera matrix 0x80098A24, per `research/working_documents/E317_callback_system.md`
  - src: `research/working_documents/E317_callback_system.md`

- **FFT's battle map projection is a pure-affine ortho at the battle viewing angle (no perspective divide): a world point `(tile_x×28+cx, −height×12, tile_z×28+cz)` (cx,cz ∈ {0,28}; +14 = center) is rotated by the 9-int16 view matrix at `0x80098A24` (4096 = 1.0; row0→sx, row1→sy, row2 = depth-only), scaled by the 3-int16 `sprite_scale` at `0x800C7CA0` (isotropic in practice), and offset by a 2-value datum calibrated live as the robust (median → mean-of-inliers) center of `screen − linear(V)` across all on-field units — the datum anchors to the game's own `+0x120` output instead of the GTE translation register, so the grid extrapolates correctly to the full 14×10 map beyond the units' x∈[1..6], z∈[0..5] hull.** — `[S·D] 2/3`
  - S: view matrix `0x80098A24` (9 int16, 4096 = 1.0, row0→sx / row1→sy / row2→depth), sprite_scale `0x800C7CA0` (3 int16, 4096 = 1.0), map width/depth u8 `0x800E4E9C` / `0x800E4EA0` — address table, per `research/working_documents/psx_tile_grid/reproject_tile_grid.md`
  - D: `reproject_tile_grid.py` live reprojection on sstate `reference-assets/orbonne_female_knight_held_by_simon.sstate` (2026-07-06): 9 live units, 0.27px RMS / 0.53px worst; one anchor outlier auto-rejected
  - R: none — PSX GTE affine map reprojection not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`; `src/debug/MapGridOverlay.gd` is the Godot-side grid overlay the tool A/Bs against, not the PSX transform)
  - src: `research/working_documents/psx_tile_grid/reproject_tile_grid.md`

## Notes

(empty — user territory)

## Related

- [[Custom Effect Hooks]]
- [[Embedded MIPS Effect Code]]
- [[Unit Sprite Object Struct]]
