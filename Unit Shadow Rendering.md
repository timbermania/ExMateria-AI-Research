# Unit Shadow Rendering

FFT battle units draw their ground shadow as a single semi-transparent textured `POLY_FT4` — a fixed ±10 world-unit quad pinned to the terrain tile under the unit's feet, GTE-projected and inserted as its own ordering-table primitive at the unit's depth. Land blend is subtractive (TPAGE `0x5F`, ABR mode 2); over water it is additive with a 4-phase UV shimmer; ledges reproject the quad onto the tile below. The footprint never scales with jump/fly height — it slides in X/Z and re-snaps Y per tile. Every element is grounded in a live Orbonne PCSX-Redux capture (raw GPU packet + VRAM texture/CLUT extraction), and godot-learning now ports it faithfully: a `blend_sub` ROM-textured quad, terrain-raycast ground-locked, draped to the terrain normal, co-sorted with the sprite on a shared OT representative point, and pixel-aspect-stretched like the rest of the scene. One known desync remains: sub-tile attack-lunge nudges (screen-space move-opcode pans) never reach the shadow, so its center lags the sprite's feet during lunges (Bug 1, deferred).

## Points

- **The unit's ground shadow is emitted by the unit render dispatch `FUN_80086640`, gated on `unit[0x298] != 0`, which calls renderer `FUN_8007d5d0` @ `0x8007D5D0` with OT slot `ot_base + unit[0x128]*4` — the shadow is inserted as its own ordering-table primitive at the unit's depth, not welded to the billboard sprite (a jumping sprite rises while the blob stays planted).** — `[S·D·R] 3/3`
  - S: `FUN_8007d5d0` @ `0x8007D5D0`, dispatch gate in `FUN_80086640`, addPrim call @ `0x8007DAF4` (BATTLE.BIN disassembly)
  - D: Orbonne entry/addPrim BP capture, 8 fires each (2026-07-03, `shadow_captures/shadow_capture.lua`, state `reference-assets/orbonne_three_actors_walk_in.sstate`)
  - D: re-verify rig register map — entry BP `0x8007D5D0` (a0=unit), addPrim BP `0x8007DAF4` (s1=packet), builder BP `0x8007B9D0`; arming script `shadow_captures/shadow_capture.lua`, `python3 -m pcsx_agent --port 8080` (2026-07-03)
  - R: `godot-learning/src/units/UnitShadow.gd` + `godot-learning/assets/shaders/shadow_blob.gdshader` (separate ground-decal node, own depth sort) — headful-verified in Orbonne Prayer, ScenarioPlayer (2026-07-03)
  - src: `research/working_documents/UNIT_SHADOW_RENDERING.md`
- **The shadow primitive is a semi-transparent textured quad `POLY_FT4` (GPU code `0x2E`) with flat color `(0x80,0x80,0x80)` = neutral 1.0 modulation; land TPAGE `0x5F` decodes to ABR mode 2 = subtractive `back − front` (full, clamped ≥0), so per channel `out = clamp(ground − clut_gray, 0, 255)` and the blob subtracts away to darken the ground.** — `[S·D·R] 3/3`
  - S: TPAGE `0x5F` ABR/4-bit decode + code built as `0x2C | 0x02` (`UNIT_SHADOW_RENDERING.md` §3; BATTLE.BIN)
  - D: live addPrim packet `code=2E len=9 rgb0=80 80 80 tpage=005F` (Orbonne, 2026-07-03)
  - R: `godot-learning/assets/shaders/shadow_blob.gdshader` `render_mode blend_sub` — framebuffer − ALBEDO matches ABR mode 2 exactly (2026-07-03, headful-verified)
  - src: `research/working_documents/UNIT_SHADOW_RENDERING.md`
- **The shadow texture is a 20×20 4-bit radial-gradient blob in VRAM page `(960,256)` with UVs `(144,64)–(163,83)`, and CLUT = `unit[0x10] + 0x40` resolves to the same 16-entry grayscale ramp for all units — `{idx0/15 transparent, idx1=0xE0 (max subtraction at center) … idx14=0x08 (≈0 at rim)}` — so shadow darkness scales with ground brightness and is effectively unit-independent.** — `[S·D·R] 3/3`
  - S: VRAM page/CLUT decode (§3.3/§4, index map `shadow_captures/shadow_texture_indexmap.txt`)
  - S: full 16-entry gray ramp re-listed by the handoff: `{–, E0, C8, B8, A8, 98, 88, 78, 68, 58, 48, 38, 28, 18, 08, –}` (2026-07-03)
  - D: 1 MB VRAM raw dump + live `clut=7900/7901/790C` across 3 captured units (Orbonne, 2026-07-03)
  - R: ROM blob copied to `godot-learning/assets/materials/unit_shadow.png` with CLUT baked (center 224, rim 8, corners α=0); shader samples it `filter_nearest` + `discard` on transparent texels (2026-07-03, headful-verified)
  - src: `research/working_documents/UNIT_SHADOW_RENDERING.md`
- **The footprint is a hard-coded ±10 world-unit square (20×20 ≈ 0.71 of a 28-unit tile) centered on the unit's *animated* ground X/Z (base position + all animation offsets), and vertex builder `FUN_8007b9d0` @ `0x8007B9D0` sets all four corner Ys to the terrain tile surface `(tile[+2]+(tile[+3]>>5))·−0xC` — the unit's own `+0x42` height is computed and discarded, so the shadow never scales on jump/fly: it stays ground-locked at constant size, slides in X/Z, and re-snaps Y per tile; slopes drape per-corner via tile-shape byte `tile[+4]` when `tile[+3]&0x1f` ≠ 0.** — `[S·D·R] 3/3`
  - S: vertex builder `FUN_8007b9d0` @ `0x8007B9D0` (BATTLE.BIN)
  - S: builder reads `base +0x40/44` plus the animation position offset (BATTLE.BIN, per `research/working_documents/handoff_unit_shadow_followups.md`)
  - S: per-shape-code corner-Y math for slope codes `0x11/0x14/0x25/0x41/0x44/0x52/0x53/0x58/0x69` in the vertex builder `FUN_8007b9d0` @ `0x8007B9D0` (BATTLE.BIN; re-cited by handoff 2026-07-03 — Godot drapes corners on the terrain mesh instead)
  - D: live corners at entry BP — ±10 square, Y = tile surface (−84 vs −96 on different tiles), center tracks `pos + off60` (Orbonne, 2026-07-03)
  - D: captured shadow center X = 154 + off60 28 = 182 (Orbonne entry BP capture, 2026-07-03)
  - R: `godot-learning/src/units/UnitShadow.gd` `FOOTPRINT = 20.0/28.0`, straight-down `RayCast3D` for terrain Y + collision normal (unit's airborne Y ignored), no height→scale/opacity (2026-07-03, headful-verified)
  - R: 0.714286² `FACE_Y` quad on `ShadowMesh` in `godot-learning/assets/scenes/Unit.tscn` (`QuadMesh` orientation=1; orientation fixed to world XZ, never billboards — ground decal per `UnitShadow.gd` header) — headful-verified against `shadow_captures/orbonne_shadow_scene.png` (2026-07-03)
  - src: `research/working_documents/UNIT_SHADOW_RENDERING.md`
- **The shadow's unit-struct state is: `+0x298` show-flag, `+0x299` rebuild bit (forced `|1` each frame and self-cleared ⇒ corners rebuilt every frame the shadow draws), `+0x29a` water-shimmer clock, and `+0x29c…+0x2b8` the four SVECTOR ground corners consumed by the GTE RTPS calls.** — `[S·D·R] 3/3`
  - S: field table §2.1 (BATTLE.BIN)
  - D: live `f298=1 f299=00 f29a=00` reads at entry BP, corners populated on builder return (Orbonne, 2026-07-03)
  - R: `godot-learning/src/units/UnitShadow.gd` `set_enabled()` show-flag seam (driven by `{4E}` / anim `0xE0`/`0xE1`) — validated by `ScenarioApplyTest._test_unit_shadow_toggles`
  - src: `research/working_documents/UNIT_SHADOW_RENDERING.md`
- **When a unit stands at the edge of a raised tile the shadow is reprojected onto the lower/adjacent tile: a second position is evaluated (`FUN_8007c80c` height resolver) and, if the unit's Y is below that height, the variant selected by the `+0x7f` flag × `tile[+0]&0x40` (base `FUN_8007b688/6a8`, step `FUN_8007b92c/94c`) re-places the quad and the packet is inserted at a different OT depth (`SUB_80044a60`-based) so it sorts against the wall/step it lands on.** — `[S] 1/3`
  - S: ledge branch, second half of `FUN_8007d5d0` + ledge helpers (BATTLE.BIN; exact variant selection still open per doc §8)
  - R: none — ledge reprojection not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/UNIT_SHADOW_RENDERING.md`
- **Land vs water is selected by `tile[+3] & 0xe0` (liquid type); over water the shadow turns additive — TPAGE `0x3F` = ABR mode 1 — while the color stays `80 80 80`: the shimmer is a 4-phase UV re-permutation driven by `water_clock & 0x30`, not a per-vertex color change (the seed doc's "water per-vertex color cycling" was wrong; confirmed UVs by packet layout).** — `[S] 1/3`
  - S: static decode §3.4 (BATTLE.BIN); live water-shield capture still an open item in the doc
  - R: none — water (additive `0x3F`) shadow not present in godot-learning, explicitly deferred (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/UNIT_SHADOW_RENDERING.md`
- **In the Godot port the shadow co-sorts with the sprite: it projects the *same* OT representative point the sprite does (`unit_depth_point`, pushed per frame from `Unit.gd` → `UnitShadow.update()`), mirroring the ROM's insertion of the shadow packet at the unit's own OT slot; `SHADOW` DepthMode (mode 9) nudges `SHADOW_FORWARD = 0.05` — one hair behind the sprite's `UNIT` nudge (0.1) — so the sprite always wins where they overlap, and the old world-space `Y_BIAS` lift was deleted entirely in favour of the unified Ordering-Table depth model (ADR-0009).** — `[S·R] 2/3`
  - S: shadow packet inserted at OT slot `ot_base + unit[0x128]*4` (§2, BATTLE.BIN)
  - R: `godot-learning/src/core/DepthMode.gd` (`SHADOW = 9`, `SHADOW_FORWARD = 0.05`) + `godot-learning/assets/shaders/shadow_blob.gdshader` `psx_ot_depth(unit_depth_point, …, 9)` — commit `5aee0a46` (2026-07-04), `tools/check_depth_shaders.py` preflight passes, headful-verified in Orbonne Prayer
  - src: `research/working_documents/UNIT_SHADOW_RENDERING.md`
- **The Godot shadow applies the full-stretch pixel-aspect correction (Pattern 1: `clip_pos.x *= psx_par` via `psx_par_full`, identical to `tile_overlay.gdshaderinc`) — it had been the one scene mesh applying zero `psx_par`, so its screen-X slid off the sprite's feet, worse the further the unit sat from screen-center.** — `[R] 1/3`
  - R: `godot-learning/assets/shaders/shadow_blob.gdshader` `psx_par_full(clip_pos)` — commit `5aee0a46` (2026-07-04), headful-verified at all camera positions
  - src: `research/working_documents/UNIT_SHADOW_RENDERING.md`
- **Shadow centering is faithful for full-tile walks (the Unit node itself moves) plus depth and PAR placement; only sub-tile attack-lunge nudges desync — the SEQ move-opcode offset is a camera-relative screen-space texture pan applied to the sprite's `shared_loc_offset` (dash/jump `distort_offset` likewise), which never reaches the shadow, so its center (unit raw X/Z only) lags the sprite's animated feet during lunges (Bug 1, deferred; impl spec in `UNIT_SHADOW_RENDERING.md` §8).** — `[R] 1/3`
  - R: `godot-learning/src/units/Unit.gd` `_on_move_offset_changed` (move-opcode pixel offset feeds the sprite-only `shared_loc_offset` uniform; raw `global_position` passed to the shadow at `Unit.gd:1154`) vs `godot-learning/src/units/UnitShadow.gd::update()` (centers on `unit_global_position` X/Z only, no sprite offset folded in) — headful Orbonne walk-in desync
  - src: `research/working_documents/handoff_unit_shadow_followups.md`
- **The on-screen shape of the PSX shadow is a flat iso diamond (~28×12 px) — just the iso projection of the ±10 flat world quad — and Godot reproduces it for free by placing the quad in world XZ, with no screen-space shaping and no billboarding.** — `[D·R] 2/3`
  - D: ~28×12 px diamond blob in the Orbonne scene capture (handoff "Concrete numbers — all live-verified", `shadow_captures/orbonne_shadow_scene.png`, 2026-07-03)
  - R: `godot-learning/assets/scenes/Unit.tscn` `ShadowMesh` world-XZ quad + `godot-learning/src/units/UnitShadow.gd` (ground-locked, no scale/fade/billboard) — headful-verified against the capture (2026-07-03)
  - src: `research/working_documents/handoff_unit_shadow_implementation.md`

## Notes

(empty — user territory)

## Related

- [[Unit Shadow Opcode]]
- [[Unit Sprite Render Pipeline]]
- [[Ordering Table & AddPrim]]
- [[PSX Texture Page Register]]
- [[PSX GPU Primitives]]
- [[GTE World-to-Screen Transform]]
