# GEDCOM versions

Three GEDCOM versions are in scope for these skills: **5.5.1**, **5.5.5**, and **7.0**. They share a family resemblance but differ in important ways. This page describes the differences that matter for parsing, writing, and validation.

## Quick comparison

| | 5.5.1 | 5.5.5 | 7.0+ |
| --- | --- | --- | --- |
| Year | 1999 | 2019 | 2021– |
| Maintainer | FamilySearch | gedcom.org | FamilySearch |
| Encoding | `UTF-8`, `ANSEL`, `ASCII`, `UNICODE` (UTF-16) | UTF-8 or UTF-16 with BOM | **UTF-8 only**, BOM allowed |
| Header `CHAR` | required | required (must match BOM) | **removed** |
| Header `GEDC` `VERS` | `5.5.1` | `5.5.5` | `7.0` (or `7.x.y`) |
| `CONC` | allowed | allowed | **removed** |
| `CONT` | allowed | allowed | allowed (only continuation) |
| Length limits | yes (255-char line cap, etc.) | yes | **none** |
| `@VOID@` null pointer | informal | informal | defined |
| Extension tags | `_TAG` convention | `_TAG` convention | defined: `SCHMA` maps `_TAG` → URI |
| GEDZIP packaging | no | no | yes (`.gdz`) |
| Same-sex / non-binary | not modeled | partial | first-class |
| Email / fax / web in addresses | implementation-defined | yes | yes |
| Multimedia | `OBJE` records or inline `BLOB` | `OBJE` records (no `BLOB`) | `OBJE` records (no `BLOB`); `MIME`/`TRAN` |

Real-world data is overwhelmingly 5.5.1. 5.5.5 cleans up 5.5.1 ambiguities but never gained traction; 7.0 is the forward path.

## Detecting the version

Always read the version from the file rather than guessing from the extension. The canonical location is:

```
0 HEAD
1 GEDC
2 VERS <version>
```

In 5.5.5 and 7.0 this is required. In 5.5.1 the structure is also required, but the value can be loose — some files state `5.5` or `5.5.1` interchangeably. Treat anything beginning with `5.5` and not `5.5.5` as 5.5.1 unless the file otherwise says otherwise.

The `2 FORM LINEAGE-LINKED` line under `GEDC` is required in 5.5.x and **removed** in 7.0. Its presence is a 5.5.x signal.

## 5.5.1 → 5.5.5 changes

5.5.5 is a backward-incompatible cleanup of 5.5.1. The changes most likely to bite a parser:

- `BLOB` (inline binary multimedia) **removed**.
- `OBJE` no longer allowed inline as a substructure; only as a top-level record referenced by pointer.
- Encoding is determined by the **BOM**, not the `CHAR` line. The `CHAR` value must match the BOM and must be `UTF-8`, `UTF-16BE`, or `UTF-16LE`. ANSEL is gone.
- Several tag definitions tightened (e.g. address structure, submitter address).
- Stricter requirement that every record referenced by a pointer must exist in the file.

If you can read 5.5.5 cleanly you can almost always read 5.5.1, but not vice versa.

## 5.5.x → 7.0 changes

7.0 is a deliberate breaking change. The big ones:

1. **UTF-8 only.** ANSEL is gone. `CHAR` is gone (encoding is implicit + may be marked by BOM).
2. **`CONC` is gone.** Multi-line payloads use `CONT` exclusively. No more soft-wrapping.
3. **Length limits gone.** Tag names, payloads, and xrefs have no length cap.
4. **Escape rules relaxed.** A leading `@` in a payload is doubled to `@@`. Other `@` characters do not need escaping. (5.5.x doubled `@` more aggressively.)
5. **Date/age phrasing.** Free-text annotations on dates, ages, and times moved into a `PHRASE` substructure rather than being interpolated into the payload. Example:
   ```
   1 BIRT
   2 DATE ABT 1850
   3 PHRASE around 1850
   ```
6. **`SNOTE` replaces `NOTE` records.** Inline `1 NOTE <text>` substructures still exist, but reusable note *records* are now `0 @N1@ SNOTE` rather than `0 @N1@ NOTE`.
7. **`SUBN` (submission record) removed.** It was an LDS-specific holdover.
8. **Extension mechanism is real.** Custom tags must be declared in `1 SCHMA / 2 TAG _NAME <uri>` in the header. Undeclared `_*` tags are still tolerated but discouraged.
9. **Calendars and dates.** First-class support for Gregorian, Julian, French Republican, and Hebrew calendars; explicit `BCE` epoch instead of `B.C.`
10. **Media types.** `OBJE` substructures use `FORM` (5.5.x) replaced by `MIME` (7.0) using IANA media types like `image/jpeg`.
11. **Translations.** A new `TRAN` substructure attaches translations to text, names, places, and dates.
12. **Bidirectional linking required.** If `INDI.FAMS` points to `FAM`, then `FAM.HUSB`/`FAM.WIFE` must point back. The spec calls this out explicitly; some 5.5.x files were sloppy here.
13. **`@VOID@` defined.** Use it explicitly when a slot is required but the value is unknown, instead of inventing a placeholder xref.
14. **GEDZIP.** A `.gdz` file is a zip archive containing a `gedcom.ged` plus referenced media files at relative paths. Useful when you need to ship media alongside the data.

## Extension semantics in 7.0

In 7.0, extension tags are first-class:

```
0 HEAD
1 GEDC
2 VERS 7.0
1 SCHMA
2 TAG _SKYPEID http://xmlns.com/foaf/0.1/skypeID
2 TAG _JABBERID http://xmlns.com/foaf/0.1/jabberID
```

Once declared, `_SKYPEID` and `_JABBERID` may be used anywhere their URI's defining schema permits. A 7.0-aware tool that doesn't recognize the URI must still preserve the structure on round-trip. A tool that *does* understand the URI may use it semantically.

## Compatibility advice for these skills

- When **reading**, try to be liberal: accept 5.5.1, 5.5.5, and 7.0; surface the detected version so callers can react.
- When **writing**, do not promote a file to a newer version unless the user explicitly asks. Round-tripping a 5.5.1 file should produce a 5.5.1 file.
- When **converting**, treat it as a deliberate operation with its own skill. Conversions lose or remap data (notably ANSEL → UTF-8, `CONC` collapsing, `NOTE` records → `SNOTE`).

## References

- [GEDCOM 5.5.1 specification (with errata)](https://familysearch.org/specifications/ged551-with-inline-errata.html)
- [GEDCOM 5.5.5 specification](https://www.gedcom.org/)
- [FamilySearch GEDCOM 7 specification](https://gedcom.io/specifications/FamilySearchGEDCOMv7.html)
- [FamilySearch GEDCOM Compatibility Guide](https://gedcom.io/compatibility/)
- [Changelog — gedcom.io](https://gedcom.io/changelog/)
