# CLAUDE.md — Working agreements for this repo

This repo is a Claude Code plugin that ships three skills for working with GEDCOM genealogy files: **`read-gedcom`**, **`search-gedcom`**, **`update-gedcom`**. The actual parser, writer, and CLI tools live in a separate Python package, [`gedcom-lite`](https://pypi.org/project/gedcom-lite/) (also on [GitHub](https://github.com/vaelen/gedcom-lite)). Skills here are thin SKILL.md wrappers that tell Claude when to trigger and which `gedcom-{read,search,update}` console script to invoke.

Read this file before doing any work in the repo.

## Repo shape

```
.
├── README.md
├── CLAUDE.md
├── .claude-plugin/
│   └── plugin.json        # plugin manifest
├── skills/
│   ├── read-gedcom/SKILL.md
│   ├── search-gedcom/SKILL.md
│   └── update-gedcom/SKILL.md
├── docs/                  # GEDCOM-domain reference, used by all three skills
└── examples/              # small demo set (full fixture suite lives in gedcom-lite)
```

There is **no Python code** in this repo. The library lives in `gedcom-lite`. If you find yourself wanting to write parsing code here, stop — fix or extend `gedcom-lite` instead, and let the SKILL.md continue to wrap its CLI.

## How the skills invoke the CLI

The canonical, dependency-free invocation is via `uvx`:

```bash
uvx --from gedcom-lite gedcom-read FILE
uvx --from gedcom-lite gedcom-search FILE [filters]
uvx --from gedcom-lite gedcom-update FILE [-o OUT | --in-place] SUBCOMMAND ...
```

`uvx` (shipped with `uv`) caches an ephemeral environment with `gedcom-lite` installed. No prior `pip install` is required.

The SKILL.md files also document a fallback form for running unreleased changes directly from git — this should remain a *secondary* option, never the primary path:

```bash
uvx --from "git+https://github.com/vaelen/gedcom-lite" gedcom-read FILE
```

If `gedcom-lite` is already installed system-wide (`pip install gedcom-lite`, `uv tool install gedcom-lite`), the bare `gedcom-read` / `gedcom-search` / `gedcom-update` commands work directly.

## SKILL.md authoring rules

- Skill names are lower-kebab-case, verb-first: `read-gedcom`, `search-gedcom`, `update-gedcom`.
- The frontmatter `description` controls triggering. Follow the conventions in `superpowers:writing-skills`:
  - third-person, "Use when…" framing
  - list specific trigger phrases users actually say
  - keywords across all skills: `GEDCOM`, `.ged`, `.gdz`, `family tree`, `genealogy`, `INDI`, `FAM`
  - `search-gedcom` additionally lists `ancestor`, `ancestors`, `descendant`, `descendants`
  - **never** summarize the skill's workflow in the description
- The body should explain when to use vs. not use, show the CLI shape, give worked examples, and link to the GEDCOM-domain docs in `docs/`.
- Keep skill bodies short. The user can always run `gedcom-{tool} --help` for full flag detail.

## GEDCOM correctness expectations

These are not enforced here (the package is) but they shape what the skills can and cannot promise to a user:

- **Round-trip fidelity.** `gedcom-update` only changes the lines targeted by an operation; everything else emits byte-identical to the source. BOM, line endings, sibling order, `CONC`/`CONT` choice, and unknown extension tags are all preserved.
- **Encoding coverage.** UTF-8 (with or without BOM), UTF-16 LE/BE (with BOM), and ANSEL (5.5.1 legacy) are detected and round-tripped.
- **Safety on update.** Default writes go to stdout (or `-o FILE`); `--in-place` is the only mode that overwrites the input.
- **No silent version promotion.** Reading a 5.5.1 file and writing it back yields a 5.5.1 file. There is no implicit conversion.

If a user asks for behavior these guarantees don't cover (e.g., 5.5.1→7.0 conversion, validation, GEDZIP packaging), say so explicitly — those are out of scope for this plugin and would belong in either a new skill plus a `gedcom-lite` extension, or a new package entirely.

## When something needs to change

- **Parser/CLI bug or missing feature** → file an issue / open a PR in `gedcom-lite`. Don't try to patch around it in a SKILL.md.
- **Wrong wording in a SKILL.md** that misroutes user intent → edit the description's trigger phrases.
- **New documentation** about GEDCOM itself → `docs/`. Each SKILL.md links there rather than duplicating spec material.
- **Plugin version bump** → update **both** `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` in the same commit. The `version` field in `marketplace.json`'s `plugins[0]` entry must always match `plugin.json`. The marketplace registry reads its own copy, so a version change in only one file ships a stale value to users.

## House style

- Match the prevailing tone in this file: short, declarative, prescriptive when it matters.
- No emojis in code or docs unless the user explicitly asks.
- Default to no comments in any auxiliary script; add one only when the *why* is non-obvious.
