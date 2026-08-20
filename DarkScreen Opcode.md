# DarkScreen Opcode

Event instruction `{76}` DarkScreen (Unknown:1, Shape:1, Screen Expansion Speed:1, Rotation Speed:2, Square Expansion Speed:1 — 6-byte body) is FFT's diamond-mosaic screen darkening: the main executor (`battle:0x80145260`) spawns a short task body `FUN_8013bd94` that sets barrier kind `0x36`, grabs the screen-FX lock (`FUN_8013bc14`, token 6), then jumps into the real renderer — a DMA-loaded runtime overlay at `0x801CA664` that paints a 288-diamond semi-transparent mosaic (PSX abr mode 0, colour RGB(48,40,16)) over the map with a top-left-anchored diagonal sweep (~1.3 s dead time + ~1.2 s fill), leaving the field dimmed to ~0.69× luminance. `{E5} WaitForInstruction(0x36)` blocks the scenario cursor until `{77}` RemoveDarkScreen posts the teardown, and the sibling `{78}` DisplayConditions draws the victory-conditions box over the darkened field. All statics were grounded live on 2026-07-02 (Orbonne scenario 4, PC 130), and the GPU-primitive capture (POLY_F4 pool at `0x801D8000`, 44 800-pixel blend regression, framebuffer time series) closed the geometry/blend/timing gaps the same day.

## Points

- **`{76}` is DarkScreen, not "Set Text Speed": the main executor's bne comparison chain routes `0x75` → `set_event_text_glyph_throttle` (`0x8013da00`) and `0x76` → DarkScreen — so `gen_opcode_catalog.py`'s provisional `0x76 → 0x8013da00` cross-reference is a misattribution. The `0x76` case at `battle:0x80145260` clears `g_deploy_open_flag` (`0x80166044`), allocates a task via `FUN_80149bec(0x10)`, and spawns body `FUN_8013bd94` through `event_task_spawn`.** — `[S·D] 2/3`
  - S: main-executor `bne`/`jal` chain `battle:0x80145248`–`0x80145294` (`battle_disassembly.txt`)
  - D: Orbonne scenario-4 live run — Exec BP at `0x80145268` fired with `s4=0x76`, `s1=0x8004A9D7` pointing at live chunk bytes `76 00 01 0C 40 00 04 E5 36 00 78`; 11/11 cited dispatch/handler words byte-match RAM (savestate `orbonne_darkscreen_dispatch.sstate`, 2026-07-02)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **The shared dispatch tail `LAB_80145f0c` wires the task up: it copies the 6 DarkScreen operand bytes into the task struct, stores the body pointer into the slot at `0x8017403C`, and the current-task slot `0x80174038` (where the overlay entry reads its operands) is set at task start.** — `[S] 1/3`
  - S: `battle:0x80145f0c` (`battle_disassembly.txt`)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **The `{76}` task body `FUN_8013bd94` is a three-call delegator: `event_task_set_kind(0x36)` → `FUN_8013bc14(0x6)` (acquire the screen-FX lock, token 6, slot `0x80166004`) → `SUB_801ca664` (the overlay that does all the work) → return — no inline work of its own.** — `[S·D] 2/3`
  - S: `battle:0x8013bd94`–`0x8013bdbc` (`battle_disassembly.txt`)
  - D: byte-matches `0x8013bda0` = `34040036` and `0x8013bdac` = `0C072999` = `jal 0x801CA664` (savestate `orbonne_darkscreen_dispatch.sstate`, 2026-07-02)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **The kind `0x36` (decimal 54) the DarkScreen task sets is the barrier `{E5} WaitForInstruction` waits on — its handler (`0x80145964`) yield-loops on the task of the given kind, and the very next instruction in the scenario-4 chunk is `{E5} WaitForInstruction(0x36, 0x00)` whose task argument equals this task's kind.** — `[S·D] 2/3`
  - S: `event_task_set_kind(0x36)` at `battle:0x8013bd9c`; `{E5}` handler `0x80145964` (`battle_disassembly.txt`)
  - D: live bytes after the 6 operands at PC 130 = `E5 36 00`, followed by `78` (DisplayConditions) (savestate `orbonne_darkscreen_dispatch.sstate`, 2026-07-02)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **`FUN_8013bc14` is the shared "screen-FX lock" — a spin-yield mutex over the two-slot semaphore `DAT_80166004`/`DAT_80166008`: it loops `event_fiber_yield` while either slot is nonzero, then claims slot 1 with token 6 (`sw s0, DAT_80166004` at `0x8013bc64`, released at `0x8013ccb8`); sibling `FUN_8013bcbc` claims slot 2 with token 4 (`0x8013bd0c`/`0x8013cbc8`), and the two slots stay mutually exclusive for the effect's lifetime.** — `[S·D] 2/3`
  - S: `battle:0x8013bc14`, `0x8013bc64`, `0x8013bd0c`, `0x8013ccb8`, `0x8013cbc8` (`battle_disassembly.txt`)
  - D: `0x80166004` transited `0 → 6 → 0` over the DarkScreen frames (savestate `orbonne_darkscreen_dispatch.sstate`, 2026-07-02)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **The diamond-mosaic renderer is a DMA-loaded runtime overlay at `0x801CA664` that is absent from the static disassembly: the word at that address flips from stale data `0x0048FE7C` to the prologue `0x27BDFF70` (`addiu sp, sp, -0x90`) a few frames after DarkScreen dispatch, so `SUB_801ca664` only ever exists in live RAM.** — `[S·D] 2/3`
  - S: `battle:0x8013bdac` = `jal SUB_801ca664`; `0x801ca6xx` body undefined in `battle_disassembly.txt` / `hacktics_disassembly.txt`
  - D: `0x801CA664` word-monitor during Orbonne scenario-4 DarkScreen — `0x0048FE7C` pre-dispatch, `0x27BDFF70` mid-effect, dumped to `darkscreen_overlay_801CA000_801CC000.bin` (2026-07-02)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **The overlay entry (`0x801CA664`) reads the current task slot `0x80174038` (populated by `LAB_80145f0c` with the 6 operands), stashes state into `DAT_80173FE0`, waits on `FUN_8013b590(0x27)` — a ~39-frame gate, ~1.3 s of on-screen dead time before the mosaic starts — and initializes a per-diamond state table at `0x801E8C3C` (each slot's `+0x3C` seeded to −12) before driving the mosaic.** — `[S·D] 2/3`
  - S: entry behaviour decoded from the live overlay dump `0x801CA664`–`0x801CA6D0` (`darkscreen_overlay_801CA000_801CC000.bin`)
  - D: framebuffer time series — no coverage change until ~1.3 s after dispatch, consistent with the `0x27` gate (savestate `orbonne_darkscreen_dispatch.sstate`, 2026-07-02)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **`0x801C3AB0` (sibling overlay function called from the entry) is the VRAM save/restore + texture-upload step, not the per-frame diamond draw: a `StoreImage`/`LoadImage`-class preserve of two framebuffer regions belonging to the deploy/conditions box, plus two `GsLoadImage` calls (`0x800248FC`) uploading the conditions-box art via rectangle descriptors at `0x801D0068`/`0x801D0070`.** — `[S·D] 2/3`
  - S: static decode of `0x801C3AB0` from the wider live dump `0x801C0000`–`0x801CC000` (48 KB)
  - D: session-2 capture of the Orbonne DarkScreen; the settled frame `darkscreen_settled_conditions.png` shows the box art rendered over the dark field (2026-07-02)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **The 6 DarkScreen operands are `[00, Shape, ScreenExpansionSpeed, RotationSpeed, 00, SquareExpansionSpeed]`, and the Orbonne scenario-4 live value `[00 01 0C 40 00 04]` matches the wiki defaults exactly (Shape=1, Screen Expansion Speed=12, Rotation Speed=0x40, Square Expansion Speed=4); `Shape` selects the mosaic start point and the speed bytes drive spread-to-fill and individual-diamond grow-to-full.** — `[S·D] 2/3`
  - S: the overlay's `lbu` of the Shape/speed bytes from the task slot (operand pointer from `FUN_8014CBF4`) — decoded from the live overlay dump (`darkscreen_overlay_801CA000_801CC000.bin`)
  - D: live read at `s1=0x8004A9D7` = `[00 01 0C 40 00 04]` (savestate `orbonne_darkscreen_dispatch.sstate`, 2026-07-02)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **The mosaic is an ordering table of 576 flat quads in a pool at `0x801D8000`–`0x801E4000`, all with identical word0 `0x2A102830` — GP0 command `0x2A` = semi-transparent POLY_F4 — with packet colour BGR `0x10, 0x28, 0x30` → RGB(48, 40, 16), a dark warm-brown/sepia tint.** — `[S·D] 2/3`
  - S: POLY_F4 packet decode of the live buffer at `0x801D8000` (`darkscreen_polyf4_buffer_801D8000.bin`)
  - D: live 48 KB slice of the packet pool captured mid-effect, Orbonne scenario 4 (2026-07-02)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **Each diamond is an axis-aligned diamond (a rotated square with cardinal vertices on its centre) with half-diagonal 12 px (24 px corner-to-corner) laid on a quincunx lattice of base pitch 12 px, basis vectors `(24,0)+(12,12)`, origin at the top-left corner — the 288-diamond pool tiles exactly at full size (cell area 24·12 = 288 px²) and no rotation is applied in scenario 4 despite `RotationSpeed=0x40`.** — `[S·D] 2/3`
  - S: lattice/size analysis of the captured buffer — 127 distinct active centres at ~47 % coverage, 12 px run-length pitch (`darkscreen_polyf4_buffer_801D8000.bin`)
  - D: coverage masks `darkscreen_mosaic_mask_{47,57}pct.png` and static-map comparison showing axis-aligned 24 px-pitch diamonds (Orbonne scenario 4, 2026-07-02)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **Each diamond is drawn twice — two overlapping POLY_F4 packets per centre (576 = 288 slots × 2, unspawned slots parked at (0,0) size 0) — so the doubled half-blend leaves diamond cores (two diamonds) darker than single-covered seams: core = ¼·bg + ¾·F, seam = ½·bg + ½·F.** — `[S·D] 2/3`
  - S: per-centre packet count from the buffer decode (116 of 127 centres × 2)
  - D: buffer + mask capture (Orbonne scenario 4, 2026-07-02)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **The mosaic blends with PSX abr mode 0 = ½·bg + ½·F (no alpha channel): a least-squares fit over all 44 800 changed map pixels gives slopes 0.457 / 0.473 / 0.469 (≈½ on R/G/B) with intercept consistent with F = the packet colour RGB(48,40,16); the settled scene is dimmed — mean luminance 74.7 → 51.2 (×0.69, steady) — not faded to black.** — `[S·D] 2/3`
  - D: 44 800-pixel least-squares regression of the pre-effect vs. settled framebuffers (Orbonne scenario 4, 2026-07-02)
  - S: packet colour from the `0x801D8000` buffer decode (authoritative F)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **Mosaic timing: ~1.3 s of dead time after dispatch (the `FUN_8013b590(0x27)` ≈39-frame gate plus setup), then a spawn front sweeping diagonally from the top-left corner (each diamond growing point→full at `SquareExpansionSpeed`) to full-screen coverage in ~1.2 s (~73 frames) as an area∝radius² S-curve — slow ~1→10 % over 1.33→1.7 s, fast 10→100 % over 1.7→2.55 s, fully covered by ~2.55 s, with the `{78}` conditions box drawn at ~4 s.** — `[S·D] 2/3`
  - S: `SUB_801ca664` calls `FUN_8013b590(0x27)` (live overlay decode, `darkscreen_overlay_801CA000_801CC000.bin`)
  - D: framebuffer time series + coverage masks (47.2 % at 1.33 s, 57.0 % at 1.7 s, 100 % at 2.55 s) (Orbonne scenario 4, 2026-07-02)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **`{77}` RemoveDarkScreen (`battle:0x8014529c`) allocates a task via `FUN_80149cbc(0x37)` and writes `0x36` into that task's kind field (`DAT_801698b8`, slot stride 0x400), posting the kind-`0x36` teardown that unwinds the DarkScreen barrier/lock — no new body pointer is passed.** — `[S] 1/3`
  - S: `battle:0x8014529c`–`0x801452a4` (`battle_disassembly.txt`)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **`{78}` DisplayConditions (`battle:0x801452d0`) is a fiber spawn, not inline: `FUN_80149bec(0x10)` alloc + `event_task_spawn(body = FUN_8013bd6c)`; in Orbonne it renders the "Conditions for Winning: Defeat all enemies!" box over the settled dark field ~4 s after the DarkScreen dispatch.** — `[S·D] 2/3`
  - S: `battle:0x801452d0`, body `FUN_8013bd6c` (`battle_disassembly.txt`)
  - D: live frame `darkscreen_settled_conditions.png` (Orbonne scenario 4, 2026-07-02)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`
- **`FUN_8013bdcc` (kind `0x46`, busy flag `0x8016607c`) is the party-deploy / save-formation coroutine that walks the 21-slot actor table (`FUN_80180afc` → `0x801908cc + idx*0x1c0`, unit record at `+0x161`) while holding a VRAM save/restore handle in `0x8016601c` — it is spawned from the battle-setup state machine at `battle:0x80142034`, not from the `{76}` body, and is a sibling DarkScreen accompanies visually (the deploy scene is dark-field + formation screen).** — `[S] 1/3`
  - S: `battle:0x80142034`, `FUN_8013bdcc`, `DAT_8016607c` writers `0x8013bde8`/`0x8013bf04` (`battle_disassembly.txt`)
  - src: `research/working_documents/DARKSCREEN_OPCODE_76_INVESTIGATION.md`

- **The `0x801C****` runtime overlay is real and its identity is now settled — it is `EVENT/ATTACK.OUT`, which loads at `0x801BF000` — but the causal reading of the `0x801CA664` prologue is wrong: what was watched was the *next* overlay landing, not the `{76}` dispatch's own code.** — `[S·D] 2/3`
  - S: matching the resident bytes against every `EVENT/*.OUT` off the ISO at the instant of a dispatch gives **`ATTACK.OUT` at 125,037 of 125,956 bytes (99.3%), with a 57,436-byte exact leading run**, against 13–41% for every rival — which is what agreeing zeros look like. The residue is a known overlap: `EVENT/ETC.OUT` sits on `ATTACK.OUT`'s first 7,548 bytes, and while it is resident `ATTACK.OUT` reads 93.9% with a leading run of 0
  - D: the word `0x27BDFF70` does flip at `0x801CA664` — at frame 14,995 against a `{76}` at 14,976 — but that word sits at `+0xB664` in **`EVENT/REQUIRE.OUT`** and in no other candidate, so it is a different overlay arriving 19 frames later. The address the `{91}` spawn row actually needs, `+0xAD68`, was already `0x27BDFFD8` at frame **3,294**, 6,302 frames *before* the `{91}` that reads it. A prologue appearing near a dispatch is an overlay swap until an address is matched to a file (web-psx `docs/event-seam.md` [event.hle.spawn]; cross-referenced 2026-08-19)
  - src: external contribution — web-psx `docs/event-seam.md` [event.hle.spawn] (see [[Web-psx Cross-Validation]])

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Color Screen Opcode]]
- [[Display Message Opcode]]
- [[PSX GPU Primitives]]
