---
name: ancestor-report
description: >
  Use when the user asks to generate, build, or write a genealogy report,
  ancestor report, ancestry report, or family tree report from a GEDCOM file —
  particularly one organized by generation with categorized immigrant tables
  (e.g., "report all my ancestors back N generations", "make a markdown report
  of [person]'s ancestry", "list immigrants in my tree by region", "build a
  colonial ancestors report"). Outputs a single markdown file with
  Ahnentafel-numbered generations, optional alternate family trees for FAMC
  conflicts, and tables grouped by region of origin and (optionally) by
  colonial-era status.
---

# ancestor-report

## Overview

Generates a categorized markdown report of direct ancestors. Walks the tree
using Ahnentafel (Sosa-Stradonitz) numbering, optionally applies parent
overrides for FAMC conflicts, and emits:

1. **Per-generation listing** of every ancestor with name, dates, birth
   place, and death place. Each ancestor is numbered (#N) using their
   Sosa-Stradonitz position; the same number always points to the same
   person and tree slot.
2. **Alternate Family Tree sections** — one per FAMC entry the user did NOT
   choose, but only for children explicitly named with `--alternate`.
   Numbered with letter prefixes (A1, A2, …, B1, …).
3. **Optional FAMC Conflicts list** showing every ancestor in scope with
   multiple recorded parent families.
4. **Immigrants section** with per-region subsections (German and
   Scandinavian, Dutch, French, Spanish and Portuguese, Italian, Irish,
   Scottish, Welsh, English by default — fully configurable). Only ancestors
   who died in the US/colonies are listed.
5. **Optional Colonial Ancestors section** (default: included; suppress
   with `--no-colonial` for non-US-focused reports) with three subsections:
   - *Colonial Immigrants* — born outside the colonies, died in the colonies
     before 1776.
   - *Ancestors Born in the Colonies* — born in the US/colonies before 1776.
   - *Early US Immigrants* — born outside the colonies before 1776, died in
     the US after 1776.

## When to use

- "Generate an ancestor report for @I1@ going back 7 generations." → invoke
  with `--root @I1@ --depth 7`.
- "Build a markdown report of John Smith's family tree." → use search-gedcom
  to find the xref, then run this skill.
- "Show me all immigrants in my family tree by region." → run the skill and
  use the *Immigrants* section.
- "List my colonial ancestors." → run the skill and use the *Colonial
  Ancestors* section.

**Do not use** when:

- The user wants a flat list of ancestors with no categorization → use
  `search-gedcom --ancestors-of` directly.
- The user wants to query specific facts → use `search-gedcom`.
- The user wants to edit GEDCOM data → use `update-gedcom`.

## How to invoke

This skill is a thin wrapper around the `gedcom-ancestor-report` console
script published by [`gedcom-reports`][gedcom-reports]. Run it via uvx:

```bash
uvx --from gedcom-reports gedcom-ancestor-report FILE --root @I1@ --depth 10 [options]
```

If `gedcom-reports` is already installed (`pip install gedcom-reports`),
invoke `gedcom-ancestor-report` directly.

[gedcom-reports]: https://github.com/vaelen/gedcom-reports

## Quick reference

```
--root @I1@                   Root individual xref. (required)
--depth N                     Generations to walk (default: 7).
--override "@C@=@F@,@M@"      Pick parents @F@ and @M@ for child @C@. Repeatable.
                              Either parent may be empty: "@C@=@F@," (no mother).
--alternate @C@               Render an Alternate Family Tree section for child @C@
                              from any FAMC entries that weren't chosen. Repeatable.
                              Without this flag, the un-chosen FAMC is silently dropped.
--regions PATH                Path to regions.yaml or regions.json (default: bundled).
--colonial / --no-colonial    Include or suppress the Colonial Ancestors section
                              (default: included). Pass --no-colonial for
                              non-US-focused reports.
--alt-depth N                 Cap depth of alternate family trees (default: --depth).
--show-conflicts              Include a FAMC Conflicts section before the tables.
--print-default-regions       Print the path to the bundled regions.yaml and exit.
                              Handy as a starting point for `--regions` overrides.
--output PATH                 Write to a file (default: stdout).
```

## FAMC default behavior

By default, the script follows **only the first FAMC** of each individual
(matching `gedcom-search --primary-famc-only`). This keeps the main tree
unambiguous. To choose a different FAMC for an individual, use `--override`.

`--override` is silent by default — the parent set you didn't pick is
dropped. To explicitly render the un-chosen FAMC entries as an **Alternate
Family Tree** section, pass `--alternate @C@` for the child whose alt tree
you want shown.

This split lets you handle two distinct cases cleanly:

- **Reconciling a data-entry error** (e.g., a person whose mother and father
  were entered as two separate FAMC families that should have been one) —
  use `--override` alone to merge them silently.
- **Documenting an unresolved parentage question** (e.g., when sources
  disagree on which couple are someone's biological parents) — use
  `--override` *and* `--alternate` so both the chosen and rejected parent
  sets are visible in the report.

If you want to see which ancestors have multiple FAMC entries before
deciding on overrides, use `--show-conflicts`.

## Region classification

A YAML/JSON config file controls how birth places are bucketed. The default
config ships inside the `gedcom-reports` package and includes these
buckets:

- **German and Scandinavian Settlers** — Germany, Austria, Prussia,
  Switzerland, Alsace, Denmark, Norway, Sweden.
- **Dutch Settlers** — Netherlands.
- **French Settlers** — France (excluding Alsace).
- **Spanish and Portuguese Settlers** — Iberia.
- **Italian Settlers** — Italy.
- **Irish Settlers** — Ireland and Northern Ireland.
- **Scottish Settlers** — Scotland.
- **Welsh Settlers** — Wales.
- **English Settlers** — England (and generic UK / Great Britain entries).

The classification is two-pass:

1. If any US keyword appears in the place string, the place is **US** and
   the person is excluded from the immigrant tables.
2. Otherwise, regions are tested in config order; first match wins.

To customize, copy the bundled `regions.yaml` (find its path with
`gedcom-ancestor-report --print-default-regions`), edit it, and pass
`--regions path/to/your.yaml`. To handle ambiguous tokens (e.g.,
"Middlesex" exists in both England and New Jersey), avoid bare keywords —
include disambiguating context like `middlesex, england`, or rely on the
country-level keyword (`england`, `united kingdom`) being present.

## Examples

```bash
# Basic 7-generation report
uvx --from gedcom-reports gedcom-ancestor-report \
  tree.ged --root @I1@ --depth 7 --output report.md

# Show every FAMC conflict so you can decide on overrides
uvx --from gedcom-reports gedcom-ancestor-report \
  tree.ged --root @I1@ --show-conflicts > conflicts.md

# Override two ambiguous FAMC entries and walk 10 generations, surfacing
# the un-chosen parent sets as alternate trees
uvx --from gedcom-reports gedcom-ancestor-report tree.ged \
  --root @I322367362260@ --depth 10 \
  --override "@I322367362470@=@I322367362647@,@I322367362648@" \
  --override "@I322367362647@=@I322410457996@,@I322548366731@" \
  --alternate @I322367362470@ \
  --alternate @I322367362647@ \
  --output report.md

# Silently fix a split-FAMC data-entry error (no alternate tree wanted)
uvx --from gedcom-reports gedcom-ancestor-report tree.ged \
  --root @I1@ --depth 7 \
  --override "@I322404467221@=@I322471147822@,@I322404467237@" \
  --output report.md

# Suppress the Colonial Ancestors section for a non-US-focused report
uvx --from gedcom-reports gedcom-ancestor-report tree.ged \
  --root @I1@ --depth 10 --no-colonial --output report.md

# Use a custom region config that adds an "Asian Settlers" bucket
uvx --from gedcom-reports gedcom-ancestor-report tree.ged \
  --root @I1@ --depth 7 --regions ./my_regions.yaml --output report.md
```

## Output contract

- Single markdown document. UTF-8.
- Sections in order: subject header, generation listings (Gen 1..N), zero
  or more Alternate Family Tree sections, optional FAMC Conflicts section,
  Immigrants section (with per-region subsections), optional Colonial
  Ancestors section (with three subsections), summary footer.
- Each ancestor in the generation listings carries an Ahnentafel number
  prefix (e.g., `**#42**`); alternate-tree entries use a letter prefix
  (`**#A1**`).
- Region subsections that have no entries are omitted.

## Reference

- [`gedcom-reports`][gedcom-reports] — the package providing the
  `gedcom-ancestor-report` console script and its bundled `regions.yaml`.
- See `gedcom-ancestor-report --help` for the authoritative flag reference.
