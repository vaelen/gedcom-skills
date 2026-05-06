---
name: search-gedcom
description: Use when the user asks to find, search, look up, query, list-matching, locate, or filter records in a GEDCOM file (`.ged` or `.gdz`); when the user asks about specific people, families, dates, or places ("find everyone named Smith", "who was born between 1900 and 1910", "people who died in Boston"); or when the user asks about ancestors, descendants, parents, children, siblings, cousins, ahnentafel, or Sosa numbers ("ancestors of @I1@", "first cousins of John Smith", "siblings of @I7@", "ahnentafel for @I1@", "anyone with conflicting parents"). Handles GEDCOM 5.5.1, 5.5.5, and FamilySearch GEDCOM 7.0+.
version: 0.2.0
---

# search-gedcom

## Overview

Query a GEDCOM file. Three search modes:

1. **Generic structure filters** — `--tag`, `--value`, `--path`, `--regex`, `--in`, `--xref`. Combinable. Returns matched structures with their record xref and file line number.
2. **Person / date / place** — `--person`, `--born-between`, `--died-between`, `--born-in`, `--died-in`. Combinable; AND together. Returns matched `INDI` records.
3. **Relationship traversal** — `--children-of`, `--parents-of`, `--siblings-of`, `--ancestors-of`, `--descendants-of`, `--cousins-of`, `--ahnentafel` (with `--depth`). Returns matched `INDI` records. Person/date/place filters compose with traversal as a *post-filter* on the result.

FAMC handling: `--primary-famc-only` (follow only the first FAMC during traversal) and `--famc-conflicts` (list INDIs with multiple FAMC entries).

Output modifiers: `--json`, `--facts`, `--show-record`, `--count`, `--limit`. `--json` / `--facts` / `--show-record` are mutually exclusive.

## When to use

- "Who was born in 1850?" → `--born-between 1850 1850`
- "Show everyone named Smith" → `--person Smith`
- "Find all the places mentioned" → `--tag PLAC`
- "What are the children of @I1@?" → `--children-of @I1@`
- "Who are the siblings of @I1@?" → `--siblings-of @I1@`
- "Show me the ancestors of John Smith" → name-lookup first, then `--ancestors-of <xref>`
- "Get the ahnentafel (Sosa list) for @I1@" → `--ahnentafel @I1@`
- "Find first cousins of @I1@" → `--cousins-of @I1@ --depth 1`
- "Pull birth/death/parents for everyone matching X as JSON" → `--facts`
- "Anyone with conflicting parents in the tree?" → `--famc-conflicts`
- "Born outside Boston, died in Boston before 1776" → combine `--born-in`, `--died-in`, `--died-between MIN 1776`
- "Find every birth date in the 1800s" → `--path INDI/BIRT/DATE --regex --value '\\b18\\d\\d\\b'`

**Do not use** when:

- The user wants an overview of the file → use **read-gedcom**.
- The user wants to add, remove, or change data → use **update-gedcom**.

## How to invoke

This skill is a thin wrapper around the `gedcom-search` console script published by [gedcom-lite](https://github.com/vaelen/gedcom-lite).

```bash
uvx --from gedcom-lite gedcom-search FILE [filters] [output options]
```

To run unreleased changes from git instead of the PyPI release:

```bash
uvx --from "git+https://github.com/vaelen/gedcom-lite" gedcom-search FILE [filters] [output options]
```

If `gedcom-lite` is already installed, invoke `gedcom-search` directly.

## Quick reference

**Generic filters** (combinable):

```
--xref @I1@ [@I2@ ...]     lookup one or more records by id (bulk; missing ids warn on stderr)
--tag NAME                 structures with this tag
--value Smith              substring match on payload
--regex                    interpret --value as a regex
--in INDI                  restrict to within INDI records
--path INDI/BIRT/DATE      tree-path query
```

**Person / date / place** (combinable; AND together):

```
--person "John Smith"      INDI whose NAME contains this string
--born-between LO HI       INDI with parseable BIRT date in [LO, HI]
--died-between LO HI       INDI with parseable DEAT date in [LO, HI]
--born-in PLACE            INDI whose BIRT.PLAC contains this string
--died-in PLACE            INDI whose DEAT.PLAC contains this string
```

`LO` and `HI` may be `MIN` or `MAX` for an unbounded side, e.g. `--died-between MIN 1776`.

Place filters are substring matches against the raw place string. There is no country-level normalization yet, so `--born-in USA` will not match `Boston, Suffolk, Massachusetts, United States of America`. Match on a discriminating substring, or normalize before importing.

**Relationships** (one at a time, plus optional `--depth`):

```
--children-of @I1@
--parents-of @I1@
--siblings-of @I1@         full and half siblings
--ancestors-of @I1@
--descendants-of @I1@
--cousins-of @I1@          collateral relatives; --depth caps cousin degree
--ahnentafel @I1@          Sosa-numbered ancestor list
--depth N                  cap traversal depth (ancestors/descendants/cousins)
```

Person/date/place filters compose as a post-filter on the traversal result, preserving `generation` / `sosa` / `degree` / `removed` metadata.

**FAMC handling**:

```
--primary-famc-only        follow only the first FAMC of any individual
--famc-conflicts           list INDIs with multiple FAMC entries
```

**Output**:

```
--json                     emit JSON (canonical record shape)
--facts                    per-INDI canonical facts JSON (xref, name, birth, death, parents, famc)
--show-record              include surrounding record dumps
--count                    emit only the number of matches
--limit N                  cap matches emitted
```

`--json` / `--facts` / `--show-record` are mutually exclusive.

## Examples

```bash
# Every NAME in the file with line numbers
gedcom-search tree.ged --tag NAME

# Names containing "Allen", limited to INDI records
gedcom-search tree.ged --tag NAME --value Allen --in INDI

# Births in the 19th century
gedcom-search tree.ged --path INDI/BIRT/DATE --regex --value '\\b18\\d\\d\\b'

# Find John Smith's xref, then their ancestors with structured facts
gedcom-search tree.ged --person "John Smith"
gedcom-search tree.ged --ancestors-of @I42@ --facts --depth 4

# Sosa-numbered ahnentafel, following only the primary FAMC
gedcom-search tree.ged --ahnentafel @I42@ --primary-famc-only

# First cousins of @I42@
gedcom-search tree.ged --cousins-of @I42@ --depth 1 --facts

# Siblings (full and half)
gedcom-search tree.ged --siblings-of @I42@ --facts

# Anyone in the tree with conflicting parents
gedcom-search tree.ged --famc-conflicts --json

# Combined filters: died in Boston before 1776
gedcom-search tree.ged --died-in Boston --died-between MIN 1776 --facts

# Combined filters as a post-filter on traversal
gedcom-search tree.ged --ancestors-of @I42@ --born-in Germany --facts

# Bulk fetch
gedcom-search tree.ged --xref @I1@ @I2@ @I3@ --facts

# Anyone born between 1900 and 1910
gedcom-search tree.ged --born-between 1900 1910

# Count every DATE structure
gedcom-search tree.ged --tag DATE --count
```

## Output contract

- **Generic structure search**: one match per line, `XREF PATH "PAYLOAD":LINE`. The `PATH` is the slash-delimited tag path from the record root.
- **Person/relationship search (text)**: one INDI per line, formatted `XREF  Name  (birth – death)`. With `--ancestors-of` / `--descendants-of`, suffixed with `(gen N)`. With `--ahnentafel`, prefixed by Sosa number.
- **`--json`**: a JSON array. Generic matches use `{xref, record_tag, tag, path, payload, line}`; record matches use the canonical `{level, tag, xref, payload, children: [...]}` shape.
- **`--facts`**: a JSON array of `{xref, name, birth: {date, place}, death: {date, place}, parents: [xref...], famc: [xref...]}`. Composable with `--xref`, `--person`, `--ancestors-of`, `--descendants-of`, `--ahnentafel`, `--cousins-of`, `--siblings-of`, `--famc-conflicts`. Conflicts are detectable as `len(famc) > 1`.
- **Generation field on `--ancestors-of` / `--descendants-of`** (`--json` / `--facts`): top-level `generation` integer. Subject = 0; shortest path wins for individuals reachable via multiple FAMC paths.
- **Sosa field on `--ahnentafel`** (`--json` / `--facts`): top-level `sosa` integer. Subject = 1; father of N = 2N; mother = 2N+1. Without `--primary-famc-only`, the same individual is emitted once per Sosa-reachable path.
- **Degree / removed on `--cousins-of`** (`--json` / `--facts` / text): `degree` (1 = first cousin, 2 = second, …) and `removed` (absolute generational offset from the most-recent common ancestor). A cousin reachable via multiple branches is emitted once at the closest relation.
- **Exit codes**: `0` on success, `1` on missing record (e.g. `--xref` not found — for bulk `--xref`, only when *every* id is missing), `2` on bad CLI arguments.

## Reference

- `docs/format.md` — line grammar and structure paths
- `docs/tags.md` — common tags by record type, including how dates and names are structured
- `docs/parsing-notes.md` — date-format coverage and limitations
