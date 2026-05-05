# gedcom-skills

A Claude Code plugin that ships three skills for working with [GEDCOM](https://www.gedcom.org/) genealogy files. The skills are thin SKILL.md wrappers around the `gedcom-read`, `gedcom-search`, and `gedcom-update` console scripts published by [**gedcom-lite**](https://github.com/vaelen/gedcom-lite) — a separate, fidelity-preserving GEDCOM toolkit.

## Skills

| Skill | What it does | Sample prompts |
| --- | --- | --- |
| **read-gedcom** | Summarize a file, list records, show one record by xref. | "What's in `tree.ged`?", "list everyone in this GEDCOM", "show me record @I42@" |
| **search-gedcom** | Query by tag/value/path or by person/date/place/relationship (ancestors, descendants, parents, children). | "find people named Smith", "who was born between 1900 and 1910", "ancestors of @I1@" |
| **update-gedcom** | Set, add, remove with round-trip fidelity; safe-by-default writes. | "change @I1@'s name to Jane Doe", "add a NOTE to @I7@", "delete record @I9@" |

For the operational details of each, read the skill's `SKILL.md`.

## Versions in scope

| Version | Year | Why we support it |
| --- | --- | --- |
| GEDCOM 5.5.1 | 1999 | Most-deployed legacy format. The de-facto interchange format used by virtually every genealogy app shipped before 2022. |
| GEDCOM 5.5.5 | 2019 | Cleanup release of the 5.5 line. UTF-8 only; not widely adopted but useful as a reference. |
| FamilySearch GEDCOM 7.0+ | 2021– | Current, actively maintained. UTF-8 only; defines a real extension mechanism and GEDZIP packaging. |

GEDCOM 7 is a breaking change from 5.5.x. `gedcom-lite` reads any of the three; on write it never silently promotes a file to a different version.

## Layout

```
.
├── README.md              # this file
├── CLAUDE.md              # working agreements for Claude Code in this repo
├── .claude-plugin/
│   └── plugin.json        # plugin manifest
├── skills/
│   ├── read-gedcom/SKILL.md
│   ├── search-gedcom/SKILL.md
│   └── update-gedcom/SKILL.md
├── docs/                  # GEDCOM-domain reference (linked from each SKILL.md)
└── examples/              # small demo set; the full fixture suite lives in gedcom-lite
```

There is no Python code in this repo. The parser, writer, ANSEL codec, CLI tools, and full test suite all live in [`gedcom-lite`](https://github.com/vaelen/gedcom-lite).

## How invocations work

The skills invoke the published `gedcom-lite` CLI via `uvx` (which ships with [`uv`](https://docs.astral.sh/uv/)). `uvx` caches an ephemeral environment with the package installed, so no prior `pip install` is required:

```bash
uvx --from gedcom-lite gedcom-read   tree.ged
uvx --from gedcom-lite gedcom-search tree.ged --person Smith
uvx --from gedcom-lite gedcom-update tree.ged -o new.ged set-payload @I1@ NAME "Jane /Doe/"
```

To run unreleased changes from git instead of the PyPI release:

```bash
uvx --from "git+https://github.com/vaelen/gedcom-lite" gedcom-read tree.ged
```

If `gedcom-lite` is installed system-wide (`pip install gedcom-lite` or `uv tool install gedcom-lite`), invoke the commands directly without `uvx`.

## Example files

`examples/` ships a small demo set so the skills can be exercised without checking out the library:

- `examples/gedcom70/minimal70.ged` — smallest valid GEDCOM 7.0 file
- `examples/gedcom70/maximal70.ged` — broad GEDCOM 7.0 coverage (every standard tag in many positions)
- `examples/gedcom555/MINIMAL555.GED` — smallest valid GEDCOM 5.5.5 file
- `examples/gedcom555/555SAMPLE.GED` — canonical GEDCOM 5.5.5 sample

For the full official test suite (UTF-16, ANSEL, extensions, escapes, voidptrs, GDZ archives, …) see [`gedcom-lite/examples/`](https://github.com/vaelen/gedcom-lite/tree/main/examples).

## References

- [gedcom-lite](https://github.com/vaelen/gedcom-lite) — the parser/CLI package this plugin wraps
- [GEDCOM 5.5.5 specification](https://www.gedcom.org/) — gedcom.org
- [FamilySearch GEDCOM 7 specification](https://gedcom.io/specifications/FamilySearchGEDCOMv7.html) — gedcom.io
- [FamilySearch GEDCOM Compatibility Guide](https://gedcom.io/compatibility/)

## License

MIT — Copyright © 2026 Andrew C. Young (andrew@vaelen.org). See [LICENSE](LICENSE) for the full text. The underlying [`gedcom-lite`](https://github.com/vaelen/gedcom-lite/blob/main/LICENSE) package is distributed under the same terms.
