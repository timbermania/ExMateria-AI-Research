# Concurrent Dialogue Boxes

During the Orbonne opening of scenario 1, FFT puts two (up to three structurally) dialogue boxes on screen at once. A `{10} Display Message` box opens as a cooperative-task kind `0x01` (foreground) and — unless its Dialog byte's `0x80` persist bit is clear, in which case it closes the moment it is advanced — demotes to kind `0x33` (background: still rendered, no longer gated by `Task=1`) when it stops being the foreground; exactly one kind-1 box exists at a time. `{E5} Wait For Instruction (Task=1)` is a player-advance gate on the foreground box only — not an auto type-out wait and not a box-close wait. Boxes occupy the screen-position slot `Dialog & 0x3` (1 = top, 2 = bottom) rather than an open-order slot, and `{51} Change Dialog`'s `Target` byte is that position slot (`Target == align`, `Message=0xFFFF` closes). Pinned by static analysis of scenario-1 idx 147–154, live PCSX capture (2026-07-01), and a full idx 55–220 script drive; the `godot-learning` reimplementation mirrors the model with a 3-box pool, a foreground-scoped kind-1 gate, Target-aware close, and the persist bit.

## Points

- **A `{10} Display Message` box opens as cooperative-task kind `0x01` (stamped by the Display Message handler `FUN_801308c0`); when it stops being the foreground it is demoted to kind `0x33` by the dialog fiber and stays rendered but no longer matches a `Task=1` barrier — the kind-1 foreground fiber (continuation `0x801316C4`) and the kind-0x33 background fiber (continuation `0x80131BD4`) are distinct loops, and exactly one kind-1 box exists at a time.** — `[S·D·R] 3/3`
  - S: `0x801308F4` (kind-1 stamp, ∈ `FUN_801308c0`), `0x80131BA4` (kind-0x33 demote stamp), `0x801316C4`/`0x80131BD4` (foreground/background fiber continuations), kind setter `0x80149d60` (`FUN_80149d48`) — address table in `research/working_documents/scenario_1_captures/concurrent_dialogue_boxes_decode.md`
  - D: scenario 1 capture — `reference-assets/orbonne_dialogue_two_boxes_open.sstate` + `probe_box_kinds.py` 3-savestate comparison + `probe_boxes_bp_orient.py` Exec-BP orientation (2026-07-01)
  - R: `godot-learning/src/scenarios/ScenarioDialogueBoxPool.gd` (background box stays rendered while a new foreground box opens) + `godot-learning/tests/ScenarioBoxedDialogTest.gd::_test_concurrent_boxes_147_154`
  - src: `research/working_documents/scenario_1_captures/concurrent_dialogue_boxes_decode.md`
- **`{E5} Wait For Instruction (Task=1)` is a player-advance gate on the foreground (kind-1) box only — held indefinitely until the player advances the box (Circle in the 2026-07-01 capture), not an auto type-out wait and not a box-close wait; background (kind-0x33) boxes do not hold it.** — `[S·D·R] 3/3`
  - S: `0x80145964` (`{E5}` WFI case entry, `s2`=TaskLo `s5`=TaskHi), kind setter `0x80149d60`/`FUN_80149d48`, advance-action word `0x80166080` (idle `2`, Circle-advance `8`)
  - D: scenario 1 capture — `probe_boxes_resume_trace.py` free-run parked on `op=E5 arg=0001` for 2 s, released only by the Circle advance tap (2026-07-01)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_wait_for_instruction` + `_task_liveness[TASK_DIALOG]` (the indefinite advance park owns the hold) + `godot-learning/tests/ScenarioBoxedDialogTest.gd::_test_concurrent_boxes_147_154` (Task=1 holds until advance)
  - src: `research/working_documents/scenario_1_captures/concurrent_dialogue_boxes_decode.md`
- **A Display Message opens into the screen-position slot `Dialog & 0x3` (align 1 = top, 2 = bottom), REPLACING any box already at that position — not "the next free slot in open order"; a naive open-order allocator leaks boxes (msg4 stranded on slot1 in the first Orbonne event, cascading into wrong-box closes by the 147 beat) while `slot = align` resolves every open/close/swap across the full idx 55–220 dialog script, and the 161–176 "three-box" beat is actually msg10 replacing msg8 at the top position (the live capture never shows more than one top + one bottom box, despite the 3-entry array at `0x80166084`).** — `[S·D·R] 3/3`
  - S: scenario-1 event script idx 147–153 (`Dialog=0x91` top / `0x92` bottom; raw `5101ffff0000`/`5102ffff0000`), `scenario_1_chunk.json`; 3-entry slot array `0x80166084`
  - D: scenario 1 capture — `reference-assets/orbonne_dialogue_two_boxes_open.sstate`: exactly one top + one bottom box (2026-07-01)
  - R: `godot-learning/src/scenarios/ScenarioDialogueBoxPool.gd::_slot_for_dialog` (`dialog & 0x3`, clamped to the wired 3-box pool) + full Orbonne drive idx 55–220 with zero leaks/warnings + `godot-learning/tests/ScenarioBoxedDialogTest.gd::_test_concurrent_boxes_147_154`
  - src: `research/working_documents/scenario_1_captures/concurrent_dialogue_boxes_decode.md`
- **`{51} Change Dialog` has the 5-byte body `Target(1) Message(2) PortraitCol(1) PortraitPal(1)`; `Message=0xFFFF` closes the target box, `Target=N` addresses box N by position slot (`Target == align` — Target=1 closes the top box, Target=2 the bottom), and one close dismisses one box: the 147 beat closes both boxes with two closes 4 frames apart (idx 151 T=1, idx 152 Wait Time=4, idx 153 T=2).** — `[S·D·R] 3/3`
  - S: scenario-1 event script idx 151/153 (raw `5101ffff0000`, `5102ffff0000`), `scenario_1_chunk.json`
  - D: scenario 1 capture — `probe_boxes_resume_trace.py` post-tap00/post-tap01: Target=1 closed the demoted box1, then Target=2 closed box2, 4 frames apart (2026-07-01)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd::_op_change_dialog` → `godot-learning/src/scenarios/ScenarioApply.gd::change_dialog` (0xFFFF → `close_dialog_slot`) + `godot-learning/tests/ScenarioBoxedDialogTest.gd::_test_change_dialog_close`
  - src: `research/working_documents/scenario_1_captures/concurrent_dialogue_boxes_decode.md`
- **Dialog bit `0x80` is the persist flag: a box with the bit set (the 0x9x messages) demotes to background and persists while it stops being the foreground; a box with the bit clear (e.g. the first event's msg2, `Dialog=0x12`) closes the instant it is advanced — so the first Orbonne event (idx 55–83) runs one box at a time, and only beats whose consecutive boxes are all 0x9x (147–154: msg6 0x91 + msg7 0x92) actually stack two boxes.** — `[S·D·R] 3/3`
  - S: `0x80130998` (per-box persist stash in the Display Message handler), `0x80131a2c` (dialog-fiber test), `LAB_80131a80` (teardown when clear) — disassembly per the doc's §3.2
  - D: PSX↔Godot side-by-side step-through of idx 55–83: PSX shows one box at a time on the first event (2026-07-01, per `dialogue_box_visual_parity_investigation`)
  - R: `godot-learning/src/scenarios/ScenarioDialogueBoxPool.gd` (`_slot_persists[slot] = dialog & 0x80`) + `godot-learning/src/scenarios/ScenarioVM.gd::_release_dialog_gate` + `godot-learning/tests/ScenarioBoxedDialogTest.gd::_test_first_event_non_persist_closes_on_advance`
  - R: `godot-learning/tests/ScenarioFirstEventParityProbe.tscn` — parity probe drives the real `ScenarioVM` + 3-box pool through idx 55–83 player-style, printing box-pool state at each stop; all four stops match the PSX capture after the fix (2026-07-01)
  - src: `research/working_documents/scenario_1_captures/concurrent_dialogue_boxes_decode.md`
- **In the first Orbonne event the kind-`0x33` stamp is the box's transient closing state, not a persistent background box: each box stamps `0x01` on open and gets exactly one `0x33` after it stops being the foreground (no demote of msg2 precedes msg3's open — msg2 had already closed), and the slot snapshots show exactly one live slot (`s2:k01`) at every stop with the slot inactive between boxes — unlike the 147-beat where `0x33` persists (`s2:k33` + `s3:k01`).** — `[S·D] 2/3`
  - S: kind-stamp BP @ `0x80149d60` (a0=kind, ra=caller): opens `0x01` from `0x801308F4`, close/demote `0x33` from `0x80131BA4` (BATTLE.BIN)
  - D: scenario-1 first-event capture (2026-07-01): kind-stamp log f7/f243/f495/f653 `0x01` opens, f363/f555/f745 one `0x33` each; slot-kind snapshots `s2:k01` alone at every stop, inactive between boxes (t01/idx 57)
  - R: none — the transient 0x33 closing state is not present in godot-learning (the port models 0x33 only as a persistent background demote in `ScenarioDialogueBoxPool.gd`; non-persist boxes close directly via `ScenarioVM.gd::_release_dialog_gate`; probed `godot-learning/src/scenarios/`, `godot-learning/tests/`)
  - src: `research/working_documents/scenario_1_captures/dialogue_box_visual_parity_investigation.md`

## Notes

(empty — user territory)

## Related

- [[Display Message Opcode]]
- [[Event Dialogue Portrait System]]
