# Formation Screen Compositing

The formation screen is a 2D overlay living inside a 3D scene whose display-space add/sub prims (gold box add/sub, orb-rim halo) now route through the ADR-0077 engine fold — the 2026-07-21 recommended rework, executed by 2026-07-30: the unit body writes real-Z depth at its unit rung and the fold's depth test occludes the farther-rung box behind it (PSX OT bucket 3–4 behind body 6), replacing the retired `FormationDisplaySpaceComposite` P2 and its stencil-silhouette occlusion. The in-scene Mobile-blend fallback keeps the depth-OFF `render_priority` design (including the `RP_BOX_SUB < RP_BOX_ADD` same-depth-sort patch), and the floor's `spot_base 1.72` brightening-pool fit (PSX floor gouraud base `0xFF` / floor `0x50`, up to ~2×) corrects the ~2×-too-dark floor that made the additive orb over-pop.

## Points

- **The formation screen deliberately runs depth OFF — "every element coplanar at z=0 with depth OFF … draw order set ENTIRELY by render_priority" — a 2D overlay living inside a 3D scene; with no depth buffer, silhouette-stencil is the only way to make the gold box hide behind the unit body.** — `[R] 1/3`
  - R: `godot-learning/src/ui3/formation/FormationScene.gd:145-146` (code comment, doc-cited 2026-07-21)
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`
  - ⚠ SUPERSEDED (2026-08-17) by: the box now occludes via ADR-0077 real-Z — the unit body writes depth at its unit rung and the fold's depth test occludes the farther-rung box behind it (PSX OT bucket 3–4 behind body 6); the stencil-silhouette path is retired/vestigial (doc 2026-07-30 correction; `formation_unit.gdshader` ADR-0077 header)
- **`FormationDisplaySpaceComposite` is the formation's own display-space `CompositorEffect` — it folds into its own UNORM scratch, so it is already renderer-portable (not a Mobile-dependency); today the formation composites via stencil occlusion + a `render_priority` ladder + ortho screen-space placement.** — `[R] 1/3`
  - R: `godot-learning/src/effects/FormationDisplaySpaceComposite.gd` (module, doc-named; doc-cited 2026-07-21) + `godot-learning/src/ui3/formation/FormationScene.gd`
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`
  - ⚠ SUPERSEDED (2026-08-17) by: the formation's add/sub prims now route through the ADR-0077 display-space engine fold (`FormationScene.fold_shader_for` → `compositor_layer` fold shaders); the Mobile-era `FormationDisplaySpaceComposite` module is deleted (see [[Display Space Blend Fold]])
- **The gold box's add-over-sub order is patched by a `RP_BOX_SUB < RP_BOX_ADD` `render_priority` ladder because Godot's transparent sort flips same-depth passes per-quadrant (`FORMATION_SCREEN.md §11.2` lines 541-543) — a workaround the recommended depth-based rework would remove in favour of the fold's OT-depth ordering + within-bucket tie-break.** — `[R] 1/3`
  - R: `godot-learning/src/ui3/formation/FormationScene.gd` (RP ladder, doc-cited 2026-07-21); companion `research/working_documents/FORMATION_SCREEN.md` §11.2 lines 541-543
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`
- **`cam.compositor` is a single slot per camera — so with the recommended depth-based formation rework the formation just uses the combat compositor, and the single slot is a feature, not a constraint.** — `[R] 1/3`
  - R: `godot-learning/src/ui3/formation/FormationScene.gd:1201`, `godot-learning/src/scenes/GPUArena.gd:189` (doc-cited 2026-07-21)
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`
- **Godot 4.6 Mobile read-modify-write: a subpass self-dependency (the color layer bound as both `color_attachments[0]` and `input_attachments[0]`) is REJECTED by RD validation ("Invalid framebuffer format attachment(0), in pass (0), it already was used for something else before in this pass"), and the color layer's usage `SAMPLING | COLOR_ATTACHMENT | INPUT_ATTACHMENT` (=515, no STORAGE, no CAN_COPY) leaves only the two-pass sample — pass A samples the color layer into an owned scratch (`A2B10G10R10_UNORM_PACK32`, usage `SAMPLING|COLOR_ATTACHMENT`), pass B samples the scratch and writes the color layer back with blend OFF; validated by a linear halve of a 0.667 pixel → measured 0.4824 vs predicted sRGB 0.486.** — `[D] 1/3`
  - D: `TwoPassSampleEffect.gd` / `SubpassSelfReadEffect.gd` spikes in `tools/compositor_prototype/` (2026-07-18); the "no CAN_COPY" ban only covers the transfer/copy OP — a shader-sample blit is fine
  - R: none — the two-pass P2 `FormationDisplaySpaceComposite` no longer present in godot-learning (P2 retired; the fold engine replaced it — see [[Display Space Blend Fold]]); probed main + `fft-monorepo-formation` worktrees
  - src: `research/working_documents/FORMATION_ORB_ADDITIVE_COLORSPACE.md`
- **P2 compositing order: each element read the FROZEN scratch, so an add overwrote the sub (no emboss); FIXED by running two batches — all sub, re-copy scratch, all add — so the gold add lands on the sub-darkened floor, mirroring the PSX OT bucket 3→4 ordering.** — `[D] 1/3`
  - D: formation oracle, sstate1 Ramza/cell 0, pcsx :8080, 256×240 1:1 (2026-07-18)
  - R: none — the two-batch scratch re-copy is not present in godot-learning (P2 retired; the fold now OT-depth-orders runs — see [[Display Space Blend Fold]])
  - src: `research/working_documents/FORMATION_ORB_ADDITIVE_COLORSPACE.md`
- **The PSX formation floor is a BRIGHTENING pool, not a darken-only vignette — floor gouraud base `0xFF` / floor `0x50` (up to ~2×) — and the port that modelled it darken-only (`spot_base = 1.0`) rendered the floor ~2× too dark (framebuffer median luma 30–37 vs PSX 69–78), making any additive orb over it over-pop; fixed with `spot_base 1.72 / swing 0.0068 / min 0.64` (fit to the PSX radial profile, rmse 0.054; re-render medians ~118 near / 42 far vs PSX 103 / 47–56), and the floor pipeline round-trip itself is faithful (at `dark = 1.0` the median luma 69 = the raw CLUT median 70, so `pow(2.2)` + present-sRGB is correct).** — `[D·R] 2/3`
  - D: formation-screen A/B, sstate1 Ramza/cell 0, pcsx :8080, corrected-floor A/B `/tmp/orb_ab_corrected_floor.png` (2026-07-18)
  - R: `godot-learning/src/ui3/shaders/formation_background.gdshader` (`spot_base = 1.72`) + `src/ui3/formation/FormationScene.gd` (`floor_spot_base = 1.72`, `floor_spot_min = 0.64`) + `tests/FormationFloorSpotlightTest.gd` (pins base 1.72 / swing 0.0068 / min 0.64)
  - src: `research/working_documents/FORMATION_ORB_ADDITIVE_COLORSPACE.md`
- **On the (retired) Mobile linear-blend path, the formation box's "too bright" read was COVERAGE, not colour: on the unoccluded left wing the per-pixel gold is the SAME brightness (PSX (221,212,135) luma 206 vs Godot (224,215,134) luma 208), clamping is NOT the differentiator (PSX 177/287 = 62% of additive px clamped vs Godot 265/402 = 66%), and the floor dst under the box lines is near-identical (luma 112.6 vs 110.6) — the real gaps are MISSING OCCLUSION (dominant; Godot's flat `POST_TRANSPARENT` overlay paints the whole diamond over the already-drawn body) + a ~1.2–1.35× thicker line (a sub-pixel `quantize5` + sRGB round-trip residual flipping the dim gradient-shoulder texels — NOT scale: the box `POLY_FT4` prim is exactly 32×16 screen == 32×16 UV 1:1, not double-draw, not trail-smear, not emboss).** — `[D] 1/3`
  - D: full oracle pass — VRAM CLUT dump + per-frame BP box-pass isolation (`/tmp/box_{off,both,add,sub}.raw` via the builder-exit re-poke at `0x80117514`, `linux-verified-methods.md` §6.6) + matched Godot 256×240 capture (`tools/probe_formation_capture.gd`), sstate1 Ramza/cell 0, pcsx :8080 (2026-07-18); gold/sub masks in `/tmp/mask_viz.png`
  - R: none — the P2 `POST_TRANSPARENT` overlay (the occlusion defect) is not present in godot-learning (P2 retired; occlusion now fold real-Z — see [[Display Space Blend Fold]]); probed main + `fft-monorepo-formation` worktrees
  - src: `research/working_documents/FORMATION_ORB_ADDITIVE_COLORSPACE.md`
- **The display-space composite reproduces the PSX orb RIM with ZERO per-mesh constant — pure `texel_srgb·gouraud/128` (base gouraud 128) — and the hard-edge / "too additive" residual is GONE (the rim is now a soft halo matching PSX); the orb CORE stays in-scene (STP=0, byte-exact), and `psx_brightness(2.2)` / `pow(psx_gamma 1.4)` / `box_add_level` are retired from the additive path.** — `[D·R] 2/3`
  - D: formation oracle, sstate1 Ramza/cell 0, pcsx :8080, 256×240 1:1 (2026-07-18); corrected-floor A/B `/tmp/orb_ab_corrected_floor.png` (orb core meanB 198 vs 196)
  - R: `godot-learning/src/ui3/shaders/formation_orb_rim_fold.gdshader` (`ALBEDO = c.rgb * brightness`, brightness IS `gouraud/128` — no pow since 2026-07-31) + `tools/check_no_pow_in_fold.py` + `tests/FormationFoldRoutingTest.gd` (routing guard)
  - src: `research/working_documents/FORMATION_ORB_ADDITIVE_COLORSPACE.md`

- **The `UNIT.BIN` roster sprites are a single 256×480 4bpp atlas + 128 BGR555 palettes; the ROM uploads each unit's own CLUT per slot — unit1 @ (944,480), unit2 @ (992,480), unit3 @ (1040,480), unit4 @ (1088,480) — and the port must do the same per-unit CLUT uploads rather than a single global palette; extra CLUT slots live at (1136,480)/(1184,480).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §11.2/§11.3 (UNIT.BIN atlas/palette layout; per-unit CLUT upload positions)
  - D: live VRAM reads at the five CLUT slots during roster display
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The out-of-battle formation background is the RANGETILE source at LBA `0xE68` + `0x1000` (256×256 4bpp — the same sheet as RANGETILE), CLUT 14 @ `0x91C0`, tpage `0x3F` → VRAM (960,256), tiled 2×2 across the 512×512 world.** — `[S·D·R] 3/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §11.1 (background source LBA/CLUT/tpage + 2×2 tiling)
  - D: live tpage-0x3F VRAM + oracle background crop match
  - R: `godot-learning/tools/parse_range_tiles.py` + `godot-learning/src/ui3/elements/RangeTileAtlas.gd` (the shared RANGETILE atlas pipeline this sheet belongs to)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The menu's VRAM map: offscreen text buffer @ (448,0) (glyphs blitted once at screen open); the UI atlas @ (576,256); menu CLUTs @ (576,506); FRAME.BIN small digits @ (576,490) and big digits @ (576,470); the selection-orb texel @ (624,508); item icons @ (896,256) tpage `0x1E`; item palettes @ (640,254); unit CLUTs (944,480)→(1184,480).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §11.4/§14.6.1/§15.20 (VRAM region table across the menu system)
  - D: live VRAM dumps matching the assigned regions
  - R: `godot-learning/tools/extract_item_sprites.py` + `godot-learning/assets/items/item_icons.tga` (the item-icon region's source extraction); `godot-learning/src/ui3/elements/NumberFont.gd` (the digit regions' consumer-side pipeline)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The roster overlay is a SEPARATE, higher overlay from the world-map overlay — resident ~`0x80100000–0x8012xxxx` (RA `0x8010DF38`) vs the world map at `0x80067BF8` — loaded by the generic overlay loader with LBA @ `0x80168EF8` / SIZE @ `0x80168F38` / DEST @ `0x80168F78` (reader `0x800448A0`); `WORLD.BIN` itself is mounted at `0x800E0000`, so ROM file offset = RAM address − `0xE0000`.** — `[S] 1/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.5/§15.11 (overlay loader table, separate-overlay discovery, WORLD.BIN base)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The formation screen's fonts: `FONT.BIN` = 77,000 B = 2,200 glyphs × 35 B, a single 10×14 **2bpp** proportional atlas (+ width table @ `0x801661FC`) — correcting the earlier 4bpp assumption; `FRAME.BIN`'s small digits use `draw_number_small_font` @ `0x8014AEC0` (U = digit·6 + 0x78, 6×10) and big digits @ `0x8014AC30` (8×16); the menu renders all text into the offscreen buffer at screen open, then draws it as one blit.** — `[S·R] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §14.6.1 (FONT.BIN 2bpp 10×14 layout + width table; FRAME.BIN digit functions; offscreen-text-buffer model) + §4 (original 4bpp assumption, corrected later)
  - R: `godot-learning/tools/parse_fft_font.py` + `godot-learning/tools/parse_frame_font.py` (+ `godot-learning/src/ui3/elements/NumberFont.gd` / `godot-learning/tests/NumberFontTest.gd` for the FRAMEFONT small-digit pipeline)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The formation→detail entry transition is three DECOUPLED phases, not one tween: (1) the roster grid slides up on the `int16[12]` keyframe table @ `0x8018ABE0` (WORLD.BIN file offset `0xAABD6`; first seven keys 144,139,67,31,13,4,0 — an ease-in settle), driven at the ~30Hz menu tick; (2) the dark band cut/swap (the band itself is static — see [[Formation Vitals And Nameplate]]); (3) the menu windows box-open under the scissor clip. The transition renderer is the `FUN_8010D0CC` banner coroutine.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.6/§15.12/§15.17 (three-phase transition; keyframe table; band cut/swap; scissor open)
  - D: frame-by-frame entry capture showing the three phases running on different schedules
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`

## Notes

(empty — user territory)

## Related

- [[Display Space Blend Fold]]
- [[Tile Overlay]]
- [[Ordering Table & AddPrim]]
