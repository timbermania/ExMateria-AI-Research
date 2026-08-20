# GTE World-to-Screen Transform

The PSX GTE (Geometry Transform Engine) world→screen transform used by FFT's effect renderers: load_gte_matrix (0x8001D0A8) and init_matrix (0x8001D138) load the camera's view rotation matrix (32 bytes at 0x80098A24) into the GTE, then rotate_vector (0x8001D578) multiplies the input vector by that camera matrix to produce screen X/Y/Z and depth — because it is a full 3D rotation, position offsets must be applied to screen coordinates after the transform, exactly as the original render_spell_charge_lines (0x801B1C04) does. The same view matrix drives the battle map projection, which is a pure affine ortho (no perspective divide): live-validated to 0.27px RMS over 9 on-field units by the PSX-side tile-grid reproject (2026-07-06), with the translation datum calibrated live from the units' own GTE-stored screen anchors. As of 2026-06-29 the camera view rotation's convention is pinned against 65 live-captured matrices (`R = Rx(pitch)·Ry(yaw)·Rz(roll)`), and the same affine-ortho model is live-confirmed on the unit-sprite billboard projector (store `+0x120/+0x122`, display datum `−128/−6` — see [[Scenario Camera Framing]]).

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
- **The camera view rotation `R` @ `0x80098A24` (9 int16, 4096 = 1.0) is `R = Rx(pitch)·Ry(yaw)·Rz(roll)` with standard right-handed elementary rotations (`Rx = [[1,0,0],[0,c,−s],[0,s,c]]`, `Ry = [[c,0,s],[0,1,0],[−s,0,c]]`), positive angle signs, angles in 4096 = 360° units at `0x800a7784/86/88` (pitch/yaw/roll) acting on the PSX world axes (X = lateral, Y = vertical-down, Z = depth); fitting all 65 live-captured matrices reproduces every sample to max error 0.0019 (the 4096-quantization floor).** — `[S·D] 2/3`
  - S: view matrix `0x80098A24` built by `FUN_800ee95c` (`battle_disassembly.txt`); `work_rotation` @ `0x800A7784/86/88` per `research/working_documents/CAMERA_SYSTEM.md` "Memory Layout"
  - D: `probe_camera_framing.py` — BP at `0x800ee9dc` (right after `RotMatrix` returns inside `FUN_800ee95c`, before ScaleMatrix), 65 distinct (angles→R) samples over yaw 3584→4432, `orbonne_three_actors_walk_in.sstate` (2026-06-29); roll was 0 in every sample, so the Rz placement is assumed last
  - R: none — the exact-`R` basis is implemented flag-gated (`ScenarioCameraDirector._psx_view_rotation` behind `camera_backrotate_pivot`, default OFF) with no validating test; the shipped baseline is the empirical Euler `PSXCameraConvert.psx_angles_to_godot_rotation` (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/scenario_1_captures/camera_framing_pivot_decode.md`
- **The per-unit sprite projection is PURE AFFINE ORTHO, not perspective: `screen = R·SV/4096 + TR` with OFX=OFY=0 and GTE H=512, where the perspective divide feeds only the sprite scale / `+0x128` (OTZ), never the X/Y — the live GTE state reproduces the unit store to the pixel (Agrias feet SVECTOR (154,−84,98): `(R·SV)>>12 + TR` = (235.7, 160.4, 573.5) vs store (235, 160, 138); the perspective form `H·v/SZ` gives (210, 143), wrong), and the screen-X slope stays flat at +20 px/tile across depth `+0x128` 124→155 (a perspective divide would swing ~25%).** — `[S·D] 2/3`
  - S: `SUB_8001d578` projection call inside `FUN_80086b44` @ `0x80086ba0` (state read @ `0x80086ba8`), stores @ `0x80086bdc`/`0x80086c14` into `+0x120/+0x122/+0x128` (`battle_disassembly.txt`)
  - D: BP at the projection call filtered to Agrias (node `0x800b7748`) at the settled savestate `orbonne_prayer_mid_dialog.sstate` (2026-06-29): feet are the exact projected point (all `FUN_8007b96c` Y-sum offsets 0 → no sprite-anchor offset); `probe_ortho_vs_perspective.py` depth sweep: +20.00 px/tile at every depth 124…155 (±0.5 = rounding); `+0x128` is the sprite scale/OT-Z, not the X/Y position divisor (it doesn't even move monotonically with base-Z)
  - R: none — the PSX GTE affine sprite projection is not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/scenario_1_captures/camera_framing_pivot_decode.md`
- **The stored unit screen coords are centre-relative with a +256 framebuffer datum baked in: on-screen X = `+0x120 − 128`, on-screen Y = `+0x122 − 6` (a constant sprite-anchor→feet offset) — verified against the emulator's actual displayed frame at two poses; all 8 live units carry the store.** — `[S·D] 2/3`
  - S: store instructions `sh v0, 0x120(s1)` @ `0x80086bdc` (X) / `sh v0, 0x122(s1)` @ `0x80086c14` (Y); the per-unit anchor offset `unit[0x58]/[0x5a]` picks ± via flag `+0x12 & 0x2/0x4` and is 0 for an idle grounded unit; OFX=OFY=0 the whole time, so the screen centre is added downstream as a framebuffer datum (`battle_disassembly.txt`)
  - D: `probe_find_unit_screen_store.py` (RAM-diff locator: moving only Agrias's world-X moved only her projected fields) + `probe_confirm_screen_store.py` screenshot ground-truth (`PCSX.GPU.takeScreenShot()`, 256×240, blue-sprite segmentation) at two poses: base (154,−84,98) → store (235,160) → on-screen (107,160) [dx 0, dy −6]; +2-tiles-X → store (275,137) → (147,137) [dx 0] (`orbonne_prayer_mid_dialog.sstate`, 2026-06-29)
  - R: none — the unit screen-store display datum is not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/scenario_1_captures/camera_framing_pivot_decode.md`

## Notes

(empty — user territory)

## Related

- [[Custom Effect Hooks]]
- [[Embedded MIPS Effect Code]]
- [[Unit Sprite Object Struct]]
- [[Scenario Camera Framing]]
