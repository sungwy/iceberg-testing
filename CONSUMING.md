# Consuming the fixtures as a submodule

A worked example of wiring this repository into an implementation's test suite. The
fixtures ship no harness and no per-language code; a consumer pins the repo, walks the
surfaces it adopts, and asserts in its own framework.

## 1. Add and pin

Add the repository as a git submodule and pin it to an immutable commit (the same
step as the root README's "Using the fixtures", repeated here for a complete example):

```
git submodule add https://github.com/sungwy/iceberg-testing.git testing/iceberg-testing
git -C testing/iceberg-testing checkout <commit>
git add .gitmodules testing/iceberg-testing
git commit -m "Pin iceberg-testing fixtures"
```

The pin is reproducibility, not propagation: a corrected fixture reaches the consumer
only when it bumps the commit, at its own pace. Dependabot can open the bump PRs.

## 2. Walk a surface

Each surface is files plus a README defining one assertion. Read that README, then
walk the directory by convention. For `table-spec/metadata`, the version is a path
level, so a consumer picks the versions it supports:

```
testing/iceberg-testing/table-spec/metadata/
  v2/  metadata.json  expected.json
  v3/  metadata.json  expected.json
```

## 3. Assert in your own framework

The consumer parses the data file with its own reader and checks the sidecar's typed
invariants through native accessors. Sketch, in Go against the `metadata` surface
(`table.ParseMetadataBytes`, `Version`, and `LastSequenceNumber` are real iceberg-go
API; the absent-check reads the raw JSON because the typed struct zero-fills):

```go
for _, v := range []string{"v2", "v3"} {
    dir := filepath.Join(root, "table-spec/metadata", v)
    exp := readExpected(filepath.Join(dir, "expected.json"))

    raw, err := os.ReadFile(filepath.Join(dir, "metadata.json"))
    require.NoError(t, err)
    meta, err := table.ParseMetadataBytes(raw)
    require.NoError(t, err)

    // invariants: spec-field -> expected value, mapped to the reader's accessors
    require.EqualValues(t, exp.Invariants["format-version"], meta.Version())
    require.EqualValues(t, exp.Invariants["last-sequence-number"], meta.LastSequenceNumber())

    // absent: v3-only keys must not appear in the raw JSON at v2
    var doc map[string]any
    require.NoError(t, json.Unmarshal(raw, &doc))
    for _, f := range exp.Absent {
        _, present := doc[f] // top-level keys; nested paths (schemas[0]...) need a walk
        require.False(t, present, "%s must be absent at %s", f, v)
    }
}
```

Each implementation writes this in its own runner (`go test`, `pytest`, `cargo test`,
JUnit, `ctest`) and maps the spec-level field names to its native accessors.

## 4. Enroll incrementally

A consumer adopting a surface incrementally keeps a local skip list, in its own repo,
of cases it does not yet satisfy, ideally with a tracking issue per entry. Adopting a
surface never requires passing every case, and the skip list is a consumer-side
concern that does not live here.
