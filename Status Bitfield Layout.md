# Status Bitfield Layout

A battle unit's status flags live in four byte fields at unit struct offsets `+0x58..+0x5C`, with bits ordered MSB-first within each byte (bit 7 = first-listed status). The damage pipeline consistently reads this byte form; a repacked 32-bit word at `+0x140` exists but is used only by the icon/palette renderers.

## Points

- **Status flags occupy unit offsets `+0x58` (Crystal, Dead, Undead `0x10`, Charging `0x08`, Jump, Defending, Performing), `+0x59` (Petrify `0x80`, Invite, Darkness, Confusion, Silence, Blood Suck, Cursed, Treasure), `+0x5A` (Oil `0x80`, Float `0x40`, Reraise `0x20`, Transparent `0x10`, Berserk `0x08`, Chicken `0x04`, Frog `0x02`, Critical `0x01`), `+0x5B` (Poison, Regen, Protect, Shell, Haste, Slow, Stop, Wall `0x01`), `+0x5C` (Faith, Innocent, Charm, Sleep, Don't Move, Don't Act, Reflect `0x02`, Death Sentence); `+0x140` holds the same set repacked into one 32-bit word for icon/palette rendering, and every disassembly branch in the damage path reads the `+0x58..+0x5C` byte form, never `+0x140`.** — `[S] 1/3`
  - S: layout from `research/wiki_articles/status_condition_visual_rendering.txt` ("Primary Status Bitfield"), confirmed by the disassembly branches at `0x80187000` (`+0x5A & 0x80`), `0x801870A8` (`+0x5A & 0x40`), `0x80188FA0` (`+0x58 & 0x10`), `0x8018BAE4` (`+0x5B & 0x01`), `0x8018C9E4` (`+0x5C & 0x02`) in `battle_disassembly.txt`
  - R: none — ROM byte offsets not present in godot-learning (`StatusEncoder.gd` packs statuses into its own 0..31 indices, e.g. `STATUS_OIL = 30`, `STATUS_UNDEAD = 1`)
  - src: `research/working_documents/status_element_interplay.md`

## Notes

(empty — user territory)

## Related

- [[Status Element Defense Interplay]]
