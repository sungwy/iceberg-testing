# table-spec/transforms

Partition transforms. Each transform is a pure function the spec fixes exactly, so
each is its own surface with its own assertion: apply the transform to a typed
input and compare the output.

| dir | what it pins |
| --- | --- |
| `bucket/` | the 32-bit hash that `bucket[N]` is built on (Appendix B) |

`truncate/` and the temporal transforms (`year`, `month`, `day`, `hour`) are
additive, each as its own surface.
