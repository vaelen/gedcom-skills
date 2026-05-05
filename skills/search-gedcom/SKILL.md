---
name: search-gedcom
description: Use when the user asks to find, search, look up, query, list-matching, locate, or filter records in a GEDCOM file (`.ged` or `.gdz`); when the user asks about specific people, families, dates, or places ("find everyone named Smith", "who was born between 1900 and 1910", "people who died in Boston"); or when the user asks about ancestors, descendants, parents, children, or any genealogy traversal ("ancestors of @I1@", "descendants of John Smith", "show the children of"). Handles GEDCOM 5.5.1, 5.5.5, and FamilySearch GEDCOM 7.0+.
version: 0.1.0
---

# search-gedcom

## Overview

Query a GEDCOM file. Three search modes — pick one per invocation:

1. **Generic structure filters** — `--tag`, `--value`, `--path`, `--regex`, `--in`, `--xref`. Combinable. Returns matched structures with their record xref and file line number.
2. **Person / date / place** — `--person`, `--born-between`, `--died-in`. Returns matched `INDI` records.
3. **Relationship traversal** — `--children-of`, `--parents-of`, `--ancestors-of`, `--descendants-of` (with `--depth`). Returns matched `INDI` records.

Output modifiers (`--json`, `--show-record`, `--count`, `--limit`) work in any mode.

## When to use

- "Who was born in 1850?" → `--born-between 1850 1850`
- "Show everyone named Smith" → `--person Smith`
- "Find all the places mentioned" → `--tag PLAC`
- "What are the children of @I1@?" → `--children-of @I1@`
- "Show me the ancestors of John Smith" → name-lookup first, then `--ancestors-of <xref>`
- "Find every birth date in the 1800s" → `--path INDI/BIRT/DATE --regex --value '\\b18\\d\\d\\b'`

**Do not use** when:

- The user wants an overview of the file → use **read-gedcom**.
- The user wants to add, remove, or change data → use **update-gedcom**.

## How to invoke

This skill is a thin wrapper around the `gedcom-search` console script published by [gedcom-lite](https://github.com/vaelen/gedcom-lite).

```bash
# After publishing to PyPI:
uvx --from gedcom-lite gedcom-search FILE [filters] [output options]

# Pre-publish, from the git repo:
uvx --from "git+https://github.com/vaelen/gedcom-lite" gedcom-search FILE [filters] [output options]
```

If `gedcom-lite` is already installed, invoke `gedcom-search` directly.

## Quick reference

**Generic filters** (combinable):

```
--xref @I1@                lookup a single record by id
--tag NAME                 structures with this tag
--value Smith              substring match on payload
--regex                    interpret --value as a regex
--in INDI                  restrict to within INDI records
--path INDI/BIRT/DATE      tree-path query
```

**Person / date / place** (one at a time):

```
--person "John Smith"      INDI whose NAME contains this string
--born-between 1900 1910   INDI with parseable BIRT date in this year range
--died-in Boston           INDI whose DEAT.PLAC contains this place
```

**Relationships** (one at a time, plus optional `--depth`):

```
--children-of @I1@
--parents-of @I1@
--ancestors-of @I1@
--descendants-of @I1@
--depth N                  cap traversal depth
```

**Output**:

```
--json                     emit JSON
--show-record              include surrounding record dumps
--count                    emit only the number of matches
--limit N                  cap matches emitted
```

## Examples

```
# Every NAME in the file with line numbers
gedcom-search tree.ged --tag NAME

# Names containing "Allen", limited to INDI records
gedcom-search tree.ged --tag NAME --value Allen --in INDI

# Births in the 19th century
gedcom-search tree.ged --path INDI/BIRT/DATE --regex --value '\\b18\\d\\d\\b'

# Find John Smith's xref, then their ancestors
gedcom-search tree.ged --person "John Smith"
gedcom-search tree.ged --ancestors-of @I42@ --depth 3

# Anyone born between 1900 and 1910
gedcom-search tree.ged --born-between 1900 1910

# Count every DATE structure
gedcom-search tree.ged --tag DATE --count
```

## Output contract

- Generic structure search: one match per line, `XREF PATH "PAYLOAD":LINE`. The `PATH` is the slash-delimited tag path from the record root.
- Person/relationship search: one INDI per line, formatted `XREF  Name  (birth – death)`.
- `--json` emits a JSON array. Generic matches use `{xref, record_tag, tag, path, payload, line}`; record matches use the canonical `{level, tag, xref, payload, children: [...]}` shape.
- Exit codes: 0 on success, 1 on missing record (e.g. `--xref` not found), 2 on bad CLI arguments.

## Reference

- `docs/format.md` — line grammar and structure paths
- `docs/tags.md` — common tags by record type, including how dates and names are structured
- `docs/parsing-notes.md` — date-format coverage and limitations
