# Parsing notes

Implementation gotchas to be aware of when reading or writing GEDCOM. These are the things that bite once you go beyond the minimal samples.

## Encoding detection

Before you can split lines, you have to decode bytes. The detection order:

1. **Look at the first 2–4 bytes for a BOM:**
   - `EF BB BF` → UTF-8 (with BOM)
   - `FE FF` → UTF-16 BE
   - `FF FE` → UTF-16 LE
2. **No BOM, GEDCOM 7.0 expected:** decode as UTF-8.
3. **No BOM, GEDCOM 5.5.x:** read just enough bytes to find `1 CHAR <encoding>` in the header (it has to be in the first ~20 lines), then redecode the whole file.
4. **`CHAR ANSEL`:** you need an ANSEL decoding table. There is no Python stdlib codec for ANSEL; use a library or a hand-rolled translation table. ANSEL has combining diacritics that come *before* the base character — easy to get wrong.
5. **`CHAR UNICODE`** in 5.5.1 means UTF-16 (typically LE) in practice.

Preserve the BOM on write if it was present on read. UTF-8-with-BOM is rare in the wild but real GEDCOM 7 emitters use it.

## Line endings

Acceptable terminators are `LF`, `CR`, and `CRLF`. Real-world files mix all three sometimes — Windows tools that munge a Unix file produce hybrids. When parsing, treat any of the three as a line break. When writing back, use the *predominant* terminator from the original file.

If you split with `str.splitlines()` you'll handle all three transparently; just remember it eats the terminator, so you have to remember it separately if you want round-trip fidelity.

## Splitting fields on a line

Naïve `line.split(' ')` is wrong because the payload may contain spaces. Correct approach:

1. Strip the line terminator.
2. Read the level by reading digits until a space.
3. Read the next field; if it starts and ends with `@`, it is the xref slot; otherwise it is the tag.
4. If the previous step consumed the xref, read the tag next.
5. Everything after the tag (skipping a single delimiter space) is the payload — including any subsequent spaces.

A line that is just `0 TRLR` (no payload) is valid; do not require a payload.

A pure-whitespace or empty line is not valid GEDCOM. Most parsers tolerate trailing blank lines after the trailer; do the same on read, but don't emit any on write.

## Continuations

When you see a child line whose tag is `CONT` or `CONC`, fold it into the parent's payload:

- `CONT` → parent_payload + `\n` + child_payload
- `CONC` → parent_payload + child_payload (no separator)

Walk all `CONC`/`CONT` children in order; they are siblings in the file, but logically they extend the parent's payload.

When **writing** a long payload back out:

- In 7.0, fold on `\n` boundaries into `CONT` lines. Never use `CONC`.
- In 5.5.x, you may break a long line with `CONC`. Prefer to **preserve the source's existing folding** if you're round-tripping; introduce new folds only when emitting fresh content.

Beware: payload trimming. The space between tag and payload is a single delimiter, but everything after it is literal. A `CONC` payload that starts with a space *includes* that space. Off-by-one bugs here corrupt addresses and notes.

## Tag-context disambiguation

The same tag means different things under different parents. Two examples:

- `DATE` under `HEAD`: file creation timestamp, fixed format `DateExact` (day month year).
- `DATE` under `BIRT`/`MARR`/etc.: a `DateValue` with optional modifiers and calendars.

A flat-table parser that doesn't carry parent context will silently corrupt these. The simplest fix is to model GEDCOM as a tree of structures — never a flat list of `(level, tag, value)` tuples.

## Pointers vs. inline values

A payload of the form `@xref@` is a pointer. A payload that *starts* with `@` but isn't pure `@xref@` (e.g. an email address quoted as `@@example.com`) is escaped — the leading `@` was doubled. Strip exactly one leading `@` when you see `@@` at the start of a payload.

Some 5.5.x files in the wild are sloppy about this and embed unescaped `@` characters. Be permissive on read, strict on write.

## Cross-reference scope

Xrefs are file-scoped. Don't assume `@I1@` in two different files is the same person. When merging files, you must remap xrefs.

`@VOID@` (7.0) is a sentinel; do not try to dereference it.

Forward references are allowed: `@F1@` may be referenced from an `INDI` record before the `FAM` record is reached in the byte stream. Two-pass parsing (collect all xrefs, then resolve pointers) is the cleanest approach.

## Bidirectional consistency

A 7.0-conforming `INDI.FAMS @F1@` requires `FAM @F1@` to contain a back-pointer (`HUSB`/`WIFE`) to that individual. 5.5.x is less strict in practice but the same convention. When updating, fix both sides; when reading, do not assume both sides are present.

## Sibling ordering

Within a structure, sibling order is significant: the first sibling of a tag is the preferred value. Never reorder siblings of the same tag on round-trip. Reordering siblings of *different* tags is technically allowed but a lot of tools rely on file order — preserve it anyway.

## Extension tags

Tags starting with `_` are extensions. Do not strip, normalize, or reject them. On round-trip, write them back unchanged. In 7.0, declared extensions appear in `HEAD.SCHMA.TAG`; preserve those declarations even if you don't understand the extension.

## Number parsing

Levels are non-negative integers in base 10. They can be more than one digit (`10`, `11`, etc.) — rare but legal. Don't `int(line[0])`.

## Common malformations seen in the wild

- Missing trailer (`TRLR`). Tolerate on read; emit on write.
- Trailing whitespace on lines. Strip before parsing the line value? **No.** Whitespace at the end of a payload may be significant. Trim only the line terminator.
- BOM in the middle of the file (cat'd files). Treat as a parse error or strip; flag it.
- Duplicate xrefs. Spec violation; flag and either reject or rename on read.
- Self-referential pointers (e.g. an INDI as its own father). Spec violation; flag.

## Library vs. roll-your-own

For Python, `ged4py` is the most maintained library and handles 5.5.x cleanly; 7.0 support is partial as of 2026. For 7.0-specific work or anything that demands round-trip fidelity, the simplest path is often a small purpose-built parser:

- Tokenize line → `(level, xref, tag, payload)`.
- Build a tree via a level-stack.
- Write back by pre-order traversal.

This fits in ~200 lines of Python and beats fighting a library that strips formatting you needed.

If a script in this repo uses a library, the rationale should be in a comment at the top of the script.
