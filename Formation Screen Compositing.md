# Formation Screen Compositing

The formation screen is a 2D overlay living inside a 3D scene: it deliberately runs depth OFF — every element coplanar at z=0 with draw order set entirely by `render_priority` — and uses silhouette-stencil as the only way to hide the gold box behind the unit body, with a `render_priority` ladder (including the `RP_BOX_SUB < RP_BOX_ADD` patch for Godot's per-quadrant same-depth sort flip) ordering the box's additive/subtractive passes. It folds its transparent content in its own display-space `CompositorEffect` (`FormationDisplaySpaceComposite`) into its own UNORM scratch, which is renderer-portable and not a Mobile dependency. A 2026-07-21 universal-routing scoping recommends the cleaner path: turn depth ON at the scene level for the unit body + its decorative prims (body depth-write, box depth-test only), which collapses the formation into the combat fold's Category A — the unit body occludes the box for free, the OT-depth fold + within-bucket tie-break replaces the RP ladder, and the stencil, the second compositor, and the ortho-placement path all delete.

## Points

- **The formation screen deliberately runs depth OFF — "every element coplanar at z=0 with depth OFF … draw order set ENTIRELY by render_priority" — a 2D overlay living inside a 3D scene; with no depth buffer, silhouette-stencil is the only way to make the gold box hide behind the unit body.** — `[R] 1/3`
  - R: `godot-learning/src/ui3/formation/FormationScene.gd:145-146` (code comment, doc-cited 2026-07-21)
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`
- **`FormationDisplaySpaceComposite` is the formation's own display-space `CompositorEffect` — it folds into its own UNORM scratch, so it is already renderer-portable (not a Mobile-dependency); today the formation composites via stencil occlusion + a `render_priority` ladder + ortho screen-space placement.** — `[R] 1/3`
  - R: `godot-learning/src/effects/FormationDisplaySpaceComposite.gd` (module, doc-named; doc-cited 2026-07-21) + `godot-learning/src/ui3/formation/FormationScene.gd`
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`
- **The gold box's add-over-sub order is patched by a `RP_BOX_SUB < RP_BOX_ADD` `render_priority` ladder because Godot's transparent sort flips same-depth passes per-quadrant (`FORMATION_SCREEN.md §11.2` lines 541-543) — a workaround the recommended depth-based rework would remove in favour of the fold's OT-depth ordering + within-bucket tie-break.** — `[R] 1/3`
  - R: `godot-learning/src/ui3/formation/FormationScene.gd` (RP ladder, doc-cited 2026-07-21); companion `research/working_documents/FORMATION_SCREEN.md` §11.2 lines 541-543
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`
- **`cam.compositor` is a single slot per camera — so with the recommended depth-based formation rework the formation just uses the combat compositor, and the single slot is a feature, not a constraint.** — `[R] 1/3`
  - R: `godot-learning/src/ui3/formation/FormationScene.gd:1201`, `godot-learning/src/scenes/GPUArena.gd:189` (doc-cited 2026-07-21)
  - src: `research/working_documents/COMPOSITOR_UNIVERSAL_ROUTING_SCOPING.md`

## Notes

(empty — user territory)

## Related

- [[Display Space Blend Fold]]
- [[Tile Overlay]]
