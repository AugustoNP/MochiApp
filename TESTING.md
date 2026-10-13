# Testing

MochiApp uses strict test-driven development. The full rules are in [docs/architecture.md](docs/architecture.md), section 5. This is the short version.

## The loop

1. **Red:** write one failing test for the next small behavior and watch it fail for the expected reason.
2. **Green:** write the least code that makes it pass.
3. **Refactor:** clean up while everything is green.

No production code without a failing test. Bug fixes start with a test that reproduces the bug.

## What is tested where

| Layer | Approach |
|---|---|
| domain (rules, progression) | plain Dart unit tests: table-driven, property tests, golden histories |
| data (Drift repositories) | real in-memory Drift database per test; migration tests per schema version |
| view models | unit tests with hand-written fake repositories; assert resulting state |
| widgets | widget tests of behavior, not pixels |
| integration | a few critical flows on an emulator |

## Conventions

- Classical style: real collaborators or simple fakes; no mocking package. Assert outcomes, not calls.
- Fakes live in `test/fakes/`. Shared helpers live in `test/support/`.
- `test/` mirrors `lib/`: `test/features/<feature>/<layer>/..._test.dart`.
- Time and IDs are injected, never read from globals in domain code.
- CI runs format, analyze and tests with coverage. The domain layer has a coverage gate.

## Commit convention

Where practical, commit the failing test and the implementation that makes it pass as separate commits, so the history shows the test coming first.
