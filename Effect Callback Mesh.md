# Effect Callback Mesh

In the Godot reimplementation, FFT's callback-driven effect geometry — the screen-effect grid/ring family (Spiral, ScreenGrid, WarpedGrid, RadialRing) and the draped/tube/ribbon meshes (`shadow_blob`, ribbons, `trap_charge_line`) — renders as real `ArrayMesh`es rebuilt each frame by `EffectCallback` (`_create_cb_mesh` / `update_render`), carrying per-vertex POSITION/COLOR/UV/CUSTOM0. A 2026-07-21 universal-routing scoping splits the family two ways for display-space routing: the in-scene grids/rings (SpiralMesh, ScreenGrid, WarpedGrid, RadialRing) are world quads/grids with per-face CUSTOM0/OT depth that already speak the fold's quad-geometry contract (Category A — only the staging differs, low cost), while the draped/tube/ribbon family is arbitrary geometry — per-vertex Gouraud colour on real triangle meshes — that the display-space fold's instanced-billboard + shared-sheet input contract cannot express (Category B, high cost, needs a mesh-carrier seam) — and `trap_charge_line` additionally has no OT depth at all today because `ImmediateMesh` silently ignores CUSTOM0.

## Points

- **The in-scene callback grids/rings (Spiral, ScreenGrid, WarpedGrid, RadialRing) and the draped/tube/ribbon meshes (`shadow_blob`, ribbons, `trap_charge_line`) are real `ArrayMesh`es with per-vertex POSITION/COLOR/UV/CUSTOM0 — per-vertex Gouraud colour on arbitrary geometry — rebuilt each frame by `EffectCallback` (`_create_cb_mesh` / `update_render`), which the fold's instanced-billboard + shared-sheet input contract cannot express.** — `[R] 1/3`
  - R: `godot-learning/src/effects/callbacks/EffectCallback.gd` (`_create_cb_mesh` / `update_render`, doc-cited 2026-07-21, live)
  - src: `research/working_documents/COMPOSITOR_INPUT_CONTRACT_GENERALIZATION.md`
  - ⚠ SUPERSEDED (2026-08-16) by: the in-scene callback grids/rings (SpiralMesh, ScreenGrid, WarpedGrid, RadialRing) are world quads with per-face CUSTOM0/OT depth that already speak the fold's quad-geometry contract (Category A, "same contract as particles"); the arbitrary-geometry classification holds only for the draped/tube/ribbon family (`shadow_blob`, ribbons, `trap_charge_line`, Category B)
- **The in-scene callback grids/rings (SpiralMesh, ScreenGrid, WarpedGrid, RadialRing) are world quads/grids with per-face CUSTOM0/OT depth that already speak the fold's two hard contracts (quad geometry + ADR-0009 CUSTOM0/OT depth) — Category A, same contract as particles, low routing cost with only the staging differing; only the draped/tube/ribbon family stays Category B.** — `[R] 1/3`
  - R: `godot-learning/src/effects/callbacks/EffectCallback.gd:166-200, 259-272` (doc-cited 2026-07-21)
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`
- **`shadow_blob` is a ground decal draped onto the terrain normal (per-vertex basis), not a flat billboard — `blend_sub`, SHADOW depth mode 9, co-sorted one hair behind its unit — and cannot be expressed as one 24-float quad transform (Category B).** — `[R] 1/3`
  - R: `godot-learning/src/units/UnitShadow.gd` (doc-cited 2026-07-21)
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`
- **`trap_charge_line` renders as `ImmediateMesh` cylinder tubes with `blend_add`, and carries a double problem for routing: arbitrary geometry *and* `ImmediateMesh` silently ignores CUSTOM0, so it has no OT depth at all today (it fades by history age, not depth) — Category B, and it would first have to move off `ImmediateMesh` to an `ArrayMesh` with CUSTOM0.** — `[R] 1/3`
  - R: `godot-learning/src/effects/TrapChargeLineEffect.gd:284-375` (doc-cited 2026-07-21)
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`

## Notes

(empty — user territory)

## Related

- [[Display Space Blend Fold]]
- [[Embedded MIPS Effect Code]]
- [[Custom Effect Hooks]]
- [[Screen Effect Gradient System]]
