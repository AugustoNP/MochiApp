# decisions

newest first. one entry per choice someone might want to undo.

## 2026-10-12: tdd evidence via docs, testing.md and commit history
- decision: strict tdd is documented in architecture.md section 5 and summarized in a public TESTING.md at the repo root. the failing test and the passing implementation are committed separately by the user where practical. ai assistance writes the failing test first and shows it failing; it never commits.
- why: ci cannot prove test-first; history, docs and review are the evidence a recruiter or contributor can inspect.
- alternatives rejected: docs only (nothing visible at repo root).

## 2026-10-12: coverage reported everywhere, gated on domain only
- decision: ci reports coverage for the whole project (generated code excluded) and fails if domain coverage is below a threshold (proposed 90%, still open).
- why: domain holds the rules and is pure, so a high bar is cheap and meaningful; a global gate pushes low-value tests for ui glue.
- alternatives rejected: no gate; one global threshold.

## 2026-10-12: strict tdd in every layer, including view models
- decision: red-green-refactor for domain, data and view models. widgets get widget tests of behavior (not pixels). no production code without a failing test; bug fixes start with a reproducing test.
- why: the layering makes each layer testable in isolation, so test-first costs little; it is the project's quality story.
- alternatives rejected: strict only in domain/data with ui test-after; strict for widgets too (high maintenance while ui churns).

## 2026-10-12: classical test doubles, no mocking package
- decision: hand-written in-memory fakes for repositories in view-model tests; real in-memory drift (`NativeDatabase.memory()`) for data tests. tests assert outcomes, not calls. fakes share a contract test with the real repository.
- why: tests survive refactors (mockist tests couple to implementation); no extra dependency; matches the pure-domain design.
- alternatives rejected: mock package with call verification; case-by-case mixing.

## 2026-10-12: casual defaults, opt-in depth
- decision: target users are casual and intermediate lifters; first launch and default screens show the simple log, advanced fields (rir, set types, volume) are opt-in.
- why: matches the "simple, reliable, not scary" goal while keeping rp-style depth available.
- alternatives rejected: intermediate-only defaults (scares casual users); casual-only (blocks the growth path).

## 2026-10-12: no absence rule in v1
- decision: if an exercise is not done for a long time, progression state is unchanged; user can set a baseline override.
- why: keeps rules predictable and testable; real use will show whether auto-reduce or ask-on-return is needed.
- alternatives rejected: auto-reduce after n weeks (another tunable rule); prompt on return (needs ui, deferred).

## 2026-10-12: first logged weight becomes the baseline
- decision: an exercise with no state takes its first logged working weight as baseline, with target_reps = rep_floor. no setup prompt.
- why: zero friction; the app learns from what the user actually lifted.
- alternatives rejected: required starting-weight prompt (setup friction).

## 2026-10-12: changenotifier view models, manual constructor injection
- decision: view models extend `ChangeNotifier`; dependencies passed by constructor from one composition root in `main.dart`; no state-management package in v1.
- why: follows flutter's documented architecture pattern, no dependency to maintain; view models hold no rules, so riverpod or bloc can replace them later without touching domain.
- alternatives rejected: riverpod or bloc from the start (extra concepts before the need is proven). revisit if manual wiring becomes painful.

## 2026-10-12: unit chosen at first launch
- decision: first run asks kg or lb; stored weights stay in kg. default increment 2.5 kg or 5 lb, configurable per exercise.
- why: works for any audience with one question; no locale guessing.
- alternatives rejected: locale-based guess (sometimes wrong); kg-only or lb-only v1.

## 2026-10-12: catalog seeded from an open dataset, then curated
- decision: start from an open-licensed exercise dataset, clean it, and own the curated result as the bundled seed. users add custom exercises on top.
- why: faster than writing hundreds of entries by hand, and curation keeps quality and muscle tags consistent.
- alternatives rejected: pure hand-written seed (slow); users build everything (poor first run).
- caveat: dataset licence must be checked before import and compatible with the chosen app licence (still undecided).

## 2026-10-12: licence deferred
- decision: no licence chosen yet; must be chosen before the first public release. check dataset licence compatibility when it is.
- why: no impact on early design; avoids picking before the catalog source is settled.

## 2026-10-12: feature-first modules with domain/data/presentation inside
- decision: lib/core for shared code; lib/features/<feature>/{domain,data,presentation}. domain is pure dart; features talk only through each other's domain interfaces. `progression` is its own feature (revisit if it never grows beyond one engine).
- why: keeps the easy-to-debug module feel of the previous project while keeping rules independent of drift/flutter, so the engine is testable without a db. matches flutter's recommended layers (views, view models, repositories, services).
- alternatives rejected: classic mvc (web idiom, not flutter's recommendation); layer-first folders (a feature is spread across the tree); services that take the database directly (rules coupled to storage).

## 2026-10-12: progression is a pure, versioned strategy
- decision: progression is a pure function behind a small strategy interface, tagged with `algorithm_version`. double progression is strategy #1. a clean pass at the ceiling is counted by `ceiling_passes`, not a general success streak.
- why: deterministic replay on every device, easy property testing, volume-aware strategies can be added without touching the core. the old shared streak could bump weight after one session at the ceiling.
- alternatives rejected: algorithm logic inside repositories or UI; stored streak that mixes climb and ceiling sessions.

## 2026-10-12: simple core schema, volume derived
- decision: core log is workout -> exercise -> sets with nullable depth fields (`rir`, `set_type`). weekly volume per muscle is a query over sets and exercise_targets. programs and mesocycles are additive tables later.
- why: simple by default, deep on demand; RP-style planning does not require a rewrite.
- alternatives rejected: building program/mesocycle tables now; storing volume.

## 2026-10-12: custom exercises in v1
- decision: users can create exercises (`is_custom`), name required, muscle tags optional.
- why: a catalog-only app is not usable; untagged exercises just skip volume stats.
- alternatives rejected: catalog-only v1.

## 2026-10-12: android first, platform-agnostic core
- decision: v1 targets android; domain and data layers must not depend on flutter UI or android-only packages.
- why: cheapest to test and ship now; ios and web stay cheap later.
- alternatives rejected: android + ios or web in v1.

## 2026-10-12: backend deferred, custom api + postgres planned
- decision: no backend in v1. when sync is built, use a small custom api with postgres.
- why: portfolio value and no vendor lock-in; cost is owning auth and ops.
- alternatives rejected: supabase (faster, less backend showcase); building backend before the app works.

## 2026-10-12: local-first, sync later; progression state not synced
- decision: ui reads only the local drift/sqlite db. sync is a later milestone, but schema keeps uuids, `updated_at`, `deleted_at`. `progression_state` is a derived cache recomputed per device and never synced. local db has no `user_id`; the sync layer attaches the account.
- why: ship a usable app first; removes the draft's two-mechanism conflict resolution; editing history just triggers a replay.
- alternatives rejected: full offline-first sync from day one (high cost before the logger is proven); online-first (fails in gyms without signal, retrofitting offline is costly); syncing the metrics cache.
