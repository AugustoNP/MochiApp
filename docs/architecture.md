# Architecture: MochiApp

**Status:** living document. Update it whenever a decision changes, and log the change in `decisions.md`. No code exists yet.

## 1. Goals and non-goals

MochiApp is a free, open-source gym tracker (Flutter) inspired by Hevy and Renaissance Periodization. It is built by one person as a portfolio piece that should behave like a real, maintainable product.

**Goals**
- Fast, simple logging that works with no signal. A casual user should never meet complexity they did not ask for.
- A suggestion engine that starts simple (double progression) and can grow toward volume-aware, RP-style planning without rewriting the core.
- Depth available on demand: RIR, set types, per-muscle volume, for users who care.
- The user owns their data: full export at any time.

**Non-goals for v1**
- Social features, coaching marketplace, billing.
- Field-level or CRDT sync merging.
- Wearable integrations.

**Target user:** casual and intermediate lifters alike. Defaults are casual (simple log, advanced fields hidden); depth is opt-in.

**Platforms:** Android first. Domain and data layers must not import Flutter UI or Android-only packages, so iOS and web remain cheap to add later.

## 2. Principles

1. **Local-first.** The UI reads and writes only the local SQLite database (Drift). No screen waits on the network. The server, when it exists, is a replica and a sync point, not a dependency.
2. **Derive, don't store.** Anything computable from logged sets (weekly volume per muscle, progression state, personal records) is computed, not synced. Stored caches are rebuildable.
3. **Simple core, optional depth.** The core log is workout -> exercise -> sets. Advanced fields are nullable and ignored by default.
4. **Pure domain core.** Progression logic is a pure function with no database, clock, or network access.
5. **Dependencies point inward.** `domain` (entities, progression) knows nothing of Drift, Flutter, or HTTP. `data` implements repository interfaces defined in `domain`.

```
 presentation (views + view models)  ->  domain (entities, repository interfaces, progression)
                                                  ^
                                                  |
                                   data (Drift repositories, later: sync client)
```

### 2.1 Project structure

Feature-first modules; each owns its layers. Flutter's architecture guide describes the layers (views, view models, repositories, services) but does not prescribe folder layout, so this layout is our own choice. Classic MVC was rejected; the layering follows the MVVM-style guidance.

```
lib/
  core/                      shared: db setup, theme, formatters, errors
  features/
    workout_log/
      domain/                entities + repository interfaces (pure Dart)
      data/                  Drift tables + repository implementations
      presentation/          screens, widgets, view models
    exercises/               same three folders
    progression/
      domain/                the pure engine; data/presentation only if needed later
    stats/                   volume queries, later
```

Rules:
1. `domain` imports no Flutter, no Drift, and nothing from `data` or `presentation`.
2. `data` and `presentation` may import their own feature's `domain`.
3. Features reach each other only through another feature's `domain` interfaces, never its tables or widgets.
4. No use-case layer until a screen needs logic that spans several repositories.
4a. View models extend `ChangeNotifier` (Flutter's documented pattern); dependencies are passed by constructor from one composition root in `main.dart`. No state-management package in v1. View models hold no business rules, so a package can be adopted later without touching `domain`.
5. Enforce rules 1 and 3 with an automated import check in CI (tool not yet chosen).

Lesson from a previous project: a service that takes the database object directly mixes business rules with storage and cannot be tested without a database. Here, rules live in `domain`, and only repositories in `data` touch Drift.

## 3. Data model (local database)

All IDs are UUIDs generated on the device. Every row has `created_at` and `updated_at`. User-log rows have `deleted_at` (tombstone); the client never runs `DELETE` on them. These are cheap now and are what make later sync possible.

The local database is single-profile: there is no `user_id` column. When sync is added, the account is attached by the sync layer (the server scopes rows by authenticated user). See `decisions.md`.

### 3.1 Catalog (shipped with the app, updated by app releases in v1)

| Table | Key fields | Notes |
|---|---|---|
| `muscle_groups` | `id`, `name`, `parent_id` | self-referencing hierarchy (Legs -> Quads) |
| `exercises` | `id`, `name`, `description`, `image_url`, `is_custom`, `deprecated_at`, `replacement_exercise_id` | custom exercises are rows with `is_custom = true`, owned by the user |
| `exercise_targets` | `(exercise_id, muscle_group_id)`, `intensity` | which muscles an exercise trains; optional for custom exercises |

Catalog rules:
- Catalog rows are never hard-deleted and UUIDs are never reused, because logs reference them forever.
- Retire an exercise by setting `deprecated_at` (and optionally `replacement_exercise_id`). Deprecated exercises disappear from pickers but stay joinable; history is never rewritten.
- A custom exercise needs only a name. Muscle tags are optional; untagged exercises simply do not count toward per-muscle volume.

### 3.2 User log

| Table | Key fields | Notes |
|---|---|---|
| `workouts` | `id`, `date`, `note`, `readiness_score` (nullable), `finished_at` | `readiness_score` is stored for future use; v1 progression ignores it |
| `workout_sections` | `id`, `workout_id`, `exercise_id`, `position`, `progression_style` | one exercise within a workout |
| `sets` | `id`, `workout_section_id`, `set_order`, `set_type`, `target_reps`, `actual_reps`, `target_weight`, `actual_weight`, `rir` (nullable) | `set_type`: `working` (default), `warmup`, later `drop`, `failure`. Only `working` sets feed progression. |

Weights are stored in kilograms as a decimal; the user's unit preference (kg/lb) is a display setting. `rir` is the optional depth field: reps in reserve.

### 3.3 Settings and derived state

| Table | Key fields | Notes |
|---|---|---|
| `exercise_settings` | `exercise_id`, `rep_floor`, `rep_ceiling`, `increment`, `baseline_override_weight`, `override_from` | user-authored configuration, with sensible defaults. This is the only progression data that would ever need to sync. |
| `progression_state` | `exercise_id`, `weight`, `target_reps`, `ceiling_passes`, `stall_count`, `algorithm_version` | **derived cache.** Rebuildable by replaying `sets` through the algorithm. Never synced. |

Because `progression_state` is a pure function of (`exercise_settings`, working sets in order, `algorithm_version`), it is recomputed in these cases:
- the user edits or deletes a past set or workout,
- `algorithm_version` changes after an app update,
- the user clears or sets a baseline override.

Replay starts at `override_from` when an override exists, using `baseline_override_weight` as the starting weight.

### 3.4 Volume and RP-style planning (future, additive)

Weekly volume per muscle needs no new table:

```
volume(muscle, week) = count of working sets in that week
                        joined through exercise_targets to that muscle (weighted by intensity)
```

Later additions are new tables that reference the core (`programs`, `mesocycles`, `planned_workouts`, per-muscle volume landmarks). The core log does not change.

## 4. Progression engine

### 4.1 Contract

```
suggest(settings, state, history_since_last_change) -> state          // pure
replay(settings, working_sets_in_order)             -> state          // pure, folds suggest
```

- No I/O, no clock, no randomness. Deterministic, so every device computes the same state.
- `algorithm_version` is stored with the state. Changing any rule bumps it, which triggers a replay.
- `progression_style` selects a strategy implementing this contract. `double_progression` is specified below; others (linear, volume-aware) follow the same contract.

### 4.2 Double progression

Plain language: add reps at a fixed weight until the top of the rep range. Once the user cleanly passes at the top for two sessions, raise the weight and drop back to the bottom of the range. Three failed sessions in a row at the same target trigger a small deload.

State: `weight`, `target_reps`, `ceiling_passes`, `stall_count`. Config: `rep_floor`, `rep_ceiling`, `increment`.

**Clean pass** (evaluated per exercise per session, working sets only):

```
clean = all working sets have actual_reps >= target_reps   at weight >= prescribed weight
```

One missed set fails the session. Warmups are excluded.

**Evaluation:**

```
if clean:
    stall_count = 0
    if target_reps < rep_ceiling:
        target_reps += 1                 # phase 1: climb reps, weight fixed
        ceiling_passes = 0
    else:                                # at ceiling
        ceiling_passes += 1
        if ceiling_passes >= 2:          # phase 2: raise weight
            weight = round_to_loadable(weight + increment)
            target_reps = rep_floor
            ceiling_passes = 0
else:
    ceiling_passes = 0
    stall_count += 1
    if stall_count >= 3:                 # micro-deload
        weight = round_down_to_loadable(weight * 0.90)
        target_reps = rep_floor
        stall_count = 0
    # else: hold weight and target
```

Notes:
- `ceiling_passes` counts clean sessions *at the ceiling only*. (An earlier draft reused a general success streak, which could trigger a weight bump after a single session at the ceiling.)
- `stall_count` resets whenever the target changes (bump, deload) or the user passes.
- `round_to_loadable` rounds to the smallest increment the user can actually load (default 2.5 kg for barbells; configurable per exercise for dumbbells and machines).
- A session that overshoots the target does not skip steps in v1. Keep rules predictable.

**First session / no baseline:** the exercise has no state. The user just logs; the first logged working weight becomes the baseline with `target_reps = rep_floor`. No setup prompt.

**Skipped exercises:** if an exercise is not performed for a long time, state is unchanged in v1; the user adjusts manually via the baseline override. An absence rule (auto-reduce, or ask on return) is deferred until real use shows it is needed.

**Bodyweight, assisted, and timed exercises:** not covered in v1. Reps-only exercises can use the same engine with `weight = 0` and a rep-only ceiling; timed exercises are out of scope.

Engine tests are specified in Section 5.5.

## 5. Testing and TDD

This project uses strict test-driven development. It is a working rule, not an aspiration: every behavior starts as a failing test.

### 5.1 The loop (red, green, refactor)

1. **Red:** write one failing test for the next small behavior. Run it and see it fail for the expected reason. A test that has never failed proves nothing.
2. **Green:** write the least production code that makes it pass.
3. **Refactor:** clean up production and test code while everything stays green. Never refactor on red.

Rules:
- No production code without a failing test that requires it.
- One behavior per cycle; keep steps small.
- A bug fix starts with a test that reproduces the bug.
- Spikes (throwaway exploration) are allowed, but are deleted and redone test-first before merging.
- Generated code (`*.g.dart`) is not tested directly.

### 5.2 What is tested, per layer

| Layer | How it is tested | Test doubles |
|---|---|---|
| `domain` (rules, progression) | Plain Dart unit tests: table-driven per rule branch, property tests, golden histories. No Flutter, no database. | None needed; the layer is pure. Clock and UUID generator are injected. |
| `data` (Drift repositories) | Real in-memory Drift database per test (`NativeDatabase.memory()`), opened in `setUp`, closed in `tearDown`. Migration tests for every historical schema version. | None; the database is real, only in memory. |
| view models | Unit tests that call commands and assert the resulting UI state. | Hand-written in-memory fake repositories. |
| widgets | Widget tests of behavior (what the user sees and can do), not pixels or layout details. | Fake view models or repositories. |
| integration | A few critical flows on an emulator, e.g. log a workout end to end. | Real stack. |

Shape: many unit and widget tests, few integration tests (Flutter's testing overview recommends the same balance: unit tests are fast and cheap but give lower confidence; integration tests give the highest confidence but are slowest and costliest).

### 5.3 Test doubles: classical style

- Use real collaborators where practical; use a **fake** (a simple working implementation) where a real one is awkward. No mocking package.
- Tests assert **outcomes** (state, return values), not that specific calls were made. This keeps tests valid when internals are refactored.
- Fakes live in `test/fakes/`, one per repository interface, and are themselves covered by a small contract test shared with the real Drift repository, so fake and real cannot drift apart.
- If a collaborator is truly impractical to fake (a platform API), wrap it behind an interface and fake that.

### 5.4 Determinism

- Time and ID generation are injected (a clock and a UUID generator interface), never read from globals inside domain code.
- Drift tests that touch SQLite time functions use a fixed clock as described in Drift's testing guide.
- Property tests print and accept a seed so failures are reproducible.

### 5.5 Engine tests (progression)

- Table-driven unit tests for each branch in Section 4.2 (climb, bump, hold, deload, resets).
- Property tests: replaying a prefix then the suffix equals replaying the whole history; weight never goes negative; `target_reps` stays within [`rep_floor`, `rep_ceiling`]; `ceiling_passes` is always below 2 after an evaluation (reaching 2 triggers the bump and resets it).
- Golden histories: recorded workout sequences with expected states, versioned with `algorithm_version`.

### 5.6 Layout

`test/` mirrors `lib/`: `test/features/<feature>/<layer>/..._test.dart`. Shared fakes and builders go in `test/fakes/` and `test/support/`.

### 5.7 Coverage and CI

- CI runs format check, analyze, and all tests with coverage on every push.
- Coverage is reported for the whole project (generated code excluded).
- CI fails if `domain` coverage drops below the threshold (proposed 90%, see Section 9).
- Coverage is a smoke alarm: high coverage does not prove test-first, and the gate exists to catch neglect, not to measure quality.

### 5.8 Definition of done

- Each new behavior was driven by a failing test.
- Format, analyze and the full test suite pass.
- The domain coverage gate passes.
- No unrelated changes in the diff.

### 5.9 Limits of enforcement

CI cannot prove tests were written first. The evidence is the commit history (the failing test and the passing implementation committed separately where practical) and code review. `TESTING.md` at the repo root gives contributors the short version of this section.

## 6. Quality and operations

- **Local schema migrations:** Drift versioned migrations, each covered by a migration test (Section 5.2).
- **Data export:** one-tap export of the full log (JSON and CSV). Import is a later goal.
- **Errors:** handled at layer boundaries; repositories return typed failures; nothing swallows exceptions.
- **CI:** see Section 5.7.
- **Licence:** to be chosen before the first public release (open question).

## 7. Future: sync and backend

Not built in v1. This section exists so v1 choices do not block it.

**Backend direction:** a small custom API with PostgreSQL (own auth, own deployment). Chosen over Supabase for portfolio value and independence; the cost is owning auth security and operations.

**Sync approach (when built):**
- Device -> server: push user-log rows (`workouts`, `workout_sections`, `sets`, `exercise_settings`) as per-workout atomic bundles in one server transaction. A bundle is `synced` only after the server acknowledges the whole transaction.
- Server -> device: pull rows changed since a cursor (`updated_at`), including tombstones. The same pull covers other devices' logs and catalog updates (upsert only, never a destructive replace).
- `progression_state` is never synced; each device replays it.
- Conflicts: user-log rows are append-mostly, so a row is either present or tombstoned. `exercise_settings` uses row-level last-write-wins by server-assigned version, not device clocks. No field-level merging unless real data loss is observed.
- Local sync status per bundle: `pending -> syncing -> synced | failed`, with retry and backoff; on launch, bundles stuck in `syncing` longer than a threshold return to `pending`.
- Tombstone retention on the server needs a defined per-device acknowledgment mechanism (for example, a per-device `last_pulled_cursor`) before any purge job is allowed. Do not purge based on a single device's acknowledgment.
- Required additions at that time: accounts and authentication, per-user authorization on every endpoint, API versioning, rate limiting, and account/data deletion.

## 8. Glossary

- **Tombstone:** a "deleted" marker kept instead of erasing a row, so deletions can be replicated.
- **Double progression:** raise reps first, then weight.
- **Deload:** a deliberate, temporary weight reduction.
- **RIR:** reps in reserve; how many more reps the user could have done.
- **Derived state:** data that can be recomputed from the log and so is never treated as authoritative.

## 9. Open decisions

- Domain coverage threshold (proposed 90%) and the coverage-report tool.
- Tool for the import-boundary check (Section 2.1, rule 5).

- Per-exercise increments for dumbbells and machines (unit is chosen at first launch; see decisions.md).
- Licence (must be chosen before first public release; check the seed dataset's licence is compatible).
- Whether overshooting the target should allow faster jumps.
- Absence rule (auto-reduce weight, or ask the user, after N weeks away).
- Deload severity (flat 10% for now) and whether it varies by exercise.
- Catalog delivery: bundled with the app (v1) versus downloaded.
