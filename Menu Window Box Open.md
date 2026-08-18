# Menu Window Box Open

The "box-open" animation formation menu windows use when they appear — corrected model: a center-out scissor (clip) reveal on a fully-rendered window, with a shared horizontal aperture across the two opening windows.

## Points

- **The menu-window "box open" is a center-out SCISSOR (UV-inset clip) reveal, NOT a geometry scale: the ROM draws each window fully rendered at its settled size, and a clip rectangle opens from the center on both axes along `world_menu_open_curve` @ `0x801533B8` = `[10,10,60,60,90,90,95,95,100,100,100,100]` via `world_menu_window_open_scale` `FUN_800EC954`, shared by 9 screens; the frame chrome appears last only because it sits at the clip edge; this supersedes the earlier geometry-scale reading (RE round 11 correction, 2026-08-05).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.17 (CORRECTION — the box-open is a scissor, not a geometry scale; curve @ `0x801533b8`, shared scaler `FUN_800ec954`)
  - D: per-frame framebuffer capture showing settled-size content appearing through the growing clip; the port's `clip_world` scissor approach reproduces it byte-identically to the ROM frame
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/; `BoxOpenAnimator` lives on the doc's port worktree, not in this checkout)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The two box-opening windows (stats band + Eqp/Ability panel) are two SEPARATE vertical scissors that share ONE horizontal aperture — same x/width every frame, same frame counter and curve; the stats band only looks like it finishes earlier (h=42 vs h=108 — a size illusion); the open advances 1 curve index per vsync (port tick fix: `_BOX_OPEN_TICK = 1/60` vs the slide's `_MENU_TICK = 2/60`).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.17 round-12 correction (shared horizontal aperture; the single-aperture conclusion no single-scissor model can produce)
  - D: frame-by-frame clip-edge tracking of both windows (x/width identical per frame; heights 42 vs 108)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`

## Notes

(empty — user territory)

## Related

- [[Start Action Menu]]
- [[Equip And Ability Panel]]
- [[Formation Screen Compositing]]
