# LFO Sub-Slot 0 Pitch LFO

FFT's sub-slot 0 pitch-LFO (mode 0, the channel-level LFO on the sub+0xE0 block): when its `active_dir` active bit is set, the mode-0 swap+accumulate path (0x800175A4–0x800175E0) fires every cadence, swaps the accumulator's sign on the countdown wrap, and arms a pitch prestage by writing `chan_word_1 |= 0x200` (`CHAN1_PITCH_PRESTAGE`), which the per-tick drain commits as SPU pitch-register writes — so an armed mode-0 LFO makes the SPU pitch register oscillate around the Note-dispatched baseline instead of holding it. The 2026-05-18 ICE V21 doc (E024, `ice_no_music`) isolated this path as the root of voice 21's post-`4cdb32b0` residual: Godot's seeder had re-armed sub-slot 0 from savestate residue (`active_dir=3`, square-wave `callback_idx=1`) on an audible primary where FFT's `play_sound` init (PC `0x80013D5C`) had cleared the active bit, driving a ±843 pitch oscillation (15464/13777 vs PCSX's constant 14596) and a 5.5× `probe_pitch_register` over-fire; smd-player's `is_silent_driver` seeder gate (iter-35) restored parity.

## Points

- **FFT's LFO mode-0 (pitch) path — the sub-slot 0 swap+accumulate block at `0x800175A4`..`0x800175E0` — decrements the sub-slot countdown, swaps the accumulator's sign on wrap, and on every fire arms a pitch prestage by writing `chan_word_1 |= 0x200` (`CHAN1_PITCH_PRESTAGE`); the per-tick drain (FUN_80017118) then commits a SPU pitch-register write (walker `WALKER_FLAG_PITCH`) for each prestage it sees — so on PCSX an SPU pitch register is written only when the bytecode dispatches a Note or the active mode-0 LFO fires, which is why a cleared sub-slot 0 leaves the register holding its Note-dispatched value.** — `[S·D·R] 3/3`
  - S: `0x800175A4`..`0x800175E0` (mode-0 swap+accumulate; `chan_word_1 |= 0x200` per `ef93e550`), drain commit via FUN_80017118 — `project-assets/fft-rom/scus_disassembly.txt` (doc §8.1, §3.1)
  - D: `probe_pitch_register` on `ice_no_music` V21 — PCSX 19 sparse Note-driven writes (value 14596 held across many cadences, next write 2298 at the cad-104 Note) vs pre-fix Godot 104 writes oscillating 15464/13777 every other cadence (5.5× over-fire, 2026-05-18)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd:230-231` (mode-0 fire sets `CHAN1_PITCH_PRESTAGE`) + `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd:495-496` (drain commits `WALKER_FLAG_PITCH`) — validated by the `probe_pitch_register` pair in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/ICE_V21_PITCH_LFO_SUBSLOT_0_SEED_DEFICIT.md`
- **The square-wave LFO callback (`LAB_80017648`, sub-slot jumptable index 1, the one ice V21's savestate sub_0 carries via `callback_idx=1`) decrements the sub-slot's 16-bit countdown at sub+0x10 and swaps the step on wrap — with mode 0 that produces the ±step alternation (accumulator −16777216 / +16777216 at the swap cadence on ice V21) the pre-fix Godot seeder was erroneously running on an audible primary.** — `[S·R] 2/3`
  - S: `LAB_80017648` (16-bit countdown at `0x10(a0)`, step swap on wrap; `callback_idx=1` per ice V21's savestate) — `project-assets/fft-rom/scus_disassembly.txt` (doc §8.1)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd` wf_idx==0 branch (port of `LAB_80017648`: accumulator toggles 0 ↔ `lfo_sub_step_source[0]`) — validated by the `probe_lfo_subslot0_state` accumulator/`active_dir` pairs in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/ICE_V21_PITCH_LFO_SUBSLOT_0_SEED_DEFICIT.md`

## Notes

(empty — user territory)

## Related

- [[LFO Subslot Residue]]
- [[Noise LFO PRNG]]
- [[LFO Sub-Slot Period Reset]]
- [[SPU Voice Engine]]
- [[PSX Pitch Conversion]]
- [[Effect Sound Audio Divergence]]
