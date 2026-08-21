# Post-Walker Lookahead

FFT's effect-SMD dispatcher (`smd_dispatcher`, `FUN_80015324`) runs a second, non-dispatching pass after its inner walker loop exits — the post-walker look-ahead (PC `0x80015534`–`0x80015694`) — which peeks ahead from the walker pointer, follows coda/loop redirects, and mutates `chan_word_0` bits `0x200` (VOL_PENDING) and `0x1000` (LAST_NOTE_FLAG) based on the byte that terminates the peek chain. The pass was the root cause of the last `probe_fun80017118_clear` diff on `cure_no_music` (call 279 / cad 349) and of the bit-`0x1000` divergence on every `probe_slur_propagation` row; it is now ported in smd-player's `post_walker_lookahead.gd` (called from `dispatcher.gd` under the same `walker_entered` gate as the SLUR propagation) and mirrored in fft-sound-driver.

## Points

- **After `smd_dispatcher` (`FUN_80015324`) exits the inner walker loop, FFT runs a second pass (PC `0x80015534`–`0x80015694`) that dispatches no opcodes: it peeks the next byte at the walker pointer, and for control opcodes outside the terminator set `{0x80, 0x81, 0x90, 0xB0, 0xB1}` advances the walker (generic opcodes by `1 + arity` via the arg-size table at `0x80028d0c`; `0x99`/`0x9a` follow the top coda frame) and re-peeks, mutating `chan_word_0` only at the terminator (a Note byte `< 0x80` or one of the set) with a single commit at `sh v0, 0x0(s1)` in `LAB_80015694`.** — `[S·D·R] 3/3`
  - S: PC `0x80015534`–`0x80015694` inside `FUN_80015324`; loop top `LAB_80015568`, generic advance `0x800155fc`–`0x80015614`, commit `sh v0,0x0(s1)` @ `LAB_80015694`; annotated disasm in the source doc
  - D: `probe_slur_propagation` capture — bit `0x1000` set on PCSX but not Godot on every row (e.g. `0x1989` vs `0x989`), `cure_no_music` (2026-05-12)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/post_walker_lookahead.gd` (`SharedPerTickPostWalkerLookahead.apply`, mirrors PC `0x80015534`–`0x80015694`) + call site in `runtime/shared/dispatcher.gd` (immediately after the SLUR-propagation block, gated on `walker_entered`) — validated by the `probe_slur_propagation` / `probe_fun80017118_clear` probe pairs (`smd-player/workspace/orchestrator/probe_validation_manifest.py`, run via `smd-player/workspace/orchestrator/run_effect_iteration.py`); mirrored in `fft-sound-driver/src/driver/per_tick.cpp::per_tick_post_walker_lookahead` + `fft-sound-driver/tests/smoke.cpp`
  - src: `research/effect_sound/working_documents/POST_WALKER_LOOKAHEAD.md`
- **The look-ahead terminator decides `chan_word_0`: a Note byte (`< 0x80`) sets bit `0x1000` (LAST_NOTE_FLAG, `ori v0,v0,0x1000` @ PC `0x80015648`); `0x80` Rest clears bit `0x200` (`andi v0,v0,0xfdff`, `LAB_80015660`) and `0x1000`; `0x81` Fermata sets bit `0x200` (`ori v0,v0,0x200`, `LAB_8001564c`) then clears `0x1000`; `0x90` EndBar with no active loop target (`chan+0x1c == 0`) clears only `0x1000` (with an active target it is a non-terminator — the walker is reassigned to `slot+0x1c` and advances); `0xB0`/`0xB1` SlurOn/SlurOff clear both `0x200` and `0x1000` — and since SLUR propagation (`chan_word_0 |= 0x200` at PC `0x80015520`–`30`) sets `0x200` before the look-ahead runs, the net effect is "bit `0x200` set iff the next dispatch is `0x81` Fermata", which on `cure_no_music` (2 EndBar dispatches + 1 Rest) clears it on virtually every walker exit.** — `[S·D·R] 3/3`
  - S: `LAB_8001564c` (0x81 → set `0x200`), `LAB_80015660` (clear `0x200` via `0xfdff`), `LAB_80015688` (clear `0x1000` via `0xefff`), PC `0x80015640`/`0x80015648` (Note → set `0x1000`), PC `0x80015568`–`0x80015584` (0x90 loop-target branch), `0x800155a0` (0xB0/0xB1) — annotated disasm in the source doc
  - D: `probe_slur_propagation` `chan_word_0_pre/post` rows — bit `0x1000` divergence on every row (`0x1989` vs `0x989`), `cure_no_music` (2026-05-12)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/post_walker_lookahead.gd` (match arms `0x80`/`0x81`/`0x90`/`0xB0,0xB1`) + `runtime/shared/slot_state.gd` (`CHAN0_VOL_PENDING = 0x200`, `CHAN0_LAST_NOTE_FLAG = 0x1000`) + `runtime/shared/dispatcher.gd` (SLUR propagation block) — validated by the `probe_slur_propagation` / `probe_fun80017118_clear` probe pairs
  - src: `research/effect_sound/working_documents/POST_WALKER_LOOKAHEAD.md`
- **The per-channel coda/loop stack lives at `chan_base + 0xb0` in 12-byte frames — counter byte at `+0`, `0x99` redirect target pointer at `+4`, `0x9a` redirect target pointer at `+8` — with the stack depth counter (u16) at `chan+0xac` and the top frame computed as `chan_base + depth*12 + 0xb0`; the look-ahead follows the top frame's `+4` pointer on `0x99` when its counter is non-zero (virtual pop at counter 0) and its `+8` pointer on `0x9a` at counter 0, otherwise advances generically.** — `[S·R] 2/3`
  - S: PC `0x80015534`–`0x8001554c` (depth read `lhu 0xaa(s0)`, top-frame arithmetic `depth*12 + 0xb0`), `0x800155b0`/`0x800155c0` (0x99 counter / `+4` redirect), `0x800155e0`/`0x800155f0` (0x9a counter / `+8` redirect) — annotated disasm in the source doc
  - R: `smd-player/addons/exmateria_sound/runtime/shared/channel_state.gd` (`loop_stack` frames with `count`/`back_pos`) + `runtime/shared/per_tick/post_walker_lookahead.gd` (`0x99`/`0x9a` arms; both FFT redirect slots map to the single `back_pos` field) — validated by the orchestrator probe-pair runs (`smd-player/workspace/orchestrator/run_effect_iteration.py`)
  - src: `research/effect_sound/working_documents/POST_WALKER_LOOKAHEAD.md`
- **The 1-row `probe_fun80017118_clear` diff at call 279 (cad 349) on `cure_no_music` was caused by FFT's post-walker look-ahead: after SLUR propagation set bit `0x200`, the look-ahead landed on a clearing opcode (`0x80` Rest / `0x90` EndBar) and cleared bit `0x200` at `LAB_80015660` before the idle-timeout gate `chan_word_0 & 0x600 == 0` in `FUN_8001714C`, so PCSX fired the `chan_word_1 |= 0x2` write (the probe's capture site) while Godot — which left bit `0x200` set — failed the gate and skipped the write.** — `[S·D·R] 3/3`
  - S: `LAB_80015660` (`andi v0,v0,0xfdff`); idle-timeout gate in `FUN_8001714C` (`& 0x600`) with the `chan_word_1 |= 0x2` write; `probe_fun80017118_clear` BP `0x800173B8` per `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - D: `probe_fun80017118_clear` 414/414 with 1 row diff at call 279 (cad 349), `cure_no_music` (2026-05-12)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` (SLUR propagation + post-walker look-ahead under `walker_entered`) + `runtime/shared/per_tick/post_walker_lookahead.gd` — validated by the `probe_fun80017118_clear` probe pair
  - src: `research/effect_sound/working_documents/POST_WALKER_LOOKAHEAD.md`
- **The look-ahead runs only when the outer smd_dispatcher gate `*param_2 != 0 && note_duration == 0` in `FUN_80015324` entered (i.e. the walker actually ran — the same gate as the SLUR propagation), and because the walker-entry snapshot+clear uses mask `0xf8ff` (`chan_word_0 &= 0xf8ff`, preserving bits `0x800` and `0x1000`), a LAST_NOTE_FLAG set by the look-ahead persists into the next walker entry's snapshot and is exposed in the next `chan_word_0` capture even though no currently-modeled gate reads bit `0x1000` (the fun80017118 gate checks `& 0x600`).** — `[S·D·R] 3/3`
  - S: gate `*param_2 != 0 && note_duration == 0` in `FUN_80015324` (doc "Insertion site" section); snapshot+clear `& 0xf8ff` at walker entry
  - D: `probe_slur_propagation` `chan_word_0_post` rows carry bit `0x1000` on PCSX across consecutive walker exits, `cure_no_music` (2026-05-12)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` (`channel.channel_word_0 &= 0xf8ff` at walker entry, current line 530; look-ahead call gated on `walker_entered`) — validated by the `probe_slur_propagation` probe pair
  - src: `research/effect_sound/working_documents/POST_WALKER_LOOKAHEAD.md`

## Notes

(empty — user territory)

## Related

- [[Effect Sound Audio Divergence]]
- [[SMD Opcodes]]
- [[SPU Voice Engine]]
- [[Effect Sound Timing]]
