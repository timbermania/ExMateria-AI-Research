# Learn Job Picker

The formation-screen "Learn" option (3rd row of the `Set / Remove / Learn` popup, substate 0xc) and the Learn job picker it opens — RE'd end-to-end over five static + live pcsx-redux rounds. Pressing ○ is two vsync beats (0xf panel teardown, then init + build + first render + band-fade start in one chain); the candidate job array is pre-populated into two non-crossing `0xFFFF`-delimited columns; the 6-byte picker record writes only on confirm; confirm is JP-gated and commits into the ability-list phase (`0x8011F5F0`). The picker renders through the shared display-list machine: the cream JOB/LV/TOTAL/NEXT/JP headers are live FT4 quads under CLUT 0x7CBC (0x7C3C = panel chrome), and the prim pool's stored X is biased +128 over screen X. Open: band-fade per-frame law + writer fn, `cd738 = -1` row, job-ID→name map.

## Points

- **○ on the "Learn" row is two vsync beats, not one: beat 1 (substate 2) the dispatcher `FUN_8011DC98` runs `FUN_8012AAF4(0xf)` to tear down only the 0xf panel and sets substate to 0x0C; beat 2 (next vsync, substate 0x0C) `FUN_8011ED18` runs INIT + BUILD + first render + band-fade start in one chain** — `[S·D] 2/3`
  - S: 0x8011DC98, 0x8012AAF4, 0x8011ED18, 0x8011F090 (WORLD decompiles at `project-assets/fft-rom/WORLD/functions/`, base 0x800E0000)
  - D: round 4 live session (2026-08-15): the beat-1 state (`ba1c`=0x0C, `bae1`=0, `bae3`=0) was never a save state; ss1 = beat 2 (`ba1c`=0x0C, `bae1`=1, `bae3`=1, tiny aperture)
  - R: none — two-beat Learn transition not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`
- **The Learn entry INIT (`FUN_8011ED18`) closes 0x400-table windows 0xc and 0xa, blanks window 8, memsets the window-9 picker record (`FUN_80118B68`), and sets learn-init `DAT_8018bae1`=1; windows 8/10/12 and the 0xf panel all close within the same transition beat, leaving only window 0 (the persistent base screen) active** — `[S·D] 2/3`
  - S: 0x8011ED18 (WORLD decompile: `FUN_8012a598(0xc)`, `FUN_8012a598(0xa)`, `FUN_8012b950(8,0)`, `FUN_80118b68(9)`)
  - D: rounds 1b/2 live (2026-08-15): active 0x400 windows ss0 = {0, 8, 10, 12, 15(0xf)}; ss1–ss4 = {0} only; `bae1`=1 from ss1 on
  - R: none — 0x400 window-table teardown not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`
- **The Learn picker's candidate job array `0x801C8568` is pre-populated at 0xf-panel entry (identical in ss0–ss4, not by the press) by the skillset scan `FUN_8012257C`; the two `0xFFFF` markers are a column break + terminator — col0 = [0x4A, 0x4B] (`cd824`=2 rows, live-verified) and col1 = 16 jobs (0x4D..0x5B + 0x5D) — and LEFT/RIGHT keys have no effect on any cursor: the two columns are separate, non-crossing lists** — `[S·D] 2/3`
  - S: 0x801C8568, 0x8012257C, 0x801263C8 (counts candidates until 0xFFFF into `DAT_801CD824`) (WORLD decompiles)
  - D: round 2 live dump of `0x801C8568` + `cd824`=2 (2026-08-15); round 4 nav probe (down×5/right×2/left/up×2/down: only `cd20c` changed, clamped 0..1 on the 2-row col0)
  - R: none — 0xFFFF two-column candidate structure not present in godot-learning (probed godot-learning/src/ui3: `UIJobPopup` is a single name-only sorted list)
  - src: `research/working_documents/LEARN_PICKER.md`
- **The picker 6-byte record at `0x801C8364 + win*6` (shorts row, col, sel&0x3ff) is written ONLY on confirm (`FUN_80118BA4`) — live it is all-zero in every state for every window; the live browse cursor is `cd20c` (row) + `cd54c` (col), both 0 in all five states, which is why the glove sits on the first item** — `[S·D] 2/3`
  - S: 0x80118BA4 / 0x80118BF0 / 0x80118B68 (confirm-write / locate / memset) and the `FUN_8011F090` chooser (WORLD decompiles)
  - D: round 2 live (2026-08-15): all 16 windows' 6-byte records all-zero (except an unrelated win5 sel=0x0006); cd20c=cd54c=0 in ss0–ss4
  - R: none — 6-byte confirm-only record not present in godot-learning (probed godot-learning/src/ui3 + tests: `UIJobPopup` threads selection through scroll-list item data)
  - src: `research/working_documents/LEARN_PICKER.md`
- **Confirm (○) on the Learn job picker is gated by the JP gate `0x8011F040` (which doubles as the Jp-column provider, slot cd7c8): pass → commit (id normalized to `id − 0x4A` by `FUN_801223B8`, record written, highlighted-job tag `cd754 = id + 0x6000` set) and return 1 → phase-1 ability list `FUN_8011F5F0`; fail → SFX `0x801134E8(0xC007, 0x30)` and stay; cancel (×) sets `bacc`=2, returns −1, and restores the formation panel** — `[S·D·R] 3/3`
  - S: 0x8011F040, 0x8011F090, 0x801223B8, 0x8011F5F0, 0x801134E8 (WORLD decompiles)
  - D: rounds 1b/2 live (2026-08-15): cd754=0x604A in ss1–ss4 (cands[0]=0x4A + 0x6000), phase `0x801C854C`=0 in all states
  - R: `godot-learning/src/ui3/UILearnPanel.gd` (`_try_learn_ability` JP gate `available_jp < jp_cost → return`, routed through `learn_ability_from_job`) + `godot-learning/tests/ProgressionTesterTest.gd` (`_test_ability_learning_from_job`)
  - src: `research/working_documents/LEARN_PICKER.md`
- **The five cream column headers (JOB / LV / TOTAL / NEXT / JP) are five live per-frame FT4 quads, all CLUT 0x7CBC, OTL 0x2D, screen y26..35; 0x7CBC is a cream recolor of the 0x7C3C font palette (slot 1: cream 0x73BD vs tan 0x10A6), while the 0x7C3C quads are the panel's chrome (top bevel y30..39, left/right edge strips, 16-px row bands) — the earlier "title tab" was the first (JOB) header quad, not a tab and not static** — `[S·D·R] 3/3`
  - S: header providers installed at 0x8011F090 (0x8011EE88 Job / 0x8011EF08 Lv. / 0x8011EFC0 Total / 0x8011EF60 Next / 0x8011F040 Jp), palette sources 0x8018DF8C.. (lower-16 = CLUT), template s4/s10 header records (WORLD decompiles)
  - D: round 5 VRAM framebuffer dump at ss4 (2026-08-15): 5-for-5 exact match of the 0x7CBC quads @0x801F2960..0x801F2A00 with the framebuffer's cream 0x73BD runs
  - R: `godot-learning/src/ui3/elements/NumberFont.gd` (menu font CLUT = 0x7CBC, live-read from VRAM; also `src/ui3/UIUnitInfoWindow.gd`) + `godot-learning/tests/NumberFontTest.gd`
  - src: `research/working_documents/LEARN_PICKER.md`
- **In the 0x801F0000–0x801FFFFF VRAM prim pool, stored X = screen X + 128 (screen = raw − 128) with Y stored raw — verified three ways: (a) the five 0x7CBC header quads map exactly onto the five cream runs, (b) the 0x7C3C edge quads at raw x156..161 / x357..362 = panel edges x28..33 / x229..234, (c) 16 raw quads x156..362 = full panel width; a raw-X-only pool scan misses the headers (round 4's "DEFINITIVE NEGATIVE" was voided)** — `[S·D] 2/3`
  - S: pool prim VAs 0x801F2960..0x801F2A00 (headers), 0x801F0890..0x801F0AC0 (bevel chrome) (WORLD.BIN VRAM pool addresses)
  - D: round 5 VRAM framebuffer dump + full 1 MB VRAM / 2 MB RAM at ss4 (2026-08-15)
  - R: none — VRAM pool X-bias law not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`
- **The transition's dark band is two double-buffered subtractive (opbyte 0x62) descriptors at `0x801C0030` (even) / `0x801C016C` (odd), +0x13C apart: core rect x=128 y=35 w=256 h=50 plus 1-px feather rows; the core foreground fades 120 → 84/96 → 84/72 → 60/48 → 0 across ss0–ss4 (even lags odd), reaching 0 in both buffers at settle, running in parallel with the aperture open; per-frame decrement law + writer fn remain OPEN** — `[S·D] 2/3`
  - S: 0x801C0030 / 0x801C016C + opbyte 0x62 (RAM layout per LEARN_PICKER §9.5; writer fn open)
  - D: rounds 1b/2/4 live bandtop core values (2026-08-15): even 120→84→84→60→0, odd 120→96→72→48→0; ss4 = `00 00 00 62` in both
  - R: none — dark-band fade not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`
- **The Learn picker's aperture open is driven by the s22 record's per-frame stage counter `0x8018E028` (++ up to cap 4, reset by setup `FUN_801263C8`) plus the s18 scale% u16[4] table @ `0x8018E048` = {10,60,90,95} — the first four unique stages of `world_menu_open_curve` 0x801533B8 [10,10,60,60,90,90,95,95,100,100,100,100], the same curve the formation/world menus use; live stage=4 and the scale% table exact at settle** — `[S·D] 2/3`
  - S: 0x8018E028, 0x8018E048, 0x801533B8, s18 0x801274D8, s22 0x80129B34 (WORLD decompiles; s22 disasm label LAB_80129B34)
  - D: round 4 live (2026-08-15): 0x8018E028=4, scale%=[10,60,90,95]; ss1–ss4 visible extents 24→80→144→187 px track the curve
  - R: none — aperture stage counter not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`
- **The VRAM pool's internal-prim record is 40 bytes (10 words), not 32: next(4) + attr(4) + 4 vertices × (x s16, y s16, u16, u16), with the CLUT at +0x0E (vertex-0's 4th word); attr = [OTL:8][24-bit pattern] (patterns seen: 0x000000 / 0x808080 / 0x565656 / 0x0B0B0B plus EEEDEE/B98888) — do NOT extract a GTE tpage from attr bits 20–23** — `[D] 1/3`
  - D: round 5 full 1 MB VRAM scan at ss4 (2026-08-15), corrected attr-based pool-scan gate validated against a known-CLUT histogram
  - R: none — pool prim record format not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`

## Notes

(empty — user territory)

## Related

- [[Formation Ability Picker]]
- [[Shared Display List Machine]]
- [[Menu Window Box Open]]
