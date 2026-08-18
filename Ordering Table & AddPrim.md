# Ordering Table & AddPrim

How custom effect primitives are submitted to the PSX GPU: the ordering table (OT) is an array of per-depth linked-list heads (24-bit pointers) whose base address is held in g_ordering_table_ptr (0x801BC0CC), and AddPrim (0x80023BB4) links each primitive into a depth chain at the head; depth values control occlusion — higher depth renders behind, lower in front — so the hook computes a depth from the caster's screen Y (screen_y>>2 + 115, empirically tuned) to place effects at the caster's depth. On the draw side, the stock libgpu DrawOTag (FUN_80024c38) DMAs each OT chain head→tail, so same-depth prims draw in reverse AddPrim-call order (last-submitted first) — oracle-confirmed (2026-07-20). For effect particles the bucket key is the GTE view-space Z: `bucket = SZ >> 2` (4 SZ units per fixed-width bucket, 383 buckets 0..0x17E, depth_mode biases on the index, per-frame base `DAT_801bc088`), and per-particle Z is computed on the fly — never stored in the particle struct. For the particle path the double head-insert (spawn head-insert + head→tail render walk + AddPrim head-insert) nets to a spawn-age tie-break — within an equal-depth bucket the newest-spawned particle is drawn last, on top (DEMI2's white additive surviving its subtractive cloud; oracle-confirmed 2026-07-20).

## Points

- **The ordering table is an array of linked-list heads, one per z-depth, each entry a 24-bit pointer to the first primitive at that depth; the OT base is read from g_ordering_table_ptr at 0x801BC0CC, and the OT entry for a depth is OT_base + depth×4.** — `[S] 1/3`
  - S: g_ordering_table_ptr 0x801BC0CC, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **OT depth semantics (verified empirically): higher depth renders BEHIND, lower depth renders IN FRONT, because the GPU processes OT entries from highest index to lowest — depth 0 renders last (in front of everything) and depth 200 first (behind everything); depth ≈ screen_y>>2 approximately matches unit sprite depth.** — `[S·D] 2/3`
  - S: g_ordering_table_ptr 0x801BC0CC (the OT it points to), per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - D: empirical OT depth test (depth 0 vs 200 vs screen_y>>2) (doc 2026-04-16)
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **AddPrim (0x80023BB4; a0 = OT entry, a1 = primitive) inserts at the head of the depth chain (*(uint*)prim = *ot_entry; *ot_entry = prim & 0x00FFFFFF), so primitives at the same depth render in reverse order of AddPrim calls (last added = first rendered = backmost); to place an effect at roughly the caster's depth, the hook uses depth = (screen_y >> 2) + 115, with the +115 offset found by binary search (120 placed the effect behind all units, 100 in front of units in front of the caster).** — `[S·D] 2/3`
  - S: AddPrim 0x80023BB4, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - D: +115 caster depth offset found by binary search (doc 2026-04-16)
  - D: ground-truth oracle confirmation of same-depth reverse-submission draw order (#214, 2026-07-20), per `research/working_documents/COMPOSITOR_DEPTH_ORDERED_FOLD.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **The OT draw side is the stock libgpu DrawOTag (FUN_80024c38), a hardware GP0 chain-DMA that walks each linked `addr` chain head→tail — so with AddPrim's (0x80023bb4) head-insertion (`p->addr = ot->addr; ot->addr = p`), within one OT bucket draw order is the reverse of AddPrim-call order: the last-submitted prim draws first (furthest back), the first-submitted draws last (on top) — and this equal-z tie-break is CONFIRMED against the ground-truth oracle (#214, 2026-07-20).** — `[S·D] 2/3`
  - S: AddPrim 0x80023bb4 / DrawOTag FUN_80024c38 (doc-cited 0x80024c5c is the function's `DrawOTag(%08x)` debug-string offset), per `research/working_documents/COMPOSITOR_DEPTH_ORDERED_FOLD.md` + `project-assets/fft-rom/scus_disassembly.txt`
  - D: ground-truth oracle (#214, 2026-07-20)
  - src: `research/working_documents/COMPOSITOR_DEPTH_ORDERED_FOLD.md`
- **The effect-particle OT bucket index is `bucket = SZ >> 2` — 4 GTE view-space Z units per fixed-width bucket, a pure function of depth (never prim-count-dynamic), with blend direction (add/sub) not part of the key; `SZ` is the `rotate_vector` (0x8001D578) output and is linear in view-space depth because the perspective divide only touches `SX`/`SY`, never `SZ`.** — `[S] 1/3`
  - S: `sra a0,v0,0x2` @ `0x801aa528`; `rotate_vector` call @ `0x801aa51c` (`SZ` in `local_50`), per `research/working_documents/COMPOSITOR_OT_BUCKET_WIDTH_PARITY.md` §4a + `project-assets/fft-rom/battle_disassembly.txt`
  - src: `research/working_documents/COMPOSITOR_OT_BUCKET_WIDTH_PARITY.md`
- **The particle OT has 383 buckets (indices 0..0x17E): the post-shift bucket index is clamped to `[0, 0x17E]` and negative bucket indices are skipped.** — `[S] 1/3`
  - S: `slti v0,a0,0x17f` @ `0x801aa57c`; `ori a0,zero,0x17e` @ `0x801aa588`; negative-skip `bltz` @ `0x801aa578`, per `research/working_documents/COMPOSITOR_OT_BUCKET_WIDTH_PARITY.md` §4a + battle disassembly
  - src: `research/working_documents/COMPOSITOR_OT_BUCKET_WIDTH_PARITY.md`
- **The depth_mode bias is applied to the bucket index AFTER the `SZ >> 2` shift: mode 1: −8, mode 2: +8, mode 3: forced to 0x17E, mode 4: forced to 0x10, mode 5: −16, mode 0: none — matching Godot's `DepthMode.Mode`.** — `[S·R] 2/3`
  - S: mode switch @ `0x801aa52c`–`0x801aa574`, per `research/working_documents/COMPOSITOR_OT_BUCKET_WIDTH_PARITY.md` §4a + battle disassembly
  - R: `godot-learning/src/core/DepthMode.gd` (depth_mode biases; `UNITS_PER_OT_BUCKET` :38, `PULL_FORWARD_8` = 8·0.19 :118, doc-cited)
  - src: `research/working_documents/COMPOSITOR_OT_BUCKET_WIDTH_PARITY.md`
- **The per-frame OT base is held in `DAT_801bc088` (set from `func_0x80044a60()`), 4 bytes per bucket, and the particle prim is inserted at `bucket_index*4 + DAT_801bc088`.** — `[S] 1/3`
  - S: `DAT_801bc088` / `func_0x80044a60`, per `research/working_documents/COMPOSITOR_OT_BUCKET_WIDTH_PARITY.md` §4a + battle disassembly
  - src: `research/working_documents/COMPOSITOR_OT_BUCKET_WIDTH_PARITY.md`
- **Per-particle Z is NOT stored in the particle struct — it is computed on the fly from the particle's world position through the GTE — so the OT linked lists themselves (walk `DAT_801bc088`, 383 × 4-byte heads, follow each chain) are the only live source of per-prim depth.** — `[S] 1/3`
  - S: disassembly dig, per `research/working_documents/COMPOSITOR_OT_BUCKET_WIDTH_PARITY.md` §4a
  - src: `research/working_documents/COMPOSITOR_OT_BUCKET_WIDTH_PARITY.md`
- **`ClearOTagR` links every EMPTY bucket head to its neighbor, so a raw OT walk always shows 383 non-null heads; counting only heads pointing outside the OT array (real prim packets) gives the true occupancy — a live walk of the whole battle scene (map + unit sprites + UI + shadows + effect) found 104 occupied buckets spanning indices 0..244, and with the machine parked at the exception vector (pc=0x80000080) the numbers are indicative, not a clean mid-effect snapshot.** — `[S·D] 2/3`
  - S: `DAT_801bc088` = 0x80057118 (OT base), per `research/working_documents/COMPOSITOR_OT_BUCKET_WIDTH_PARITY.md` §4b + battle disassembly
  - D: live OT walk via pcsx-agent (port 8080), effect-editor `demi2` session, savestate `SCUS94221.sstate8` (2026-07-20)
  - src: `research/working_documents/COMPOSITOR_OT_BUCKET_WIDTH_PARITY.md`- **The effect-particle submit path nets to a spawn-age tie-break inside an equal-depth OT bucket — the active particle list is head-inserted at spawn (newest first), the render loop walks it head→tail, and each prim head-inserts into its bucket via AddPrim — so the newest-spawned particle is drawn last (on top); PSX orders a shared add/sub bucket by age, not blend direction, because direction is a per-particle packet field read from the sprite-def flags word (`andi 0x200` → poly code 0x2E sub / 0x2C add) and is not part of the bucket key.** — `[S·D] 2/3`
  - S: `submit_sprite_to_ordering_table` @ 0x801a5390 (loop @ 0x801a5550), blend-direction flag read @ 0x801a55f4, per `project-assets/fft-rom/battle_disassembly.txt`
  - D: `demi2` effect-editor session PSX oracle capture, savestate `SCUS94221.sstate8` (2026-07-20) — the white additive core (248,248,248) survives on top of the subtractive cloud in the (113,131)–(137,157) box, i.e. the additive folded last at that pixel
  - src: `research/working_documents/DEMI2_E046_ADDITIVE_SUBTRACTIVE_ORDERING.md`

## Notes

(empty — user territory)

## Related

- [[PSX GPU Primitives]]
- [[Embedded MIPS Effect Code]]
- [[Custom Effect Hooks]]
- [[Display Space Blend Fold]]
