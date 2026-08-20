# Unit Sprite Height Table

FFT keeps a per-sprite-type unit height table in BATTLE.BIN used across the battle system for head positioning, collision, targeting, and VFX anchor placement. The table is a 4-byte-per-entry array at 0x80094748 whose byte +3 is the sprite height; lookups gate on unit flags (dead/intangible/hidden return 0), and charge VFX (spell charge lines, summon charge orbs) position their convergence point at the caster's head via height + 8.

## Points

- **The per-sprite-type height table is at 0x80094748 (4 bytes/entry); the sprite height is byte +3 (DAT_8009474B) and bytes 0–2 (DAT_80094748–4A) are referenced by collision/targeting/height-adjustment functions.** — `[S] 1/3`
  - S: table at 0x80094748, height byte DAT_8009474B, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **FUN_8008DC24 returns 0 when the unit struct is NULL, when flags_0x144 & 0x9 (dead/intangible), or when flags_0x140 & 0x4 (hidden); otherwise it returns DAT_8009474B[sprite_type_0x06 × 4], and FUN_8008DC74 is the by-unit-ID wrapper.** — `[S] 1/3`
  - S: FUN_8008DC24/FUN_8008DC74 lookup code, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Most sprite types share height 0x24 (36, standard humanoid); non-standard heights: sprite_type 0 = 0, 1 = 63 (tall humanoid special character), 17 = 37, 65 = 60 (large monster), 73 = 120 (tallest, boss-type).** — `[S] 1/3`
  - S: height values table, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - ⚠ SUPERSEDED (2026-08-19) by: The shipped US table has exactly 4 distinct heights — sprite_type 0 = 0, type 65 = 60 (0x3c), type 73 = 120 (0x78), and every other type 36 (0x24); types 1 and 17 are 36, not 63 and 37
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The table is cross-referenced at 17+ sites: unit collision/height checks 0x80070340/0x800703EC/0x80070540/0x800706CC, targeting height calculation 0x8007E4A0, internal height lookups 0x8008DBB8/0x8008DC68, spell charge lines 0x801B1E4C, and summon charge orbs 0x801B36E8.** — `[S] 1/3 CONTESTED`
  - S: cross-reference call-site table, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The charge VFX handlers (4 = spell charge lines, 22 = summon charge orbs) position their convergence point at the caster's head: head_offset = height + 8, center_y = camera_y − head_offset, and DAT_801B8798 = −head_offset (config_12 pos_scatter_y sparkle anchor).** — `[S] 1/3`
  - S: charge VFX head-targeting code, DAT_801B8798, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The shipped US table holds 4 distinct heights over sprite ids 0..154 — 0 for id 0, 60 (0x3c) for id 65, 120 (0x78) for id 73, and 36 (0x24) for the other 152 — and each 4-byte row is (shape id, sequence id, flying, height), so bytes 0–2 are not opaque collision data.** — `[S] 1/3`
  - S: `BATTLE.BIN+0x2d748` dumped whole off the US retail disc image (web-psx `src/cdrom/iso.ts`, `src/sprite/catalog.ts` `SHAPE_TABLE`) — type 1 = `00 00 00 24`, type 17 = `00 00 00 24`, type 65 = `06 06 00 3c`, type 73 = `07 07 00 78`; the parallel file table at `+0x2dcdc` (8 bytes: LBA + byte count of the `.SPR`) has 154 entries and sprite id is that index plus one, so ids above 154 are past the table and 63/37 look like readings from beyond it or hex/decimal slips (2026-08-19)
  - src: external contribution — web-psx `src/sprite/catalog.ts`, `docs/combat-tables.md` (see [[Web-psx Cross-Validation]])

## Notes

(empty — user territory)

## Related

- [[ENTD Unit Deployment Table]]
- [[Effect Execution Model]]
- [[Web-psx Cross-Validation]]
