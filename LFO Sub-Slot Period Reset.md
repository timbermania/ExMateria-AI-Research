# LFO Sub-Slot Period Reset

FFT's per_channel_tick (FUN_8001517C) contains a per-tick LFO sub-slot period reset (PC 0x800157AC..0x80015804) gated by s4, which is set only when a Note dispatches with CHAN0_KON_ARM already armed (PC 0x80015488). It snaps every active first-segment sub-slot to a clean baseline (acc = 0, countdown = 1, dir = 3, `chan_word_0 |= 0x100`) so the LFO swap resumes at +step_source instead of drifting through the `(dir | 0x4) ^ 0x8` toggle. Godot lacked the reset until 2026-05-16: on protect_no_music the pan-LFO (sub-slot 2, chan_8a) sign-flipped, flipping the pan and swapping vol_l/vol_r; the fix (originally `_apply_lfo_period_reset` in dispatcher.gd, later refactored to `runtime/shared/per_tick/lfo_period_reset.gd`, which also models the delay/depth reloads) took all three vol probes from FAIL to OK.

## Points

- **FFT's per_channel_tick (FUN_8001517C) contains an s4-gated per-tick LFO sub-slot period_reset at PC `0x800157AC`..`0x80015804` which, for every sub-slot with dir_flags & 0x3 == 0x3 (active + first-segment), zeroes the accumulator (sub+0x04), resets the countdown to 1 (sub+0x10), clears dir bits 0x4/0x8 (dir → 3), and sets `chan_word_0 |= 0x100` (CHAN0_VOL_PRESTAGE) — snapping the next LFO swap back to `step_current = +step_source`, and its absence in Godot made the protect_no_music pan-LFO (sub-slot 2, chan_8a) sign-flip, which flips `pan_base = clampi(0x4000 + chan_8a_signed + pan, 0, 0x7F00)` (play_sound.gd) and swaps vol_l/vol_r.** — `[S·D·R] 3/3`
  - S: PC `0x800157AC`..`0x80015804` (LAB_800157AC..LAB_80015808): acc clear `0x800157D0`, countdown reset `0x800157D4`, CHAN0_VOL_PRESTAGE `0x800157F0`..`0x800157F4`, dir clear `0x80015800`/`0x80015804` — RAM disassembly listing in doc §3
  - D: `diag_chan_13e_writers.jsonl` (Write-BP on voice 21 chan+0x13E, chan_base 0x8003787A) period_reset writes dir 7→3 at cad 0/241/301 + `probe_vol_inputs` 50 row-diffs, all on chan_8a, all exact negations (e.g. +1040 vs −1040 at cad 241+) — protect_no_music (effect_id 9), 2026-05-16
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/lfo_period_reset.gd` (the doc's `smd-player/scripts/effect_sound/dispatcher.gd::_apply_lfo_period_reset`, later refactored into this module) called from the `note_dispatched and (s2_snapshot & CHAN0_KON_ARM) != 0` gate in `runtime/shared/dispatcher.gd` — validated by the post-fix protect_no_music `probe_vol_inputs` / `probe_vol_lr_staging` OK 233/233 and `probe_vol_register` OK 226/226 (drift ±1, 2026-05-16)
  - src: `research/effect_sound/working_documents/CHAN_8A_PAN_LFO_SIGN_FLIP_INVESTIGATION.md`
- **The s4 flag inside per_channel_tick — cleared at entry (PC `0x80015388`) and read at PC `0x80015718` to skip the LFO body — has exactly one write, at PC `0x80015488`, reached only when a Note byte (< 0x80) dispatches this tick AND the s2_snapshot (chan_word_0 read at per_channel_tick entry, BEFORE the per-tick clear at PC `0x80015398`) had CHAN0_KON_ARM (bit 0x400) set, i.e. a Rest, note-duration drain, or effect-load handler armed the next KON in the prior tick; the same block sets `chan_word_1 |= 0x1` (PC `0x8001548C`/`0x80015490`).** — `[S·D·R] 3/3`
  - S: PC `0x80015478` (gate `andi v0, s2, 0x400`), `0x80015488` (`ori s4, zero, 0x1`), `0x8001548C`/`0x80015490` (`chan_word_1 |= 0x1`), `0x80015388` / `0x80015398` / `0x80015718` — disassembly in doc §5
  - D: protect_no_music trace alignment — first chan_8a divergence at cad 241 exactly matches the `diag_chan_13e_writers` period_reset write (row 10, dir 7→3), and all cadences before 241 pair on chan_8a (2026-05-16)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` — the `if note_dispatched and (s2_snapshot & _SS.CHAN0_KON_ARM) != 0: channel.channel_word_1 |= 0x1` gate (port of PC `0x8001548C`/`0x80015490`, the doc's dispatcher.gd line 1099) with the period_reset call added under the same gate — validated by the vol probes turning OK on protect_no_music (2026-05-16)
  - src: `research/effect_sound/working_documents/CHAN_8A_PAN_LFO_SIGN_FLIP_INVESTIGATION.md`
- **FFT's LFO swap path (PC range `0x800174D0`..`0x800177BC`, swap write at PC `0x800177A4`) toggles the sub-slot direction via the bit-0x8 XOR every time the countdown reaches 0, and because `(dir | 0x4) ^ 0x8` leaves bit 0x4 set permanently after the first swap the countdown reload is doubled — so without the period_reset the LFO triangle's direction/cadence drifts from PCSX (observed: Godot's chan_8a keeps accumulating negative instead of flipping ascending at cad 241).** — `[S·D·R] 3/3`
  - S: PC `0x800177A4` (dir_flags swap write), swap+accumulate range `0x800174D0`..`0x800177BC` — doc §3
  - D: `diag_chan_13e_writers.jsonl` swap sequence dir 3→0xF (cad 40), 0xF→7 (cad 121), 7→0xF (cad 201), 0xF→7 (cad 281), 7→0xF (cad 342), between the 0xEC arm at PC `0x800169E8` and the 0xEF disarm at PC `0x80016AFC` (clears bit 0x1) — protect_no_music, 2026-05-16
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd` (swap+accumulate port; period boundary mirrors PC `0x80017760`–`0x800177A4`; the doc's dispatcher.gd:284-552 later moved here) — the drift only resolves with the period_reset (point 1), validated by `probe_vol_inputs` OK 233/233 (2026-05-16)
  - src: `research/effect_sound/working_documents/CHAN_8A_PAN_LFO_SIGN_FLIP_INVESTIGATION.md`
- **The period_reset also reloads the sub-slot delay_counter (sub+0x14 from sub+0x16) and depth (sub+0x18 from sub+0x1A); those reloads are no-ops in FFT for the current opcode stream because the LFO body skips the depth-multiplied branch whenever depth >= 0x100, which is the constant every existing LFO-arm path (0xE5 / 0xED / 0xEC) sets.** — `[S·R] 2/3`
  - S: reload instructions in the period_reset listing — loads at `0x800157C8` / `0x800157CC`, stores at `0x800157D8` / `0x800157E4` — doc §3
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/lfo_period_reset.gd` now models both reloads (`lfo_sub_delay_counter = lfo_sub_delay_reload`, `lfo_sub_depth = lfo_sub_depth_delta`) in the later refactor — at the doc's date they were unmodeled (`channel_state.gd` carried `lfo_sub_depth` as a per-arm constant only); no validating test named in the doc
  - src: `research/effect_sound/working_documents/CHAN_8A_PAN_LFO_SIGN_FLIP_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[Noise LFO PRNG]]
- [[SPU Voice Engine]]
- [[KON KOFF Mask Dispatch]]
- [[SMD Opcodes]]
- [[Effect Sound Parity Ladder]]
