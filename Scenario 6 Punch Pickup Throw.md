# Scenario 6 Punch Pickup Throw

The scenario-6 abduction beat where Delita (unit 5) punches Ovelia (unit 12), knocks her back, picks her up, throws her over his shoulder, and turns to walk down the stairs to the chocobo — event PCs 193–231 (map 56, ENTD 387). Live PSX polling and the Godot probe confirm the billboards' world feet-anchors of both leads match PSX exactly at every beat rest point, so the reported "looks off" is not world placement: the beat's motion model is the absolute sub-tile offset triple on a fixed tile position, and the divergence is in the sprite ART riding those correct anchors — the EVTCHR carry poses' baked per-frame Y-offset/pivot and Delita's facing (Godot faces the camera, PSX faces away).

## Points

- **In scenario 6 ("Abducting the Princess", map 56, ENTD 387), event PCs 193–231 drive the Delita-(unit-5) punch → Ovelia-(unit-12) knockback → pickup → throw-over-shoulder → turn beat: punch 193–198 (`0x71` on U5 @193, lunge +Y16 T1, anim 520), knockback 199–215 (three Time-2 staggers with negative X/height and rising depth, SFX 83 thud @206, anims 510/511), pickup/lift 216–224 (reach-down 519, camera push-in @218, lift anims 525/512, both rising +Y32/+Y28 T1), throw+turn 225–231 (shadows off @225/226, throw 526, long T14 settles, first eased step @231 with Type2/w8); every Unit Anim sits in the mid-cinematic band [0x1F4, 0x258).** — `[S·D·R] 3/3`
  - S: `godot-learning/assets/scenarios/chunks/scenario_006_chunk.json` instructions 193–231 (opcode + operand dump)
  - D: live scn6 beat on pcsx-redux port 8080, savestate `scenario6_abduct_punch_pickup_start` (one CIRCLE tap plays the whole beat), both leads' offset trajectories (2026-07-06/07)
  - R: `godot-learning/tools/probe_scenario6_punch_pickup.gd` — both leads render throughout, walkers armed on the cinematic anims, poses progress 520→519→525→526 (Delita) and 509→510→511→512 (Ovelia) in opcode order, `segment_001.tga` bound, palettes 05/0C stable
  - src: `research/working_documents/SCENARIO6_PUNCH_PICKUP_THROW.md`
- **Godot's feet-anchors match PSX exactly at every sampled rest point of the beat: the PSX sub-tile offset triple `+0x60`(+X)/`+0x62`(+Z height)/`+0x64`(+Y depth) ↔ Godot `gp − home` agree to <0.01 tile (baseline pc193, knockback pc206/pc213, lift pc220) — so the reported "sprite positions off" is not the billboard world placement.** — `[S·D·R] 3/3`
  - S: unit-struct offsets `+0x60/+0x62/+0x64`, bases Delita `0x800B8848` / Ovelia `0x800BA608` (struct A, base `0x800B7308`, stride `0x440`)
  - D: same-PC sampling of the live poll (PSX offset triple) and the Godot probe (`gp − home`), Delita home ≈ (8.5, 4.8, 2.5), PSX→world mapping +X→+worldX, +Z→−worldY, +Y→−worldZ, ÷28 (2026-07-07)
  - D: BP-free live offset-field poll across the ride-off — all three units' `[0x60/0x62/0x64]` settle *exactly* on the `(+X,+Z,+Y)` operands (chocobo 139 @ `0x800b8c88`: (0,0,0)→(−140,0,0); Delita 5 @ `0x800b8848`: →(−150,−10,23); Ovelia 12 @ `0x800ba608`: →(−212,45,66)); live base/home `[0x40..44]` reads confirm `[0x62]` is the height field (chocobo (154,−84,378), leads (238,−132,322)); savestate `scenario6_rideoff_start.sstate` parked at instr 344, pcsx-redux port 8080 (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `sprite_move_offset` + `src/scenarios/ScenarioActor.gd` `capture_home` (sticky home; `target = home + offset`) — validated by the home/offset trace in `tools/probe_scenario6_punch_pickup.gd`
  - src: `research/working_documents/SCENARIO6_PUNCH_PICKUP_THROW.md`
- **All of the beat's motion lives in the sub-tile offset triple: both leads' tile position `+0x40/+0x42/+0x44` stays fixed at (238,−132,322) from pc193 through the throw, the big X shift (0→−58) and height rise (10→35) belong to the walk past PC 231, and the offset keeps growing until the home rebase (POS → (182,−96,378), offset → (0,0,0)) hands off to the ride-off — the seam the ride-off warp-disarm fix guards.** — `[D·R] 2/3`
  - D: live poll of Delita/Ovelia `+0x40..44` (tile), `+0x60/62/64` (offset), `+0x29C/+0x29E/+0x2A0` (home anchors 228,−132,352 / 228,−132,332), pcsx-redux port 8080, savestate `scenario6_abduct_punch_pickup_start` (2026-07-06/07)
  - R: godot-learning home capture rock-stable at (8.5, 4.75, 2.5) for both leads with no drift and no re-capture across PCs 102–231 (`ScenarioActor.capture_home`; probe trace 2026-07-07)
  - src: `research/working_documents/SCENARIO6_PUNCH_PICKUP_THROW.md`
- **Side-by-side captures show the carry beat's divergence is in the sprite ART riding the correct anchors, not the anchors: Godot draws Delita facing the camera (his reddish front) through the carry where the PSX lift shows his dark back (facing away), and Godot pivots Ovelia's carry frame up-away from her feet anchor where the PSX A/B shows her blue dress tucked tight against Delita.** — `[D] 1/3`
  - D: grid-off captures `/tmp/punch_f2035.png` (pc220), `/tmp/punch_f2075.png` (pc231) vs PSX `/tmp/psx_lift.png` (2026-07-06) and PSX A/B `/tmp/sxs/cmp_knock.png` (2026-07-07)
  - R: none — carry-pose per-frame Y-offset/pivot and Delita facing not A/B'd in godot-learning; PSX `+0x130` mount-mode carry composition not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/SCENARIO6_PUNCH_PICKUP_THROW.md`

## Notes

(empty — user territory)

## Related

- [[Scenario 6 Carry Composition]]
- [[Scenario 6 Ride Off]]
- [[Scenario Beat Capture]]
- [[Event Opcode Catalog]]
- [[Unit Shadow Opcode]]
- [[Reset Palette Opcode]]
- [[Unit Sprite Object Struct]]
