# table-spec/transforms/bucket

The 32-bit hash that the `bucket[N]` partition transform is built on.

`bucket[N](x) = (hash(x) & 0x7fffffff) % N`. The hash is the spec-determined part,
and this surface pins it. The spec's Appendix B provides reference vectors per type.

## Why this surface

Bucket hashing must agree across implementations or the same value lands in
different partitions across engines. It is sensitive to how each type is serialized
to the bytes that are hashed — the minimal two's-complement of a decimal, the full
UTF-8 encoding of a string — which is exactly where readers diverge.

## Format: `cases.jsonl`

| field | meaning |
| --- | --- |
| `id` | stable case id |
| `type` | the Iceberg type of the value |
| `value` | the input value as a string: decimal/date/time/timestamp/uuid as their text form, fixed/binary as hex |
| `hash` | the expected signed 32-bit hash |
| `clause` | the spec clause pinned |
| `note` | what the case pins |

## Assertion to implement

Serialize `value` per the Appendix B hashing rules for its type — int, long, date,
time, and timestamp as an 8-byte little-endian long (date is days, time/timestamp
are microseconds); decimal as the minimal two's-complement big-endian of the
unscaled value; string as UTF-8; uuid as 16 bytes big-endian; fixed and binary
as-is. Compute the 32-bit Murmur3 (x86 variant, seed 0) and assert it equals
`hash`. `bucket[N]` is then `(hash & 0x7fffffff) % N`.

## Expected values are derived from the spec

The hashes are computed from the spec algorithm and verified against the spec's own
Appendix B vectors, never copied from an implementation. This is not a detail: an
implementation can share a bug with the value you would copy, so a fixture seeded
that way would certify the bug instead of catching it. The boundary cases (decimal
`-1.28`, `-327.68`, the non-BMP string) exist for the same reason: they pin
encodings where a naive implementation diverges, with expected values taken from the
spec.
