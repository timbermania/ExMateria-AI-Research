# GTE World-to-Screen Transform

The PSX GTE (Geometry Transform Engine) world→screen transform used by FFT's effect renderers: load_gte_matrix (0x8001D0A8) and init_matrix (0x8001D138) load the camera's view rotation matrix (32 bytes at 0x80098A24) into the GTE, then rotate_vector (0x8001D578) multiplies the input vector by that camera matrix to produce screen X/Y/Z and depth — because it is a full 3D rotation, position offsets must be applied to screen coordinates after the transform, exactly as the original render_spell_charge_lines (0x801B1C04) does.

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

## Notes

(empty — user territory)

## Related

- [[Custom Effect Hooks]]
- [[Embedded MIPS Effect Code]]
