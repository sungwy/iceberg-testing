# table-spec/manifests

Manifest Avro files paired with their expected decode, read by field-id. Like
`metadata/`, this is a version-in-path surface: the `manifest_entry` and `data_file`
structs have version-dependent columns (the spec's field table has `v1 | v2 | v3`),
so the format version is a directory level (`v2/`).

## Why this surface

Manifest encoding is where the sharpest cross-implementation drift has surfaced,
because a reader resolves the Avro fields by field-id and each field's physical type
is spec-fixed. The pinned case is `equality_ids`: the spec types it `list<136: int>`,
but iceberg-go once wrote the elements as Avro `long`, and Java threw a
`ClassCastException` reading them (iceberg-go#880). A flat decode does not catch this
- `int(1)` and `long(1)` both decode to `1` - so the fixture retains the physical
type: `equality_ids` (field 135) carries `element-type: int` for its element
(field 136). That is the spec-semantic rule (normalize where the spec is open, retain
the physical type where it is pinned) on the manifest side, mirroring `data-files/`.

## Sidecar format

Each `<name>.avro` has a `<name>.avro.expected.json` sidecar, as in `delete-formats/`
and `data-files/`.

| field | meaning |
| --- | --- |
| `data-file` | the manifest Avro file this sidecar describes |
| `format-version` | the manifest version under `v2/` |
| `accept` | `true` if a conformant reader reads it, `false` if it rejects the file |
| `entries` | the decoded `manifest_entry` records, keyed by field-id |
| `clause` | the spec behavior the case pins |
| `note` | what the case pins |

An `entries` record is keyed by manifest field-id (`0` status, `1` snapshot_id,
`2` data_file, `3` sequence_number); the nested `data_file` is keyed by its own
field-ids (`134` content, `100` file_path, `101` file_format, `102` partition,
`103` record_count, `104` file_size_in_bytes, `135` equality_ids). 64-bit integers are
JSON strings. `equality_ids` is a typed node `{element-field-id, element-type, value}`
so the element's physical type is pinned, not just its value.

## Assertion to implement

For `accept: true`: read the manifest with your reader, resolve each field by
field-id, and assert the decoded records equal `entries`. For `equality_ids`, assert
both the values and that the element type is `int` (field 136), not `long`.

For `accept: false`: reading the manifest must raise an error.

## Provenance

The `equality-deletes.avro` file was written with iceberg-go's manifest writer as a
v2 manifest holding one equality-delete entry, then verified two ways before use: the
OCF header schema types the `equality_ids` element as `int` (`{"element-id":136,
"items":"int","type":"array"}`) matching the spec's `list<136: int>`, and the file
decodes back to the values pinned here. The expected element type is taken from the
spec, not from the writer.
