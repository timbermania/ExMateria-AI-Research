# Change Job Screen

The START-"Change Job" screen: a computed elliptical job ring (rcos/rsin), gender-gated membership, and its entry/exit/rotation animations.

## Points

- **The job ring is a COMPUTED rcos/rsin elliptical ring (static-derived, validated to 0.0px on all 19 live members), NOT a baked 2D layout: top-left basis `DAT_8018C918/19 = (124,126)`; the sprite-CENTRE ellipse = centre (128,126), radiusX 100, radiusY 60 (`scaleX*100>>12` / `scaleY*0x3C>>12` at settled scale `0x1000`); `step = int(4096/n)`; `angle = (k·step + rotOffset + 0x400) & 0xFFF` (`0x400` = +90° = front-down); member k=0 = the cursor's job and the slot index DECREASES as the angle increases, so ascending job-array index walks the oval counter-clockwise; the builder `FUN_80118F14` / placement `FUN_80119AA0` are reached only through the function-pointer table `PTR_FUN_8018BAEC[DAT_8018BA1C]` (the placement calls trig INDIRECTLY via `func_0x8001BC28`=rcos / `func_0x8001BB5C`=rsin, so grep for sin/cos finds nothing); the selected avatar (centre (128,123)) is drawn separately by `FUN_80117DB8`, not by the ring loop; this supersedes RE28/29's "no trig ring / baked layout".** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.24 RE round 31 (full ROM model of `FUN_80119AA0(rotOffset, scaleX, scaleY)`, function-pointer reach, indirect trig calls, retracted RE28/29 conclusion)
  - D: live ring primscan (19 tpage 0x64/0x65 CLUT 0x38xx quads) — every member's top-left matched the formula at distance 0.0px
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/; `ChangeJobWheel` lives on the doc's port worktree, not in this checkout)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **Ring membership is 19 for a male oracle: the 20 generic jobs `0x4A..0x5D` minus the opposite-sex gender-locked job (male drops Dancer `0x5C`; female drops Bard `0x5B`) — one body per gender-appropriate job; the gender lock comes from the sex bits of the selected unit's flags word `DAT_801CD52C` (male `0x20` / female `0x40` / monster `0x80`).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.24 (static NEGATIVE for the ring + static PROVEN gender filter via `DAT_801cd52c`)
  - D: live ring count (19 members, male-Ramza oracle) matching the 20-generic-minus-1 rule
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **Change Job entry/exit: the roster splits by a FIXED top-half→LEFT / bottom-half→RIGHT row split (NOT relative to the selected row — `cell.y·2 < ROWS` → top rows left, else right), the ring members SLIDE IN radially from outside the final oval and contract onto it (`start ≈ centre + k·(final−centre)`, k≈1.85 → 1.0; no fade, no scale — pure position), and the job-title plate BOX-OPENS center-out opaque in place over ~2 vsyncs (same mechanism as every other FFT window); EXIT is the reverse — the ring expands and rotates off-screen (k→3.4, ease-in) + the roster re-splits; ←/→ rotation is a SLOT GLIDE, not per-frame trig recompute (`DAT_801C83B8` accumulates 2–3/frame to `0x24`, then the cursor `DAT_801C83F4` snaps by `DAT_801C83BC`(±1) and resets); the centre avatar is fixed; draw order `iVar6` starts `0x1E`, `+1` for front-hemisphere members / `−1` for back, so nearer members draw over farther ones; the plate holds a "Lv. N" line (RANGETILE "Lv." word cell + cream `0x7CBC` FRAMEFONT digits, drawn half ABOVE the plate's top edge) and the job name as a run of per-glyph FONT.BIN quads; the top-right nameplate does NOT change (stays the unit's current job) until confirm.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.24/§15.29 (entry timeline f0–~50, fixed row split, plate box-open, slot-glide rotation) + §15.24 RE32 (exit animation) + §15.24 RE31 (plate content + re-measured plate rect (84,207,88,27))
  - D: oracle entry/exit + rotation captures (upper-left-selected case proving the fixed split; k≈1.9/1.78/1.85 start positions top/bottom/left; plate collision at y186 fixed by the (84,207) re-placement)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`

## Notes

(empty — user territory)

## Related

- [[Equip Sub Screen]]
- [[Roster Display Struct]]
- [[Menu Window Box Open]]
