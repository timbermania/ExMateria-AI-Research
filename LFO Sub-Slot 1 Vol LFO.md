# LFO Sub-Slot 1 Vol LFO

FFT's vol-LFO sub-slot 1 is driven by the 0xE5/0xE6/0xE7 trio operating on the active byte at `chan+0x11e`: 0xE5 (3 params, `FUN_8001676c`) does the full init (MODE=1 into `chan+0x11c`, direction into `chan+0x11d`, active flag `0x1` or `0x3` into `chan+0x11e`), 0xE6 (0 params, PC `0x8001686c`) re-arms by setting bit 0, and 0xE7 (0 params, PC `0x80016884`) disarms by clearing bit 0. Bit 0 of `chan+0x11e` gates the per-tick LFO dispatch at PC `0x80017564`, which sets `CHAN1_VOL_PRESTAGE` (chan_word_1 bit 0x100) every iteration while armed, and the disarm leaves the accumulator/step intact so a later 0xE6 re-arm resumes from the pre-disarm value. The 2026-05-16 `reraise_no_music` investigation pinned this state machine down from the fact that Godot's dispatcher had no 0xE6/0xE7 handlers — voice 21's vol-LFO stayed armed for the entire spell, producing 143 spurious vol_register writes and cos_dist 0.705; both handlers are now wired in smd-player's shared opcode table and mirrored in fft-sound-driver.

## Points

- **Opcode `0xE5` (3 params) fully initializes the vol-LFO sub-slot 1: it writes the direction byte to `chan+0x11d`, MODE=1 to `chan+0x11c`, and the active flag — `0x1`, or `0x3` when p2 carries bit `0x10` — to `chan+0x11e`; handler `FUN_8001676c` (jumptable slot `0x80028CA0`), and the flag's secondary bit `0x2` survives 0xE6/0xE7.** — `[S·D·R] 3/3`
  - S: `FUN_8001676c` (stores at `0x800167f8`/`0x800167fc`/`0x80016800`) + jumptable slot `0x80028CA0` — doc §4 MIPS listing
  - D: `probe_lfo_subslot1_state`, `reraise_no_music` (2026-05-16) — chan_base `0x80037878` active on 168/313 rows, first active cad 0, from bytecode arm `0xE5 4C 1B 02`
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/e5_3param.gd` (shared table `0xE5`; sequencer routes to shared) + `fft-sound-driver/src/driver/opcodes_lfo_subslot.cpp` — validated by the `reraise_no_music` orchestrator run (`probe_lfo_subslot1_state` pair in `smd-player/workspace/orchestrator/probe_validation_manifest.py`)
  - src: `research/effect_sound/working_documents/RERAISE_VOICE_21_LFO_E7_DISARM_MISSING.md`
- **Opcode `0xE6` (0 params) re-arms the vol-LFO sub-slot 1: it sets bit 0 of `chan+0x11e` without rebuilding the LFO step/period state — handler at PC `0x8001686c`–`0x80016880` (jumptable slot `0x80028CA4`).** — `[S·R] 2/3`
  - S: PC `0x8001686c`–`0x80016880` (`lhu`/`ori 0x1`/`sh` on `chan+0x11e`) + jumptable slot `0x80028CA4` — doc §4
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/set_subslot1_active.gd` (shared table `0xE6`; sequencer-local `sequencer/opcodes/flag_set_e6.gd`) + `fft-sound-driver/src/driver/opcodes_lfo_simple.cpp` — no validating test named in the doc (0xE6 never fires in the reraise capture; open question Q1)
  - src: `research/effect_sound/working_documents/RERAISE_VOICE_21_LFO_E7_DISARM_MISSING.md`
- **Opcode `0xE7` (0 params) disarms the vol-LFO sub-slot 1: it clears bit 0 of `chan+0x11e` — handler at PC `0x80016884`–`0x80016890` (jumptable slot `0x80028CA8`); with no handler, a 0xE5-armed LFO stays active to end of spell.** — `[S·D·R] 3/3`
  - S: PC `0x80016884`–`0x80016890` (`lhu`/`andi 0xfffe`/`sh` on `chan+0x11e`) + jumptable slot `0x80028CA8` — doc §4
  - D: `reraise_no_music` probe deltas (2026-05-16) — PCSX chan_base `0x80037878` LFO active cads 0..419 then disarmed; Godot without 0xE7 stayed armed to cad 781, contributing all 143 extra rows of the voice-21 `probe_walker_flag_word_entry`/`probe_vol_register`/`probe_vol_inputs`/`probe_vol_lr_staging` cluster (voice-21 cos_dist 0.705 pre-fix)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/clear_subslot1_active.gd` (shared + sequencer table `0xE7`) + `fft-sound-driver/src/driver/opcodes_lfo_simple.cpp` — validated by the `reraise_no_music` orchestrator run (doc §9 expectations: `probe_walker_flag_word_entry` 785/785, `probe_vol_register` 585/585, voice-21 cos_dist 0.705→≤0.05, KEYONs stay 77/77)
  - src: `research/effect_sound/working_documents/RERAISE_VOICE_21_LFO_E7_DISARM_MISSING.md`
- **Bit 0 of `chan+0x11e` gates the per-tick LFO dispatch: the consumer at PC `0x80017564` reads the active byte each tick and, while bit 0 is set, sets `CHAN1_VOL_PRESTAGE` (`chan_word_1 |= 0x100`) every iteration, which drains into `WALKER_FLAG_VOL_LR_RAW` — with the bit cleared the gate fails and the next walker pass sees `walker_flag_word == 0` and skips the slot entirely.** — `[S·D·R] 3/3`
  - S: PC `0x80017564` (per-tick LFO dispatch loop) — doc §4
  - D: `reraise_no_music` voice-21 trace (2026-05-16) — all 143 Godot-only extra `walker_flag_word_entry` rows carry `walker_flag_word = 1` (`WALKER_FLAG_VOL_LR_RAW` alone, no opcode-dispatch bits)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd:235` (`lfo_sub_active[sub_idx] == 0` skip) + `:292` (mode-1 `CHAN1_VOL_PRESTAGE` arm) — validated by the `reraise_no_music` orchestrator probe pairs
  - src: `research/effect_sound/working_documents/RERAISE_VOICE_21_LFO_E7_DISARM_MISSING.md`
- **The disarm is bit-only: 0xE7 touches only bit 0 of `chan+0x11e` and never resets the LFO accumulator/step, so a later 0xE6 re-arm resumes from the pre-disarm accumulator value — consistent with the 0xE5 re-init path, where FFT's `pitch_lfo_period_reset` (PC `0x80016DC0`) resets the accumulator and reloads the countdown but does not zero `step_current`.** — `[S·D·R] 3/3`
  - S: PC `0x80016884`–`0x80016890` (bit-0-only RMW) + PC `0x80016DC0` (period reset) — doc §4/§8.4
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/clear_subslot1_active.gd` / `set_subslot1_active.gd` clear/set only `lfo_sub_active[1]` and `slot.word_11e` bit 0 (accumulator/step untouched) — no validating test named in the doc
  - D (additive): `probe_lfo_subslot1_state`, `cure_4_no_music` voice 19 at the cad-497 0xE5 re-arm entry (2026-05-14) — both engines enter with the identical sub-slot 1 state (cd=1, dir=0x03, acc=0, mode=1, inner_reload=8); the only diff is a benign 1-LSB step_source gap (−227690788 PCSX vs −227690787 Godot) that the 16-bit right-shift truncates
  - R (additive): `smd-player/addons/exmateria_sound/runtime/shared/opcodes/e5_3param.gd` (0xE5 re-init preserves `lfo_sub_step_current[1]` — the unconditional zero was dropped; in-code comment cites the probe verification) — validated by the post-fix `probe_vol_inputs` `chan_88` 0/264 (v19) / 0/217 (v18) on `cure_4_no_music` (2026-05-14)
  - src: `research/effect_sound/working_documents/RERAISE_VOICE_21_LFO_E7_DISARM_MISSING.md`

## Notes

(empty — user territory)

## Related

- [[LFO Sub-Slot 0 Pitch LFO]]
- [[LFO Sub-Slot 2 Pan LFO]]
- [[LFO Callback Jumptable]]
- [[LFO Sub-Slot Period Reset]]
- [[SMD Opcodes]]
- [[Effect Sound Audio Divergence]]
