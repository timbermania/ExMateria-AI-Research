# AGENTS.md

This repo is **not a code project**. It is an Obsidian vault of distilled,
evidence-graded knowledge notes about **Final Fantasy Tactics (PSX)** —
the knowledge base for the fft-project reverse-engineering and Godot
reimplementation effort. 70+ flat, Title Case markdown notes at the vault
root. No folders; organization is via `[[wikilinks]]` and index notes.

## Where the notes come from (read this first)

Topic notes are **machine-generated and merged** by the `vault-sync`
pipeline in the companion monorepo:

- Runner: `~/Repos/fft-monorepo/vault-sync/` (`vault_sync.py`, Python
  stdlib-only; one headless `pi` parse per research doc, local model via
  the vLLM endpoint).
- Spec (source of truth): `vault-sync/PARSE_PROMPT.md` — the exact prompt
  given to every parse agent, including the point format, conflict rules,
  and the result-file contract below.
- Source corpus: `research/**/*.md` in the monorepo (`key_documents/`,
  `wiki_articles/`, `working_documents/`, `effect_sound/`, …). The
  `src:` lines inside notes are **monorepo-relative paths**, not vault
  paths.

Consequences for editing:

- **Merge, don't rewrite.** A re-parse of a source doc will merge into the
  same topic note. Search the vault for an existing note covering a topic
  before creating one — never re-create or restructure an existing note.
- **Human edits to point bullets are sacred** — later merges keep them
  verbatim (human > machine). You may edit point bullets by hand; the
  pipeline will not clobber them.
- **`## Notes` is user territory**: never write in it, never reformat it.
  It doubles as a **command channel** — instructions left there are acted
  on by the parse agent the next time a doc feeding that note is
  re-parsed.
- Keep diffs minimal; preserve the exact point-bullet shape (it is
  greppable for the merge logic).
- To force a re-parse: modify the source doc in the monorepo (content
  change), or run `python3 vault-sync/vault_sync.py --doc PATH`
  (repo root). `--reset [DOC]` recovers a state left by a killed run.
- Sync state lives in the **monorepo** (`vault-sync/inventory.json`,
  `vault-sync/vault-sync.log`), not in this vault.

## Note format (exact)

```markdown
# <Topic Title>

<one-paragraph summary of the topic's current state of knowledge>

## Points

- **<Claim in one sentence>** — `[S·D·R] 2/3`
  - S: <ROM address/symbol + file>
  - D: <capture/probe identifier + date>
  - R: <code path + validating test>
    OR
    R: none — <term> not present in godot-learning
  - src: `research/<path of origin doc>.md`

## Notes

(empty — user territory)

## Related

- [[Other Topic Note]]
```

Score-badge rules:

- Badge letters = support types present, joined by `·`; number = count of
  distinct types (0–3). `S` = static analysis, `D` = dynamic analysis,
  `R` = reimplementation. No evidence → `[ ] 0/3`.
- A type earns credit only with locatable evidence: `S` → ROM
  address/symbol (never line numbers) + which file; `D` →
  capture/dump/probe identifier + date; `R` → code path in
  `godot-learning`, `smd-player`, `fft-sound-driver`, or `effect-editor`
  + the validating test.
- Omit the `S:`/`D:` sub-bullet only when that type is absent. The `R:`
  sub-bullet is **always present** — either the implementation evidence
  or an explicit `R: none — …` line from probing; absence of evidence
  earns no credit, but probing for it is part of the parse.
- **Negative results are not stored** (rejected hypotheses produce no
  points). Superseded points are kept with a
  `⚠ SUPERSEDED (<date>) by: <new claim>` sub-bullet. Unresolvable
  contradictions get `CONTESTED` appended to both badges and are listed
  in a `Needs Review` index note.
- Conflict precedence: explicit correction > higher score > newer doc
  date (doc date = git commit date in the monorepo, not mtime) >
  CONTESTED.

## Index structure

- `AI Research Index.md` — root index. Links **domain index notes only**
  under `## Topics` (topic notes are reached through domain indexes, not
  the root index).
- Domain indexes (list of `[[wikilinks]]` + one-line scope description):
  `Ability Execution Index`, `Effect System Index`, `Event VM Index`,
  `Scenario Index`, `SFX Index`, `Unit Deployment Index`,
  `Unit Sprite Index`.
- When adding a new topic note: link it from the relevant domain index
  (create the index if the domain has none). New *index* notes
  additionally go in the root index.

## Companion repo (where evidence resolves)

Canonical monorepo: `~/Repos/fft-monorepo` (sibling git worktrees exist
for parallel work; the vault-sync tooling lives at the repo root, shared
by all of them). Key locations (see its `CLAUDE.md` for the full map):

| What | Where |
|---|---|
| Monorepo conventions | `~/Repos/fft-monorepo/CLAUDE.md` (router), `research/CLAUDE.md` |
| Godot reimplementation | `godot-learning/` (Godot 4.6) |
| Authoritative RE docs | `research/key_documents/` (e.g. `STRUCTURE_DEFINITIONS.md`) |
| Disassembly | `project-assets/fft-rom/{scus,battle}_disassembly.txt` — **gitignored, local-only**; `project-assets/` holds all ROM-derived content |
| Ghidra label exports | `fft-ghidra/` (source of truth for the label set) |

Domain vocabulary in notes: PSX RAM addresses (`0x800xxxxx`),
`BATTLE.BIN` / `SCUS.BIN` disassembly, `E###.BIN` per-scene effect files,
`EVTCHR.BIN` (unit animation scripts), `ENTD` unit deployment tables,
SPU/GTE/PSX GPU primitives, "chapel" = the Orbonne chapel cinematic
(scenario 1).

## Useful commands

No build/test/lint — this is a notes vault.

```bash
# Find notes by name / content
ls *.md | grep -i keyword
grep -rl "keyword" --include="*.md" .

# Backlinks to a note
grep -rl '\[\[Note Title\]\]' .

# Sync pipeline state (from the monorepo root)
tail vault-sync/vault-sync.log
python3 vault-sync/vault_sync.py --once --dry-run          # what would run
python3 vault-sync/vault_sync.py --once --max 1 --doc PATH # watch one parse live
```

Git: branch `main`, loose commit messages ("update").

## Gotchas

- Don't look for sync state, inventories, or source docs inside the
  vault — they live in the monorepo.
- Don't "clean up" point bullets or section orderings: the shape is a
  machine contract, and reformatting fights the next merge.
- A `## Notes` section that is non-empty is the user talking to the parse
  agent — treat its contents as instructions to be honored, not text to
  be edited.
- Notes are flat and Title Case; wikilinks must match filenames exactly
  (e.g. `Ordering Table & AddPrim`, `Color Screen Opcode`).
- Point bullets cite ROM addresses/symbols — `0x800xxxxx`-style prose is
  expected content, not a leak.
