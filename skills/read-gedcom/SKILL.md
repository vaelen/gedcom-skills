---
name: read-gedcom
description: Use when the user asks to read, open, summarize, inspect, view, parse, or describe a GEDCOM file (`.ged` or `.gdz`), or asks "what's in this family tree", "show me this genealogy file", "list everyone in this GEDCOM", "show record @I1@", "what version is this GEDCOM", or wants a JSON dump of GEDCOM contents. Handles GEDCOM 5.5.1, 5.5.5, and FamilySearch GEDCOM 7.0+; preserves encoding, BOM, and extension tags.
version: 0.1.0
---

# read-gedcom

## Overview

Parse a GEDCOM file and report on its contents. Three modes:

- **Summary** — version, encoding, originating software, declared extensions, and a per-record-type count. The default mode.
- **List** — enumerate all records of a given type (`INDI`, `FAM`, `SOUR`, …) one per line.
- **Record** — show one record by xref as a level-indented outline with pointer payloads annotated by the record they target.

Add `--json` to any mode for a structured dump.

## When to use

- "What's in `tree.ged`?" → summary
- "List the people in this GEDCOM" → `list INDI`
- "Show me record @I42@" → `record @I42@`
- "Dump this file as JSON" → `--json`

**Do not use** when:

- The user wants to find records matching some criterion → use **search-gedcom**.
- The user wants to change the file → use **update-gedcom**.

## How to invoke

This skill is a thin wrapper around the `gedcom-read` console script published by [gedcom-lite](https://github.com/vaelen/gedcom-lite).

The recommended invocation uses `uvx`, which installs the package into a cached ephemeral environment on first use — no global pip install required:

```bash
uvx --from gedcom-lite gedcom-read FILE [args]
```

To run unreleased changes from git instead of the PyPI release:

```bash
uvx --from "git+https://github.com/vaelen/gedcom-lite" gedcom-read FILE [args]
```

If `gedcom-lite` is already installed system-wide (`pip install gedcom-lite` or `uv tool install gedcom-lite`), invoke `gedcom-read` directly.

## Quick reference

```
gedcom-read FILE                       # summary (default)
gedcom-read FILE list TAG              # enumerate records of a type
gedcom-read FILE record XREF           # show one record, pointers resolved
gedcom-read FILE record XREF --depth N # limit nesting depth
gedcom-read FILE --json                # dump full document as JSON
gedcom-read - < FILE                   # read from stdin
```

`TAG` is `INDI`, `FAM`, `SOUR`, `REPO`, `OBJE`, `SUBM`, `NOTE`, `SNOTE`, etc.
`XREF` is the cross-reference id including `@…@`, e.g. `@I1@`.

## Examples

```
$ uvx --from gedcom-lite gedcom-read tree.ged
GEDCOM version: 7.0
Encoding: utf-8-sig (with BOM)
Line ending: LF
Records:
  FAM    47
  INDI   142
  …

$ uvx --from gedcom-lite gedcom-read tree.ged list INDI
@I1@  John /Smith/  (1 JAN 1900 – 5 MAR 1965)
@I2@  Jane /Smith/  (1902 – ?)

$ uvx --from gedcom-lite gedcom-read tree.ged record @I1@
@I1@ INDI
  NAME John /Smith/
  SEX M
  BIRT
    DATE 1 JAN 1900

$ uvx --from gedcom-lite gedcom-read tree.ged --json | jq '.records | length'
142
```

## Output contract

- Human output is plain text: one record per line for `list`, indented tree for `record`.
- JSON output uses the canonical document shape: `{level, tag, xref?, payload?, children?: [...]}`. The full-document form also includes `version`, `encoding`, `bom`, `line_terminator`, `declared_char`, and `warnings`.
- Exit codes: 0 on success, 1 on missing record / parse error, 2 on bad CLI args.

## Reference

For GEDCOM semantics see the docs at the repo root:

- `docs/format.md` — line grammar, levels, xrefs, CONT/CONC, encoding
- `docs/tags.md` — common tags by record type
- `docs/versions.md` — 5.5.1 vs 5.5.5 vs 7.0 differences
