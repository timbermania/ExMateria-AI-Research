# Monster Ability Element Defense

Monster ability formula handlers bypass the element-defense pipeline entirely in BATTLE.BIN: they never reach the status overlay or the equipment/job element matrix, so monster damage abilities are element-defense-blind (consistent with most monster ability data carrying `Element = NoElement`). The single exception is `4D_MonAbsorb`, which still runs the Undead heal⇄damage sign-flip.

## Points

- **The monster formula handlers `4C_MonHeal` (`0x8018A398`), `4D_MonAbsorb` (`0x8018A3CC`), `4F_GoblinPunch` (`0x8018A454`), `50_MonPhysSpell` (`0x8018A49C`), and `51_MonEsuna` (`0x8018A4E4`) do not call `FUN_8018877C` and therefore skip both the status overlay (`FUN_80186FF8`: Oil×Fire / Float×Earth) and the equipment matrix (`FUN_80184E98`), so a monster ability carrying an element bit deals damage with no equipment/status reduction; exception: `4D_MonAbsorb` calls `FUN_80187248`, so Undead status still flips its heal sign.** — `[S] 1/3`
  - S: handler addresses `0x8018A398`, `0x8018A3CC`, `0x8018A454`, `0x8018A49C`, `0x8018A4E4` (`battle_disassembly.txt`)
  - R: none — monster-formula element-blindness not present in godot-learning (no "MonAbsorb"/monster-formula carve-out in `src/gpu/` or `tests/`)
  - src: `research/working_documents/status_element_interplay.md`

## Notes

(empty — user territory)

## Related

- [[Status Element Defense Interplay]]
