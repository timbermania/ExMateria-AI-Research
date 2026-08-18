# Cinematic Atlas Row

The per-unit atlas row of FFT's cinematic (EVTCHR) sprite rendering — which character row of the shared EVTCHR atlas each unit's blocks sample. Proven NOT to be a per-unit `+0x7a` delta in the chapel cinematic (zero for all 8 units; the non-zero writer never fires), with the row instead carried by the animation's EVTCHR frame data; plus the static decode of the `FUN_80083570` bit-switch that would write `+0x7a` from `unit[+0x144]`, and the renderer dispatch that shifts each block's source tile_y by the signed delta.

## Points

- **The PSX cinematic atlas row is not a per-unit `+0x7a` delta in the chapel: `+0x7a == 0` for all 8 cinematic units (only the allocation zero-init `0x80087C58` fires; the genuine non-zero writer `0x80083620` fired zero times during the cinematic, and `FUN_80083570` itself fires event-driven on anim-arm — 1× in 25 s / 1× in 8 s — not per-frame), so the per-unit row comes from the animation's EVTCHR frame data (anim id → `cinematic_seq` → frame byte → source rect), the same mechanism that already decodes correctly for anims 0x258–0x25f at offset 0.** — `[S·D] 2/3`
  - S: writer enumeration — zero-inits `0x80087C58` / `0x800834CC`, non-zero writer `0x80083620` (`battle_disassembly.txt`)
  - D: chapel cinematic BP capture (renderer + `FUN_80083570` entry + all five `+0x7a` writers): `0x80083620` fired 0×, live roster `+0x7a==0` and `+0x144==0` for slots 0,1,2,3,4,5,6,12 (2026-06-28)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`
- **`FUN_80083570` sets `+0x7a` purely from `unit[+0x144] & 0xf` — bit0 → 0x60 (`0x800835bc`), bit2 → 0x30 (`0x80083620`), bit1 → 0 (`0x80083610`), bit3 → 0 (`0x800835e4`), nibble 0 → `LAB_80083680` default-sprite-remap path which forces `+0x7a=0` (`0x800836cc`, often early-returning at `0x800836bc`) — and `+0x144` is a pending-bits bitfield folded each tick by the caller `FUN_80068d08` (`+0x144 = (old & ~unit[+0x154]) | unit[+0x14c]`), the set-mask `+0x14c` being written by an anim-type switch at `0x80068ef4`.** — `[S] 1/3`
  - S: `FUN_80083570` bit-switch stores `0x800835bc`/`0x80083620`, `LAB_80083680` default path, `FUN_80068d08` fold + set-mask write `0x80068ef4` (`battle_disassembly.txt`; addresses re-verified live against BP instruction words, not +0x800-shifted)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`
- **The cinematic renderer `FUN_80085c0c` (update_and_animate_unit_wep_eff) dispatches on the anim slot `unit[+0x0c]` — `0` → `LAB_80086308` (zero-anim / pose-octant), `>= 0x1f5` → `LAB_80086164` (EVTCHR re-derive) — and the frame commit shifts each block's source tile_y by the signed `unit[+0x7a]` delta (sites `0x80084258` / `0x8008435c` in `FUN_80084214`).** — `[S·D] 2/3`
  - S: `FUN_80085c0c`, `LAB_80086308`, `LAB_80086164`, shift sites `0x80084258` / `0x8008435c` (`battle_disassembly.txt`)
  - D: renderer BP at `0x80085c0c` (~227 Hz in chapel) walked both cohorts, always-present and late-adds, in the EVTCHR branch (2026-06-28)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`

## Notes

(empty — user territory)

## Related

- [[Cinematic Palette Pipeline]]
- [[Unit Anim Opcode]]
- [[ENTD Unit Deployment Table]]
