# GEDCOM file format

This is a developer-oriented summary of how a GEDCOM file is laid out on disk. It applies to all versions in scope (5.5.1, 5.5.5, and 7.0+) except where called out. For per-version differences see [versions.md](versions.md).

## High-level shape

A GEDCOM file is a flat, line-oriented text file made of three pieces in this order:

```
HEAD          (header — exactly one)
0 @X1@ INDI   (zero or more records, in any order)
0 @F1@ FAM
...
TRLR          (trailer — exactly one, payload-less)
```

Every meaningful unit in the file is a **structure**, written across one or more lines. A record is a top-level structure (level 0). Substructures nest by indentation expressed in the leading level number — there is no whitespace-based indentation; the level integer alone defines the hierarchy.

## Line grammar

Each line has the form:

```
Level [Space Xref] Space Tag [Space LineValue] EOL
```

- **Level** — non-negative decimal integer. `0` for top-level records, header, and trailer; `n+1` for direct substructures of a level-`n` line.
- **Xref** (optional) — `@identifier@` form. Only valid on level 0 records and on lines that *define* a record. Pointer payloads also use the same `@xref@` form, but as a `LineValue`, not as the xref slot.
- **Tag** — required. Standard tags match `[A-Z][A-Z0-9_]*`. Extension tags start with an underscore: `_[A-Z0-9_]+`.
- **LineValue** (optional) — payload. May be free text, a pointer (`@xref@`), or a typed value (date, age, enumeration, etc.) depending on the tag and its superstructure.
- **EOL** — `CR`, `LF`, or `CRLF`. All three are valid; preserve whatever the source uses on round-trip.

A single space (`U+0020`) is the only allowed delimiter between fields. Tabs and multiple spaces are not delimiters; if you see them inside the payload, they are part of the payload.

## Hierarchy

A level-`n+1` line is a substructure of the most recent preceding line at level `n`. There is no closing token — nesting ends as soon as a line at level ≤ `n` appears.

```
0 @I1@ INDI
1 NAME John /Smith/
2 GIVN John
2 SURN Smith
1 BIRT
2 DATE 1 JAN 1900
2 PLAC London, England
0 @I2@ INDI         <-- starts a new top-level record
```

Sibling order matters within a structure: per the spec the **first** sibling of a given tag is treated as the preferred value, and software is expected to keep them in order on round-trip.

## Cross-references and pointers

Cross-references are file-scoped identifiers in `@id@` form. They do two jobs:

1. **Defining** — written in the xref slot of a record line, e.g. `0 @I42@ INDI`.
2. **Referencing** — written as a payload, e.g. `1 FAMS @F7@`.

Bidirectional links (e.g. `INDI.FAMS` ↔ `FAM.HUSB`/`FAM.WIFE`/`FAM.CHIL`) must be mutual: if an INDI record points at a FAM record, the FAM record should point back. Tools that update one side should update both.

GEDCOM 7 defines `@VOID@` as an explicit null pointer for cases where a slot is required but the target is unknown. In 5.5.x the convention varies; see [versions.md](versions.md).

## Continuations: CONT and CONC

Payloads cannot contain a line terminator. To represent multi-line text, GEDCOM uses pseudo-substructures:

- **`CONT`** — start a new line. Equivalent to `\n` in the original payload.
- **`CONC`** — string concatenation, no newline. Used in 5.5.x to wrap a long line at a soft column boundary. **Removed in 7.0.**

Example (5.5.x):

```
1 NOTE This is a longer note that
2 CONC  has been concatenated, and
2 CONT then continues on a new line.
```

Reading: reassemble the payload as `payload + (CONC payload)... + ('\n' + CONT payload)...` walking the children in order.

Writing: in 7.0 only `CONT` is valid. In 5.5.x either is valid; preserve the source style on round-trip rather than re-folding.

## Escaping

Inside a payload, a leading `@` (the first character of a non-pointer line value) must be doubled to `@@` so it cannot be confused with a pointer. Other `@` characters in the payload do not need to be escaped in 7.0; 5.5.x has stricter rules — see [versions.md](versions.md).

## Encoding and BOMs

- **7.0+** — UTF-8 only. A leading `U+FEFF` BOM is allowed and should be preserved on write.
- **5.5.5** — UTF-8 or UTF-16 (LE/BE), each with a BOM. The encoding is determined by the BOM, not by `1 CHAR`.
- **5.5.1** — encoding is declared in the header by `1 CHAR`. Possible values include `UTF-8`, `ANSEL` (a legacy MARC-derived encoding), `ASCII`, and `UNICODE` (UTF-16, in practice). ANSEL is its own can of worms; if you see it, decode using a dedicated ANSEL table — Python has no stdlib codec for it.

## The header (`HEAD`)

Required at the top of every file. Common substructures (most are version-dependent):

```
0 HEAD
1 GEDC                # spec-compliance block (7.0 requires this)
2 VERS 7.0
1 CHAR UTF-8          # 5.5.x only — 7.0 has no CHAR
1 SOUR <software>     # originating program
2 VERS <version>
2 NAME <product name>
1 DATE <creation date>
2 TIME <creation time>
1 SUBM @U1@           # pointer to a SUBM record
1 LANG <BCP-47 tag>   # default language for free text
1 SCHMA               # 7.0 only — extension tag → URI map
2 TAG _MYTAG https://example.com/mytag
```

The trailer is the literal line `0 TRLR` (no payload, no children) and marks end-of-file.

## Record types (level 0)

All versions share the same core record types; payload semantics drift between versions. See [tags.md](tags.md) for substructure details.

| Tag | Meaning | Notes |
| --- | --- | --- |
| `INDI` | Individual (a person) | Most common record type. |
| `FAM` | Family relationship | Holds spouses, children, family events. |
| `SUBM` | Submitter | The contributor of the data. |
| `SOUR` | Source citation record | Bibliographic source. |
| `REPO` | Repository | Where a source is held. |
| `OBJE` | Multimedia object | Reference to a media file. |
| `NOTE` | Free-text note (5.5.x) | Removed-as-record in 7.0; replaced by `SNOTE`. |
| `SNOTE` | Shared note (7.0) | Reusable note record. |
| `SUBN` | Submission record (5.5.1) | LDS-related; not in 7.0. |

The trailer (`TRLR`) and the header (`HEAD`) are also level-0 structures but are not "records" in the cross-reference sense — they have no xref id.

## Worked example

A minimal valid 7.0 file is just three lines:

```
0 HEAD
1 GEDC
2 VERS 7.0
0 TRLR
```

A minimal 5.5.5 file (from the official samples) adds the `CHAR` declaration, the `FORM`, and a submitter:

```
0 HEAD
1 GEDC
2 VERS 5.5.5
2 FORM LINEAGE-LINKED
3 VERS 5.5.5
1 CHAR UTF-8
1 SOUR gedcom.org
0 @U@ SUBM
1 NAME gedcom.org
0 TRLR
```

Both files are in `examples/`; round-tripping them byte-for-byte is a reasonable smoke test for any parser/writer pair we ship.
