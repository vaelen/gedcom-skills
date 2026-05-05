---
name: update-gedcom
description: Use when the user asks to edit, modify, change, update, set, alter, fix, correct, add, remove, or delete a record, value, name, date, or any field in a GEDCOM file (`.ged`). Examples that should trigger this skill: "change the name on @I1@", "set the birth date for John Smith", "add a NOTE to this record", "remove the @VOID@ ASSO from @I1@", "create a new INDI", "delete record @I7@". Preserves round-trip fidelity — only the targeted line is rewritten — and never modifies the input file unless `--in-place` is passed. Handles GEDCOM 5.5.1, 5.5.5, and FamilySearch GEDCOM 7.0+.
version: 0.1.0
---

# update-gedcom

## Overview

Modify a GEDCOM file with structural operations that respect round-trip fidelity. Only the lines you target are rewritten; every untouched line emits byte-identical to its source. The default writes the modified file to stdout (or `-o FILE`); the input file is **never** touched unless `--in-place` is passed.

Operations:

- `set-payload XREF PATH VALUE` — change a structure's payload.
- `add-substructure XREF PATH NEW_TAG [VALUE]` — append a child under an existing structure.
- `remove XREF PATH` — delete the structure at `PATH` within record `XREF`.
- `add-record TYPE [--xref XREF]` — create a new top-level record (xref auto-generated unless given).
- `delete-record XREF [--orphan-pointers void|strip]` — remove a top-level record. Refuses by default if other records still point at it.

`PATH` is a slash-delimited chain of tags relative to the record root (`NAME`, `BIRT/DATE`, …). The first matching child at each level is targeted, consistent with the GEDCOM convention that the first sibling is preferred. An empty path (`""`) refers to the record itself.

## When to use

- "Change @I1@'s name to Jane Doe" → `set-payload @I1@ NAME "Jane /Doe/"`
- "Add a note to @I1@" → `add-substructure @I1@ "" NOTE "..."`
- "Set @I1@'s birth date" → `set-payload @I1@ BIRT/DATE "1 JAN 1900"`
- "Add a new family record" → `add-record FAM`
- "Delete @I7@" → `delete-record @I7@` (will refuse if pointers exist; rerun with `--orphan-pointers`)

**Do not use** when:

- The user just wants to view or summarize → use **read-gedcom**.
- The user wants to find records → use **search-gedcom**.

## How to invoke

This skill is a thin wrapper around the `gedcom-update` console script published by [gedcom-lite](https://github.com/vaelen/gedcom-lite). Global flags must come **before** the subcommand:

```bash
uvx --from gedcom-lite gedcom-update FILE [global-flags] SUBCOMMAND ...
```

To run unreleased changes from git instead of the PyPI release:

```bash
uvx --from "git+https://github.com/vaelen/gedcom-lite" gedcom-update FILE [global-flags] SUBCOMMAND ...
```

If `gedcom-lite` is already installed, invoke `gedcom-update` directly.

## Quick reference

**Global flags:**

```
-o FILE       write result here (default: stdout)
--in-place    overwrite the input file (mutually exclusive with -o)
```

**Subcommands:**

```
set-payload      XREF PATH VALUE
add-substructure XREF PATH NEW_TAG [VALUE]
remove           XREF PATH
add-record       TYPE [--xref XREF]
delete-record    XREF [--orphan-pointers void|strip]
```

## Examples

```
# Change a name (writes to stdout; pipe or redirect as needed)
gedcom-update tree.ged set-payload @I1@ NAME "Jane /Doe/" > new.ged

# Set a birth date and write to a file
gedcom-update tree.ged -o new.ged set-payload @I1@ BIRT/DATE "1 JAN 1900"

# Add a NOTE substructure to an INDI
gedcom-update tree.ged -o new.ged add-substructure @I1@ "" NOTE "Met at family reunion"

# Add a DATE under an existing BIRT
gedcom-update tree.ged -o new.ged add-substructure @I1@ BIRT DATE "ABT 1900"

# Remove a substructure
gedcom-update tree.ged -o new.ged remove @I1@ NAME

# Create a new INDI record (xref auto-generated; printed to stderr when -o is used)
gedcom-update tree.ged -o new.ged add-record INDI

# Delete a record after voiding inbound pointers
gedcom-update tree.ged -o new.ged delete-record @I7@ --orphan-pointers void

# Apply edits to the source file (be sure first)
gedcom-update tree.ged --in-place set-payload @I1@ NAME "Jane /Doe/"
```

## Safety contract

- The default invocation **does not modify** the input file. The result goes to stdout, or to the path given to `-o`.
- `--in-place` is the only mode that overwrites the source. It is mutually exclusive with `-o` and stdin input.
- For `add-record`, the new record's xref is printed to **stderr** when output goes to a file (so stdout stays clean for the document body).
- `delete-record` aborts with a non-zero exit if any other record still points at the target, listing up to ten of the pointers. Re-run with `--orphan-pointers void` (rewrite payloads to `@VOID@`) or `--orphan-pointers strip` (remove the pointing substructures) to proceed.
- Round-trip fidelity is preserved: every untouched line is emitted byte-identical to the source, including BOM, line endings, sibling order, and unknown extension tags.

## Reference

- `docs/format.md` — line grammar; understand `level`, `xref`, `tag`, payload before issuing edits
- `docs/tags.md` — tag semantics so you know which `PATH` to target
- `docs/parsing-notes.md` — encoding, continuation, and pointer-handling caveats relevant to in-place edits
