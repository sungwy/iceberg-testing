# Contributing to iceberg-testing

A fixture encodes one agreed reading of the spec. Adding or changing one is a
pull request, reviewed and approved like any other. This guide covers how the
fixtures are organized, what a contribution must include, and what happens when an
expected value is debated or changed.

## How the fixtures are organized

A **surface** is one spec behavior verified by one assertion. Its directory holds
the inputs and their co-located expected results, and its README defines the single
assertion a consumer implements against them. The assertion is what defines a
surface; it rarely changes once written.

- `table-spec/types/` is one surface: parse a type string, compare the parsed
  fields, and assert the canonical serialization. `decimal/`, `fixed/`, and
  `geometry/` only partition its cases by type; they share the one assertion.
- `table-spec/transforms/bucket/` is one surface: hash a typed value with the
  spec's 32-bit Murmur3 and compare. It sits under the `transforms/` grouping
  alongside future `truncate/` and temporal surfaces.
- `table-spec/delete-formats/` is one surface: decode a delete file's columns by
  field-id. `positional/` and `equality/` only partition its cases by delete kind;
  they share the one assertion.

A **grouping directory** only organizes and documents; it holds no fixtures of its
own. `table-spec/` groups the surfaces. A surface that grows several distinct
assertions can become a grouping directory whose subdirs are each a surface; until
then, partitions like `types/decimal/` or `delete-formats/positional/` are just
case folders under one surface.

Where a behavior differs by format version, the version is part of the path (for
example `metadata/v2` and `metadata/v3`). There is no global index; the
per-directory README is the contract.

## Adding to a surface, or making a new one

Decide by the assertion:

- **Same behavior, same assertion -> add a case.** A new edge case is a new line in
  a `cases.jsonl`, a new input-plus-expected file set, or an edit to an existing
  binary and its sidecar. The surface README does not change. This is the common
  contribution.
- **A different assertion -> add a surface.** If the new case needs a different
  assertion than the surface defines, it is a new surface with its own README (for
  example a future predicate-evaluation surface for filter semantics over nulls).
- **A different spec behavior -> add a surface.**

If you find yourself bending a surface's assertion to fit a new case, that is the
signal to make a new surface instead.

## What a contribution must include

- The input artifact and its co-located expected file, named so the input maps to
  its expected file by convention.
- The spec clause the fixture pins, named in the expected file.
- An expected value that encodes the agreed reading of the spec, not one
  implementation's current behavior.
- Generic wording. Fixtures and READMEs state the spec behavior a case pins; they do
  not name which implementation currently diverges or its tracker issue, because
  those date as implementations change. Record that correlation in the pull request,
  not in the committed files.
- For a binary fixture, a `<file>.expected.json` sidecar with the logical decode,
  never a byte expectation. Generation is provenance: record it in the PR, not as a
  committed build step.

A new surface also needs a README that states:

- the input format and the expected-file format,
- the single assertion to implement: what to compare, and for data surfaces that it
  is logical values only, never bytes,
- any encoding discipline the expected values follow,
- why the surface exists: the spec behavior it pins and the kind of divergence it
  guards against.

## Deriving expected values

An expected value is the answer the spec fixes, computed from the spec — not copied
from an implementation's output. An implementation can share the bug with the value
you would copy, so a fixture seeded that way certifies the bug instead of catching
it. Compute the value from the spec, and where the spec provides reference vectors
(for example the Appendix B hash vectors), verify your computation reproduces them
before extending to new inputs.

The value of a fixture is in its boundary cases. The spec's own examples are the
easy path that most implementations already pass; the drift lives at the edges,
where a value's serialization or encoding differs across languages. When adding
cases, deliberately hunt for inputs that stress how values are encoded, and pin
them:

- numbers at byte boundaries (a negative whose minimal two's-complement is a byte
  shorter, like decimal `-1.28`; the min/max of a width),
- non-ASCII and non-BMP text (multi-byte UTF-8, emoji, combining characters),
- temporal values before the epoch and at sub-second precision,
- null, empty, and `0.0`/`-0.0` values.

A boundary case that every implementation gets right today still guards against
regressions and new implementations; one that an implementation gets wrong today is
exactly the drift this repository exists to surface.

## Additions versus corrections

Every change is one of two kinds, and the PR should say which:

- **Addition.** A new case or surface. A consumer that was green did not test it
  before, so it may newly fail, but nothing it relied on changed.
- **Correction.** A change to an existing expected value. It can flip a case a
  consumer was passing, so it is a breaking change for that consumer. Use it when
  the agreed reading was wrong or the spec was clarified.

Writing or changing an expected value forces a concrete reading of the spec, so a
reviewer who reads it differently says so on the PR. If the behavior is genuinely
debated rather than merely unfamiliar, do not invent an expected value and merge it.
The question goes to `dev@iceberg` (or an IIP), the spec decides, and the fixture
lands afterward. This is the norm any implementation already follows at an undecided
spec point; the fixture only surfaces it in one place, earlier.

## How changes reach consumers

Each consumer pins this repository at a commit (a git submodule) and runs the
fixtures in its own CI. A change reaches a consumer only when it bumps that pin,
which it does manually and at its own pace. There is no central gate, and no
consumer is obligated to adopt a fixture.

- An **addition** a consumer does not yet satisfy can sit on that consumer's local
  skip list until it conforms; enrolling never requires passing everything.
- A **correction** flips a previously passing case. The consumer sees it on the next
  bump and adopts when ready, optionally skipping the case meanwhile.

Because the PR here states whether a change is an addition or a correction, a
consumer bumping the pin knows whether a newly red case is expected.
