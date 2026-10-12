t# Architecture: Offline-First Progressive Overload Workout App

**Status:** Living document — update this file whenever an architectural decision changes. Do not let source-code comments silently diverge from what's written here.

---

## 0. How to read this document

This document is written for two audiences at once:

- **Non-technical readers** (product, QA, support): read the plain-language summary at the top of each section — it explains *what* the system does and *why*, without requiring you to read SQL or code.
- **Technical readers** (engineers): each section also includes schema, state machines, and formulas needed to implement or modify the system.

If you only remember one thing from this document: **the phone is the source of truth for "what should I do right now," and the server is the source of truth for "what actually happened, everywhere."** Almost every design decision here exists to make those two things reconcile safely when the phone comes back online.

---

## 1. System Overview

### 1.1 Plain-language summary

The app works fully offline. A user can open the app on an airplane, log an entire workout, close the app, and everything just works — nothing requires the internet in the moment. Later, whenever the phone has a connection, it quietly uploads what happened to the cloud, so the same user can pick up their phone or tablet later and see the same history.

Two kinds of data live in the app, and they behave very differently:

1. **The catalog** — the list of exercises and muscle groups. This is like a reference book: the app downloads it and only reads from it. Users don't edit it.
2. **The user's log** — the workouts, sets, and progress the user actually creates. This is written on the phone first, and only travels to the cloud after the fact.

### 1.2 Technical summary

- **Client:** local embedded database (SQLite via Drift) is the transactional source of truth for all reads/writes during app usage.
- **Server:** Supabase/PostgreSQL is the durable, multi-device source of truth, reconciled via a background sync process.
- **Sync direction:** catalog flows server → device (pull, delta-based). User log flows device → server (push, bundle-based), with periodic pulls to reconcile multi-device state.
- **Consistency model:** eventual consistency with explicit, documented conflict-resolution rules per table (Section 4) — not a general-purpose CRDT or operational-transform system. This is a deliberate scope decision (see 4.1).

```
┌─────────────────────────┐         ┌──────────────────────────┐
│         DEVICE           │         │          SERVER           │
│   (SQLite / Drift)       │         │   (Supabase / PostgreSQL) │
│                          │         │                            │
│  ┌────────────────────┐  │  pull   │                            │
│  │  Catalog (RO cache) │◄─┼─────────┤  Catalog (authoritative)  │
│  └────────────────────┘  │ deltas  │                            │
│                          │         │                            │
│  ┌────────────────────┐  │  push   │                            │
│  │  User log (RW)      │──┼────────►│  User log (authoritative) │
│  │  workouts/sections/ │  │ atomic  │  (durable, multi-device)  │
│  │  sets, metrics      │◄─┼─────────┤                            │
│  └────────────────────┘  │  pull   │                            │
└─────────────────────────┘         └──────────────────────────┘
```

---

## 2. Data Model

### 2.1 Plain-language summary

Think of it in two layers:

- **The exercise library** (muscle groups, exercises, which muscles an exercise targets) — a shared reference, the same for every user.
- **The user's own workout history** (workouts, the exercises within them, the sets within those) — private to that user, created on their device.
- **A "progress save file"** for each exercise the user has ever done — a small summary row that says "here's where they currently are" so the app never has to replay their entire history just to plan tomorrow's workout.

### 2.2 Entity-relationship diagram

```
=================== SERVER-MANAGED CATALOG ===================
(Read-only on device; additive/upsert sync only — see 5.3)

 ┌───────────────────────┐        ┌──────────────────────────┐
 │     MUSCLE_GROUPS      │        │     EXERCISE_TARGETS      │
 │  id            UUID PK │◄───────┤  muscle_group_id  PK,FK   │
 │  name          TEXT    │        │  exercise_id      PK,FK   │──┐
 │  parent_id     UUID FK │        │  intensity        TEXT    │  │
 │   (self-ref, nullable) │        └────────────────────────────┘  │
 └───────────────────────┘                                        │
                                    ┌──────────────────────────┐   │
                                    │        EXERCISES          │◄──┘
                                    │  id                UUID PK│
                                    │  name              TEXT   │◄──┐
                                    │  description        TEXT  │   │
                                    │  image_url          TEXT  │   │
                                    │  deprecated_at   TIMESTAMP│   │
                                    │  replacement_       │     │   │
                                    │    exercise_id    UUID FK │   │
                                    └──────────────────────────┘   │
                                                                    │
================ DEVICE-MANAGED USER LOG =====================     │
(Created offline; synced up via atomic bundles — see 5.2)          │
                                                                    │
 ┌────────────────────────────┐                                    │
 │   USER_EXERCISE_METRICS     │   "the save file" — see Section 6 │
 │  user_id            PK      │                                    │
 │  exercise_id        PK,FK   │────────────────────────────────────┤
 │  current_baseline_weight    │                                    │
 │  baseline_weight_source     │                                    │
 │  progression_rate           │                                    │
 │  target_reps                │                                    │
 │  rep_floor / rep_ceiling    │                                    │
 │  success_streak             │                                    │
 │  stall_count                │                                    │
 │  version            INT     │  ← conflict resolution, see 4.2     │
 │  sync_status                │                                    │
 └────────────────────────────┘                                    │
                                                                    │
 ┌───────────────────┐    ┌─────────────────────────┐               │
 │      WORKOUTS       │    │    WORKOUT_SECTIONS      │               │
 │  id         UUID PK │    │  id             UUID PK │               │
 │  date         TEXT  │───►│  workout_id        FK   │               │
 │  readiness_score INT│    │  exercise_id       FK   │───────────────┘
 │  deleted_at   TS *   │    │  progression_style TEXT │
 │  sync_status  TEXT * │    │  deleted_at      TS *   │
 └───────────────────┘    │  sync_status    TEXT *  │
                            └────────────┬─────────────┘
                                         │
                              ┌──────────▼─────────────┐
                              │           SETS           │
                              │  id           UUID PK    │
                              │  workout_section_id  FK  │
                              │  set_order       INT     │
                              │  target_reps     INT     │
                              │  actual_reps     INT     │
                              │  target_weight   REAL    │
                              │  actual_weight   REAL    │
                              │  deleted_at      TS *    │
                              │  sync_status    TEXT *   │
                              └──────────────────────────┘

* = tombstone + sync-queue fields, see Section 5
```

### 2.3 Table reference

#### Catalog layer (read-only on device)

| Table | Key fields | Notes |
|---|---|---|
| `muscle_groups` | `id` PK, `name`, `parent_id` (self-FK, nullable) | Recursive hierarchy (e.g. Legs → Quads) |
| `exercises` | `id` PK, `name`, `description`, `image_url`, `deprecated_at`, `replacement_exercise_id` | Append-only; see 5.3 |
| `exercise_targets` | `(exercise_id, muscle_group_id)` composite PK, `intensity` | Join table |

#### User log layer (read/write, offline-first)

| Table | Key fields | Notes |
|---|---|---|
| `user_exercise_metrics` | `(user_id, exercise_id)` composite PK, `current_baseline_weight`, `baseline_weight_source`, `progression_rate`, `target_reps`, `rep_floor`, `rep_ceiling`, `success_streak`, `stall_count`, `version`, `sync_status` | The progression "save file" — see Section 6 |
| `workouts` | `id` PK, `date`, `readiness_score`, `deleted_at`, `sync_status` | Root of an atomic bundle |
| `workout_sections` | `id` PK, `workout_id` FK, `exercise_id` FK, `progression_style`, `deleted_at`, `sync_status` | One exercise within a workout |
| `sets` | `id` PK, `workout_section_id` FK, `set_order`, `target_reps`, `actual_reps`, `target_weight`, `actual_weight`, `deleted_at`, `sync_status` | Individual working set |

---

## 3. Architectural Patterns

### 3.1 UUID primary keys

**Why:** two devices can create a workout offline at the exact same millisecond. An auto-incrementing integer ID would collide the moment both devices sync. A UUID generated client-side guarantees no collision, ever, without coordinating with the server first.

**Rule:** all IDs are generated on the device at creation time, not assigned by the server.

### 3.2 Tombstones (soft deletes)

**Plain-language:** when a user deletes a set or workout, the app doesn't actually erase it — it just marks it "deleted" and hides it. That marker itself needs to travel to the server, or the server would never find out the item was removed.

**Rule:** no `DELETE` statement is ever run against user-log tables from the client. Deleting a row sets `deleted_at = CURRENT_TIMESTAMP` locally; the UI filters out anything with `deleted_at IS NOT NULL`.

**Retention policy** (this was previously undocumented — now explicit):

```
Tombstoned rows are hard-deleted server-side by a scheduled job when:
    confirmed_synced_at < NOW() - INTERVAL '30 days'

confirmed_synced_at is set by the sync endpoint once the client has
pulled and acknowledged the tombstone — this prevents a stale local
replica from resurrecting a row the server already purged.

Local (on-device) tombstones are purged immediately after a pull confirms
the server has them; no 30-day local retention is needed, since the phone
doesn't need history of its own deletions once the server has the record.
```

`workouts`, `workout_sections`, and `sets` all carry `confirmed_synced_at TIMESTAMP` for this purpose.

### 3.3 Version-based conflict resolution — scope, not a general mechanism

**Plain-language:** if you edit the same thing on your phone and your tablet before they've had a chance to talk to each other, one of those edits has to lose. We track a version number so we can tell which edit is newer, without trusting either device's clock (clocks drift and lie).

**Rule (explicit, deliberate scope):** this is **row-level last-write-wins**, not field-level merging.

```
If a user edits user_exercise_metrics on two offline devices before either
syncs, one edit is discarded in full — not merged field-by-field.

This is accepted because:
  - the data is single-user (no multi-party conflicts to reconcile)
  - manual edits to this table are rare relative to workout logging
  - for the 'computed' case (see 6.4), the discarded edit is recoverable
    by recomputing from synced `sets` history anyway

Do NOT implement field-level or CRDT-based merging without a concrete,
observed case of user-harming data loss. This is a deliberate simplicity
choice, not an oversight — see userPreferences principle #2 (don't
abstract speculation).
```

### 3.4 Atomic bundles

**Plain-language:** a workout and everything inside it (its sections, its sets) travel to the server together, as one package, or not at all. There's no state where the server has half a workout.

**Rule:** the sync engine uploads `workouts → workout_sections → sets` as a single server-side transaction. The server either commits the entire bundle or rejects the entire bundle. A bundle is marked `synced` on the client only after the server acknowledges the whole transaction — never per-row.

### 3.5 Sync status as a state machine, not a flag

`sync_status` is not just `pending`/`synced` — it needs a state for "in flight" (to detect a sync interrupted by the app being killed) and a state for "the server rejected this" (to distinguish a transient network error from something that needs user attention).

```
        ┌─────────┐
   ┌───►│ pending │◄────────────────┐
   │    └────┬────┘                 │
   │         │ sync engine picks up │ retried after
   │         ▼                      │ backoff
   │    ┌─────────┐            ┌────┴────┐
   │    │ syncing │───────────►│ failed  │
   │    └────┬────┘  network/  └─────────┘
   │         │       server error
   │         │ server acks full bundle
   │         ▼
   │    ┌─────────┐
   └────┤ synced  │  (terminal, until row is edited again)
        └─────────┘

Watchdog rule: on app launch, any row stuck in `syncing` for longer than
N minutes (crash/kill mid-upload) is reset to `pending` and retried.
```

`sync_status` on `workouts`/`workout_sections`/`sets` uses the full four-state enum above. `user_exercise_metrics` intentionally uses a reduced two-state version — see 6.5 for why.

### 3.6 Catalog mutation policy — append-only

**Plain-language:** if an exercise is retired from the library, old workouts that used it must keep working and keep showing accurate history — we never quietly rewrite the past.

```
Catalog rows are never hard-deleted, and a UUID is never reused, since
user logs reference these rows indefinitely.

To retire an exercise: set `deprecated_at`, optionally point
`replacement_exercise_id` at a successor.

Deprecated exercises:
  - are excluded from "add exercise" pickers going forward
  - remain fully joinable for historical workout_sections
  - are NOT auto-migrated into old workout_sections — history reflects
    what the user actually did at the time, not what the catalog
    currently recommends

Client catalog sync is delta-based on `updated_at`, additive/upsert
only — never a destructive replace of the local catalog tables.
```

---

## 4. Conflict Resolution — Full Picture

Two genuinely different mechanisms exist in this system. Conflating them was an earlier design risk; they're now named separately.

| Table(s) | Mechanism | Why |
|---|---|---|
| `workouts`, `workout_sections`, `sets` | Tombstone + atomic bundle (3.2, 3.4) — no field-level conflicts possible, since these are append-mostly logs, not editable state | A logged set doesn't get "merged," it either exists or is tombstoned |
| `user_exercise_metrics` (`baseline_weight_source = 'computed'`) | **Recompute-on-conflict** (6.4) | The stored value is a cache of `sets` history; on conflict, discard both versions and recompute from the (already-reconciled) `sets` table |
| `user_exercise_metrics` (`baseline_weight_source = 'user_override'`) | **Version-based LWW** (3.3) | This is genuinely user-authored data with no derivable source of truth to recompute from |

**The rule that matters:** version-number comparison only governs the `user_override` case. For `computed` rows, sync "resolution" isn't a comparison at all — it's an invalidation-and-recompute pass that runs after `sets` data has synced. Do not write code that runs both mechanisms on the same row.

---

## 5. Sync Engine

### 5.1 Plain-language summary

Sync happens in the background, in two directions: the app periodically asks the server "anything new in the catalog?" (small, cheap, frequent), and separately asks "do I have anything of mine that hasn't gone up yet?" (uploads pending workouts as complete packages).

### 5.2 Upload flow (device → server)

```
[App: user taps "Finish Workout"]
        │
        ▼
[Local write: workouts/sections/sets marked sync_status='pending']
        │
        ▼
[Progression algorithm runs locally — Section 6]
        │
        ▼
[Background sync worker wakes on connectivity]
        │
        ▼
[Select all 'pending' bundles, oldest first]
        │
        ▼
[POST bundle to server as ONE transaction] ──fails──► [mark 'failed', backoff, retry]
        │ succeeds
        ▼
[Server commits workout+sections+sets, returns ack]
        │
        ▼
[Client marks bundle 'synced', sets confirmed_synced_at]
```

### 5.3 Catalog pull flow (server → device)

```
[App launch / periodic tick]
        │
        ▼
[Request catalog rows WHERE updated_at > last_catalog_sync_at]
        │
        ▼
[Upsert into local catalog tables — never delete/replace wholesale]
        │
        ▼
[Update last_catalog_sync_at]
```

### 5.4 Why this scales

Because progression state lives in a single small row (`user_exercise_metrics`) rather than being recalculated from the full `sets` history on every screen load, the app never needs to scan years of workout data to answer "what weight should I lift today?" It reads one row, applies the rules in Section 6, and writes the result back — locally, instantly, offline.

---

## 6. Progressive Overload Algorithm

### 6.1 Plain-language summary

This is a **double progression** model: instead of adding weight every single workout (which is unsustainable), the app first asks the user to do more *reps* at the same weight. Only once they've comfortably hit the top of their rep range for a couple of sessions in a row does the app bump the *weight* — and when it does, it drops the reps back down and starts the rep climb again at the new weight. If the user stalls — can't make progress at all for a few sessions — the app automatically backs the weight off a little so they can build momentum again, rather than leaving them stuck.

### 6.2 State fields (in `user_exercise_metrics`)

| Field | Meaning |
|---|---|
| `current_baseline_weight` | The weight the user is currently working at |
| `baseline_weight_source` | `computed` (algorithm-owned) or `user_override` (user-set, frozen — see 6.4) |
| `target_reps` | Current rep target for the next session |
| `rep_floor` | Reps to reset to after a weight bump (e.g. 8) |
| `rep_ceiling` | Reps required before a weight bump is considered (e.g. 12) |
| `progression_rate` | The weight increment used on a bump (e.g. 2.5 kg) |
| `success_streak` | Consecutive sessions that hit `target_reps` on every working set |
| `stall_count` | Consecutive sessions that **failed** at the *same* weight/rep target — separate counter from `success_streak`, since a streak reset doesn't tell you whether the user is stalling at one target or bouncing between different failures |

`workout_sections.progression_style` selects which rule set applies (e.g. `double_progression`, `linear`) — this document specifies `double_progression`; other styles follow the same shape but are out of scope here.

### 6.3 Evaluation flow

```
                     [ Workout Section Completed ]
                                  │
                                  ▼
              Did actual_reps >= target_reps on EVERY
                    working set in this section?
                       (Rule A — Clean Pass check)
                    /                          \
                 YES                            NO
                  │                              │
                  ▼                              ▼
      success_streak += 1              success_streak = 0
      stall_count = 0                  stall_count += 1
                  │                              │
                  ▼                              ▼
      Is target_reps >= rep_ceiling?    Is stall_count >= 3?
       AND success_streak >= 2?          (Rule C — stall trigger)
              /        \                     /          \
           YES          NO                 YES            NO
            │             │                  │              │
            ▼             ▼                  ▼              ▼
    [PHASE 2: BUMP    [PHASE 1: BUMP    [MICRO-DELOAD]   [HOLD]
     WEIGHT, reset     REPS +1, hold    weight *= 0.90    weight & reps
     reps to floor,    weight]          stall_count = 0   unchanged
     success_streak                     success_streak=0
     = 0]
```

### 6.4 Rule A — the Clean Pass check

A session counts as a pass **only if every working set** in that section met or exceeded its rep target at the prescribed weight:

```
clean_pass = ∀ set ∈ section.sets : set.actual_reps >= set.target_reps
```

One missed set fails the whole section for streak purposes — partial credit isn't given, since the point is confirming the current target is sustainable across the full session, not just a good first set.

### 6.5 Rule B — the Double Progression Matrix

**Phase 1 — Rep Accumulation:** weight stays fixed; reps climb toward the ceiling.

```
next_target_reps = min(actual_reps, rep_ceiling)
```

**Phase 2 — Weight Overload:** triggers only when the user has been sitting at `rep_ceiling` and cleanly passed it for `success_streak >= 2` consecutive sessions.

```
new_baseline_weight = current_baseline_weight + progression_rate
next_target_reps    = rep_floor
success_streak       = 0
```

The two-session requirement at the ceiling (rather than one) exists to reduce the chance a single easy day triggers a jump the user can't actually sustain.

### 6.6 Rule C — Stalls and micro-deloads

A single missed session does **not** drop the weight — the plan holds steady at the same weight and rep target for the next attempt, since one off day (poor sleep, stress) shouldn't erase progress.

```
On failure:
    success_streak = 0
    stall_count += 1
    weight and target_reps: UNCHANGED

If stall_count reaches 3 (three consecutive failed sessions at the
SAME weight/rep target):
    new_baseline_weight = current_baseline_weight * 0.90   (10% deload)
    stall_count = 0
    success_streak = 0
    next_target_reps = rep_floor
```

`stall_count` only accumulates against a *fixed* target — if a deload or a weight bump changes the target, `stall_count` resets to 0, since it's now a different target being evaluated, not a continuation of the old stall.

### 6.7 `user_exercise_metrics` source-of-truth rule (ties back to Section 4)

```
current_baseline_weight and progression state are DERIVED by default
(baseline_weight_source = 'computed') from sets history via the
algorithm above — they are a cache, not independently authoritative.

Exception: a user can manually override their baseline (e.g. "I know my
real working weight is X"). This sets baseline_weight_source =
'user_override', and the algorithm STOPS recomputing this field until
the user explicitly clears the override. While overridden, edits to this
row are resolved via version-based LWW (3.3) — not recomputed, since
there's no longer a `sets`-derived value to fall back to.
```

### 6.8 Interaction with the offline-first architecture

```
[User taps "Finish Workout"]
        │
        ▼
[Section 6 algorithm runs LOCALLY, synchronously, on-device]
        │
        ▼
[user_exercise_metrics updated locally: version += 1,
 new baseline/reps/streak/stall_count written]
        │
        ▼
[Next workout's sets pre-generated from the new state]
        │
        ▼
[Entire payload flagged sync_status = 'pending']
        │
        ▼
[Uploaded as part of the next atomic bundle sync — Section 5.2]
```

Because the algorithm only ever reads one small row plus the current session's sets — never the user's entire history — it runs instantly offline with no network dependency, which is the whole point of the offline-first design.

---

## 7. Open Decisions / Explicitly Deferred

Documented here so nobody assumes these were overlooked — they were considered and deliberately deferred:

- **Field-level conflict merging** for `user_exercise_metrics` — deferred until there's evidence row-level LWW causes real user-visible data loss (3.3).
- **Alternate `progression_style` values** (e.g. `linear`) — schema supports them via the same field, but their rule sets aren't specified in this document yet.
- **Deload severity tuning** (currently a flat 10%) — may need to vary by exercise type or user experience level; not yet parameterized.
- **Tombstone retention window** (currently 30 days) — arbitrary starting value, not yet validated against real storage growth.

---

## 8. Glossary (for non-technical readers)

- **Tombstone:** a "marked as deleted" flag instead of actually erasing data, so deletions can be communicated to the server later.
- **Atomic bundle:** a group of related records (a workout and everything inside it) that must all save together or not at all.
- **Baseline weight:** the weight the app currently thinks the user should lift for an exercise.
- **Double progression:** increase reps first, then weight — instead of increasing weight every session.
- **Deload:** a deliberate, temporary reduction in weight to help the user recover and break through a plateau.
- **Sync:** the background process of sending the phone's data to the cloud (and pulling down anything new) whenever there's a connection.est
