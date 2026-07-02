# table-spec/data-files

Encoded data files paired with their expected logical decode.

A data file is a static artifact: decode its columns to logical rows and compare
the values. This grouping holds the data-file surfaces; each subdir is one surface
with its own README and assertion.

| dir | what it pins |
| --- | --- |
| `variant/` | the shredded-variant Parquet layout (v3): reassemble the shredded `value`/`typed_value` columns and decode the variant |

`primitives/` and `nested/` (plain and nested non-variant column decode) are
additive, each as its own surface.
