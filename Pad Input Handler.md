# Pad Input Handler

The FFT runs on the PSX kernel pad subsystem with an inverted-button convention: the kernel pad buffer at `0x80032A78` is refreshed each VSync by the BIOS SIO ISR (with pcsx-redux's `setOverride` mask applied), high-low byte-swapped and bit-inverted, and every button poll goes through the `SCUS_get_inverted_button_input` wrapper at `0x8001DB58` — the input consumer of the event-dialogue advance tick.

## Points

- **The kernel pad button-invert mask lives at `0x80032A78` (`button_invert_mask`, populated by `PadInit(mode=0x20000001, buf=0x80032A78)` at `0x8001DB08`) and is updated each VSync by the BIOS pad ISR with the SIO override applied; the mask is high-low byte-swapped vs the SIO wire layout (high-byte bit 6 = CROSS) and inverted (a non-pressed button reads 1, a pressed one 0), so `setOverride(14)` flips byte 0 of the buffer `0xFF` → `0xBF`; the per-button-poll wrapper `SCUS_get_inverted_button_input` @ `0x8001DB58` returns the masked value on the next VSync, and it fired 141×/30 s during the chapel cinematic and 0× in the post-load idle.** — `[S·D] 2/3`
  - S: `SCUS_get_inverted_button_input` `0x8001DB58` (SUB_8001db58, battle_disassembly.txt), `PadInit` caller `0x8001DB08` + buffer `0x80032A78` (PLATE comment, `fft-ghidra/content/comments_battle_bin.jsonl`)
  - D: live probes — `setOverride(14)` byte-flip `0xFF`→`0xBF` verified, 141 fires/30 s during cinematic vs 0 during pure idle (2026-06-20)
  - R: none — pad buffer / inverted-button polling not present in godot-learning (probed `src/`, `tests/` for `0x80032A78`/`PadInit`/`inverted`)
  - src: `research/working_documents/SCENARIO_LOADING.md`

## Notes

(empty — user territory)

## Related

- [[Display Message Opcode]]
