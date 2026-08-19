# SMD Opcodes

Ground truth for the FFT SMD music-opcode dispatcher and its gaps against the smd-player reimplementation, established by a 2026-05-27 sweep cross-referencing the ROM music jumptable at `0x80028b0c` and arg-size table at `0x80028d0c` against smd-player's dispatch table and a 100-file vanilla `MUSIC_NN.SMD` batch survey (0 unaccounted bytes, `--verify-opcodes` clean). Four opcodes appearing in the vanilla corpus fell through smd-player's `_StubNoop`: `0x87` (true FFT noop, cosmetic) and `0x9C`/`0xC3`/`0xC8` (real FFT handlers — 145 occurrences across 6 files, dominated by 132× `C8 05` in MUSIC_58); 25 more real FFT handlers never fire in vanilla (latent), and 35 opcode bytes are confirmed FFT noops. `0x9B` was a phantom binding (smd-player runs `SaveLoopTarget`; FFT's music dispatcher noops it) and `0xFE` carried a parser arity mismatch (ROM: 2 params, parser: 1). Root cause of the missed noops: the 2026-05-21 coverage audit scanned effect bytecode only. As of 2026-08-19: `0xC3`/`0xC8` are wired in smd-player (2026-05-27) and fft-sound-driver (2026-05-28); `0x9C` and the `0x9B` phantom remain open. The MUSIC_41 capture work (2026-04) additionally pinned down timing semantics: key-off on a note's own delta expiry, pre-note rests separate from post-note rest accumulation, repeat count = param minus 1, Coda rewinds to the captured start index, and all tracks open with a preamble opcode block.

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
  - src: `research/working_documents/SMD_PARSER_GAPS.md`

- **The FFT arg-size table entry for `0xFE` (BankSelect) is `0x03` → 2 params, but smd-player declares arity 1 — a parser-arity mismatch that persists in the current tree.** — `[S·R] 2/3`
  - S: `ram:80028d8a = 0x03`, `scus_disassembly.txt`
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
## Notes

(empty — user territory)

## Related

- [[SMD Header Layout]]
- [[SMD Header Field 0C]]
- [[SPU Voice Engine]]
- [[WAVESET Instrument Bank]]
- [[SFX Index]]
