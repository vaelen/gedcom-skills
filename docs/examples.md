# Example files

`examples/` ships a small demo set so the skills can be tried without checking out the library. The full official test suite — UTF-16 LE/BE, ANSEL, extensions, escapes, voidptrs, GDZ archives, every standard tag — lives in [`gedcom-lite/examples/`](https://github.com/vaelen/gedcom-lite/tree/main/examples).

## What ships here

### `examples/gedcom70/`

| File | Description |
| --- | --- |
| `minimal70.ged` | The smallest valid GEDCOM 7.0 file. Three lines plus the trailer. |
| `maximal70.ged` | Stress test that exercises every standard tag in many positions. The big one. |

### `examples/gedcom555/`

| File | Description |
| --- | --- |
| `MINIMAL555.GED` | The smallest valid GEDCOM 5.5.5 file. |
| `555SAMPLE.GED` | The canonical 5.5.5 specification sample (UTF-8). |

## Sources

- GEDCOM 5.5.5 — [gedcom.org/samples.html](https://www.gedcom.org/samples.html)
- GEDCOM 7.0 — [gedcom.io/tools/](https://gedcom.io/tools/)

## When you need more coverage

If you want to test the parser against UTF-16 files, ANSEL files, GDZ archives, or every standard tag in isolation, install `gedcom-lite` from source and use its `examples/` tree:

```bash
git clone https://github.com/vaelen/gedcom-lite
ls gedcom-lite/examples/
```

That repo also ships a `tests/` directory that round-trips every fixture byte-identically.

## Treat these as read-only

Skills should never modify the files in `examples/`. If a worked example needs to demonstrate `update-gedcom`, write the modified output to `/tmp/` or to a path the user gives.
