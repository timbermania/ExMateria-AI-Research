# Sprite Move Opcode

Event instruction `{3B}`/`{6E}` Sprite Move: the arm step does not move the unit — it spawns a per-frame lerp coroutine (`FUN_80146940`, task kind `0xB`) into a free scheduler slot and returns; the coroutine writes the unit's `+0x60/+0x62/+0x64` offset triple one lerp step per yielded frame (true per-frame lerp, not snap-at-Wait), and `{6F} Wait Sprite Move` is a pure main-thread barrier that spins until that unit's move coroutine ends. The offsets change only at elapsed-frame boundaries, and PSX and Godot spend the slide on the same parks (202/206/210/213/216/225 on the scn6 carry). It drives no animation and no facing — a scenario walk cycle across a slide must come from a concurrent low-range `Unit Anim` holding the body SEQ. A 2026-06-28 refinement proved the `+X/+Z/+Y` operand is the **absolute value of the sub-tile offset field `+0x60..+0x64`** (the move ends at `home + operand`, not a world coordinate), not a cumulative delta; bit-exactly verified all four `Type` easing curves against live per-frame traces (each curve parameterised by the `Unknown` weight byte); and completed the ScenarioVM reimplementation (per-unit concurrent moves via `ScenarioApply`/`ScenarioMotion`) with a dedicated 52-assertion test. The cinematic move is also confirmed to run through a separate path from the in-battle walk/animation state machine (`FUN_80069E68`/`FUN_80079A98`), whose `0x3C` animation state at `+0x7f` was once misread as the `0x3B` opcode.

## Points

- **[S·D·R] 3/3** — The `{3B}` Sprite Move arm step changes nothing: it spawns a per-frame interpolation coroutine (`FUN_80146940 @ 0x80146940`, task kind `0xB`) into a free slot and returns; the coroutine's loop `do { yield(); offset = lerp(start, target, frame/Time); } while (frame < Time)` writes `unit+0x60/62/64` one lerp step per yielded frame (true per-frame lerp, not snap-at-Wait) — and `{6F} Wait Sprite Move` (`FUN_80146F5C @ 0x80146F5C`) is a pure main-thread barrier, `do { yield(); } while (any slot kind == 0xB && unit matches)`: it gates the main thread, it does not create the motion.
  - S: `FUN_80146940 @ 0x80146940` (coroutine loop), `FUN_80146F5C @ 0x80146F5C` (barrier spin) (battle_disassembly.txt)
  - S: `{3B}`/`{6E}` arm installs the handler pointer `FUN_80146EE4` ({3B}) / `FUN_80146F20` ({6E}) + state `0x0B` in the slot table at `DAT_8016986C + slot*0x400`; the slot comes from allocator `FUN_80149C48` (a0 < 0x10 = direct slot, a0 == 0x10 = scan from `DAT_80174038+1` via `FUN_8014CC94`), `FUN_8014c8a0` sets slot+0x48 ACTIVE with tags `DAT_801698b8[slot]=0xb` / `DAT_801698bc[slot]=resolve(Unit)`, the worker completes by calling `FUN_8014c958` (clears slot+0x48), and `{6F}` dispatches at `0x80145018` (battle_disassembly.txt, per `SPRITE_MOVE_INVESTIGATION.md` §3b + completion-signal decode)
  - D: single-pass `scn_step` sweep pc203→225 (2026-07-08): no field changes at the `{3B}` arm parks (204/208/211/214); bit-exact live trace in `scenario_1_captures/SPRITE_MOVE_INVESTIGATION.md`
  - D: scenario 1 capture (2026-06-28): live decode stream `… 3b 6f …` for unit 0x13; baked `scenario_1_chunk.json` `{3B}` at offset 452 (unit 0x13, `+X=0x1C`, `Time=0x28`), trigger `orbonne_priest_walk.sstate` + Circle
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` (`sprite_move_intent` / `sprite_move_offset`) + `godot-learning/src/scenarios/ScenarioApply.gd` (`sprite_move`, `{6F}` barrier) + `godot-learning/tests/ScenarioSpriteMoveTest.gd`
  - src: research/working_documents/SCENARIO_WAIT_SEMANTICS.md

- **[D·R] 2/3** — PSX and Godot spend the Sprite-Move slide on the same parks: every `{3B}` offset change lands on the `{6F}` that spins its coroutine, or on the immediately-following `{F1}` when no `{6F}` follows (`{3B}`@211 spends its frame in `Wait(4)@212`) — matching park-for-park across parks 202/206/210/213/216/225; the Round-0 eyeball "no move" at the `{6F}`@205/@209 was actually a perceptually tiny lift (Y −3→−5, Z +8→+16) that the eye reads as stillness.
  - D: single-pass PSX `scn_step` sweep pc200→225, base `scenario6_letgo_full_base` (2026-07-08), read against Godot's `probe_beat_sweep`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (post-§8k step-boundary fix) + `godot-learning/tools/probe_beat_sweep.gd`
  - src: research/working_documents/SCENARIO_WAIT_SEMANTICS.md

- **`Sprite Move` is a pure position lerp by design — it drives no animation and no facing; a scenario walk cycle across a slide must come from a concurrent low-range `Unit Anim` holding the body SEQ (chapel priest: the inst-73 `Unit Anim` walk is what animates his legs through the inst-75–76 slide).** — `[S·D·R] 3/3`
  - S: sibling ROM handlers `FUN_80149398` ({11} Unit Anim), `FUN_80148284` ({2D} Rotate Unit), dispatcher `FUN_80143bd8` (`SPRITE_MOVE_INVESTIGATION.md`, `scenario_1_captures/`)
  - D: chapel trace `last_run/godot_all_units_trace.jsonl` (2026-06-28, burn-through): uid 0x13's anim field changes only on the inst-73/77 `Unit Anim` opcodes, never during the 75–76 slide
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` (`sprite_move` — position only) + `godot-learning/tests/ScenarioSpriteMoveTest.gd`
  - src: `research/working_documents/scenario_1_captures/HANDOFF_priest_walk_animation.md`

- **The `{3B}`/`{6E}` position operands (`+X/+Z/+Y`, each 16-bit signed) are an absolute target position, not a delta — the move slides the unit to that target and repeated moves don't accumulate** — `[S·D·R] 3/3`
  - S: `FUN_80149C48 @ 0x80149C48` (arm handler), dispatcher `FUN_80143bd8` (`SPRITE_MOVE_INVESTIGATION.md`, `scenario_1_captures/`)
  - S: the actual per-move lerp is `FUN_80146940`, which reads the current value via `FUN_8008ca48` (returns `&unit+0x60`, the sub-tile offset field zeroed at spawn by `FUN_8008ca7c`) and writes it through the ADD/accumulate accessor `FUN_8008c9c4`; the renderer draws at base `+0x40` + offset `+0x60` (`0x80042b1c` callers), so the operand is the absolute value of the OFFSET field and the move's end position is `home + operand`, not a world coordinate (`SPRITE_MOVE_INVESTIGATION.md` §3b + 2026-06-28 correction entry)
  - D: live check at the priest's move start: base `[0x40]=(154,-84,154)` (≈ tiles (5.5,-3,5.5)) with offset `[0x60]=(0,0,0)` (2026-06-28, `SPRITE_MOVE_INVESTIGATION.md` correction entry)
  - D: operand-patched trace `probe_sprite_move_delta_vs_abs.py` on the three-actor walk-in `orbonne_three_actors_walk_in.sstate` (2026-06-28): absolute-target vs delta behaviour proven
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` (`sprite_move` — endpoint = home + operand, "endpoint is absolute-from-home, so repeated moves don't accumulate") + `godot-learning/tests/ScenarioSpriteMoveTest.gd` (multi-move absolute test)
  - src: research/working_documents/scenario_1_captures/HANDOFF_sprite_move_implementation.md

- **`{3B}` parameter layout is `Unit:2, +X:2, +Z:2, +Y:2` (signed), `Type:1, Unknown:1, Time:2` with `Time` in 1/60 s frames, and the move is a straight-line slide that ignores terrain (not a pathfind)** — `[S·D·R] 3/3`
  - S: `FUN_80149C48 @ 0x80149C48` (`SPRITE_MOVE_INVESTIGATION.md`) + opcode catalog `godot-learning/assets/scenarios/event_instructions.json` (`verified:true` for `{3B}`/`{6E}`; doc-era path `event_opcodes.json`)
  - D: scenario 1 capture (2026-06-28): baked `scenario_1_chunk.json` `{3B}` at offset 452 — unit 0x13, `+X=0x1C`, `Time=0x28` (≈1 tile east over ≈0.67 s); live trigger `orbonne_priest_walk.sstate` + Circle
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`_op_sprite_move` — sign-extends the 16-bit fields, Time → duration; conversion divisor 28 opcode-units/tile, axis remap X→world X, Z→world −Y, Y→world Z) + `godot-learning/tests/ScenarioSpriteMoveTest.gd` (delta/sign + reaches-target assertions)
  - src: research/working_documents/scenario_1_captures/HANDOFF_sprite_move_implementation.md

- **`{6E} Sprite Move Beta` is identical to `{3B}` except the last field is `Speed` (units/frame) instead of `Time` — duration derives from distance/Speed — and it shares the same ROM handler `FUN_80149C48`** — `[S·R] 2/3`
  - S: `FUN_80149C48 @ 0x80149C48` (shared handler; `SPRITE_MOVE_INVESTIGATION.md`, `scenario_1_captures/`)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`_op_sprite_move_beta`, duration = 4·distance/Speed) + `godot-learning/tests/ScenarioSpriteMoveTest.gd` (Beta speed assertion)
  - src: research/working_documents/scenario_1_captures/HANDOFF_sprite_move_implementation.md
  - ⚠ SUPERSEDED (2026-08-19) by: `{6E} Sprite Move Beta` differs from `{3B}` only in the last field — `Speed` (units/frame) instead of `Time`, with frame count = `sqrt(Σd²·0x10)/Speed` = 4·dist/Speed (×4 over raw distance) — and both opcodes fall into the same dispatch case at `LAB_80144FA0` (`0x80144f70` ori s0,zero,0x3b / `0x80144f94` ori v0,zero,0x6e) into wrappers `FUN_80146EE4` ({3B}) / `FUN_80146F20` ({6E}), each calling the shared per-frame lerp `FUN_80146940` (mode 0/1); `FUN_80149C48` is only the task-slot allocator, not a handler

- **The ROM easing for all four `Type` values (0 linear, 1 ease-out, 2 ease-in-out, 3 ease-in) is a closed-form curve parameterised by the `Unknown` weight byte and reproduces the hardware bit-exact** — `[S·D·R] 3/3`
  - S: `FUN_80149C48 @ 0x80149C48` (handler stores Type/Unknown and steps position; closed forms in `SPRITE_MOVE_INVESTIGATION.md`, `scenario_1_captures/`)
  - S: closed forms (f = t/T, w = weight, cw = 16−w, integer 8.8 fixed-point, `muldiv_64` truncated toward zero, final write `(pos + (pos<0?0xff:0)) >> 8`): Type 0 `f`; Type 1 `f + w·f·(1−f)/16` (w=16 → `1−(1−f)²` ease-out, w=1 ≈ linear + ~1.5% bump); Type 2 piecewise-symmetric about f=0.5, branching on `2t < T`, mirrored about T/2 (w=16 → quadratic ease-in/out); Type 3 `(w·f² + cw·f)/16` (w=16 → `f²` ease-in, w=0 → linear); baked scenario-1 data is all weight=1, so every curve is near-linear (2026-06-28 verify entry, `SPRITE_MOVE_INVESTIGATION.md`)
  - D: live per-frame position trace `probe_sprite_move_trace.py` + operand-patched traces for Types 1/2/3 `probe_sprite_move_curves.py` (2026-06-28, deterministic trigger `orbonne_priest_walk.sstate` + Circle)
  - R: `godot-learning/src/scenarios/ScenarioMotion.gd` (`curve(easing, f, weight)`) + `godot-learning/tests/ScenarioSpriteMoveTest.gd` (easing endpoints + midpoints, trace-pinned integer-ROM reference)
  - src: research/working_documents/scenario_1_captures/HANDOFF_sprite_move_implementation.md

- **Sprite Move is per-unit and concurrent — multiple units (Ramza + Gafgarion + a squire) walk in at once from independent start/end blocks, so move state is tracked per unit, never as one global lerp** — `[D·R] 2/3`
  - D: scenario 1 capture `orbonne_three_actors_walk_in.sstate` (2026-06-28): three concurrent walk-ins from independent blocks
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` (per-unit `world.arm_motion(key, m)` registration) + `godot-learning/tests/ScenarioSpriteMoveTest.gd` (concurrency assertion)
  - src: research/working_documents/scenario_1_captures/HANDOFF_sprite_move_implementation.md
- **The Sprite Move offset triple (`+0x60/+0x62/+0x64`) is also zeroed by `{70} Jump` — in addition to re-placement (warp re-bases and zeroes it) — so the offset is a transient, not a persistent unit property.** — `[S] 1/3`
  - S: `{70}` Jump handler `FUN_8013e708` loads the offset, negates it, and calls the add/accumulate `FUN_8008c9c4`, which zeroes `+0x60..+0x64` (battle_decompilation.c, per `SPRITE_MOVE_INVESTIGATION.md` 2026-06-28 correction entry)
  - R: none — no `{70} Jump` handler in godot-learning's scenario VM (probed `godot-learning/src/scenarios/`, `godot-learning/tests/`)
  - src: `research/working_documents/scenario_1_captures/HANDOFF_unit_tile_alignment.md`
- **The cinematic `{3B}`/`{6E}` Sprite Move does not enter the in-battle unit animation/movement machinery: that region is a 58-case per-unit action/animation state machine (`FUN_80079a98`), not the 176-opcode event VM, whose real frame-by-frame position interpolator `FUN_80069E68`/`FUN_8006A20C` works on target coords `+0x80..+0x82`, current `+0x7c..+0x7e`, moving flag `+0x140` bit 1 (cleared by `FUN_80069254` on completion), and writes animation state `0x3C` to unit byte `+0x7f` — an adjacent value a static pass mis-read as opcode `0x3B`.** — `[S·D] 2/3`
  - S: `FUN_80079a98` 58-case switch; `*(param_1+0x7f)=0x3c` in `FUN_80069e68`; interpolator fields `+0x7c..+0x82`, moving flag `+0x140` bit 1, `FUN_80069254` (battle_decompilation.c, battle_disassembly.txt, per `SPRITE_MOVE_INVESTIGATION.md` §0/§3)
  - D: Exec BP `BEGIN_80069e68` at `0x80069E68` stayed 0 across every chapel-cinematic run while the Layer-A event-VM handlers fired (`probe_sprite_move_safe.py`, 2026-06-27)
  - R: none — in-battle walk interpolator (FUN_80069E68/FUN_8006A20C) not present in godot-learning (probed `godot-learning/src/scenarios/`, `godot-learning/tests/`)
  - src: research/working_documents/scenario_1_captures/SPRITE_MOVE_INVESTIGATION.md

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Scenario Wait Semantics]]
- [[Unit Sprite Object Struct]]
- [[Unit Sprite Render Pipeline]]
- [[Walk To Opcode]]
- [[Unit Anim Opcode]]
- [[Rotate Unit Interpolation]]
- [[Block Execution]]
- [[Dialogue Box Geometry]]
