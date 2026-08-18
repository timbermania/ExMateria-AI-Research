# Gold Selection Box

The gold selection box (diamond outline behind the selected formation unit): its primitive/OT structure, its baked-in fade LUT and sprite descriptor (WORLD.BIN), and its one-frame-lagging glide law.

## Points

- **The gold selection box is drawn only when a unit is selected and is built from two ¼-texel 32×16 4-way-mirror pieces (tpage `0x5F`, CLUT `0x7F65`); per box the ROM emits 8 polys — 4 quadrants × an add/sub pass pair, the sub pass offset −1px top / −1px bottom — double-buffered (×2), landing in two OT buckets (3,4) whose OT order is the draw order.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §11.5 (box primitive structure, add/sub `structB.y = structA.y − 1` offsets, OT buckets 3/4)
  - D: per-frame OT dumps across cursor moves show both boxes present with the add-over-sub ordering; settled-state OT shows only slot 7 drawing
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The box fade is a 3-state colour ramp (levels `{0.62, 0.34, 0.18}` tinted `(0.62, 0.71, 0.87)`), NOT alpha — PSX has no per-poly alpha: a 48-colour RGB32 fade LUT is baked into WORLD.BIN at file offset `0xAC88C` (RAM − `0xE0000`), next to a baked sprite descriptor at `0xAC87C` (x=68, y=123, 33×18, `TP2=0x25F`, CLUT `0x7F65`); the data is shared by all 256 units via `BUNIT.OUT` `0x10894` — only the per-unit fade colour index changes. Fade steps follow the table `{20,35,50,65,80,90,100,128}` @ `0x8018C88C` (128 = opaque clamp) driven by a 1/60 timer, faster than the 2/60 menu-slide tick.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §11.5.5 (ISO parsing, byte-anchored: fade LUT @ file `0xAC88C`, descriptor @ `0xAC87C`, `BUNIT.OUT` `0x10894`) + Ghidra label for the fade table @ `0x8018C88C`
  - D: live fade-step sampling matches the 8-entry ramp; the 1/60-vs-2/60 timer split measured across the open/close
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The box position is a one-frame-lagging glide, not direct follow: an 8-slot trail array at `0x801C8344` (slot 7 @ `0x801C8360`) is shifted each frame and the newest entry obeys `slot7 = (3·old + (target − 0x1F)) >> 2` — the ROM writes `target − 31` into the subtrahend slot, a −31 quirk a faithful port must compensate by adding 31 back; slots 0–6 (oldest/dimmest → newest/brightest) are skipped when the cursor is still, so a settled box is static; a live trail is visible in OT dumps (both boxes moving on every cursor move).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §11.5 (smoothing law `(3*old + (target.x - 0x1f)) >> 2`, `k = 31`, trail slots 0–6 skipped at rest)
  - D: OT dumps show the dimmed trail slots appearing only during a move; the settled box draws from slot 7 alone
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`

## Notes

(empty — user territory)

## Related

- [[Formation Screen Compositing]]
- [[Selection Orb And Floor Spotlight]]
- [[Menu Window Box Open]]
