# Status Element Defense Interplay

ROM (BATTLE.BIN) implements status × element interactions as hardcoded overlays in the spell-damage pipeline, not as element-mask changes: Oil doubles Fire damage before the equipment chain, Float hard-cancels Earth damage (via the same `FUN_80184E40` null path as an equipment Cancel bit, reading the mount host's status when mounted), Undead is a per-formula sign-flip consulted at four formula-handler sites, and Wall/Petrify are element-blind generic cancels. All of these run ahead of the equipment/job-driven Absorb > Cancel > Half > Weak matrix at `FUN_80184E98`; the weapon-element path (`FUN_80186FD0`) skips status overlays entirely, and monster ability handlers skip the overlay and matrix altogether. Reflect, Transparent, Charging, Berserk, and Frog have no element-defense role. godot-learning mirrors the spell-path overlay, the weapon no-status path, the 1.25× BoostElem strengthen, the Undead heal inversion, and the four-mask matrix in its GPU combat shaders, validated by dedicated GPU tests; the ROM byte layouts (element bytes, `+0x58..+0x5C` status bytes) and the caster-terrain multiplier are not mirrored.

## Points

- **A unit's element affinity is stored as five 1-byte per-element bitfields at unit offsets `+0x6D..+0x71` (AbsorbElem, NullElem/Cancel, HalfElem, DoubleElem/Weak, and attacker-side BoostElem), OR-folded from every equipped item's `ItemAttributes` element bytes (absorb at item attribute offset `+0x14` through boost at `+0x18`) by the SCUS battle-prep aggregation at `0x8005C784` (per-slot loop `0x8005C788`), with job-innate element defense OR'd into the same bytes in parallel.** — `[S·R] 2/3`
  - S: `SCUS::0x8005C784` / `0x8005C788` aggregation loop + `BATTLE.BIN::0x800642C4` `ItemAttributes` struct dump (`project-assets/fft-rom/hacktics_disassembly.txt`, `battle_disassembly.txt`)
  - R: `godot-learning/src/data/ElementEncoder.gd` (`defense_for_equipment` ORs the four target masks, `strengthen_for_equipment` folds BoostElem, both across every equipped slot; packed via `GPUCombatPacker._extract_unit_config`) + tests `godot-learning/tests/GPUElementAbsorbTest.gd`, `GPUElementCancelTest.gd`
  - src: `research/working_documents/status_element_interplay.md`
- **The equipment/job element-defense matrix (`FUN_80184E98` at `0x80184E98`) applies target-side masks in strict precedence: `+0x6D` Absorb match flags `CurActData+0x10 |= 0x400` (damage sign-flips into HP gain), else `+0x6E` Cancel tail-calls `FUN_80184E40` (damage zeroed, miss-kind tag set), else `+0x6F` Half halves `CurActData+0x4`, else `+0x70` Weak flags `CurActData+0x10 |= 0x800` and doubles it — and it consults no `+0x58..+0x5C` status byte.** — `[S·R] 2/3`
  - S: `FUN_80184E98` at `0x80184E98`, `FUN_80184E40` at `0x80184E40` (`battle_disassembly.txt`)
  - R: `godot-learning/src/gpu/shaders/combat_common.glslinc` `element_multiplier` / `apply_element_defense` (absorb -256 > cancel 0 > half 128 > weak 512, ×256-scaled) + tests `GPUElementAbsorbTest.gd`, `GPUElementCancelTest.gd`
  - src: `research/working_documents/status_element_interplay.md`
- **Attacker-side element strengthen: `FUN_80185FFC` at `0x80185FFC` reads the attacker's `+0x71` (BoostElem) against the ability element and, on any matching bit, scales `CurrentAbilityData.AbPower` by 5/4; its 13 XREF callers cover every spell formula handler, and it runs prior to target-side defense.** — `[S·R] 2/3`
  - S: `FUN_80185FFC` at `0x80185FFC` (`battle_disassembly.txt`)
  - R: `godot-learning/src/gpu/shaders/combat_common.glslinc` `apply_strengthen_elem` (1.25× AbPower on attacker strengthen-mask overlap) + test `GPUStrengthenFireTest.gd`
  - src: `research/working_documents/status_element_interplay.md`
- **Oil (`+0x5A & 0x80`) against a Fire ability (element bit `0x80`) doubles AbPower in the status overlay of `FUN_80186FF8` at `0x80187000`–`0x80187070` (also ORs `Target_CurActData+0x22 |= 0x80` and, via `FUN_80184B24`, writes damage-type tag `+0x25 = 8`), and because it lands before the equipment-matrix call at `0x801870E0`, an Oiled target with equipment Fire-Half nets 2× × ½ = 1× and an Oiled Fire-Absorb target still absorbs the doubled amount.** — `[S·R] 2/3`
  - S: `FUN_80186FF8` at `0x80187000` (`lbu target+0x5A; andi 0x80` → `AbPower <<= 1`), post-Oil fall-through `LAB_80187074` → `0x801870E0` (`battle_disassembly.txt`)
  - R: `godot-learning/src/gpu/shaders/combat_common.glslinc` `apply_element_defense` status overlay (`amount *= 2` for Fire + OIL before the mask chain) + tests `GPUOilFireDoubleTest.gd`, `GPUOilFireHalfCompositionTest.gd`, `GPUOilFireAbsorbCompositionTest.gd`
  - src: `research/working_documents/status_element_interplay.md`
- **Float (`+0x5A & 0x40`) against an Earth ability (element bit `0x08`) deals 0 damage — `FUN_80186FF8` tail-calls `FUN_80184E40` at `0x801870D0` (zero damage, miss-kind 7, effect-identical to a Cancel-Earth bit); when the target is mounted the Float bit is read from the mount host's `+0x5A` (host resolution at `0x80187074`–`0x801870A4` via `UnitBattleData[]` at `0x801908CC`, stride `0x1C0`), and the AI predictor duplicates the check at `0x8019E840`.** — `[S·R] 2/3`
  - S: `FUN_80186FF8` at `0x801870A8`/`0x801870D0`, mount-host resolution `0x80187074`, `UnitBattleData[0]` `0x801908CC` (`battle_disassembly.txt`)
  - R: `godot-learning/src/gpu/shaders/combat_common.glslinc` `apply_element_defense` status overlay (Earth + FLOAT returns 0, preempting the mask chain) + test `GPUFloatEarthAbsorbCompositionTest.gd`
  - src: `research/working_documents/status_element_interplay.md`
- **Undead (`+0x58 & 0x10`) is handled per-formula, not as an element-mask change: `0E_DeathSpell` entry at `0x80188FA0` sets `CurrentAbilityData.UndeadReverse`, `FUN_80187248` flips heal to damage (`Target_CurActData+0x25 |= 0x40`), `FUN_801873D8` inverts Phoenix Down (called only from `0E_DeathSpell` at `0x8018906C`), and `FUN_80187350` writes the post-damage display tag (`+0x25 = 0x80`); the community-lore "weak to Holy / absorb Dark" comes from the Undead *job*'s innate element table, not the status bit.** — `[S·R] 2/3`
  - S: `0x80188FA0`, `FUN_80187248` at `0x80187248`, `FUN_801873D8` at `0x801873D8`, `FUN_80187350` at `0x80187350` (`battle_disassembly.txt`)
  - R: `godot-learning/src/gpu/shaders/stage_compute.glsl` (heal inverted to damage under `STATUS_UNDEAD`) + test `GPUUndeadInvertTest.gd`
  - src: `research/working_documents/status_element_interplay.md`
- **Wall (`+0x5B & 0x01`) nullifies all magic damage against the target unless the action's `BATTLE_Copy_Of_CurActionTarget_State+0x2` has the ignores-Wall bit `0x01` — an element-blind immunity in the routine at `0x8018BAC8` (Wall check at `0x8018BAE4`, force-miss + clear-damage branch at `0x8018BB10`–`0x8018BB58`, routed through `BATTLE_force_attack_miss_by_catch`), and Petrify (`+0x59 & 0x80`) is the adjacent generic cancel in the same routine at `0x8018BA94`.** — `[S] 1/3`
  - S: immunity routine `0x8018BAC8`, `+0x5B & 0x01` read at `0x8018BAE4`, Petrify at `0x8018BA94` (`battle_disassembly.txt`)
  - R: none — Wall immunity not present in godot-learning (no `STATUS_WALL` in `src/gpu/shaders/`)
  - src: `research/working_documents/status_element_interplay.md`
- **The spell-damage post-processing helper `FUN_8018877C` chains, in fixed order: `FUN_80186568` (damage finalizer, `damage = AbPower × UnitPower`, no status reads) → `FUN_80186ED0` (caster terrain × element) → `FUN_80186FF8` (status overlay) → `FUN_80184E98` (equipment matrix); formula dispatch runs off the `AbilityFormulaCodePtrs[]` table at `0x8018F614`.** — `[S] 1/3`
  - S: `FUN_8018877C` chain + `AbilityFormulaCodePtrs` at `0x8018F614` (`battle_disassembly.txt`)
  - R: none — the ROM call chain not present in godot-learning (the overlay-before-matrix ordering is mirrored inside `apply_element_defense`, but the chain itself is not implemented)
  - src: `research/working_documents/status_element_interplay.md`
- **Reflect (`+0x5C & 0x02`), Transparent (`+0x5A & 0x10`), Charging (`+0x58 & 0x08`), and Berserk/Frog (`+0x5A & 0x08`/`0x02`) have no element-defense role: Reflect (`FUN_8018C9E4` at `0x8018C9E4`) is pure re-routing to the caster after the element handler has run, Transparent (`TransparentCheck` at `0x801854DC`) is an attacker-side hit% override that zeros the target's evasion bytes, Berserk scales attacker `AbPower × 3/2` and Frog clamps `AbPower = 1` in `FUN_80186254`/`0x801862A0`, and Charging appears in no element-defense function.** — `[S] 1/3`
  - S: `0x8018C9E4`, `0x801854DC`, `FUN_80186254` at `0x80186254`, `0x801862A0` (`battle_disassembly.txt`)
  - R: none — "no element role" negative result; no status×element branch for these statuses present in godot-learning (`STATUS_*` registry constants only)
  - src: `research/working_documents/status_element_interplay.md`
- **The weapon-element-only defense path `FUN_80186FD0` at `0x80186FD0` (1-instruction wrapper) passes `CurrentAbilityData.CurrentWeaponElement` straight to `FUN_80184E98` without the status overlays, so weapon-elemental physical attacks ignore Oil/Float status; all four ROM callers sit in basic-attack post-process chains.** — `[S·R] 2/3`
  - S: `FUN_80186FD0` at `0x80186FD0` (`battle_disassembly.txt`)
  - R: `godot-learning/src/gpu/shaders/combat_common.glslinc` `apply_weapon_element_defense` (mask-only, no status reads, called at the basic-attack and counter queue sites) + test `GPUWeaponElementNoStatusTest.gd`
  - src: `research/working_documents/status_element_interplay.md`

## Notes

(empty — user territory)

## Related

- [[Element Byte Encoding]]
- [[Status Bitfield Layout]]
- [[Caster Terrain Element Multiplier]]
- [[Monster Ability Element Defense]]
