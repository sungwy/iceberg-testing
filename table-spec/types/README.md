# table-spec/types

Type strings as they appear in schema JSON: `decimal`, `fixed`, `geometry`.

The spec pins exactly one serialization for each type string, so a consumer asserts
both that the input parses to the expected fields and that re-serializing yields the
canonical form, byte for byte.

## Why this surface

Type-string parsing is where readers most visibly disagree: whitespace tolerance in
`decimal(...)` and `fixed[...]`, and CRS quoting in `geometry(...)`. Per-repo unit
tests miss it because each repo validates the spec against its own reading. The
cases pin the spec's reading:

- `decimal( 9 , 2 )` and `fixed[ 8 ]`: the spec allows optional whitespace around
  the parameters, and the canonical form removes it.
- `geometry(srid:4326)`: the CRS is unquoted, matching the spec example.

A reader stricter or looser than the spec diverges on these.

## Format: `<type>/cases.jsonl`

JSON Lines, one self-contained case per line.

| field | meaning |
| --- | --- |
| `id` | stable case id, unique within the repo |
| `input` | the type string as it appears in schema JSON |
| `accept` | `true` if a conformant reader parses it, `false` if it rejects it |
| `parsed` | expected parsed fields, present when `accept` is `true` |
| `canonical` | the single spec-canonical serialization to re-emit |
| `clause` | the spec clause the case pins |
| `note` | what the case pins |

## Assertion to implement

For `accept: true`:

1. `parse(input)` succeeds and its fields equal `parsed`.
2. `serialize(parse(input))` equals `canonical`, byte for byte.

For `accept: false`: `parse(input)` raises a parse error.

`parsed` keys are spec-level field names (`precision`, `scale`, `length`, `crs`).
Map them to your implementation's native accessors for the parsed type (its
precision and scale, its length, its CRS), and serialize with your reader's
type-string output.

In pseudo-code, framework-independent:

```
for case in read_jsonl("<type>/cases.jsonl"):
    if case.accept:
        t = your_reader.parse(case.input)
        assert native_fields(t) == case.parsed       # map parsed keys to accessors
        assert your_reader.serialize(t) == case.canonical
    else:
        assert_raises(lambda: your_reader.parse(case.input))
```

## Staged adoption

Some implementations do not satisfy every case yet. A consumer enrolling
incrementally keeps a local skip list of the cases it does not yet satisfy, in its
own harness, ideally with a tracking issue linked to each entry so the list stays
temporary and reviewable rather than a silent quarantine. When the implementation
conforms, the entry is removed. That list is a consumer-side convenience; it does
not live here, and these fixtures stay implementation-neutral.
