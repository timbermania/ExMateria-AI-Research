# Unit Visibility Flag

FFT PSX on-screen unit visibility is a dedicated per-sprite halfword `unit[+0xa]` (1 = drawn, 0 = hidden; companion `+0x1d8` moves in lockstep), toggled at runtime by `{44} Draw Unit` / `{46} Erase Unit` and held hidden by the INVERTED `Draw` byte of `{45} Add Unit` (same 0=now/1=held convention as `{47}`'s `xDR`). The flag is code-initialised at EVTCHR-load in `evtchr_unit_clut_writer`, gated on the caller's held flag `s5` — the chunk's Draw byte for chunk-added units, and for boot units the ENTD loader's held bit, set only when ENTD flags2 is `0xC0` (both `always_present` AND `randomly_present`). `always_present` alone is a formation/presence gate, never a render gate — a unit whose first visibility op is a reveal starts hidden even with `always_present=True` (scn6 Ovelia at the doorway). Godot implements this model as chunk-derived frame-0 visibility (`_chunk_reveals_first` under a presence gate) plus runtime show/hide handlers, replacing the old wrong `unit.visible = always_present` wiring; live-verified on the scenario-6 doorway beat (2026-07-09).

## Points

- **The unit on-screen render on/off flag is halfword `unit[+0xa]` (1 = drawn / 0 = hidden), with `unit[+0x1d8]` moving in lockstep; the show/hide helpers' display object is the unit struct itself (0x440 stride), so `unit_base+0xa` is directly pollable in RAM.** — `[S·D·R] 3/3`
  - S: `unit_sprite_object_show` @0x8008d138 (`sh 1, 0xa(v1)` @0x8008d158) and `unit_sprite_object_hide` @0x8008d18c (clears `+0xa` and `+0x1d8`) — labeled in `fft-ghidra/content/renames_high.tsv` (cross-ref `UNIT_FADE_COLOR_UNIT_OPCODE.md` §5: same helpers for the door-exit fade)
  - D: scenario 6 doorway-beat live poll (savestate9, pcsx-redux port 8080 `rd16`): Delita show=1/+0x1d8=1/anim=7/scr=(261,138), Ovelia show=0/+0x1d8=0/scr=(252,139) — staged 9 px apart, Ovelia not yet drawn; Agrias visible, Chocobo hidden (2026-07-09)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `draw_unit`/`erase_unit` → `ScenarioWorld.set_unit_visible` (Godot's `Unit.visible` mirrors this flag) — validated by `godot-learning/tests/ScenarioUnitVisibilityTest.gd`
  - src: `research/working_documents/SCENARIO6_UNIT_REVEAL_VISIBILITY.md`
- **`{44} Draw Unit` is the on-screen reveal: its backend `unit_sprite_object_show` @0x8008d138 looks up the unit's sprite object and stores 1 into `+0xa` (show store @0x8008d158).** — `[S·D·R] 3/3`
  - S: `unit_sprite_object_show` @0x8008d138, show store @0x8008d158 (`fft-ghidra/content/renames_high.tsv`, BATTLE.BIN disassembly)
  - D: live write-watch on the four scn6 units' `+0xa` (Circle-driven): Delita's `{44} Draw@88` and Ovelia's `Draw@104` both write through `0x8008D158` (2026-07-09)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_draw_unit` → `ScenarioApply.draw_unit` (`visible = true`) — validated by `godot-learning/tests/ScenarioUnitVisibilityTest.gd`
  - src: `research/working_documents/SCENARIO6_UNIT_REVEAL_VISIBILITY.md`
- **`{46} Erase Unit` is hide-only: its backend `unit_sprite_object_hide` @0x8008d18c clears both `+0xa` and `+0x1d8`, leaving the unit allocated in the sprite list (distinct from the full teardown of `{3D}` Remove Unit).** — `[S·D·R] 3/3`
  - S: `unit_sprite_object_hide` @0x8008d18c (`sh zero, 0xa(v1)` + `sh zero, 0x1d8(v1)`) (`fft-ghidra/content/renames_high.tsv`)
  - D: scenario 6 doorway-beat live poll — hidden Ovelia/Chocobo read `+0xa=0` and `+0x1d8=0` together (savestate9, 2026-07-09)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `erase_unit` (`visible=false`, registry intact — hide only) — validated by `godot-learning/tests/ScenarioUnitVisibilityTest.gd`
  - src: `research/working_documents/SCENARIO6_UNIT_REVEAL_VISIBILITY.md`
- **The `{45} Add Unit` `Draw` byte is INVERTED: `Draw=1` holds the unit hidden, `Draw=0` draws it now — the same 0=now/1=held convention as `{47}`'s ghost `xDR`.** — `[S·D·R] 3/3`
  - S: `evtchr_unit_clut_writer` @0x80087a28 init stores gated on the caller held flag `s5` — the chunk `{45}` producer passes the Draw byte (`Draw=1` → HIDE store @0x80087bc4) — plus the `{47}` `xDR` inverted template (`ADD_GHOST_UNIT_OPCODE_47.md`)
  - D: live — Chocobo's `+0xa` went `1→0` across its `Add@11` (no other op touches it), Delita `1→0→1` across `Add@10`/`Draw@88`; write-watch shows Delita/Chocobo's first `+0xa` write = HIDE store `0x80087BC4` via `Add Draw=1` (2026-07-09; the §3.2 pre-events baseline read landed on pc=0x80000080 and is treated as weak)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `add_unit` — `world.set_unit_visible(uid, draw == 0)` (inverted: Draw=0 draw-now / Draw=1 held) + `commit_unit_graphic` — validated by `godot-learning/tests/ScenarioUnitVisibilityTest.gd`
  - src: `research/working_documents/SCENARIO6_UNIT_REVEAL_VISIBILITY.md`
- **The initial `+0xa` is written by code at EVTCHR-load, not a struct default: `evtchr_unit_clut_writer` @0x80087a28 stores it immediately after the per-unit CLUT store, gated on `s5` (`lw s5, Stack[0x20]` @0x80087a50, branch @0x80087bac) — `s5==0` → show store `sh 1, 0xa(s2)` @0x80087bb8, `s5!=0` → hide store `sh zero, 0xa(s2)` @0x80087bc4 (each also writes `+0x1d8`); `s5` is the caller's held/Draw byte — the chunk `{45}` Draw / `{47}` xDR, or the boot loader's per-ENTD held bit.** — `[S·D·R] 3/3`
  - S: `lw s5, Stack[0x20]` @0x80087a50, branch @0x80087bac, show store @0x80087bb8, hide store @0x80087bc4 (BATTLE.BIN disassembly; `fft-ghidra/content/renames_high.tsv`)
  - D: live write-watch on the four units' `+0xa` (Circle-driven): Delita/Chocobo's first write = HIDE `0x80087BC4` (via `Add Draw=1`); Agrias never took the HIDE store — only the `0x8008D158` show (2026-07-09)
  - R: `godot-learning/src/scenarios/ScenarioPlayerScene.gd` `_chunk_reveals_first` — the chunk's first `{45} Add Draw=1` / `{44} Draw` IS the on-disk encoding of `s5` — validated by `godot-learning/tests/ScenarioUnitVisibilityTest.gd`
  - src: `research/working_documents/SCENARIO6_UNIT_REVEAL_VISIBILITY.md`
- **A boot (ENTD) unit loads held/hidden iff ENTD flags2 & 0xC0 == 0xC0 — BOTH `always_present` (bit 7) AND `randomly_present` (bit 6): `entd_to_roster_loader_16` @0x8017f8a0 reads the flag byte at `roster+0x5` (= flags2; its `&0x30` is team_color, so the old "flags1 & 0xC0" reading was a mislabel), sets the held `s1=1` at the 0xC0 check (0x8017faec–faf8) through the `randomly_present` RNG path, and passes it into `evtchr_queue_enqueue` as `s5` (`sw s1, sp+0x20` @0x8017fb78) — Ovelia (flags2 0xC4) → held → hidden, Agrias (0x84) → visible.** — `[S·D] 2/3`
  - S: `entd_to_roster_loader_16` @0x8017f8a0: `lbu v0, 0x5(s0)`, 0xC0 check 0x8017faec–faf8, `sw s1, sp+0x20` @0x8017fb78, RNG helper `FUN_8017fbec` (`fft-ghidra/content/renames_high.tsv`, BATTLE.BIN disassembly)
  - D: scn6 live-consistent — Ovelia (0xC4) hidden at the doorway beat while Agrias (0x84) is visible (poll + write-watch, 2026-07-09)
  - R: none — boot-held flags2-0xC0 gate not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`: frame-0 visibility is the presence gate + chunk-derived `_frame0_visible`/`_chunk_reveals_first`; `randomly_present` is not read for visibility)
  - src: `research/working_documents/SCENARIO6_UNIT_REVEAL_VISIBILITY.md`
- **A boot unit with `randomly_present` but NOT `always_present` (flags2 & 0x40 without 0x80) is RNG-deactivated from the formation at 0x8017fa20 — a branch separate from the 0xC0 held gate; so `always_present`/`randomly_present` answer "does this unit exist in the formation," not "is its sprite drawn this frame".** — `[S] 1/3`
  - S: cull branch @0x8017fa20 in `entd_to_roster_loader_16` (BATTLE.BIN disassembly, per the doc's §4.4)
  - R: none — formation RNG-cull not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`: `always_present` feeds only the `scenario_present` presence gate in `ScenarioPlayerScene._spawn_units`, never visibility)
  - src: `research/working_documents/SCENARIO6_UNIT_REVEAL_VISIBILITY.md`
- **`FUN_8017fbec` is a 3-mode stateful RNG helper implementing the randomly-present formation bookkeeping (which randomly-present units spawn this battle), not a visibility path: `a0=0xff` resets the counter `DAT_8018f89c`; `a0=0` inits it to 1 and stores the candidate in `DAT_8018f870`; `a0`=else selects over the accumulated count — and `always_present` cinematic principals are never RNG-culled, so their held-at-load is deterministic.** — `[S] 1/3`
  - S: `FUN_8017fbec`, counter `DAT_8018f89c`, candidate list `DAT_8018f870` (BATTLE.BIN disassembly; `fft-ghidra/content/renames_high.tsv`)
  - R: none — randomly-present formation RNG bookkeeping not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/SCENARIO6_UNIT_REVEAL_VISIBILITY.md`
- **A unit's frame-0 visibility is CHUNK-DERIVED, not ENTD-flag-derived: it starts hidden iff its first visibility-affecting opcode in the chunk is `{44} Draw Unit` or `{45} Add Unit Draw=1` (a reveal/hold on cue); a first op of `{46} Erase Unit` (implies already on-screen), `{45} Add Unit Draw=0` (disasm-predicted `s5==0 → +0xa=1`, live-untested), or no visibility op at all → visible. The rule collapses both PSX held paths — the chunk `s5` gate and the boot flags2 `0xC0` gate (a boot-held unit must carry a `{44} Draw Unit` to ever appear) — and matches every observed scn6 and scn1 unit.** — `[S·D·R] 3/3`
  - S: faithful proxy for the `evtchr_unit_clut_writer` `s5` gate @0x80087bb8/@0x80087bc4 and the boot 0xC0 pair gate @0x8017faec (doc §4.3/§4.4; BATTLE.BIN disassembly)
  - D: scn6 observed units — Delita 5 (`Add Draw=1`@10 held → `Draw@88` reveal), Ovelia 12 (`Wait Time=28`@99 → `Draw@104`, landing ~28+ ticks after Delita), Agrias 52 (`Erase@71` first → starts visible), Chocobo 139 (`Add Draw=1`@11 held → `Draw@184`) — live poll + write-watch (2026-07-09); scn1 walk-ins `Add Draw=1` → hidden → `Draw`-revealed, Simon no vis-op → visible
  - R: `godot-learning/src/scenarios/ScenarioPlayerScene.gd` `_chunk_reveals_first` (the exact first-op rule) + `_frame0_visible` (presence master gate) — validated by `godot-learning/tests/ScenarioUnitVisibilityTest.gd` (synthetic branch coverage, scn6 ordering fixture, real scn1/scn6 oracle)
  - src: `research/working_documents/SCENARIO6_UNIT_REVEAL_VISIBILITY.md`
- **The top-level `{46}` Erase dispatch `jal` sits at `0x8013EB74` in `FUN_8013e904`, but erases inside a `0x2A`/`0x2B` Block run under the block-body executor `FUN_8013edd8` @ `0x8013EDD8` — so a dispatch-site BP misses block-internal erases, while the handler BP `0x8008D18C` catches every erase regardless of which dispatcher called it (scenario 1's Exit-B erases are block-internal, chunk instrs 339–378).** — `[S·D] 2/3`
  - S: `0x46` case `jal` @ `0x8013EB74`, `unit_sprite_object_hide @ 0x8008d18c`, block processor `FUN_8013edd8` @ `0x8013EDD8` (`battle_disassembly.txt`)
  - D: observed live driving scenario 1's Exit-B — the block-internal Erases did not fire at the top-level dispatch site (savestate `orbonne_three_actors_walk_in.sstate`, 2026-06-30)
  - R: none — the block-body-executor dispatch distinction is not present in godot-learning (probed `godot-learning/src/scenarios/`, `godot-learning/tests/`; blocks are pure brackets — `{46}` itself is `ScenarioApply.erase_unit`, validated by `ScenarioUnitVisibilityTest.gd`)
  - src: `research/working_documents/UNIT_FADE_COLOR_UNIT_OPCODE.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[ENTD Unit Deployment Table]]
- [[Add Ghost Unit Opcode]]
- [[Wait Add Unit Opcode]]
- [[Unit Sprite Object Struct]]
- [[EVTCHR CLUT Resolution]]
- [[Scenario Beat Capture]]
- [[Dead Unit Fade]]
- [[Color Unit Opcode]]
