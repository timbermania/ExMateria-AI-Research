# Terrain Render Pipeline

How FFT's PSX renderer turns map polygons, unit sprites, and effect particles into one shared Ordering Table. Every terrain polygon (all four types) passes the same 10-stage pipeline: per-vertex GTE RTPS, a two-stage backface cull (pre-computed quadrant bits, then NCLIP screen winding), AVSZ Z-averaging into the OT bucket, a range check, and head insertion. Flat-shaded black border walls carry a two-mode depth toggle: pinned to near-front bucket 4 during normal gameplay, natural GTE depth while spell effects run. Three AddPrim rectangles at OT bucket 383 form the sky. Godot reimplements the depth model as a calibrated per-face depth (AVSZ centroids baked to CUSTOM0, ADR-0009) with the black skirt as a static MAP_SKIRT mode.

## Points

- **Terrain, units, and effect particles all insert into the single shared Ordering Table — there is no floor/wall layering, the OT alone decides front vs back; the main frame renders FUN_800ee95c (view/rotation matrices) → FUN_800e840c (terrain) → FUN_801a1c40 (effects) → FUN_80074bf8 (units) → GPU OT draw, and this call order does NOT determine layering, only same-bucket tie-breaks (terrain inserted 1st = behind, effects 2nd = middle, units 3rd = in front).** — `[S] 1/3`
  - S: main game-loop call order + shared-OT insertion pattern, per `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md` §6.1/§6.4 + battle disassembly (FUN_800ee95c / FUN_800e840c / FUN_801a1c40 / FUN_80074bf8)
  - R: none — PSX call-order same-bucket tie-break not present in godot-learning (single Z-buffer depth model, ADR-0009)
  - src: `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md`
- **Every terrain polygon passes the same 10-stage pipeline regardless of type — no floor/wall distinction for depth, purely camera distance — with each vertex transformed individually through the GTE RTPS command (copFunction 0x480012), screen Z pushed to the GTE SZ FIFO (0x8800→0x9000→0x9800) and screen XY to the SXY FIFO (0x6000→0x6800→0x7000) for later averaging.** — `[S] 1/3`
  - S: per-vertex RTPS in the terrain renderers (FUN_8012cc54 et al.), per `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md` §6.2 Stages 1–2 + battle disassembly
  - R: none — per-vertex RTPS pipeline not present in godot-learning
  - src: `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md`
- **Terrain polygons are backface-culled in two stages: first a fast early-out against pre-computed quadrant-visibility bits (16-bit per-polygon flags at vertex-data offset 0x0E, bits 2–13 = visible-from which camera quadrant, quadrant index = camera angle & 0xC00 >> 10, between-quadrant angles use the alternate bit), then a screen-space winding cull via GTE NCLIP (copFunction 0x1400006, MAC0 register 0xc000 ≥ 0 = front-facing) catching edge-on polygons; a conservative screen-bounds clip (at least one vertex inside [0,0x280)×[0,0x1E0)) runs alongside.** — `[S] 1/3`
  - S: quadrant-cull test + NCLIP winding cull in the terrain renderers, per `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md` §6.2 Stages 4–6 + battle disassembly
  - R: none — pre-computed quadrant backface cull not present in godot-learning
  - src: `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md`
- **Polygon OT depth is the GTE-averaged vertex Z — AVSZ3 (copFunction 0x158002d) for triangles, AVSZ4 (copFunction 0x168002e) for quads, result read from the OTZ register (0x3800) — only Z values are averaged, screen X/Y play no role in bucket assignment; terrain depth outside 0..0x17F (one bucket wider than the 0..0x17E particle/unit clamp) is dropped, not clamped.** — `[S·R] 2/3`
  - S: AVSZ3/AVSZ4 + OTZ range check, per `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md` §6.2 Stages 7–8 / §6.3 + battle disassembly
  - R: godot-learning/src/core/DepthMode.gd (tri_centroid/quad_centroid mirror AVSZ3/AVSZ4, baked to CUSTOM0 per face) + godot-learning/src/map/DynamicGeometryBuilder.gd (reads the pre-computed face centroid) + godot-learning/tests/DepthModeTest.gd (AVSZ centroid fixture test, ADR-0009)
  - src: `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md`
- **The terrain mesh has four polygon types, each with its own renderer and a pre-baked world-space vertex array loaded at map load: textured triangles FUN_8012cc54 / DAT_8011a2d8 (0x18 bytes/poly), textured quads FUN_8012cf44 / DAT_8011c498 (0x20), flat-shaded triangles FUN_8012d2b4 / DAT_80122004 (0x18), flat-shaded quads FUN_8012d568 / DAT_80122604 (0x20); orchestrator FUN_800e840c dispatches all four.** — `[S] 1/3`
  - S: FUN_8012cc54 / FUN_8012cf44 / FUN_8012d2b4 / FUN_8012d568 + DAT_8011a2d8 / DAT_8011c498 / DAT_80122004 / DAT_80122604, per `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md` §6.2 Stage 1 / §6.5 + battle disassembly
  - R: none — four-type terrain mesh pipeline not present in godot-learning
  - src: `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md`
- **Flat-shaded (black) border-wall depth is modified before OT insertion as `depth & mask | offset` (mask = DAT_80121d58, offset = DAT_800fc558), toggled by FUN_800ef6a0: 0x88 on effect start → mask 0xFFFFFFFF, offset 0 (natural GTE depth sorting), 0x87 on idle → mask 0, offset 4 (every flat poly pinned to near-front bucket 4) — pinning avoids wall Z-fighting and paints over border seams in normal play, while effects switch walls to natural depth since the camera may move; initialized to the 0x87 state at startup.** — `[S] 1/3`
  - S: FUN_800ef6a0 + DAT_80121d58 / DAT_800fc558, per `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md` §6.2 Stage 9 + battle disassembly
  - R: none — two-mode wall depth toggle not present in godot-learning (black skirt is a static MAP_SKIRT depth mode, ADR-0009)
  - src: `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md`
- **The black border-wall color is DAT_800f5b58, initialized to R=0, G=0, B=1 — B=1 avoids the PSX GPU treating pure (0,0,0) as transparent — and is settable at runtime via map command 99 and reset to default via map command 100.** — `[S] 1/3`
  - S: DAT_800f5b58, per `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md` §6.2 Stage 9 + battle disassembly
  - R: none — map-command 99/100 wall color not present in godot-learning
  - src: `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md`
- **Animated terrain (windmills, drawbridges, water wheels, multi-part maps) works because orchestrator FUN_800e840c loops over multiple mesh groups — each with its own rotation matrix, world position, GTE matrix reload, and independent polygon slice — while all groups insert into the same OT.** — `[S] 1/3`
  - S: FUN_800e840c mesh-group loop, per `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md` §6.6 + battle disassembly
  - R: none — multi-mesh-group animated terrain not present in godot-learning
  - src: `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md`
- **The sky/background is three AddPrim (0x80023BB4) inserts of colored rectangles at OT bucket 383 (ot_base + 0x5FC — one past the 0..0x17E terrain/particle range), drawn first by the highest-to-lowest OT walk so they sit behind everything; colors come from DAT_801251c8 / DAT_801251cc.** — `[S] 1/3`
  - S: AddPrim 0x80023BB4 + DAT_801251c8 / DAT_801251cc, per `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md` §6.8 + battle disassembly
  - R: none — OT-bucket-383 background fill not present in godot-learning
  - src: `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md`
- **Gouraud shading (bit 0 of the polygon flags word) is computed AFTER OT insertion via GTE DPCT (copFunction 0xd80420) depth-cueing — it fades far polygons toward the fog color but does not change the OT bucket; depth ordering is locked in at insertion.** — `[S] 1/3`
  - S: DPCT pass in the terrain renderers, per `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md` §6.2 Stage 10 + battle disassembly
  - R: none — DPCT depth-cue Gouraud pass not present in godot-learning
  - src: `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md`

## Notes

(empty — user territory)

## Related

- [[Ordering Table & AddPrim]]
- [[GTE World-to-Screen Transform]]
- [[PSX GPU Primitives]]
- [[Particle Depth Mode]]
