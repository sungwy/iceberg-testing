# iceberg-testing

A standalone, language-neutral repository of conformance fixtures for Apache
Iceberg's implementations.

The Iceberg spec is prose, and each implementation parses and serializes it
independently. Two implementations can read the same clause differently, so an
artifact written by one may be read differently by another. This repository
provides shared fixtures that pin one reading: each fixture is a static input
artifact paired with the language-neutral expected result the spec fixes for it,
co-located in the same directory. A consumer runs the fixtures against its own
implementation to check it against that shared reading.

## What this repository is

It is a set of files: input artifacts and their expected values, plus a README per
surface that documents the assertion to run against them.

It ships no harness and no per-language code. Integrating the fixtures into a test
suite is each consumer's responsibility. Keeping the accessors and the enrollment
decisions in the consumer is what keeps the fixtures language-neutral.

## Layout

```
table-spec/
  types/            # parse + exact re-serialize  (decimal, fixed, geometry)
  transforms/       # partition transforms  (bucket: the 32-bit hash KAT)
  delete-formats/   # logical decode of delete files  (positional, equality)
```

There is no global index. Each surface directory carries a README that states its
input format, its expected-file format, and how its fixtures are used. Each
expected file names the spec clause it pins.

## Using the fixtures

A consumer writes its own suite in its own test framework and runs it alongside its
existing unit tests.

1. Add this repository as a git submodule pinned to a commit:

   ```
   git submodule add https://github.com/sungwy/iceberg-testing.git iceberg-testing
   git -C iceberg-testing checkout <commit>
   ```

2. For each surface adopted, read that directory's README for the input format, the
   expected-file format, and the assertion to implement. Walk the fixtures by
   directory and naming convention; there is nothing to register centrally.

3. Keep a local skip list, in the consumer's repo, for cases the implementation
   does not yet satisfy. Adopting a surface does not require passing every case.
   That list is a consumer-side concern and is not part of these fixtures.

Updating to newer fixtures is a manual change of the pinned commit, made by each
consumer when it chooses to. There is no central gate.

## Contributing

A new fixture or a changed expectation is a pull request, reviewed like any other.
See `CONTRIBUTING.md` for how the fixtures are organized, what a contribution must
include, and how additions and corrections reach consumers.

## License

Apache License 2.0. See `LICENSE.txt`.
