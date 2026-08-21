# Effect Sound Pipeline Layout

Where the effect-sound pipeline lives in ROM: the BATTLE.BIN region owns the battle-side trigger chain (timeline sound tracks, mode-based sound-id selection, sound-system globals), the always-resident SCUS code owns the SPU voice engine (SMD interpreter, FEDS opcode handlers, KON/KOFF dispatch), and the PSX SPU MMIO page is the hardware sink. The 2026-05-18 SOUND_PIPELINE call-graph doc (end-to-end map from timeline keyframe to SPU KON register, every node address-cited) records the four regions and the key data globals in each.

## Points

- **The effect-sound pipeline spans four regions: BATTLE.BIN `0x801A0000`–`0x801C0000` (effect-system dispatcher, timeline tracks, `lookup_sound_effect`; sound-system globals at `0x801B9250` / `0x801BACC8` / `0x801BBF74` / `0x801BC0DC`), always-resident SCUS code `0x80012000`–`0x80017FFF` (SPU voice management, SMD interpreter, FEDS opcode handlers), SCUS data at `0x80028B0C` (FEDS opcode jump table) and `0x80032A00` (`g_sound_resource_list` head), and PSX SPU MMIO `0x1F801C00`–`0x1F801FFF` (voice registers, KON, KOFF).** — `[S] 1/3`
  - S: layout table of the call-graph doc (static analysis; all addresses corroborated elsewhere in this vault, e.g. `scus_disassembly.txt` for the SCUS-side symbols) — per `research/effect_sound/working_documents/SOUND_PIPELINE.md` §Layout
  - R: none — ROM binary region layout, not present in godot-learning or smd-player (probed both)
  - src: `research/effect_sound/working_documents/SOUND_PIPELINE.md`

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[Effect Sound Timing]]
- [[KON KOFF Mask Dispatch]]
- [[SMD Interpreter Per-Channel Tick]]
- [[SFX Index]]
