# Element Byte Encoding

FFT encodes an ability's or weapon's element as a single-byte bitmask with bit 7 = Fire down to bit 0 = Dark, so the element order is Fire, Lightning, Ice, Wind, Earth, Water, Holy, Dark (high bit → low bit) — not the Fire/Ice/Lightning order some wiki text implies. Every bit was confirmed against raw bytes in the spell data table, and the Hacktics struct dump's named `Element` enum labels are not the raw byte values.

## Points

- **The ability/weapon element field is a 1-byte bitmask: bit 7 Fire (`0x80`), bit 6 Lightning (`0x40`), bit 5 Ice (`0x20`), bit 4 Wind (`0x10`), bit 3 Earth (`0x08`), bit 2 Water (`0x04`), bit 1 Holy (`0x02`), bit 0 Dark (`0x01`), confirmed against raw bytes in the spell data table at `SCUS::SecondaryAbilityData` (`0x8005FBF0..`), where each spell entry's `Element` byte at entry offset `+0x07` reads Fire 1 [16] `0x80` at `0x8005FCD7`, Bolt 1 [20] `0x40` at `0x8005FD0F`, Ice 1 [24] `0x20` at `0x8005FD47`, Holy 1 [15] `0x02` at `0x8005FCC9`, Dark `0x01` at `0x8005FFF5`, Earth `0x08` at `0x8005FF77`, Water `0x04` at `0x8005FFBD`, Wind `0x10` at `0x80060321`.** — `[S] 1/3`
  - S: raw `Element` bytes at the listed addresses, read from `scus_disassembly.txt` (`SecondaryAbilityData` table base `0x8005FBF0`)
  - R: none — raw 8-bit element-byte layout not present in godot-learning (renumbered 1..8 in `GPUAbilityLoader.ELEMENT_MAP` / `ElementEncoder.ELEMENT_MAP`)
  - src: `research/working_documents/status_element_interplay.md`

## Notes

(empty — user territory)

## Related

- [[Status Element Defense Interplay]]
