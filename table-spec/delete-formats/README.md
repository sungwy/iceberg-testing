# table-spec/delete-formats

Row-level delete files paired with their expected logical decode.

A delete file is a static artifact: decode its columns to logical rows, keyed by
Iceberg field-id. This surface pins that decode. It does not apply the deletes to a
data file; computing surviving rows is a read-time operation, not an artifact
decode, and is out of scope here.

Parquet, ORC, and Avro admit many valid serializations, so a consumer asserts
logical decode equality only: decode the file, map columns to field-ids, and
compare the values. Never assert a byte round-trip. Each `<name>.parquet` is paired
with a `<name>.parquet.expected.json` sidecar describing its expected logical
decode.

## Kinds

The two delete-file kinds share one assertion (decode by field-id) and differ only
in which field-ids the columns carry. Each is a subdir.

| dir | what it pins |
| --- | --- |
| `positional/` | a positional delete file: `file_path` and `pos` at the reserved field-ids `2147483546` and `2147483545` |
| `equality/` | an equality delete file: each column carries its table field-id, so equality values are read by field-id, not by column position |

## Sidecar format

| field | meaning |
| --- | --- |
| `clause` | the spec clause pinned |
| `delete-file` | the binary this sidecar describes |
| `schema[]` | `field-id`, `name`, `type` per column |
| `decoded-rows` | the logical decode, rows keyed by field-id |

Values are encoded with the discipline that keeps the JSON unambiguous across
languages: rows are keyed by field-id, 64-bit integers are JSON strings, `null` is
JSON `null`, and binary is uppercase hex.

## Assertion to implement

Decode `delete-file` with your reader, mapping each column to its Iceberg field-id
(from `PARQUET:field_id`), and encode values per the discipline above. Assert the
result equals `decoded-rows`. Logical values only, never bytes.

## Provenance

The binaries were generated with pyarrow. The exact bytes are not significant: the
assertion is logical decode only, so a regenerated file need not be byte-identical,
and a different writer or library version is fine as long as it decodes to the
values in the sidecar. The script that produced these files lives in the pull
request that introduced them.
