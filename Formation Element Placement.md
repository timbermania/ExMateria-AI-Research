# Formation Element Placement

Byte-exact placement and scale of every formation-cell element (unit body, drop shadow, gold box, orb, HP numbers), grounded static (address-cited Ghidra decompile in `roster_overlay_decompile.c`) → dynamic (live pcsx RAM under roster sstate0) → visual (256×240 VRAM framebuffer vs oracle `t04.png`). The body is centred at a fixed cell offset and drawn at its descriptor-native size 1:1 (centre = cell + (31, 24)); the drop shadow's flat-on-the-floor look is a 2:1 vertical squash — a 20×10 on-screen rect sampling a 20×20 texel — gated on body width Uw < 0x19, not a POLY skew; every element is emitted by one axis-aligned POLY_FT4 setter so nothing is ever skewed. The Godot port landed this on 2026-07-17 (`rom_body_rect`/`rom_shadow_rect`, feet-baseline placement, `FormationPlacementTest` green).

## Points

- **The formation unit body is placed at a fixed cell offset and drawn at its descriptor-native size 1:1 — top-left = (cellX − Uw/2 + 31, cellY − Vh/2 + 24), centre = (cellX + 31, cellY + 24) independent of sprite size, with no scale factor in the path; `FORMATION_SCREEN.md` §12.3's "cellX − 31 + ½Uw" had both signs flipped (corrected there)** — `[S·D·R] 3/3`
  - S: `FUN_80117db8` @ `0x80117db8` (X = cellX − (Uw/2 − 31), Y = cellY − (Vh/2 − 24); W = Uw, H = Vh ⇒ native), descriptor `FUN_801256c8` @ `0x801256c8` — `roster_overlay_decompile.c` lines 21915–21954
  - D: roster sstate0 live-RAM cross-check (2026-07-17, `SCUS94221.sstate0`, `/tmp/rr_read_prims.lua` via curl `/lua/exec`): shadow template X=211 Y=130 pins the cell7 body rect to (211,100,24,40) on screen
  - R: `godot-learning/src/ui3/formation/FormationScene.gd` `rom_body_rect` / `BODY_ANCHOR_DX/DY = 31/24` (branch `import-godot-game`) + `godot-learning/tests/FormationPlacementTest.gd` asserting cell7 body (211,100,24,40) — green
  - src: `research/working_documents/FORMATION_ELEMENT_PLACEMENT.md`
- **The drop shadow's "perspective on a tile" look is a 2:1 vertical squash, not a POLY skew: it is drawn into a 20 (W) × 10 (H) on-screen rectangle while sampling a 20 × 20 texel, producing the wide flat horizontal-ellipse battle-shadow blob** — `[S·D·R] 3/3`
  - S: `FUN_8011814c` @ `0x8011814c` (shadow template static at `0x8018c704`; subtractive tpage `GetTPage(0,2,0x3c0,0x100)` = 0x5F) — `roster_overlay_decompile.c` lines 22000–22019
  - D: sstate0 live read of shadow template `0x8018c704` (2026-07-17): W=20 H=10 Uw=20 Vh=20, clut=0x7EE4, tpage=0x005F; squash confirmed visually in `placement_fb_20260717.png`
  - R: `godot-learning/src/ui3/formation/FormationScene.gd` `rom_shadow_rect` / `SHADOW_SCREEN_PX = 20×10` + `tests/FormationPlacementTest.gd` asserting cell7 shadow (211,130,20,10) — green
  - src: `research/working_documents/FORMATION_ELEMENT_PLACEMENT.md`
- **The shadow-offset gate is on the body's width Uw (< 0x19), not Vh: narrow sprites (roster units) place the shadow at (bodyLeft, bodyTop + 30), wide sprites (monsters) at (bodyLeft + 12, bodyTop + 35)** — `[S·D·R] 3/3`
  - S: `FUN_8011814c` @ `0x8011814c` (gate `param_1[6] < 0x19`; `sRam8018c704/06` narrow vs wide branch) — `roster_overlay_decompile.c` lines 22000–22019
  - D: sstate0 live read (2026-07-17) pins the narrow branch: cell7 Uw=24 < 25, shadow Y=130 = bodyTop + 30
  - R: `godot-learning/src/ui3/formation/FormationScene.gd` `SHADOW_UW_GATE = 25` + `tests/FormationPlacementTest.gd` wide-sprite-branch assertion — green
  - src: `research/working_documents/FORMATION_ELEMENT_PLACEMENT.md`
- **The roster grid is 56×48 cells with on-screen anchors cellX = col·0x3e + 6 (6/68/130/192) and cellY = 36 + row·0x3c (36/96); the setter's `+0x80` X offset is a draw-env centring that cancels (screen is 256 wide), so on-screen X = the local X the code computes** — `[S·D·R] 3/3`
  - S: `FUN_80116264` @ `0x80116264` (grid), `FUN_8012c8bc` @ `0x8012c8bc` (vertex X = localX + 0x80, decompile 32744–32750) — `roster_overlay_decompile.c`
  - D: sstate0 live read (2026-07-17): shadow Y=130 = on-screen cellY 96 + 34 — second independent data point validating the on-screen coordinate model
  - R: `godot-learning/src/ui3/formation/FormationScene.gd` `GRID_ORIGIN=(6,36)`, `COL_PITCH=62`, `ROW_PITCH=60`, `CELL=56×48` + `tests/FormationPlacementTest.gd` `expect_cellx = [6, 68, 130, 192]` — green
  - src: `research/working_documents/FORMATION_ELEMENT_PLACEMENT.md`
- **Every formation-cell element (body, shadow, box, orb, numbers) is emitted by the one sprite setter `FUN_8012c8bc`, which writes an axis-aligned POLY_FT4 (v0/v2 share X, v1/v3 share X, v0/v1 share Y, v2/v3 share Y — a perfect rectangle, zero skew); none of the elements are skewed, so the shadow's "tilt" is purely the vertical squash plus body occlusion** — `[S·D] 2/3`
  - S: `FUN_8012c8bc` @ `0x8012c8bc` (40-byte prim from pool `_DAT_801cd528[4]`, four vertex corners; `param_3 ∈ {0,1,2,3}` only permutes the UV corners = the box's 4-way mirror; tpage at +0x16, clut at +0x0e) — `roster_overlay_decompile.c` lines 32722–32800
  - D: `placement_fb_20260717.png` (2026-07-17, 256×240 framebuffer, captured while running): shadows are wide-flat 20×10 ellipses at each unit's feet, no skew
  - R: none — POLY_FT4 vertex setter not present in godot-learning (probed main + `import-godot-game` worktrees; the port composites quads/shaders instead)
  - src: `research/working_documents/FORMATION_ELEMENT_PLACEMENT.md`
- **The prim-template short array consumed by the setter has fields [0]=X [1]=Y [2]=W [3]=H [4]=U [5]=V [6]=Uw [7]=Vh [8]=CLUT [9]=TPage, where W×H is the on-screen size and Uw×Vh is the texture window from (U,V); the sprite is scaled only when W×H ≠ Uw×Vh, and the shadow is the only roster element that exploits this** — `[S·D] 2/3`
  - S: `FUN_8012c8bc` @ `0x8012c8bc` (screen rect from W/H, texture window from Uw/Vh) — `roster_overlay_decompile.c`
  - D: sstate0 live shadow-template read (2026-07-17) confirms the layout field-by-field (X=211 Y=130 W=20 H=10 U=144 V=64 Uw=20 Vh=20 clut=0x7EE4 tpage=0x005F)
  - R: none — prim-template short array not present in godot-learning (probed main + `import-godot-game` worktrees)
  - src: `research/working_documents/FORMATION_ELEMENT_PLACEMENT.md`
- **Roster unit descriptors are Uw = 0x18 (24) × Vh = 0x28 (40) — the 24×40 UNIT.BIN atlas cell — at tpage 0x64 (UNIT.BIN atlas in VRAM at (256,0)), so every roster body draws 24×40 with no per-job scatter** — `[S] 1/3`
  - S: `FUN_801256c8` @ `0x801256c8` (descriptor read) — `roster_overlay_decompile.c`
  - R: none — tpage 0x64 / UNIT.BIN descriptor read not present in godot-learning (probed main + `import-godot-game` worktrees)
  - src: `research/working_documents/FORMATION_ELEMENT_PLACEMENT.md`
- **Under sstate0 the live roster display struct reads as 8 consecutive cell structs at 0x801C8638 + n·0x128 with per-cell job/palsel/f70 fields, and f70 bit6 set (f70 = 0x50) marks the female units (cells 4–6); cell0 is Ramza/Squire (f70=0x80), cell7 Delita (f70=0x81)** — `[D] 1/3`
  - D: sstate0 live-RAM read via `/tmp/rr_read_state.lua` + curl `/lua/exec` (2026-07-17): cellcount=8, cell0 ptr=0x801C8638 … cell7 ptr=0x801C8E50, 8/8 matching `FORMATION_SCREEN.md` §13.5
  - R: none — live display-struct read not present in godot-learning (probed main + `import-godot-game` worktrees; the port builds cells from its own data)
  - src: `research/working_documents/FORMATION_ELEMENT_PLACEMENT.md`
- **The gold box's gold CLUT is a gradient — idx1 (255,222,123) bright core → idx2..5 dimmer gold → idx6..15 near-black — and the add pass `base·brightness·psx_brightness(2.2)` at brightness 1.0 clips idx1..5 all to max, flattening the gradient into a thick uniform band; dropping the per-box additive level to ~0.45 (0.45·2.2 ≈ 1.0) keeps the gradient so the line reads thinner and falls off at its edges** — `[R] 1/3`
  - R: `godot-learning/src/ui3/formation/FormationScene.gd` `_bind_tunables` — `formation.box_add_level` / `box_sub_level` (+ `box_emboss_dy`) Tune slugs, calibrated headful 256×240 A/B vs `t04.png` (2026-07-17)
  - src: `research/working_documents/FORMATION_ELEMENT_PLACEMENT.md`
  - ⚠ SUPERSEDED (2026-08-17) by: the box level is not a tunable fudge — the ROM writes the trail-fade table value VERBATIM (head slot 128/128 = full 1.0, `FUN_8011712c` / `0x8018C88C`), and on the display-space fold the faithful add is the raw `texel·gouraud/128` with no ×2.2; the sub-full level (0.22/0.45 era) masked the real residuals (see the trail-fade point below + [[Display Space Blend Fold]])
- **The landed port stands bodies on a common feet baseline — visible-bbox bottom-centre on (cellX+31, cellY+40), X at the ROM body centre 31 — with a fixed uniform body_scale ≈ 9 instead of normalizing each unit's visible height to the descriptor Vh 40, because the UNIT.BIN roster mini fills only ~26 px of its 40-px cell and the ROM draws all minis at one common scale (oracle row-1 heads at y 96/97/107/109 — units genuinely differ in height)** — `[D·R] 2/3`
  - D: `t04.png` oracle head-position read + headful 256×240 SubViewport A/B, round 2 (2026-07-17)
  - R: `godot-learning/src/ui3/formation/FormationScene.gd` `derive_body_offset_px` / `feet_target_px = (31, 40)` + `SpriteLayerManager.compute_body_bbox` — validated by headful A/B vs `t04.png` (2026-07-17); `tests/FormationPlacementTest.gd` guards the underlying ROM rects
  - src: `research/working_documents/FORMATION_ELEMENT_PLACEMENT.md`
- **The Godot formation orbs sit at the EXACT same screen coords as PSX — detected centroids 12,40 / 73,41 / 135,41 in both — and the PSX orbs are NOT at the feet: they float up-left of the head, same as Godot (the "placement confounder" fear — Godot orb over dark head-stone vs PSX over lit feet-floor — is disproven by the oracle).** — `[D] 1/3`
  - D: formation oracle, detected centroids in both the PSX and Godot 256×240 captures, sstate1 Ramza/cell 0, pcsx :8080 (2026-07-18)
  - R: none — no test in godot-learning pins the orb centroids to the PSX values (probed main + `fft-monorepo-formation` worktrees `tests/`)
  - src: `research/working_documents/FORMATION_ORB_ADDITIVE_COLORSPACE.md`
- **The formation box gouraud is FULL, not sub-full: `FUN_8011712c` (WORLD.BIN — the BATTLE-region address 0x8011712c is BSS zeros, RAM base 0x800E0000) writes r0/g0/b0 VERBATIM from the trail-fade table `0x8018C88C = {20,35,50,65,80,90,100,128}` indexed by slot (colour assembly `0x801172b4`–`0x801172cc`, setter `FUN_8012c8bc` `0x8012c930`–`0x8012c950`); head slot 7 = 0x80 = 128 = full `gouraud/128 = 1.0`, IDENTICAL for the additive (OT bucket 4) and subtractive (bucket 3) prims, no per-channel diff, no base×fade — so the faithful box level is just `trail_fade` (head 1.0), and the earlier oracle-fit 0.22 `box_add_level` was a fudge masking the real residuals.** — `[S·D·R] 3/3`
  - S: `FUN_8011712c`, trail-fade table `0x8018C88C`, colour assembly `0x801172b4`–`0x801172cc`, setter `FUN_8012c8bc` (`0x8012c930`–`0x8012c950`) — WORLD.BIN
  - D: formation oracle, sstate1 Ramza/cell 0, pcsx :8080 (2026-07-18)
  - R: `godot-learning/src/ui3/formation/FormationScene.gd` `TRAIL_RAMP_FALLBACK = [20, 35, 50, 65, 80, 90, 100, 128]` (the verbatim ROM table; no test pins the values — `tests/FormationFoldRoutingTest.gd` guards routing only)
  - src: `research/working_documents/FORMATION_ORB_ADDITIVE_COLORSPACE.md`
- **The gold-box CLUT is byte-exact in the port: live VRAM CLUT `0x7F65` (VRAM (592,509), 16 u16) == `BOX.palette.tga` under `<<3|>>2` expansion — idx1 (255,222,123) … idx15 (8,0,0) — with all idx1–15 STP bit15=1 and idx0 = 0x0000; re-confirmed byte-exact 2026-07-30 (the +7/+6/+3 per-channel deltas are pure expansion-method artifact, not data drift), refuting the "palette gold brighter than the live CLUT" suspicion.** — `[S·D·R] 3/3`
  - S: CLUT `0x7F65`, VRAM (592,509), 16 u16
  - D: VRAM CLUT dump, sstate1 Ramza/cell 0, pcsx :8080 (2026-07-18); re-confirmed (2026-07-30)
  - R: `godot-learning/assets/ui/formation/BOX.palette.tga` (byte-exact vs the VRAM dump; no dedicated test)
  - src: `research/working_documents/FORMATION_ORB_ADDITIVE_COLORSPACE.md`
- **The A/B compare carries a ~3% systematic offset from the 5→8-bit expansion method: the pcsx screenshot expands `v<<3` (max 248) while the .tga palette uses `v<<3|v>>2` (max 255) — a compare artifact, not data drift.** — `[D] 1/3`
  - D: palette re-confirmation, sstate1 Ramza, pcsx :8080 (2026-07-30)
  - R: none — the expansion-method compare is not present in godot-learning
  - src: `research/working_documents/FORMATION_ORB_ADDITIVE_COLORSPACE.md`

- **The roster body sprite is the unit's `TYPE1` SEQ/SHP sequence 2 ("Face Front") held static and un-mirrored — a static body frame per party slot, not a pose animation.** — `[D] 1/3`
  - D: oracle body-frame identification on the formation roster (user-confirmed the "Face Front" seq-2 match)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`

## Notes

(empty — user territory)

## Related

- [[Formation Screen Compositing]]
- [[Unit Shadow Opcode]]
