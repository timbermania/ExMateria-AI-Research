# SMD Opcodes

Ground truth for the FFT SMD music-opcode dispatcher and its gaps against the smd-player reimplementation, established by a 2026-05-27 sweep cross-referencing the ROM music jumptable at `0x80028b0c` and arg-size table at `0x80028d0c` against smd-player's dispatch table and a 100-file vanilla `MUSIC_NN.SMD` batch survey (0 unaccounted bytes, `--verify-opcodes` clean). Four opcodes appearing in the vanilla corpus fell through smd-player's `_StubNoop`: `0x87` (true FFT noop, cosmetic) and `0x9C`/`0xC3`/`0xC8` (real FFT handlers — 145 occurrences across 6 files, dominated by 132× `C8 05` in MUSIC_58); 24 more real FFT handlers never fire in vanilla (latent), and 37 opcode bytes are confirmed FFT noops (corrected 2026-08-21: the sink `LAB_8001586c` takes exactly 37 of the 128 jumptable slots, and that set is identical to the arg-size table's 37 zero entries; the earlier 35 omitted `0x9B`, which this note documents as a sink entry elsewhere, and `0xB9`, which was mis-filed as latent). `0x9B` was a phantom binding (smd-player runs `SaveLoopTarget`; FFT's music dispatcher noops it) and `0xFE` was thought to carry a parser arity mismatch (ROM: 2 params, parser: 1), but the ROM entry reads `0x02` = 1 param and smd-player's arity 1 is correct — the mismatch was a misread of this note's own arg-size table (corrected 2026-08-21). Root cause of the missed noops: the 2026-05-21 coverage audit scanned effect bytecode only. As of 2026-08-19: `0xC3`/`0xC8` are wired in smd-player (2026-05-27) and fft-sound-driver (2026-05-28); `0x9C` and the `0x9B` phantom remain open. The MUSIC_41 capture work (2026-04) additionally pinned down timing semantics: key-off on a note's own delta expiry, pre-note rests separate from post-note rest accumulation, repeat count = param minus 1, Coda rewinds to the captured start index, and all tracks open with a preamble opcode block. A 2026-05-15 `protect_no_music` effect-session analysis recovered four more real handlers hit by effect bytecode — `0xC0` (instrument reload), `0xD2` (relative pitch-bend ×8), `0xD5` (chan+0x6 bit-0x2 toggle), `0xDC` (portamento stop) — all wired in smd-player's shared opcode table and mirrored in fft-sound-driver.

## Points

- **The FFT music opcode jumptable at `0x80028b0c` holds 128 4-byte entries indexed by `opcode - 0x80` (dispatch step XREF'd from `lw v0,-0x74f4(at)` at `ram:800154e4`), and the per-opcode arg-size table at `0x80028d0c` (XREF `ram:8001560c`) stores `1 + arity` per entry — `0x01` = no params, `0x02` = 1 param byte, `0x04` = 3 param bytes; a jumptable entry pointing at `LAB_8001586c` marks a runtime noop.** — `[S·R] 2/3`
  - S: jumptable @ `ram:80028b0c`, arg-size table @ `ram:80028d0c`, dispatch `lw` @ `ram:800154e4` — `scus_disassembly.txt`
  - R: `smd-player/addons/exmateria_sound/runtime/smd_opcodes.gd:80-85` (`_EXTRA_OPCODES` arities explicitly "derived from FFT size table at ROM 0x80028d0c (table value - 1)"); validated by the smd-player music parity Gate A (per-opcode dispatch counts, `smd-player/workspace/regression/verify_all.sh`)
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **Opcode `0x87` is a runtime noop in FFT — the dispatcher entry at `ram:80028b28` points at the shared noop sink `LAB_8001586c` (`jr ra; move v0,a0`, the "unimplemented opcode" sink XREF'd from 21 jumptable entries) and the size table entry `ram:80028d13 = 0x00` gives it arity 0; smd-player correctly noops it (not in the dispatch table).** — `[S·D·R] 3/3`
  - S: `ram:80028b28` = `6c580180` → `LAB_8001586c` body @ `ram:8001586c`; size table @ `ram:80028d13`; `scus_disassembly.txt`
  - D: vanilla corpus batch survey (2026-05-27): exactly 1 occurrence, `MUSIC_11.SMD @ 0x1CED`
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer/opcodes/_table.gd` (only `0x80`/`0x81` in the 0x8x range, current tree) → fall-through to `opcodes/_stub_noop.gd` (`SequencerOpStubNoop.apply = pass`); no named test
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **Opcode `0x9C` (arity 3) is a real FFT handler — entry `ram:80028b7c` → `FUN_80015bb8` packs params as `(u16 = param[1]<<8 | param[0], u8 = param[2], fixed flag 0x40)` and calls `FUN_80012664`, which when bit `0x1000` of `DAT_80032a54` is set fires the per-channel init `FUN_80013b20` (the iter-51 `chan+0x60` env-multiplier writer); smd-player noops it, so the 3 vanilla MUSIC_09 hits silently skip the re-init.** — `[S·D] 2/3`
  - S: handler body `ram:80015bb8`–`ram:80015bf8`; `FUN_80012664` chain (`andi v0,v0,0x1000` @ `ram:80012688`, `jal FUN_80013b20` @ `ram:800126c4`); `scus_disassembly.txt`
  - D: vanilla corpus batch survey (2026-05-27): 3 hits in `MUSIC_09.SMD` @ 0x0E17/0x0E28/0x0E38, each immediately before a NOTE event
  - R: none — 0x9C not present in godot-learning, smd-player, or fft-sound-driver (smd-player holds it only as `_EXTRA_OPCODES` arity-3 metadata, `smd_opcodes.gd:87`; dispatch falls through to `_StubNoop`)
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **Opcode `0xC3` (arity 1) is a real FFT handler — entry `ram:80028c18` → `LAB_800161c4` sets `chan[+0x4] |= 0x20` (`WALKER_FLAG_ADSR1_MID`; next SPU-IRQ walker pass fires `FUN_8001B79C`, writing ADSR1 mid-nibble bits 4-7) and stores `param[0]` at `chan[+0x66]` (ADSR-staging halfword, same field family as `0xC7`'s sustain byte); the 10 vanilla hits were nooped by smd-player at survey time (parity hole).** — `[S·D·R] 3/3`
  - S: handler body `ram:800161c4`–`ram:800161dc` (`ori v0,v0,0x20`, `sh v0,0x4(a2)`, delay-slot `sh v1,0x66(a2)`); walker flag 0x020 → `FUN_8001B79C` per `smd-player/.../shared/slot_state.gd`; `scus_disassembly.txt`
  - D: vanilla corpus batch survey (2026-05-27): 10 hits — MUSIC_11 ×1 @ 0x1CE8, MUSIC_17 ×4, MUSIC_19 ×3, MUSIC_97 ×2
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer/opcodes/_table.gd:165` (`0xC3 → _AdsrDecayRate`, wired 2026-05-27) + `opcodes/adsr_decay_rate.gd`; also `fft-sound-driver/src/driver/opcodes_music_adsr.cpp` (`smd_op_c3_adsr_decay_rate` @ `LAB_800161C4`, ported 2026-05-28); validated by the smd-player music parity Gate A (per-opcode dispatch counts, `smd-player/workspace/regression/verify_all.sh`)
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **Opcode `0xC8` (arity 1) is a real FFT handler — entry `ram:80028c2c` → `LAB_80016260` sets `chan[+0x4] |= 0x10` (`WALKER_FLAG_ADSR1_HIGH`; walker fires `FUN_8001B938`) and word-stores `param[0]` at `chan[+0x58]` (ADSR2-mode HIGH byte, overriding the WAVESET-installed mode); MUSIC_58 fires it 132×, always `C8 05` inside the fixed `AC → C8 → C2 → C5` instrument-setup macro, and smd-player nooped it at survey time (severe parity hole).** — `[S·D·R] 3/3`
  - S: handler body `ram:80016260`–`ram:80016278` (`ori v0,v0,0x10`; delay-slot `sw v1,0x58(a2)` — word store, not halfword); `chan+0x58` = ADSR2 mode high per `smd-player/.../shared/channel_state.gd`; `scus_disassembly.txt`
  - D: vanilla corpus batch survey (2026-05-27): 132 hits, all in MUSIC_58, all param `0x05`, each preceded by `AC` (Instrument) and followed by `C2 35 C5 0E`
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer/opcodes/_table.gd:170` (`0xC8 → _AdsrAttackMode`, wired 2026-05-27) + `opcodes/adsr_attack_mode.gd`; also `fft-sound-driver/src/driver/opcodes_music_adsr.cpp` (`smd_op_c8_adsr_attack_mode` @ `LAB_80016260`, ported 2026-05-28); validated by the smd-player music parity Gate A
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **25 further opcodes have real FFT handlers (jumptable `0x80028b0c`) but zero vanilla MUSIC occurrences — latent parity holes: `0x8A, 0x8D, 0x8E, 0x8F, 0x9D, 0x9E, 0xA1, 0xA2, 0xA4, 0xA5, 0xA6, 0xA7, 0xAA, 0xAE, 0xAF, 0xB8, 0xB9, 0xC1, 0xE9, 0xEA, 0xEE, 0xF5, 0xFD, 0xFE, 0xFF`.** — `[S·D] 2/3`
  - S: jumptable entries + size-table arities per opcode, `scus_disassembly.txt` (e.g. `0x9D` → `FUN_80015bfc` and `0x9E` → `LAB_80015c38`, siblings of `0x9C`; `0xAE`/`0xAF` = PercussionOn/Off)
  - D: vanilla corpus batch survey (2026-05-27): 0 occurrences of any of the 25 across all 100 files
  - R: none — none of the 25 bytes is a key in smd-player's `_table.gd` (all fall through to `_StubNoop`); not present in godot-learning or fft-sound-driver
  - ⚠ SUPERSEDED (2026-08-21) by: the list is **24**, not 25 — `0xB9` is not a latent handler, it is a runtime noop. Its jumptable slot at `0x80028B0C + 0x39×4` targets the sink `LAB_8001586c` and its arg-size entry is `0x00`, so it belongs in the noop census above (which is likewise 37, not 35). Verified by decoding both full 128-entry tables out of `project-assets/fft-extract/SCUS_942.21`; the remaining 24 are unaffected (ExMateria-AI-Research#3)
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **The FFT arg-size table entry for `0xFE` (BankSelect) is `0x03` → 2 params, but smd-player declares arity 1 — a parser-arity mismatch that persists in the current tree.** — `[S·R] 2/3`
  - S: `ram:80028d8a = 0x03`, `scus_disassembly.txt`
  - ⚠ SUPERSEDED (2026-08-21) by: the ROM entry is `0x02`, not `0x03` — `0xFE` takes **1** parameter and smd-player's arity 1 is correct, so there is no mismatch to close. Re-read off `project-assets/fft-extract/SCUS_942.21` at file offset `0x1950C` (= RAM `0x80028D0C`, via `t_addr=0x80010000` + the 0x800 PS-X EXE header): the arg-size table's entry `0x7E` (`0xFE − 0x80`) reads `0x02`. The cited address is right and is exactly `0x80028D0C + 0x7E` — this note read one byte of its own arg-size table as a loose fact and got the value wrong. Acting on the `0x03` reading misaligns every stream containing `0xFE`, so the direction of the error is the expensive one (ExMateria-AI-Research#3; web-psx `docs/sequence-format.md`)
  - R: `smd-player/addons/exmateria_sound/runtime/smd_opcodes.gd:74` (`0xFE: ["BankSelect", 1]` — still mismatched as of 2026-08-19)
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **`0x9B` is phantom on the music side: smd-player binds it to `SaveLoopTarget` (writes `slot+0x1c`/`slot+0x2b`), but the FFT music jumptable entry at `ram:80028b78` routes it to the noop sink `LAB_8001586c` — the "one table for music + SFX" premise is overgeneralized; 0 vanilla MUSIC occurrences, so latent.** — `[S·D·R] 3/3`
  - S: `ram:80028b78` = `6c580180` → `LAB_8001586c` (music jumptable @ `0x80028b0c`), `scus_disassembly.txt`
  - D: vanilla corpus batch survey (2026-05-27): 0 SaveLoopTarget occurrences in the 100-file MUSIC corpus
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer/opcodes/_table.gd:195` (`0x9B → _dispatch_shared.bind(_SharedSaveLoopTarget, 0x9B)`, still bound as of 2026-08-19; `shared/opcodes/save_loop_target.gd` mirrors SFX-side `LAB_800159DC`) — contrast `fft-sound-driver/src/driver/opcodes_bytecode_flow.cpp:205`, which registers `0x9B` as `op_nop_sled`
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **35 opcode bytes are confirmed runtime noops in FFT — their jumptable slots hold `6c580180` (`LAB_8001586c`): `0x82–0x89, 0x8B, 0x8C, 0x92, 0x93, 0x9F, 0xA3, 0xA8, 0xAB, 0xBC–0xBF, 0xCB–0xCF, 0xDD–0xDF, 0xF3, 0xF4, 0xF8–0xFC`; smd-player noops all of them too (fall-through), so no parity gap.** — `[S·R] 2/3`
  - S: jumptable slots @ `0x80028b0c + (op - 0x80)*4` = `6c580180`, `scus_disassembly.txt`
  - R: none of the 35 bytes is a key in `smd-player/addons/exmateria_sound/runtime/sequencer/opcodes/_table.gd` (current tree) → `opcodes/_stub_noop.gd` fall-through; no named test
  - ⚠ SUPERSEDED (2026-08-21) by: the census is **37**, not 35 — the jumptable's noop sink `LAB_8001586c` is the target of exactly 37 of the 128 slots, and that set is *identical* to the arg-size table's 37 zero entries. The two missing bytes are `0x9B` and `0xB9`. `0x9B` this note **already documents as a sink entry** two points above (`ram:80028b78` = `6c580180`), so the 35-byte list contradicted its own file; `0xB9` was mis-filed in the 25-opcode latent-handler list below. Verified by decoding all 128 jumptable words at `0x80028B0C` and all 128 arg-size bytes at `0x80028D0C` out of `project-assets/fft-extract/SCUS_942.21`: 91 non-sink entries / 91 non-zero widths, sink set == zero set exactly. Corrected set: `0x82–0x89, 0x8B, 0x8C, 0x92, 0x93, 0x9B, 0x9F, 0xA3, 0xA8, 0xAB, 0xB9, 0xBC–0xBF, 0xCB–0xCF, 0xDD–0xDF, 0xF3, 0xF4, 0xF8–0xFC` (ExMateria-AI-Research#3)
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **Cross-auditing smd-player's shared bindings against the FFT music jumptable confirmed four as legitimate (real FFT handlers, no phantom behavior): `0xAD` → `LAB_80015e68` (Byte76), `0xD9` → `FUN_800164d4` (SharedLfo), `0xDC` → `LAB_800163d4` (PortaStop), `0xEC` → LfoArmSubslot2 handler.** — `[S·R] 2/3`
  - S: jumptable entries @ `0x80028b0c`, `scus_disassembly.txt`
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer/opcodes/_table.gd:197,210,211,217` (`_SharedByte76`, `_SharedLfo`, `_SharedPortaStop`, `_SharedLfoArmSubslot2`; current line numbers)
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **The research master parser desyncs from FFT runtime on `0x9C`: each of the 3 MUSIC_09 hits lands inside a per-track loop where the parser exits the track region after the opcode byte, so it reports `!! truncated: opcode declares arity 3, only 0 bytes read` and treats the 3 param bytes as the start of the next event, whereas FFT consumes them as params (handler returns `a0 + 3` at `ram:80015be4`).** — `[S·D] 2/3`
  - S: `ram:80015be4` (`addiu v0,s0,0x3` — 3-byte advance), `scus_disassembly.txt`
  - D: vanilla corpus batch survey (2026-05-27): 3 MUSIC_09 hits, all flagged `!! truncated` by `research/key_documents/smd_master_parser.py`
  - R: none — smd-player consumes 0x9C's 3 params via `_EXTRA_OPCODES` arity (`smd_opcodes.gd:87`), so its stream stays aligned; the desync is specific to the research parser (outside the reimplementation repos)
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **The `0x9C`/`0xC3`/`0xC8` dispatcher noops went undetected because smd-player's `_EXTRA_OPCODES` coverage audit (2026-05-21) scanned only the 14 `effects/*_no_music.bin` files — effect bytecode never hits those opcodes, while the music corpus carries 145 occurrences across 6 files.** — `[D·R] 2/3`
  - D: 2026-05-21 effect-bytecode audit (all 14 `E###.BIN` no-music files, 0 hits) vs 2026-05-27 vanilla MUSIC batch survey (145 hits: 0x9C ×3, 0xC3 ×10, 0xC8 ×132)
  - R: `smd-player/addons/exmateria_sound/runtime/smd_opcodes.gd:82` — comment still states the audit scope ("audited 2026-05-21 across all 14 effects/*_no_music.bin", current tree)
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **The PSX SMD interpreter fires key-off (ADSR release) when a note's own `delta_time` expires — not when the next note's key-on arrives; Fermata (0x81) extends the sustain and Rest (0x80) arms key-off (verified: Note A#4 with dur=67 enters Release at dur=1).** — `[D·R] 2/3`
  - D: Lua duration traces on MUSIC_41.SMD (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer/helpers/note_life_ticks.gd` (`idle_timeout` from the accumulated note sustain; default gate 0x0F = duration−1) + `smd-player/addons/exmateria_sound/runtime/sequencer/opcodes/rest.gd` (KOFF arm, FFT `smd_rest` @ `0x80015874`) + music parity Gate D
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **SMD duration semantics: pre-note rests are processed as separate waits before the note fires, and post-note Rests + the note's own delta + subsequent Rests accumulate into one wait per note — lumping preamble rests with post-note rests made note-to-note intervals too long (the fix took voice 2 correlation 0.39 → 1.00, voice 3 0.22 → 1.00, voice 8 0.20 → 0.80).** — `[D·R] 2/3`
  - D: MUSIC_41.SMD per-voice correlation before/after fix (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer/per_tick/advance_track.gd` (pre-note `accumulated > 0 and not note_fired → return` wait; post-note scan accumulates Fermata/Rest into `note_duration`)
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **`smd_repeat` (0x98) stores `param − 1` as the repeat count, so `Repeat(2)` plays the body twice total (initial + 1 repeat) — reference voice 5 shows 6 evenly-spaced notes at 96-tick intervals consistent with that count.** — `[D·R] 2/3`
  - D: MUSIC_41.SMD reference capture, voice 5 note spacing (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer/opcodes/repeat.gd` (`count = params[0] - 1`, FFT `smd_repeat` @ `LAB_80015960`) + smd-player music parity Gate A
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **Coda loop-back target: when Repeat fires, the event index already points past the repeat byte, so Coda (0x99) must rewind to the captured start index — not start+1 — or the loop body's first event is skipped (the fix took voice 5 intervals from 97,97,97,49,97,97 to uniform 97 ticks, spectral 0.999 → 1.000).** — `[D·R] 2/3`
  - D: MUSIC_41.SMD reference comparison, voice 5 intervals (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer/opcodes/coda.gd` (rewinds to `ts.loop_stack[-1][0]` captured by `repeat.gd` after the event index advanced) + `smd-player/addons/exmateria_sound/runtime/sequencer/per_tick/advance_track.gd` (Coda-follow lookahead)
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **All tracks of MUSIC_41.SMD open with a preamble opcode block (ReverbOn, Loop, Dynamics, Pan, Instrument, Octave) before their first note; tracks 1 and 6 carry valid instruments — the earlier "tracks 1/6 have no instrument" reading was an artifact of decoding from the wrong file offset.** — `[D·R] 2/3`
  - D: full MUSIC_41.SMD track decode from the correct SMD data offset (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer/per_tick/advance_track.gd` (pre-note Rest/opcode dispatch spreads conductor preambles across cadences) + smd-player music parity Gate A
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **Opcode `0xC0` (arity 0) is a real FFT handler — entry `0x80028c0c` → `smd_op_c0_instrument_reload` @ `FUN_8001613C` re-issues the 0xAC instrument loader with no payload byte: `lbu a0,0x2c(a1)` then `jal` the instrument loader (`Hyp_instrument_data_loader` / `FUN_80016fb4`), re-arming the walker bits `0x1FF` (ADSR1/ADSR2 registers, sample start/repeat addrs) for the channel's stored instrument index; `protect_no_music` ch0 dispatches it once (event #28, right after 0xDC), and the pre-fix Godot skip left a −1 row deficit across the whole ADSR/sample-register probe family (`probe_adsr1_high/low/mid_register` 7 vs 6, `probe_adsr2_low_register` 45 vs 44, `probe_sample_start/repeat_addr_register` 7 vs 6, `probe_vol_register_sweep` 7 vs 6).** — `[S·D·R] 3/3`
  - S: jumptable entry @ `0x80028c0c` (= `0x80028b0c` + (0xC0−0x80)×4) → `0x8001613c`; handler body `0x8001613c`–`0x8001616c` (`lbu a0,0x2c(a1)`, `jal FUN_80016fb4` at `0x80016154`, 0-byte payload return) — `scus_disassembly.txt`
  - D: `protect_no_music` dispatch trace + probe counts (2026-05-15, HEAD `6810465c`) — 1 dispatch on ch0 (`... E0 DC C0 80 AC 94 ...`); the −1-across-the-family row deficit listed in the claim
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/instrument_reload.gd` (`SharedOpInstrumentReload.apply` re-runs the waveset loader from `channel.instrument_idx` and re-arms `walker_flag_word |= 0x1FF` / `chan_word_1 |= 0x300`), bound at `runtime/shared/opcodes/_table.gd:146`; mirrored in `fft-sound-driver/src/driver/opcodes_instrument.cpp` — validated by the `protect_no_music` parity re-run (probe row counts + audio cos_dist) via `smd-player/workspace/orchestrator/run_effect_iteration.py`
  - src: `research/effect_sound/working_documents/PROTECT_NO_MUSIC_UNHANDLED_OPCODES.md`

- **Opcode `0xD2` (arity 1) is a real FFT handler — entry `0x80028c54` → `smd_op_d2_pitch_bend_rel` @ `LAB_80016304`: the 1-byte payload is treated as signed s8, scaled ×8 (`sll 24; sra 21` ≡ s8×8), and added to `chan+0x86` (pitch_bend), then `CHAN1_PITCH_PRESTAGE` (chan_word_1 bit 0x200) is set — the 0xD1 mechanism with the byte scaled by 8; `protect_no_music` dispatches it 7× (ch0 events #34/#38 after the dt=24/dt=3/dt=3 Note cluster, ch2 mirror, plus `events_from(N)` flow-on into adjacent pre-EndBar channels), and each missed dispatch costs one `probe_pitch_inputs` row plus a stale pitch_bend baseline on every subsequent Note (pre-fix voice-20 `probe_pitch_inputs` 66 vs 26, first miss at cadence 261, gap pattern [3,2,3,2,…]).** — `[S·D·R] 3/3`
  - S: jumptable entry @ `0x80028c54` → `0x80016304`; handler body `0x80016304`–`0x8001632c` (`sll v1,v1,0x18; sra v1,v1,0x15` scale, `ori a1,a1,0x200` prestage arm, `sh v0,0x86(a2)` commit) — `scus_disassembly.txt`
  - D: `protect_no_music` dispatch trace + pitch deficit (2026-05-15, HEAD `6810465c`) — 7 dispatches; voice-20 `probe_pitch_inputs` 66 vs 26 (−40, 60% missing), voice-21 197 vs 121 (−76)
  - D (additive): `cure_4_no_music` dispatch trace at HEAD `ba9acddc` (2026-05-15) — 0xD2 dispatches once on cure_4 (cadence 497, slot 2 / voice-18 area, byte=8), contradicting the prior doc's "none of D2/D5/DC/C0/EC/EF fire on cure_4" projection; post-fix vs pre-fix cure_4 cos_dist deltas all within run-to-run jitter (max +0.011 on voice_19)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/pitch_bend_rel.gd` (`SharedOpPitchBendRel.apply`: `word_86 += s8*8` with 16-bit wrap, arms `CHAN1_PITCH_PRESTAGE` + `WALKER_FLAG_PITCH`), bound at `runtime/shared/opcodes/_table.gd:148`; mirrored in `fft-sound-driver/src/driver/opcodes_pitch.cpp` (`op_pitch_bend_rel` @ `LAB_80016304`) — validated by the `protect_no_music` parity re-run (probe row counts + audio cos_dist) via `smd-player/workspace/orchestrator/run_effect_iteration.py`
  - src: `research/effect_sound/working_documents/PROTECT_NO_MUSIC_UNHANDLED_OPCODES.md`

- **Opcode `0xD5` (arity 0) is a real FFT handler — entry `0x80028c60` → `smd_op_d5_chan6_bit2_toggle` @ `LAB_800163BC`: toggles bit 0x2 of `chan+0x6` with a 0-byte payload (`xori v0,v0,0x2`); it fires immediately after each 0xD4 portamento init in `protect_no_music` (ch0 event #19, ch1 event #13), and the per-tick reader at PC `0x80015208` (`andi v0,a3,0x2; bne` → `LAB_80015234`) skips the portamento target-counter decrement + `porta_active` clear while the bit is set, so portamento runs until 0xDC clears bit 0x1 or a fresh 0xD4 reinitializes.** — `[S·D·R] 3/3`
  - S: jumptable entry @ `0x80028c60` → `0x800163bc`; toggle body `0x800163bc`–`0x800163d0`; reader `andi v0,a3,0x2` @ PC `0x80015208` + `bne v0,zero,LAB_80015234` @ `0x8001520c` (per_channel_tick) — `scus_disassembly.txt`
  - D: `protect_no_music` dispatch trace + pitch deficit (2026-05-15, HEAD `6810465c`) — 2 dispatches, each immediately after 0xD4; voice-21 `probe_pitch_inputs` 197 vs 121 (−76, 39% missing) concentrated in the dt=96 Note's porta-driven cluster
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/chan6_bit2_toggle.gd` (`SharedOpChan6Bit2Toggle.apply` flips `channel.chan_6_bit_2`; the cadence_body porta-counter decrement is gated on it), bound at `runtime/shared/opcodes/_table.gd:149`; mirrored in `fft-sound-driver/src/driver/opcodes_pitch.cpp` (`op_chan6_bit2_toggle` @ `LAB_800163BC`) — validated by the `protect_no_music` parity re-run via `smd-player/workspace/orchestrator/run_effect_iteration.py`
  - src: `research/effect_sound/working_documents/PROTECT_NO_MUSIC_UNHANDLED_OPCODES.md`

- **Opcode `0xDC` (arity 0) is a real FFT handler — entry `0x80028c7c` → `smd_op_dc_porta_stop` @ `LAB_800163D4`: clears bit 0x1 of `chan+0x6` (`portamento_active`) with a 0-byte payload (`andi v0,v0,0xfffe`) — the literal mirror of 0xDB's chan+0xFE LFO sub-slot-0 clear; `protect_no_music` ch0 dispatches it at event #27 (after the `E0 E2 N(24) E0 E2 N(24)` cluster, before `C0 80`), terminating ch0's portamento where pre-fix Godot kept it active until `pre_pitch_delta` decayed to 0 — the largest single contributor to voice-20's ~40 missed `probe_pitch_inputs` rows, all post-cadence 261.** — `[S·D·R] 3/3`
  - S: jumptable entry @ `0x80028c7c` → `0x800163d4`; clear body `0x800163d4`–`0x800163e8`; the portamento tick gate reading bit 0x1 sits at PC `0x80015200` (`andi v0,a3,0x1`) — `scus_disassembly.txt`
  - D: `protect_no_music` dispatch trace + pitch deficit (2026-05-15, HEAD `6810465c`) — 1 dispatch on ch0; voice-20 `probe_pitch_inputs` 66 vs 26 with the 40 misses concentrated in the dt=24 + dt=24 + DC region (gap pattern [3,2,3,2,…] from cadence 261)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/porta_stop.gd` (`SharedOpPortaStop.apply` sets `channel.portamento_active = false`), bound at `runtime/shared/opcodes/_table.gd:151`; mirrored in `fft-sound-driver/src/driver/opcodes_pitch.cpp` (`op_porta_stop` @ `LAB_800163D4`) — validated by the `protect_no_music` parity re-run via `smd-player/workspace/orchestrator/run_effect_iteration.py`
  - src: `research/effect_sound/working_documents/PROTECT_NO_MUSIC_UNHANDLED_OPCODES.md`

## Notes

(empty — user territory)

## Related

- [[SMD Header Layout]]
- [[SMD Header Field 0C]]
- [[SPU Voice Engine]]
- [[WAVESET Instrument Bank]]
- [[Portamento Tick Ordering]]
- [[SFX Index]]
