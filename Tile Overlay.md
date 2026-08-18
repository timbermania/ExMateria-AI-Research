# Tile Overlay

The Godot reimplementation's tile overlay system draws per-tile map effects (placement markers, cursor, barber-pole shimmer) as flat map decals with CLUT-based per-type colour. Verified: the barber-pole is a 15-phase CLUT-index rotation (a permutation of palette entries 1..15, not a texture animation); every transparent overlay type is a uniform flat CLUT fill (`flat_fill = true`, `flat_index = 2`) rather than a textured sheet sample — `CURSOR_ACTIVE`, the only sheet-sampled (barber-pole) overlay, is opaque (blend mode 4) and never enters the transparent compositor; and each tile's colour resolves to a single solid colour per (type, phase) computed on the CPU each frame, collapsing the per-type `srgb_gamma` / `mono` / `tint` divergences into that one colour. These facts are what the display-space fold's `tile_overlay` routing must mirror (only blend modes 1/2/3 route; modes 0/4 stay in-scene). A 2026-07-21 universal-routing scoping pass (`COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`) characterized the in-scene construction: per-tile 4-vertex `ArrayMesh` quads with a 4-vertex CUSTOM0 centroid, `TILE_OVERLAY` depth mode 7, `depth_draw_never` (overlapping overlays rely on correct fold order), modes 0–3 mapping 1:1 to the fold's mode space, hardware-blended unclamped into the base buffer — the high-volume Category-A case — alongside a 2-surface tile cursor (`TileCursor.gd`) whose opaque body stays in-scene while only the semi surface routes (per-object OT depth).

## Points

- **The tile overlay's barber-pole is a CLUT index rotation, not a texture animation: `tile_overlay_color` rotates the palette entry as `rot = ((idx-1+phase)%15)+1` over palette entries 1..15 — a pure permutation of palette entries driven by a per-frame phase.** — `[R] 1/3`
  - R: `godot-learning/assets/shaders/tile_overlay.gdshaderinc` (`tile_overlay_color`, doc-cited 2026-07-21, live)
  - src: `research/working_documents/COMPOSITOR_INPUT_CONTRACT_GENERALIZATION.md`
- **Every transparent tile overlay type is a flat CLUT fill, not a textured sheet sample: `PLACEMENT_PLAYER`, `PLACEMENT_ENEMY`, `PLACEMENT_CONTESTED` are all `flat_fill = true` with a uniform CLUT-index fill (`flat_index = 2`), and `CURSOR_ACTIVE` — the only `flat_fill = false` (barber-pole-sheet) overlay — is opaque (blend mode 4) so it never enters the transparent compositor.** — `[R] 1/3`
  - R: `godot-learning/src/map/TileOverlayConfig.gd` (`_seed_defaults()`; `flat_index = 2` "fixed CLUT hue for flat tiles", doc-cited 2026-07-21, live)
  - src: `research/working_documents/COMPOSITOR_INPUT_CONTRACT_GENERALIZATION.md`
- **A tile overlay's colour resolves to a single solid colour per (type, phase) — `pow(rotate(CLUT[flat_index], phase), srgb_gamma)`, optional `mono`, then `× tint` — computed on the CPU each frame, so the per-type `srgb_gamma` / `mono` / `tint` divergences all collapse into that one colour.** — `[R] 1/3`
  - R: `godot-learning/src/map/TileOverlayConfig.gd` (per-type `srgb_gamma`/`mono`/`tint` params) + `godot-learning/assets/shaders/tile_overlay.gdshaderinc` (doc-cited 2026-07-21, live)
  - src: `research/working_documents/COMPOSITOR_INPUT_CONTRACT_GENERALIZATION.md`
- **The tile overlays `tile_overlay_mode0..3` are built as per-tile 4-vertex `ArrayMesh` quads with a 4-vertex CUSTOM0 centroid, `TILE_OVERLAY` depth mode 7, and `depth_draw_never` — so overlapping overlays rely on correct fold order — with modes 0–3 mapping 1:1 to the fold's mode space and the blend hardware-run unclamped into the base buffer in-scene (the high-volume case).** — `[R] 1/3`
  - R: `godot-learning/src/map/Tile.gd:107-148` (doc-cited 2026-07-21)
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`
- **The tile cursor `tile_cursor_semi_mode0..3` is a billboard quad with per-object OT depth on a 2-surface mesh (opaque body + semi outline) — the opaque surface stays in-scene as canvas+depth (like `RM_OPAQUE`) and only the semi surface routes to the fold.** — `[R] 1/3`
  - R: `godot-learning/src/scenes/TileCursor.gd:194-232` (doc-cited 2026-07-21)
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`

## Notes

(empty — user territory)

## Related

- [[Display Space Blend Fold]]
- [[Map Tint]]
