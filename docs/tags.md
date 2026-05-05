# Common GEDCOM tags

A working reference for the tags you will hit most often when reading or writing GEDCOM. This is **not** a complete tag list — see the official specs in [versions.md](versions.md) for that. Each entry notes the contexts where the tag appears and any version differences that matter.

GEDCOM tags are **context-sensitive**: the same tag means different things under different parents. `DATE` under `BIRT` is a birth date; `DATE` under `HEAD` is the file creation date. Always interpret a tag in light of its superstructure.

## Record-level tags (level 0)

| Tag | Record | Notes |
| --- | --- | --- |
| `HEAD` | Header | Exactly one; first record. |
| `TRLR` | Trailer | Exactly one; last line; no payload, no children. |
| `INDI` | Individual | A person. |
| `FAM` | Family | A relationship grouping spouses and children. |
| `SUBM` | Submitter | The contributor. |
| `SOUR` | Source | A bibliographic source. |
| `REPO` | Repository | Holding institution for sources. |
| `OBJE` | Multimedia | Reference to a media file. |
| `NOTE` | Note record (5.5.x) | Reusable note. Removed as a record type in 7.0. |
| `SNOTE` | Shared note (7.0) | Replaces `NOTE` records. |
| `SUBN` | Submission (5.5.1) | LDS submission metadata; **removed in 7.0**. |

## INDI substructures

The hot path. An individual record looks like:

```
0 @I1@ INDI
1 NAME John /Smith/
1 SEX M
1 BIRT
2 DATE 1 JAN 1900
2 PLAC London, England
1 DEAT
2 DATE 5 MAR 1965
2 PLAC Boston, Massachusetts, USA
1 FAMC @F1@
1 FAMS @F2@
```

Common substructures:

| Tag | Meaning |
| --- | --- |
| `NAME` | Personal name. Surname is delimited by `/.../`. May have multiple (preferred first). |
| `SEX` | `M`, `F`, `U` (5.5.x); 7.0 also defines `X` for non-binary/intersex. |
| `BIRT`, `DEAT`, `BURI`, `BAPM`, `CHR`, `CREM` | Life events. Each holds a `DATE` and `PLAC` substructure. |
| `EVEN` | Generic event with a `TYPE` payload. |
| `OCCU`, `EDUC`, `RELI`, `NATI`, `RESI` | Attributes (more like states than events). |
| `FAMC` | Pointer to the family in which this individual is a *child*. |
| `FAMS` | Pointer to the family in which this individual is a *spouse*. |
| `ASSO` | Association with another `INDI` (godparent, witness, etc.). |
| `NOTE` | Inline note text or pointer to a NOTE/SNOTE record. |
| `SOUR` | Source citation. May be inline text or a pointer to a SOUR record. |
| `OBJE` | Multimedia citation — pointer to an OBJE record. |
| `CHAN` | Last-changed metadata; holds a `DATE` substructure. |

## FAM substructures

```
0 @F1@ FAM
1 HUSB @I1@
1 WIFE @I2@
1 CHIL @I3@
1 CHIL @I4@
1 MARR
2 DATE 1 JUN 1899
2 PLAC London, England
```

| Tag | Meaning |
| --- | --- |
| `HUSB` | Pointer to husband INDI. (Spousal role 1.) |
| `WIFE` | Pointer to wife INDI. (Spousal role 2.) |
| `CHIL` | Pointer to a child INDI. May appear multiple times; order is birth order by convention. |
| `MARR`, `DIV`, `ENGA`, `ANUL` | Family events. |
| `NCHI` | Number of children (count, not pointers). |
| `RESI` | Family residence. |

In 7.0, the `HUSB`/`WIFE` slot names are kept for backward compatibility, but a same-sex couple may use either slot for either partner. Some 7.0 emitters prefer `1 ASSO`-style modeling for non-traditional families; do not assume `HUSB` is male.

## Names

A `NAME` payload uses slashes around the surname:

```
1 NAME John Quincy /Adams/ Sr.
2 GIVN John Quincy
2 SURN Adams
2 NSFX Sr.
```

Substructures (all optional, all in any combination):

- `NPFX` — name prefix (`Dr.`, `Sir`)
- `GIVN` — given name(s)
- `NICK` — nickname
- `SPFX` — surname prefix (`van`, `de la`)
- `SURN` — surname
- `NSFX` — name suffix (`Jr.`, `III`)
- `TYPE` — `birth`, `married`, `aka`, `immigrant`, `maiden`, `adoptive`, `religious`, etc.

If a person has multiple `NAME` structures, the first one is the preferred name.

## Dates

GEDCOM dates are typed and can be surprisingly rich:

```
1 BIRT
2 DATE 1 JAN 1900
2 DATE ABT 1850
2 DATE BET 1850 AND 1855
2 DATE FROM 1900 TO 1910
2 DATE BEF JUN 1900
2 DATE AFT 1 JAN 1900
2 DATE EST 1900
2 DATE CAL 1900
2 DATE @#DJULIAN@ 1 JAN 1700
2 DATE @#DHEBREW@ 1 TSH 5760
2 DATE @#DFRENCH R@ 1 VEND 8
```

Calendar escapes `@#D...@` select a non-Gregorian calendar. Modifier keywords:

- `ABT` (about), `EST` (estimated), `CAL` (calculated)
- `BEF`, `AFT`
- `BET ... AND ...` (range)
- `FROM ... TO ...` (period)

In 7.0, free-text comments accompanying a date go into a `PHRASE` substructure, not the `DATE` payload itself:

```
2 DATE ABT 1850
3 PHRASE shortly before the war
```

In 5.5.x, parenthesized phrases inside the `DATE` payload were the convention.

## Places

`PLAC` payloads are a comma-delimited hierarchy from finest to coarsest. The header may declare the format with `1 PLAC / 2 FORM`:

```
1 BIRT
2 PLAC Boston, Suffolk, Massachusetts, USA
3 LATI N42.3601
3 LONG W71.0589
```

7.0 adds a `MAP` substructure for coordinates; 5.5.x put `LATI`/`LONG` directly under `PLAC`.

## Sources and citations

A `SOUR` record is the bibliographic record:

```
0 @S1@ SOUR
1 AUTH Jane Doe
1 TITL A History of the Smith Family
1 PUBL 1972, Boston: Smith Press
1 REPO @R1@
```

A *citation* references a source from inside another record. It may be a pointer plus page detail, or inline text:

```
1 BIRT
2 SOUR @S1@
3 PAGE p. 42
3 QUAY 3
```

`QUAY` is the certainty: 0 (unreliable) – 3 (direct primary evidence).

## Notes

Inline notes attach text directly to a structure. Reusable notes are records.

- **5.5.x:** `0 @N1@ NOTE …` defines a reusable record. Inline form: `1 NOTE <text>` with optional `2 CONT`/`2 CONC`.
- **7.0:** `0 @N1@ SNOTE …` (renamed). Inline form: `1 NOTE <text>` with optional `2 CONT` (no `CONC`). `SNOTE` records can carry `MIME` and `LANG`.

## Multimedia (OBJE)

```
0 @M1@ OBJE
1 FILE photos/john_smith.jpg
2 FORM jpg          (5.5.x)
2 MIME image/jpeg   (7.0)
1 TITL John Smith, c.1930
```

In 5.5.x, `1 OBJE` could appear inline as a substructure. In 5.5.5 and 7.0 only the pointer form is allowed inside other records.

## Address structure

Used inside `SUBM`, `REPO`, `INDI`, etc.:

```
1 ADDR 15 East South Temple Street
2 CONT Salt Lake City, UT 84150 USA
2 CITY Salt Lake City
2 STAE UT
2 POST 84150
2 CTRY USA
1 PHON +1 555 555 1212
1 EMAIL john@example.com
1 FAX +1 555 555 1213
1 WWW http://example.com
```

In 5.5.1, `EMAIL`/`FAX`/`WWW` were unofficial extensions before being formalized in errata. 5.5.5 and 7.0 standardize them.

## Change tracking

`CHAN` (change) attaches a last-modified timestamp to most records:

```
1 CHAN
2 DATE 5 MAY 2026
3 TIME 14:23:00
```

Updaters should refresh `CHAN` on records they modify, but this is convention, not a hard requirement.

## See also

- [format.md](format.md) — line-level syntax
- [versions.md](versions.md) — what differs between 5.5.1, 5.5.5, and 7.0
- [parsing-notes.md](parsing-notes.md) — implementation gotchas
