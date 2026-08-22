# World Map Screen

The FFT world map (overworld) screen — the dispatcher of the whole game and target of every `post_scenario_step=0x80` scenario exit — is now reverse-engineered end to end. `WLDCORE.BIN` loads flat at RAM `0x80067000` (its baked absolute pointers prove the base; it shares that base with `BATTLE.BIN` as a mutually exclusive overlay), with `WORLD.BIN` resident alongside at `0x800E0000` as the front-end/graphics overlay the map code calls into. The view is a fixed, non-scrolling 1:1 window into a map stored in 240×240-texel slabs; the background is 81 quads in one pass over a 17×13 vertex grid, the aperture is a two-layer CLUT-ramp vignette over an 8 bpp distance field (additive glow off-centre), and the HUD is node-table-driven — current node `DAT_800D0BB4`, table at `0x800D3CD0`, one cel/animation system underneath. `WLDTEX.TM2` is a sector-packed `LoadImage` stream that replays bit-exact into all map-specific VRAM, and the ~11 s entry load is captured frame-by-frame (first paint at vsync 645). The dispatcher is solved (2026-08-21, round 12): `FUN_80091238` runs a 41-opcode bytecode over the per-node event table at `0x80097234`, `var[110]` is the story counter that orders the 55 type-8 reveal emits, and the travel model (48 routes, 12-bit bearings, 92-bit progression) is double-sourced and exported by `travel.py json`. Open: the roles of `WLDPIC/WLDBK/WLDMES.BIN`, the node map-space projection, the aux-DMA vs pool-reset ordering, the black top 8 rows, and the ~6 unaccounted seconds of the entry load. No Godot counterpart yet.

## Points

- **A savestate pair brackets the entire world-map entry load: `world_map_ss0_scenario_end_prequicksave.sstate` is parked at a scenario end with the screen black and `WLDCORE.BIN` not resident anywhere in RAM, and `world_map_ss1_settled_dialog_closed.sstate` is 27.9 s later with it resident — the uncaptured `0x80` fork at selector `0x801C3740` lies strictly between the two states.** — `[D] 1/3`
  - D: PCSX-Redux savestate pair `reference-assets/world_map_ss0_scenario_end_prequicksave.sstate` + `world_map_ss1_settled_dialog_closed.sstate` (2026-08-21)
  - ⚠ SUPERSEDED (2026-08-21) by: the `0x80` fork is captured frame-by-frame — the full `Scenario → World map` entry load runs from `WLDCORE.BIN` resident at vsync 61 to first paint at vsync 645 (~11 s of game time); the bracket claim itself still stands
  - R: none — world map / `WLDCORE` not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`

- **`WLDCORE.BIN` loads flat at RAM `0x80067000` (confirmed by the file's own baked absolute pointers), and the world-map render is solved end to end: the map is a flat 1:1 scrolled 8 bpp bitmap stored in 240×240-texel slabs, the view aperture is a two-layer CLUT ramp over a distance field applied once per frame, and every HUD element's tpage/CLUT/uv is pinned.** — `[S·D] 2/3`
  - S: static check of the absolute pointers baked in the ROM file `WLDCORE.BIN`, pinning the `0x80067000` load base (reported in doc §4.3; byte-level detail in `research/working_documents/WORLD_MAP_SCREEN.md`)
  - D: dynamic measurement + offline RAM analysis on the world-map savestate pair (2026-08-21)
  - D: frame-simulation rebuild (`research/working_documents/world_map_captures/simulate_frame.py`): background + the 40 aperture quads, one application each, 31,456/32,680 (96.3%) pixel-exact against the `ss1` framebuffer (2026-08-21)
  - R: none — `WLDCORE` / world-map render not present in godot-learning (probed godot-learning/src, godot-learning/tests; only an unrelated BATTLE.BIN-base constant in `tests/CinematicPoseLUTTest.gd`)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`

- **`WORLD.BIN` is resident at `0x800E0000` the whole time the world map is on screen, and `WLDCORE` calls into it — the display init `FUN_80067D70` issues `func_0x800E10F4` / `func_0x800E17C4` / `func_0x800E1864` — so `WORLD.BIN` is the front-end/graphics overlay (ordering-table setup, `AddPrim`, `DrawOTag`, primitive-buffer allocator) for the whole world-map context, and the formation screen is one consumer of it, not the whole of it.** — `[S·D] 2/3`
  - S: `FUN_80067D70` @ `0x80067D70` decompilation (Ghidra import of `WLDCORE.BIN` @ `0x80067000`; `research/working_documents/world_map_captures/round3_decompiled.txt`)
  - D: RAM probe on `world_map_ss1_settled_dialog_closed.sstate` — 914,872/973,144 bytes (94.01%) identical at `0x800E0000`, resident range `0x800E0000..0x801CD958` (2026-08-21)
  - R: none — world-map overlay / `WORLD.BIN` graphics layer not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **`WLDCORE.BIN` and `BATTLE.BIN` share the load base `0x80067000` as mutually exclusive overlays — a battle has `BATTLE.BIN` there, the world map has `WLDCORE` — so addresses in `battle_disassembly.txt` above that line must not be cited for world-map code.** — `[S] 1/3`
  - S: `WLDCORE.BIN` base `0x80067000` (baked absolute pointers, file offset `0x000` table) + `fft-ghidra/README.md` "BATTLE.BIN covers 0x80067000+"
  - R: none — world-map overlay at `0x80067000` not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **World-map GPU primitives carry signed screen-centred coordinates (`x ∈ [−128,+128]`, `y ∈ [−120,+120]`) with a `(128,120)` drawing offset, not the `+128` x-convention the `WORLD.BIN` roster screen uses; the overlay's own copy of the `E5` offset reads `0x801CD508 = 0x00780080` (low half 128, high half 120).** — `[S·D] 2/3`
  - S: drawing-offset word `0x801CD508` = `0x00780080` (`WORLD.BIN` file offset `0xED508`, Ghidra export `world_disassembly.txt` / `WLDCORE` import)
  - D: war-funds label `SPRT` packet at screen `(48,84)` vs the rendered frame position in `ss1` — agrees to within a pixel (2026-08-21)
  - R: none — world-map coordinate model not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **Per-frame draw order is fixed: a full-screen black `FillRect` (GP0 `02`, 3-word tag at `0x80195C14`, mirrored at `0x80195C24` for the other buffer) → background chain (aux `+0x8`) → travel-path chain (aux `+0x4`) → main OT; MAIN is the 16-slot OT at `0x800D487C`/`0x800D48BC` and AUX the 4-slot OT at `0x800D4908`/`0x800D4918`, two render contexts set up by `FUN_80067D70` (structs at `0x800BB364`/`0x800BB3C4`, stride `0x14`) and built per frame by `FUN_800677A4`; within the main list the submission order is pins → Ramza → additive aperture → subtractive aperture → cursor → date / location name / war funds.** — `[S·D] 2/3`
  - S: `FUN_80067D70` init + `FUN_800677A4` frame builder decompilation (Ghidra `WLDCORE` import; `research/working_documents/world_map_captures/round3_decompiled.txt`)
  - D: ordering-table walk of `ss1` — 61 packets in submission order, fill tag and chain heads at `0x800A017C`/`0x8009FF9C` measured (`round1_ot_walk.txt`, 2026-08-21)
  - R: none — world-map render loop / OT layout not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **The map background is emitted as 81 quads in a single pass — a 9×9 mesh over the 17×13 vertex grid at `0x800C7320` (stride `0x18`, cull on `flags & 0x300`), drawer `FUN_8006a9d8` — and because the main primitive list is then written from the same bump-allocated pool base `0x8009F31C + buffer × 0xE000`, 56 of the 81 quads are overwritten, so any savestate holds only the 25 survivors.** — `[S·D] 2/3`
  - S: `FUN_8006a9d8` @ `0x8006A9D8` + grid table `0x800C7320`; pool-base formula from `FUN_800674E0`/`FUN_80068308` (Ghidra `WLDCORE` import; `round3_decompiled.txt`)
  - D: OT walk + pool addresses in `ss1`/`ss2` — `0x8009F31C + 81×40B = 0x8009FFC4` (first path quad), main list ends at `0x8009FBD8`, `0x8009FBD8 − 0x8009F31C = 56 × 40B` exactly (2026-08-21)
  - R: none — world-map background drawer not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **The visible map is a straight 1:1, unscaled, unrotated window — `u = fb_x`, `v = fb_y + 60`, anchored at VRAM `(768,60)` — and the view never scrolls: across 6,675 consecutive vsyncs including an 11 s held-LEFT, the sampled nodes' projected screen positions (`(−4,0)` and `(−60,−49)`) never moved; the cursor moves inside the fixed window and clamps at the screen edge.** — `[S·D] 2/3`
  - D: exact-colour search from `ss1` framebuffer pixels back to a unique VRAM source; round-6 input trace `round6_pulse_trace.csv`, 6,675 rows (2026-08-21)
  - S: grid table `0x800C7320` stores each vertex's map-space (`+0x08/+0x0C`) and post-scroll screen (`+0x10/+0x14`) positions — a pure translation (`round3_decompiled.txt`)
  - R: none — world-map window/scroll model not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **The map art is stored in 240×240-texel slabs (pitch 240, not 256, in both axes), each with a 16-texel gutter on the right and bottom edges — the gutter is what forces the mid-row tpage switches; slabs A/B sit at VRAM `(768,0)`/`(896,0)` (tpages `0x008C`/`0x008E`) and slab C at `(256,256)` (`0x0094`), all 8 bpp under CLUT `0x7800`.** — `[D] 1/3`
  - D: `ss1` VRAM renders — the gutter strip is visible between the two map panels at VRAM `x ≈ 888..896` (`map_slabs_A_B_vram768.png`, 2026-08-21)
  - R: none — map slab storage not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **The world-map aperture vignette is two layers of one 8 bpp distance field at VRAM `(448,256)`: 24 subtractive quads (`abr=2`, `B−F`, CLUT `0x7880` ramp, `v` band 8..128) centred on the screen, plus 16 additive quads (`abr=1`, `B+F`, CLUT `0x7940` ramp, `v` band 136..216) whose glow centre is off-screen-centre at `(−44,−32)`; the texel values are distances and the falloff shape lives entirely in the two 256-entry ramps, and all 40 quads are byte-identical across captures at different cursor positions — the vignette does not move with the cursor.** — `[D] 1/3`
  - D: 40-quad ordering-table walk + full 256-entry ramp capture (`round1_cluts.json`); 96.3% pixel-exact single-application frame-simulation; `ss1` vs `ss2` byte-identical aperture quads (2026-08-21)
  - R: none — aperture vignette not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **The location name is one 80×10 `POLY_FT4` quad from a pre-rasterised string atlas at VRAM `(384,256)` (tpage `0x0016`, 4 bpp, CLUT `0x7980`): all 43 location names in two columns at one per 10-texel row (row index = `v/10`), the 4×3 month grid at `v=200/210/220`, and a numeral row that spells `I` for the digit 1 (hence the date reads `Jan. I`).** — `[D] 1/3`
  - D: atlas render + transcription (`string_atlas_384_256.png`); confirmed on two captures with two names — `ss1` row 6 = Gariland Magic City, `ss2` row 24 = Mandalia Plains (2026-08-21)
  - R: none — string atlas / location-name quad not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **The current location is a 1-based node index in `DAT_800D0BB4` (atlas row = node−1, cel id = node+0x36), and the name quad is placed from that node's entry in the node table at `0x800D3CD0` (stride `0x34`: projected screen x/y, live pulsing colour, two chapter-tier fields, packed map-space position) — not from the cursor, which merely sits on the same node (label centre = `node.x + 4`, label top = `node.y + 1`).** — `[S·D] 2/3`
  - S: `FUN_8006C108` @ `0x8006C108` decompilation + cel part word `node.x + (−36), node.y + 1` (`research/working_documents/world_map_captures/round4_decompiled.txt`)
  - D: verified on both captures — `ss1` node 7 = Gariland, `ss2` node 25 = Mandalia Plains, atlas row and cel id exact in both (2026-08-21)
  - R: none — node table `0x800D3CD0` / `DAT_800D0BB4` not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **The HUD date and war funds are plain game values: month = game variable `0x2E`, day = game variable `0x2F` (cel ids `month + 0x1F` and `digit + 0x2C`, the day drawn as one or two 8 px cells), and war funds = `DAT_800D09AC` (4500 in both captures), drawn by `FUN_8006B78C` anchored at `DAT_8009F2B0/B4` = `(48,84)` as a right-aligned 8-digit `SPRT` field with placeholder `⓪` and value glyphs at different uvs.** — `[S·D] 2/3`
  - S: `FUN_8006BF9C` @ `0x8006BF9C` and `FUN_8006B78C` @ `0x8006B78C` decompilations (`round4_decompiled.txt`)
  - D: values read from `ss1`/`ss2` match the measured label/digit packets' screen positions, uvs and CLUTs in both captures (2026-08-21)
  - R: none — HUD date / war funds variables not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **Every world-map HUD element is drawn through one cel/animation system (drawer `FUN_8006AED0`): descriptor id → frame list at `0x80092B38` → cel list at `0x8009324C` → parts, where part word0 = `(x+0x80, y+0x80)` and word1 = `(h, w, v, u)` — word1.byte1 is the per-entry width of the variable-pitch atlas, and a frame duration of `0xFFFF` marks a static element; the cel data reproduces the measured packets' tpage, CLUT, uv, size and screen position exactly.** — `[S·D] 2/3`
  - S: `FUN_8006AED0` @ `0x8006AED0` decompilation + frame/cel list fields (`round4_decompiled.txt`)
  - D: cel 61 (location name, node 7) and cel `0x2C` (day digit) byte-exact against the live ordering-table packets in both captures (2026-08-21)
  - R: none — cel/animation system `0x80092B38`/`0x8009324C` not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **The dotted travel path is one of 48 routes, each gated on its own game variable `0x22C + n`, each a map-space polyline from a pointer table at `0x80095004`; drawer `FUN_8008D51C` transforms every vertex (`func_0x8001D3E8`), clips in screen space to `−127 ≤ x ≤ 127`, `−111 ≤ y ≤ 111`, and smears a single 16×16 dot texture (tpage `0x0017`, CLUT `0x7A01`) into degenerate quads along consecutive polyline segments.** — `[S·D] 2/3`
  - S: `FUN_8008D51C` @ `0x8008D51C` decompilation — its `GetTPage(0,0,0x1C0,0x100)` / `GetClut(0x10,0x1E8)` arguments match the measured `tp=0x0017` / `clut=0x7A01` exactly (`round3_decompiled.txt`)
  - D: the 12-quad path chain measured off `ss1`'s aux OT head `[+0x4]`, segments from the destination pin `(−66,−57)` down to Ramza `(−12,−16)` (2026-08-21)
  - R: none — travel-path routes / `FUN_8008D51C` not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **The two live map pins are 12×12 quads (uv `(200,0)`, tpage `0x001F`) distinguished by CLUT — `0x784A` for dim/unselected, `0x784B` for bright/current — and their per-frame brightness is a linear triangle wave on the node table's live colour field: grey steps exactly ±2 per frame between 96 and 224, period 128 frames (~2.13 s), with the two markers in exact antiphase so their greys always sum to 320.** — `[S·D] 2/3`
  - S: pin table loop `FUN_80069810` @ `0x80069810` (entries at `0x800D0AD0`) + node colour field `0x800D3CD0+0x08` (`round3_decompiled.txt`, `round4_decompiled.txt`)
  - D: 6,675-vsync colour trace (`round6_pulse_trace.csv`, 2026-08-21); round-1's three scattered frame pairs (144/176, 100/220, 142/178) all sum to 320
  - R: none — map pins / pin pulse not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **Ramza's idle marker ping-pongs the first three cels of an 8-cel strip (`u = 0 → 16 → 0 → 32`, 10 frames each — a 40-frame / 0.667 s cycle, cels 3–7 unaccounted for, presumably travel); the strip sits on row `v=240..255` of tpage `0x000E` at VRAM `(896,0)` at 16-texel pitch, and the 15×15 body quad is drawn one texel smaller than the 16-pitch cell.** — `[D] 1/3`
  - D: per-vsync `u` scan of the body quad (CLUT `0x78C0`) in the packet pool + the strip render (`ramza_walk_strip.png`, 2026-08-21)
  - R: none — Ramza world-map marker strip not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **The hand cursor is a 16×16 sprite (uv `(168,0)`, CLUT `0x7847`, tpage `0x001F` → VRAM `(960,256)`) drawn last of everything so it is never darkened by the aperture, with its drop shadow as a subtractive silhouette (tpage `abr=2`, uv `(184,0)`, CLUT `0x7844`) offset `(+2,+2)`; the cursor moves freely rather than node-snapping — `DAT_800D0BB4` goes to 0 whenever it is off a node and the location-name quad is simply not emitted.** — `[D] 1/3`
  - D: `ss1` cursor/shadow packets + round-6 cursor-driven capture at the far-left edge with no location name (`cursor_free_at_edge.png`, 2026-08-21)
  - R: none — hand cursor / `DAT_800D0BB4` zeroing not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **The whole `Scenario → World map` (`post_scenario_step = 0x80`) entry load is ~11 s of game time in six measured phases: `WLDCORE.BIN` resident at vsync 61, `WORLD.BIN` at 170, the "Now loading" text (lower-right, ~304 halfwords) up from v≈120 to v≈630, `WLDTEX.TM2` streamed from CD into VRAM over v380–v445 with coverage climbing 8% → 100%, the HUD mode live at v596, and the first paint — map, aperture, date, war funds and the "Check" tutorial box all at once — a single frame at v645; the fork long filed as never captured is now captured frame-by-frame.** — `[D] 1/3`
  - D: round-5 `GPU::Vsync` trace (`round5_load_trace.csv`, 2,864 rows) + full-savestate beats at v62/v120/v171/v385–v445/v645/v655, resumed from `world_map_ss0_scenario_end_prequicksave.sstate` (end of scenario 12, `0x8016A014` = 12) (2026-08-21)
  - R: none — world-map load sequence not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **`WLDTEX.TM2` is not a TIM2 despite its extension: it is a sector-packed stream of PSX `LoadImage` blocks (per 2048-byte CD sector: `u32` block count, then `u16 x,y,w,h` destination VRAM rect + raw row-major 16-bit VRAM pixels; blocks never straddle sectors), 134 sectors / 386 blocks / 274,432 bytes; replaying all 386 blocks into a blank 1024×512 VRAM is 100.0000% bit-exact (126,416/126,416 halfwords) against `ss1` and carries every world-map-specific region — map slabs, string atlas, aperture field, Ramza strip and the CLUT strip at `(0,480)` — while the shared UI sheet at `(960,256)` is not in it.** — `[D] 1/3`
  - D: `research/working_documents/world_map_captures/wldtex_replay.py --verify` against `world_map_ss1_settled_dialog_closed.sstate` (2026-08-21)
  - R: none — `WLDTEX.TM2` parser not present in godot-learning (doc §18.4 notes it deliberately stays in `research/` as an RE record, not wired into `godot-learning/tools/bootstrap_assets.sh` per ADR-0001)
  - src: `research/working_documents/WORLD_MAP_SCREEN.md`

- **The world map's dispatch logic is a per-node event table inside `WLDCORE.BIN` at `0x80097234`, run by a 41-opcode bytecode interpreter (`FUN_80091238`): 43 script lists, 182 scripts, each a run of conditions followed by one typed emit. The main-story condition is `var[110] == n` — `var[110]` is the story counter — and the 55 type-8 emits read back in `var[110]` order are the whole main story, one node per value (Gariland 1, Mandalia Plains 2, Igros Castle 3, Sweegy Woods 4, Dorter 5, Zeklaus Desert 6, … Murond Holy Place 52); nine more emits are gated on side flags and the party roster instead (Goug, Nelveska, Zarghidas …); an emit's two operands become `var[0x27]` (the scenario id) and `FUN_8008047C`'s transition mode — so "which run unlocks next" is `var[110]` plus whichever node the marker stands on.** — `[S] 1/3`
  - S: round-12 (2026-08-21) static read of the WLDCORE import — `FUN_80091238`, event table `0x80097234`, `FUN_8008047C`; byte-level detail in `research/working_documents/WORLD_MAP_SCREEN.md` §29
  - R: none — world-map dispatcher / event table not present in godot-learning (probed `src/`, `tests/` for `WLDCORE`/`0x80097234`/`world_map`)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`

- **The `World map → Scenario` trigger is `FUN_8008E2BC`, reached from `FUN_8006C894`'s ○ branch: it queries the per-node event table about the node the *marker* stands on, and when a live type-8 script is there it sets `var[0x27]` from the emit's first operand, calls `FUN_8008047C(second)`, and ORs `0x2000` into `0x8004D950` — which is why the game forces you into the pending story battle before it will consider a path; otherwise it runs the pathfinder and the marker walks.** — `[S·D] 2/3`
  - S: round-12 (2026-08-21) decompilation of `FUN_8008E2BC` / `FUN_8006C894` (WLDCORE import; `research/working_documents/WORLD_MAP_SCREEN.md` §29.3, §29.6)
  - D: round-12 live traversal watched on the emulator — the marker walk matched the static prediction at all twelve waypoints (2026-08-21)
  - R: none — world-map trigger / `0x8004D950` dispatch bit not present in godot-learning (probed `src/`, `tests/`)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`

- **World-map travel is a 48-entry route graph with 12-bit bearings — `FUN_8008F434` maps a bearing to its frame list, and route/node progression lives in 92 contiguous bits (node `i` ⇔ bit `512+i`, route `r` ⇔ bit `556+r`); the model is now double-sourced, with the per-node event table's reveal-emit guards and operands agreeing with it, and `travel.py json` exports the graph, the 48-route waypoint/polyline tables, the game-variable store's layout and named indices, the facing rules and all 182 event scripts as one blob whose check row replays a real traversal out of that blob alone.** — `[S·D] 2/3`
  - S: round-12 (2026-08-21) static — `FUN_8008F434` + the 92-bit progression layout, cross-checked against the type-8 reveal emits in the `0x80097234` event table
  - D: round-12 traversal replay — `world_map_captures/travel.py json` check row replays a real traversal from the exported blob (2026-08-21)
  - R: none — travel model / route graph not present in godot-learning (probed `src/`, `tests/`, `tools/`)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`

- **The world-map main loop branches on `*(0x8004D950) & 2`: when the bit is clear it runs `FUN_80067A78`, one table of 61 primitives, with the background DMA'd by `GsLoadImage` from a 256×238 cache at `0x801D0000`.** — `[S] 1/3`
  - S: round-12 (2026-08-21) decompilation of `FUN_80067A78` (WLDCORE import; `research/working_documents/WORLD_MAP_SCREEN.md` §29)
  - R: none — world-map frame path not present in godot-learning (probed `src/`, `tests/`)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`

## Notes

(empty — user territory)

## Related

- [[Scenario Transition Graph]]
- [[Scenario Table]]
- [[Inter Scene Orchestration]]
- [[CDROM DMA Load Pipeline]]
- [[Formation Screen Compositing]]
- [[Ordering Table & AddPrim]]
