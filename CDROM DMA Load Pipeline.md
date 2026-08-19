# CDROM DMA Load Pipeline

How FFT pulls CD data into RAM: every file load — the 1.5 MB BATTLE.BIN battle overlay, the 128 KB ATTACK.OUT overlay, and the small per-event sub-loads — funnels through one SCUS file-load state machine (`FUN_80011E38`, ctx at `0x8004EAF4`) and PSX DMA channel 3 (CDROM→RAM), kicked per 0x800-byte sector by `FUN_80020650` @ `0x800206FC`, the only ROM site that writes the CDROM channel's CHCR-GO. The CDROM→RAM copy is invisible to per-byte CPU write BPs; only the register-programming sites are observable.

## Points

- **All CD file loads funnel through one chain: `BATTLE_open_ATTACK_OUT_deployment` (`0x8013D320`) jal's the thunk `0x8014CF28` (via `0x8013D38C`) which dispatches `*0x80173CA8 = FUN_80044954` (SCUS thin wrapper) → `FUN_800448A0` (polling loop) → `FUN_80011E38` (file-load state machine; ctx global `0x8004EAF4` with +0x04 state 0..6, +0x10 sector_count, +0x1C filename buffer, +0x20 RAM destination) → caseD_5 `FUN_80020C3C(LBA, dest, 0x80, count)` → `FUN_80020A64` (registers the per-sector IRQ callback `FUN_80020840` via `FUN_8001EB70`) → at each CDROM data-ready IRQ (one per 0x800-byte sector) `FUN_80020840` → `FUN_8001EF54` → `FUN_80020650` (the DMA-3 kicker), with 1976 transfers firing in the 14-s battle-setup window.** — `[S·D] 2/3`
  - S: `BATTLE_open_ATTACK_OUT_deployment` `0x8013D320`, thunk `0x8014CF28..0x8014CF50` (`lui at,0x8017` / `lw t0,0x3CA8` / `jalr t0`), `SUB_80011e38` call sites, thunk-store comment @ `0x8013D384` (battle_disassembly.txt)
  - D: live Exec-BP pass through `0x800206FC` — 1976 CDROM-DMA transfers in the 14-s load window (sstate2 + Enter, 2026-06-20)
  - R: none — CDROM/DMA load chain not present in godot-learning (probed `src/`, `tests/`, `tools/` for `CDROM`/`DMA`/`800206FC`)
  - src: `research/working_documents/SCENARIO_LOADING.md`
- **Battle setup issues two big `FUN_80044954` loads (outer ra `0x80044964`): first 752 sectors (0x2F0, 1.5 MB / 0x178000 bytes, LBA `0x3E8`) to `0x80067000` — the BATTLE.BIN battle overlay + adjacent overlays, entered directly from the SCUS jal site `0x800409E0` because the BATTLE.BIN thunk is not resident yet — then 64 sectors (0x40, 128 KB / 0x20000 bytes, LBA `0x990`) to `0x801BF000`, which is ATTACK.OUT (its scenario table at file offset `0x10938` → RAM `0x801CF938`); per-event sub-loads are a third actor whose caller `0x80079908` (secondary path `0x8007A018`) skips the `FUN_800448A0` wrapper and calls `FUN_80011E38` directly for 1–64-sector reads into the `0x801DF000` scratch region.** — `[S·D] 2/3`
  - S: per-event sub-load call sites `battle:80079900` / `battle:8007a010` (`jal SUB_80011e38`) (battle_disassembly.txt)
  - D: 14-s sstate2 + Enter capture — load table with caller rAs, sector counts, RAM dests, LBAs (2026-06-20, `scenario_1_captures/file_load_capture.json`)
  - R: none — battle-overlay / per-event sub-load pipeline not present in godot-learning
  - src: `research/working_documents/SCENARIO_LOADING.md`
- **PSX DMA channel 3 is the CDROM channel — D3_MADR `0x1F8010B0` (RAM destination), D3_BCR `0x1F8010B4` (block low-16 | count high-16), D3_CHCR `0x1F8010B8` (bit 24 = start) — and `FUN_80020650`'s GO write at `0x800206FC` is the only CPU site in the ROM that starts channel 3, so every ATTACK.OUT/BATTLE.BIN sector passes through that single instruction and the CDROM→RAM copy is invisible to per-byte CPU-store write BPs (Write BPs on `0x801BF000`/`0x801BF004`/`0x801CF938` stayed silent across the load window even though the bytes demonstrably landed).** — `[S·D] 2/3`
  - S: D3_MADR/BCR/CHCR `0x1F8010B0..0x1F8010B8`, CHCR-GO write @ `0x800206FC` inside `FUN_80020650` (battle_disassembly.txt)
  - D: silent Write-BP results across the 12–14 s load window + 64× sector GO observations for the ATTACK.OUT load (2026-06-20)
  - R: none — PSX DMA register programming not present in godot-learning
  - src: `research/working_documents/SCENARIO_LOADING.md`
- **Filename-driven opens resolve MSF→LBA in caseD_3: the ctx `+0x1C` filename buffer holds the 3-byte BCD-MSF location passed to `CdlSetloc` (PSX BIOS CD command `0x15`), with `lba = (bcd(m)*60 + bcd(s))*75 + bcd(f) − 150`.** — `[S·D] 2/3`
  - S: `SUB_80011e38` file-load state machine + `CdlSetloc` (BIOS CD cmd `0x15`) (battle_disassembly.txt)
  - D: per-event sub-load table — filenames with LBAs matching the BCD-MSF conversion (sstate2 + START + 48 s capture, 2026-06-20, `scenario_1_captures/file_load_capture.json`)
  - R: none — MSF/LBA resolution not present in godot-learning
  - src: `research/working_documents/SCENARIO_LOADING.md`
- **Scenario 1's battle-prep sub-loads all land in the `0x801DF000` scratch region, each file being consumed (parsed/copied out) before the next load overwrites it: `MAP062.GNS` (548 B, the scene/group descriptor), `MAP062.8` (27 sectors, 55008 B, alternate-arrangement mesh), `MAP062.7` (64 sectors, 131072 B, primary mesh), `ENTD3.ENT` (LBA 60433, 40 sectors, the full 80 KB bank covering entd_idx 256–383), the `TYPE1/2/3`, `CYOKO`, `MON`, `OTHER`, `WEP1/2`, `EFF1` `.SHP`/`.SEQ` sprite+animation families plus the weapon/other sprite atlases, and the sprite atlases `0B/12/33` (loaded with battle prep) + `01/16/5F/61/63` (loaded mid-cinematic as the units are introduced).** — `[D] 1/3`
  - D: 36-load LBA table (sstate2 + START + 48 s, 2026-06-20; `scenario_1_captures/file_load_capture.json`)
  - R: none — per-event sub-load inventory not present in godot-learning (its assets are parsed straight from the disc)
  - src: `research/working_documents/SCENARIO_LOADING.md`

## Notes

(empty — user territory)

## Related

- [[Scenario Table]]
- [[Inter Scene Orchestration]]
- [[ENTD Unit Deployment Table]]
- [[Map State Selection]]
