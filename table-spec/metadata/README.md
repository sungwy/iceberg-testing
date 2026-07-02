# table-spec/metadata

Table `metadata.json` files paired with their expected typed invariants. This is a
version-in-path surface: the same struct has a version-dependent shape, so the format
version is a directory level (`v2/`, `v3/`), unlike the feature surfaces (`types/`,
`transforms/`, `delete-formats/`, `data-files/`) whose behavior does not vary by
version.

## Why version in the path

The spec authors table metadata as one struct whose required fields and members
differ by format version (its field table has `v1 | v2 | v3` columns). Splitting by
version keeps each expected file a flat statement of one version's reading and makes
the delta between versions explicit. The two files here are the *same table* at v2
and v3; the only differences are the v3 additions:

- `next-row-id` (row lineage), required in v3, absent in v2.
- `initial-default` / `write-default` on a schema field (default values), v3 only.

A reader that carries a v2 default for a v3 field, or misses `next-row-id`, diverges
here.

## Sidecar format

Each version directory holds the `metadata.json` under test and an `expected.json`
beside it. The binary surfaces (`delete-formats/`, `data-files/`) append
`.expected.json` to the data-file name, but that doubles the extension for a JSON
input (`metadata.json.expected.json`), so a JSON-input case pairs `metadata.json`
with `expected.json` in its own directory instead.

| field | meaning |
| --- | --- |
| `data-file` | the metadata JSON file this sidecar describes |
| `format-version` | the format version under `v2/`, `v3/` |
| `accept` | `true` if a conformant reader parses it, `false` if it rejects it |
| `invariants` | spec-field name (or path) to expected value; typed invariants, not a byte decode |
| `absent` | spec fields that must not be present at this version |
| `clause` | the spec behavior the case pins |
| `note` | what the case pins |

`invariants` are compared as decoded values through native accessors, never a byte
round-trip: `metadata.json` field order and optional-field omission are not
spec-fixed, so only the parsed values are pinned.

## Assertion to implement

For `accept: true`: parse `data-file`, and for each `invariants` entry assert the
parsed value equals it (mapping the spec-field name to your reader's accessor). Assert
each `absent` field is not present. For `accept: false`: parsing must raise an error.

## Provenance

The metadata files are the minimal valid v2 and v3 examples from Iceberg Java's core
test resources (`TableMetadataV2ValidMinimal.json`, `TableMetadataV3ValidMinimal.json`),
unmodified. The invariants are read directly from those files. Field order and
whitespace carry no meaning and are not asserted.
