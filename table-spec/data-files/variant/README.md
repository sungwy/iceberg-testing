# table-spec/data-files/variant

Shredded-variant Parquet files paired with their expected logical decode, keyed by
Iceberg field-id. A variant column may be written *shredded*: frequent fields are
stored in typed Parquet columns (`typed_value`) alongside the residual `value`
binary. This surface pins the reader side: resolve the variant column by its
field-id, reassemble it, and decode each row.

## Why this surface

The shredded layout itself is a Parquet concern that `parquet-testing` owns. What
this surface pins is the Iceberg part: the shredded group is the value of an Iceberg
field-id, and a reader must resolve it by field-id, not by name or position (the same
contract `delete-formats` pins, and the one `equality_ids` / iceberg-go#880 turns on).
In these files the top-level columns carry `PARQUET:field_id` (`id` = 1, `var` = 2);
the inner shred columns carry none.

Reassembly is authored in the Parquet spec `VariantShredding.md` (each `clause` points
at a section) and the value's physical type in `VariantEncoding.md`. The comparison is
spec-semantic: normalize where the spec leaves representation open, but retain the
physical type where it pins one, so `int32(34)` and `int64(34)` do not both collapse to
a bare `34` and let a wrong-width read pass.

## Sidecar format

Each `case-NNN.parquet` has a `case-NNN.parquet.expected.json` sidecar, as in
`delete-formats/`.

| field | meaning |
| --- | --- |
| `data-file` | the shredded Parquet file this sidecar describes |
| `iceberg-schema` | `field-id`, `name`, `type` per top-level column, read from the file |
| `accept` | `true` if a conformant reader reads it, `false` if it rejects the layout |
| `decoded-rows` | present when `accept`: one entry per row, keyed by field-id |
| `reason` | present when not `accept`: the spec invariant the layout violates |
| `clause` | the spec behavior the case pins |
| `note` | what the case pins |

A `decoded-rows` entry maps the variant field-id to that row's type-annotated decode; a
physical/SQL-null row is a bare `null`, distinct from a present variant null
(`{"type": "null"}`). A decoded value is a tree:

| node | shape |
| --- | --- |
| object | `{"type": "object", "fields": {<name>: <node>}}` |
| array | `{"type": "array", "elements": [<node>]}` |
| primitive | `{"type": "<physical-type>", "value": <scalar>}` |

`<physical-type>` is the `VariantEncoding.md` primitive name (`int8`..`int64`,
`decimal4/8/16`, `float`, `double`, `boolean`, `string`, `binary`, `date`, the
timestamp variants, `uuid`). A 64-bit integer is a JSON string (it can exceed 2^53);
narrower integers are JSON numbers.

## Assertion to implement

For `accept: true`: load the file under `iceberg-schema`, read the variant column by
its field-id, reassemble and decode each row to the type-annotated form above, and
assert the per-row results equal `decoded-rows`. Logical values only, never bytes.

For `accept: false`: reading the variant column must raise an error. The `reason` is
documentation; any read error passes.

## Provenance

The Parquet files are lifted unmodified from
[apache/parquet-testing](https://github.com/apache/parquet-testing) `shredded_variant/`
at commit `a3d96a65e11e2bbca7d22a894e8313ede90a33a3`; the `iceberg-schema` field-ids are
read from each file's `PARQUET:field_id`. Each decoded value was derived with one
implementation and corroborated value- and type-for-type against parquet-testing's
Java-produced record, so two implementations agree on every pinned value. Its raw
`variant` field is a Java `toString()` and is not cross-language comparable, so it is
not used. The `-INVALID` files (`case-043`, `case-084`) are held out: parquet-testing
labels them invalid yet supplies a value and implementations disagree, so per the
contribution rule the ambiguity is escalated to the spec rather than pinned here.
