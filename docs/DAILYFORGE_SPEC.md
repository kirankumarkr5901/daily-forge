# DailyForge — Build Specification

**Version:** 1.0
**Purpose:** Hand this file to Claude Code as the source of truth for building the app.
**Stack:** Angular (+RxJS/Signals) · Java 21 / Spring Boot 3 · H2 (dev) → PostgreSQL (prod) · Capacitor for Android

---

## 0. How to use this document

- Sections 1–7 are **contracts**. Do not change them without asking.
- Section 8 is the **UI spec**, page by page.
- Section 9 is the **design system**. Follow the tokens exactly.
- Section 11 is the **build order**. Work milestone by milestone; each one must be runnable and demoable before starting the next.
- Section 13 lists **open decisions**. Defaults are given for every one of them, so you can build without blocking. Flag them in the PR description rather than inventing new answers.
- Anything marked **[ADD]** is a suggested addition to the original plan, not something the owner asked for. Build it only if it is inside the current milestone.

---

## 1. Product summary

DailyForge is a manual-logging fitness and habit tracker where every logged action converts into points, and points are the single currency across the whole app. Workouts, runs, habits, one-off good/bad activities, and goals all feed one score. The score is spendable on user-defined rewards.

The premise is that the app should be worth opening every day even when the user does not feel like training: logging is fast, the feedback is immediate, and the history is visible.

**Non-goals for v1:** GPS tracking, wearable sync, social feed, AI chat logging (explicitly deferred), nutrition tracking.

### 1.1 Scope gap worth flagging
The original plan opens with "points earning and spending it on rewards" but no rewards screen appears in the page list. A minimal **Rewards** feature is specified in §8.9 because without it the points have no sink and the core loop is incomplete. If it should be deferred, cut it at M8, not earlier.

---

## 2. Naming

Keep **DailyForge**. It is concrete, ownable, matches the "effort compounds into something" idea, has an obvious mark (anvil / spark / ingot), and the .com-style handles are less contested than generic fitness words.

If a change is wanted, ranked alternatives: **Anvil**, **Forgeline**, **Emberlog**, **Streakforge**, **Tallyforge**, **Ironkeep**.

Do not rename anything in code. Use `dailyforge` as the package root (`com.dailyforge`) and npm scope.

---

## 3. Architecture and technology decisions

### 3.1 Repository layout (monorepo)

```
dailyforge/
├── backend/                 # Spring Boot 3.3+, Java 21, Gradle (Kotlin DSL)
│   └── src/main/java/com/dailyforge/
│       ├── common/          # errors, time, config, security
│       ├── points/          # POINTS ENGINE — the core module
│       ├── identity/        # users, auth, settings
│       ├── workout/         # plans, exercises, logs, PRs
│       ├── habit/           # habits, logs, streaks
│       ├── activity/        # positive/negative one-off activities
│       ├── run/             # runs, run PRs
│       ├── goal/
│       ├── job/
│       ├── body/            # body metrics
│       ├── reward/
│       └── insight/         # daily summaries, heatmap, quotes
├── frontend/                # Angular 20, standalone components
├── android/                 # Capacitor Android project (generated)
├── docs/
│   ├── DAILYFORGE_SPEC.md   # this file
│   └── decisions/           # one ADR per non-obvious decision
└── CLAUDE.md
```

Each backend module is a package with its own `api/` (controllers + DTOs), `domain/` (entities + services), `repo/`. Modules talk to each other through published service interfaces only, never through each other's repositories. Points are awarded exclusively by calling the points module.

### 3.2 Database recommendation

The plan says "in-memory database for now". Take that as "no infrastructure to install yet", not "lose data on restart".

- **Dev default:** H2 in **file mode** — `jdbc:h2:file:./.data/dailyforge;MODE=PostgreSQL;DATABASE_TO_LOWER=TRUE`. Data survives restarts, which matters a lot when you are testing 7-day streaks.
- **Tests:** H2 in-memory, fresh per test class.
- **Production:** **PostgreSQL 16**. It is the right long-term answer: real date/time handling with time zones, partial and expression indexes for the ledger, window functions for streaks, and `jsonb` for the flexible bits (goal targets, rule config).
- **Migrations:** Flyway from commit one. `MODE=PostgreSQL` plus plain-SQL migrations means the same migration files run on both. Never use `hibernate.ddl-auto` beyond `validate`.
- Avoid H2-only syntax. If a query cannot be expressed portably, put it behind a repository interface with two implementations rather than branching in the service.

### 3.3 Frontend decisions

- **Angular 20**, standalone components, no NgModules.
- **State:** Angular signals for component and feature state; RxJS for anything with time or streams (HTTP, debounced search, offline queue, day rollover ticks). Use `@ngrx/signals` signal stores for the three shared stores (`SessionStore`, `PointsStore`, `SyncStore`). Do not add full NgRx.
- **Routing:** lazy-loaded standalone routes per feature. Home is eager.
- **UI components:** hand-built with `@angular/cdk` for behaviour (drag-drop for reordering, overlay for sheets/popups, a11y for focus traps and live regions). Do **not** pull in Angular Material — the design in §9 is not Material and fighting it costs more than building the ~15 primitives needed.
- **Charts:** `chart.js` + a thin Angular wrapper component (`<df-chart>`). One wrapper, one place to theme it.
- **Forms:** typed reactive forms.
- **HTTP:** functional interceptors for auth token, error normalisation, and the offline queue.
- **PWA:** `@angular/pwa` service worker, installable, offline shell.
- **i18n-ready:** no hardcoded strings in templates; use a simple `t()` lookup even if English-only for now. Cheap now, expensive later.

### 3.4 Android packaging

Wrap the built Angular app with **Capacitor 7**. Do not build a separate native app.

Requirements to bake in from M1 so the wrap is not a rewrite later:
- `<base href="/">` configurable per build target; API base URL from environment file, never hardcoded to `localhost`.
- Safe-area insets respected (`env(safe-area-inset-*)`) on header, bottom nav, and sheets.
- Auth token stored via `@capacitor/preferences` on native, `localStorage` on web, behind one `TokenStorage` interface.
- Plugins: `@capacitor/status-bar` (theme-aware), `@capacitor/haptics` (light tap on log, heavier on PR), `@capacitor/local-notifications` (habit reminders **[ADD]**), `@capacitor/app` (resume → refresh today's data).
- Back button on Android must map to router navigation, closing sheets first.
- Because the backend is a separate service, the app needs a reachable API host. For local development on device, use the LAN IP; document it in the README.

---

## 4. Cross-cutting rules

These apply everywhere and are the most common source of subtle bugs. Treat them as invariants.

### 4.1 Identity and anonymous browsing
- Anonymous visitors can browse every page. Any request that writes data returns `401` with `{"code":"AUTH_REQUIRED"}`, and the frontend opens the login sheet, then **replays the intended action** after login. Do not lose the user's input.
- Anonymous mode is backed by a read-only **demo dataset** **[ADD]** (a seeded fake user with two months of history) so the heatmap, charts, and PR cards are not empty. Label it clearly: "Sample data. Sign in to start your own."
- Auth: email + password, BCrypt, stateless JWT access token (15 min) + rotating refresh token (30 days) stored server-side so it can be revoked. Spring Security 6 with a filter chain, no sessions.

### 4.2 Time and the definition of a "day"
- Every user has a `timeZone` (IANA, defaulted from the browser at signup) in `user_settings`.
- All "day" logic — streaks, heatmap cells, daily bonuses, today's points — uses the **user's local date**, computed as `Instant → ZonedDateTime(userZone) → LocalDate`.
- Store instants in UTC (`timestamptz`), store logical dates as `LocalDate` (`date`). Never store a local date as a timestamp.
- One `DayService` in `common/` owns this conversion. No `LocalDate.now()` anywhere else in the codebase.

### 4.3 Edit windows and backdating
- Habits: user may change **today and yesterday** only (from the original plan). Older days are locked and the UI shows a lock icon with a tooltip explaining why.
- Habits may be logged for **future days** (from the original plan), but future logs award points only when that day arrives; the rollover job settles them. Show them as "planned", not "earned".
- Workouts, runs, activities, body metrics: editable for **7 days** back, then locked. **[ADD]** — the original plan does not say, and an unbounded window makes points trivially gameable.
- Locked data can still be corrected by an explicit "Request correction" flow that writes an `ADJUSTMENT` ledger entry. Defer to M9.

### 4.4 Undo means reversal, never deletion
The plan's hard rule is that undoing an action reverts its points. Implementation rule:

> Ledger entries are **immutable and append-only**. Undo writes a compensating entry; it never deletes or edits the original.

This gives a correct score, a truthful activity log ("+15 PR bonus" then "−15 reversed"), and an audit trail. See §5.

### 4.5 Idempotency
Every write endpoint accepts an `Idempotency-Key` header. The frontend generates a UUID per user action and reuses it on retry (critical for the offline queue and flaky mobile networks). The points ledger enforces uniqueness on `(user_id, idempotency_key)`.

### 4.6 Offline-first logging **[ADD]**
Mobile users log in gyms with no signal. Queue mutations in IndexedDB, replay on reconnect in order, using the idempotency key. Show a small "N pending" indicator. Reads fall back to the last cached response. Do not attempt offline point calculation — the server is the only authority on score; show the optimistic local delta as "pending" and reconcile on sync.

### 4.7 Error contract
All errors return:
```json
{ "code": "HABIT_LOCKED", "message": "This day can no longer be changed.", "field": null, "details": {} }
```
`code` is a stable enum the frontend maps to copy. Never surface a stack trace or a raw Java message.

---

## 5. The Points Engine (core module)

This is the heart of the app. Build it first, test it hardest. Every other feature is a thin producer of events into it.

### 5.1 Ledger model

Table `points_entry`:

| column | type | notes |
|---|---|---|
| `id` | uuid pk | |
| `user_id` | uuid fk | |
| `occurred_on` | date | user-local date the points belong to |
| `created_at` | timestamptz | when the row was written |
| `category` | enum | `WORKOUT`, `RUN`, `HABIT`, `ACTIVITY`, `GOAL`, `JOB`, `REWARD`, `ADJUSTMENT` |
| `rule_code` | varchar | e.g. `RUN_DISTANCE`, `HABIT_CONSISTENCY`, `WORKOUT_PR` |
| `amount` | int | **signed**. Penalties and reversals are negative |
| `source_type` | varchar | `WORKOUT_SET`, `RUN`, `HABIT_LOG`, … |
| `source_id` | uuid | nullable |
| `reverses_id` | uuid fk | non-null if this is a compensating entry |
| `reversed` | boolean | true once a compensating entry exists |
| `description` | varchar | user-facing line for the activity log |
| `idempotency_key` | varchar | unique per user |

**Score = `SUM(amount)` over all entries for the user.** Never store a mutable running total as the source of truth. Cache it in `user_score_cache` for fast reads, updated in the same transaction, and provide an admin `recalculate` endpoint that rebuilds it from the ledger.

Indexes: `(user_id, occurred_on)`, `(user_id, category, occurred_on)`, `(source_type, source_id)`, unique `(user_id, idempotency_key)`.

### 5.2 Engine API (internal)

```java
public interface PointsService {
    PointsResult award(AwardCommand cmd);          // idempotent
    void reverseBySource(String sourceType, UUID sourceId, String reason);
    void reconcile(UUID userId, ReconcileScope scope); // recompute derived bonuses
    ScoreSnapshot snapshot(UUID userId);           // total, today, week, month, by category
}
```

`AwardCommand` carries userId, occurredOn, category, ruleCode, amount, source, description, idempotencyKey.

`PointsResult` returns the entries written **and** a list of `Celebration` descriptors (`PR`, `STREAK_7`, `MILESTONE_21K`, `ALL_HABITS_DONE`, `WORKOUT_COMPLETE`) so the frontend knows which animation to play. The frontend must never decide what was earned.

### 5.3 Reconciliation (the hard part)

Some bonuses depend on history, so editing the past can invalidate an already-granted bonus. Example: a 14-day habit streak earned a consistency bonus; the user then unticks day 9. The 14-day bonus is no longer valid.

Rule: after any mutation to a source that feeds a derived bonus, run a **recompute-and-reconcile** pass inside the same transaction:

1. Recompute what *should* be awarded for the affected scope (one habit, or one exercise's PR chain, or one day's commitment bonus) from raw logs.
2. Diff against non-reversed ledger entries for that scope.
3. Write reversals for entries that should not exist, and awards for entries that should.

Scopes: `HABIT(habitId, fromDate)`, `EXERCISE_PR(exerciseId)`, `DAY_COMMITMENT(date)`, `RUN_PR(userId)`, `GOAL(goalId)`.

This must be covered by property-based tests: apply a random sequence of log/unlog/edit operations, then assert that the ledger sum equals a naive from-scratch recomputation. This test is non-negotiable.

### 5.4 Earning rules

All numbers below live in a `points_rule_config` table seeded by migration, editable per-user where noted. No magic numbers in code.

#### Habits
| rule_code | when | amount |
|---|---|---|
| `HABIT_BASE` | habit ticked for a day | `habit.points` (set by user at creation) |
| `HABIT_PENALTY` | strict habit **not** completed at day rollover | `−habit.penaltyPoints` |
| `HABIT_CONSISTENCY` | streak reaches a multiple of 7 | see formula |
| `HABIT_COMMITMENT` | every active habit for that day is complete | `settings.commitmentBonus` (one global value, asked once) |

Consistency bonus formula, with `n = streak / 7` (so n=1 at 7 days, n=2 at 14…):

```
bonus(n) = round(habit.baseBonus * pow(habit.bonusMultiplier, n - 1))
```
Both `baseBonus` and `bonusMultiplier` are captured at habit creation (defaults 20 and 1.5). Cap `n` at 12 (84 days) so the number does not explode; after that keep paying `bonus(12)` every 7 days. **[ADD — the plan has no cap; without one, a 1.5 multiplier pays ~2.4 million points at one year.]**

A "strict" habit that is missed also breaks the streak. A normal habit that is missed breaks the streak with no penalty.

#### Runs
| rule_code | when | amount |
|---|---|---|
| `RUN_DISTANCE` | every run | `floor(distanceMeters / 100)` |
| `RUN_MILESTONE` | run distance crosses a milestone | table below, **highest single milestone reached in that run** |
| `RUN_FIRST_MILESTONE` | first time ever reaching that milestone | milestone value again (i.e. doubled) **[ADD]** |

Milestones (config): 10 km → 50, 15 km → 80, 21.1 km → 150, 25 km → 180, 42.2 km → 400, 50 km → 500.
Note: the plan lists "21" and "42"; interpret as half and full marathon distances (21.1 / 42.2 km). See §13.

#### Workouts
| rule_code | when | amount |
|---|---|---|
| `WORKOUT_SET` | each logged set | `1` (config, default on) |
| `WORKOUT_PR` | new personal record for an exercise | see formula |
| `WORKOUT_SESSION_COMPLETE` | all exercises for the day logged | `0` — celebration only, per the plan |

PR bonus:
```
prBonus = round(typeFactor * (totalWeightKg / 5)) + repsOnlyBonus
```
- `typeFactor`: barbell 1.2, dumbbell 1.1, machine 0.9, bodyweight 1.0, cardio n/a.
- `totalWeightKg`: single-hand weight × 2, or combined weight as entered. Bodyweight exercises use **added weight only** (see §5.5).
- If the PR is a reps-only PR (same weight, more reps) the weight term is skipped and `repsOnlyBonus = 5`.
- Minimum award 1, maximum award 100 per PR.

#### Activities
`ACTIVITY_POSITIVE` = `+activity.points`, `ACTIVITY_NEGATIVE` = `−activity.points`. User-defined.

#### Goals
`GOAL_COMPLETE` = `goal.rewardPoints` (user sets at creation, default 100). Reversed if the goal is reopened.

#### Jobs
`JOB_STAGE_ADVANCE` = 0 by default, config-enabled. Job tracking is a productivity feature, not a fitness one; awarding points for it invites inflation. Home page still shows job **metrics** as the plan requires.

#### Rewards
`REWARD_REDEEM` = `−reward.cost`.

### 5.5 PR definition (exact)

For an exercise, a set is compared as the tuple `(totalWeightKg, reps)`.
- Ordering: higher `totalWeightKg` wins; on a tie, higher `reps` wins.
- Display format: `weight × reps` (e.g. `42.5 kg × 8`).
- **Recent PR** = best set in the last 90 days. **Lifetime PR** = best set ever.
- Bodyweight exercises: `totalWeightKg` for PR purposes is the **added** weight only (0 if none). Display shows `BW + 10 kg × 12`, and hides the `+ 0` case, showing just `BW × 12`.
- Weight entry type is per set: `SINGLE` (per hand, doubled) or `COMBINED` (as entered). Store both `enteredWeight`, `weightMode`, and the derived `totalWeightKg`.

### 5.6 Daily rollover job

A scheduled task runs hourly. For each user whose local date has just advanced:
1. Apply `HABIT_PENALTY` for missed strict habits on the day that just closed.
2. Settle any future-dated habit logs that are now current.
3. Evaluate `HABIT_COMMITMENT` for the closed day.
4. Rebuild `daily_summary` rows for the last 3 days.
5. Break streaks that ended.

The job must be idempotent — re-running it for the same date must produce no new entries.

### 5.7 Anti-gaming guardrails **[ADD]**
- Daily cap per category (config, default: 500 workout, 400 run, unlimited habit).
- Sanity limits on inputs: run ≤ 200 km and ≤ 24 h; set weight ≤ 500 kg; reps ≤ 100.
- Editing outside the window is rejected (§4.3).
These are guardrails, not accusations. Error copy should be neutral: "That looks out of range. Check the distance."

---

## 6. Data model

Entities by module. All tables have `id uuid pk`, `created_at`, `updated_at`, and where user-owned, `user_id`.

### identity
- **user** — email (unique, citext), passwordHash, displayName, createdAt, status.
- **user_settings** — timeZone, unitSystem (`METRIC`|`IMPERIAL`), theme (`SYSTEM`|`LIGHT`|`DARK`), commitmentBonus, weekStart (fixed `MONDAY` per plan), onboardingCompletedAt, reminderTime.
- **refresh_token** — tokenHash, expiresAt, revokedAt, deviceLabel.

### points
- **points_entry** (§5.1), **points_rule_config**, **user_score_cache**.

### workout
- **exercise** — name, muscleGroups (set), kind (`STRENGTH`|`CARDIO`), equipment (`DUMBBELL`|`BARBELL`|`BODYWEIGHT`|`MACHINE`|`NONE`), isElite, ownerUserId (null = system catalog), searchName (normalised for suggestions).
  Exercises are **shared across plans** and reusable across future plans, per the plan.
- **workout_plan** — name, dayCount, isActive, archivedAt.
- **plan_day** — planId, dayIndex (1..dayCount), label (e.g. "Push"), isOptionalBucket (the swap pool).
- **plan_day_exercise** — planDayId, exerciseId, sortOrder, targetSets, targetReps, notes. Unique `(planDayId, exerciseId)`.
- **elite_assignment** — planId, exerciseId, dayIndexes (int[]). Elite exercises show on multiple days but track **once per day** — the log is keyed by `(user, exercise, date)`, not by day slot.
- **workout_session** — userId, date, planId, planDayId, startedAt, completedAt.
- **workout_set** — sessionId, exerciseId, setNumber, enteredWeight, weightMode, addedWeight, reps, totalWeightKg (derived), isPr (derived), loggedAt, deletedAt (soft delete).
- **personal_record** — userId, exerciseId, scope (`RECENT`|`LIFETIME`), totalWeightKg, reps, achievedOn, setId. Recomputed by the reconciler, never hand-edited.

### habit
- **habit** — name, icon, points, type (`NORMAL`|`STRICT`), penaltyPoints, baseBonus, bonusMultiplier, sortOrder, activeFrom, archivedAt, scheduleDays (bitmask, default all 7) **[ADD — lets a habit be "weekdays only" instead of failing every weekend]**.
- **habit_log** — habitId, date, state (`DONE`|`SKIPPED`), loggedAt, settledAt. Unique `(habitId, date)`.
- **habit_streak** — habitId, currentStreak, bestStreak, lastAwardedMultipleOf7, lastCompletedDate. Derived, rebuilt by the reconciler.

### activity
- **activity_type** — name, polarity (`POSITIVE`|`NEGATIVE`), points, icon, sortOrder, archivedAt.
- **activity_log** — activityTypeId, date, count, note.

### run
- **run** — date, distanceMeters, durationSeconds, type (`LONG`|`INTERVAL`|`TEMPO`), paceSecPerKm (derived), note, feltEffort (1–5) **[ADD]**.
  `LONG` is auto-assigned when distance ≥ 10 km; below that the user picks `INTERVAL` or `TEMPO` (per the plan).
- **run_record** — userId, bracket (`D5K`,`D10K`,`D15K`,`D21K`,`D25K`,`D42K`,`D50K`, plus `OVERALL_DISTANCE`, `OVERALL_PACE`), rank (1–3), runId, value. Derived.

### goal
- **goal** — title, description, kind (`HABIT_ADHERENCE`|`EXERCISE_TARGET`|`RUN_DISTANCE`|`BODY_METRIC`|`CUSTOM`), periodType (`WEEK`|`MONTH`|`TARGET_DATE`), startDate, endDate, rewardPoints, status (`ACTIVE`|`COMPLETED`|`FAILED`|`ARCHIVED`), completedAt.
- **goal_target** — goalId, targetJson (jsonb: `{"habitId":…,"days":20}` or `{"exerciseId":…,"weightKg":100}`), currentValue, targetValue.
  Progress is recomputed on every relevant write and cached in `currentValue`.

### job
- **job_application** — company, role, roleId, city, jobUrl, resumeVersion, source (`APPLIED`|`REFERRAL_REQUESTED`|`REFERRED`|`RECRUITER`), referrerName, status (`APPLIED`|`ASSESSMENT`|`INTERVIEW`|`OFFER`|`REJECTED`|`WITHDRAWN`|`GHOSTED`), currentRound, nextFollowUpOn, note, appliedOn.
- **job_event** — applicationId, fromStatus, toStatus, roundNumber, occurredOn, note. Full history, so the pipeline is auditable and the metrics are computable.

### body
- **body_metric** — date, weightKg, heightCm, bodyFatPct (optional), note. Height is usually static; store it on settings **and** allow a per-log override.
  BMI and the underweight/normal/overweight band are computed, never stored.

### reward
- **reward** — name, cost, icon, isRepeatable, stock, archivedAt.
- **reward_redemption** — rewardId, redeemedAt, pointsSpent, ledgerEntryId.

### insight
- **daily_summary** — userId, date, pointsTotal, pointsByCategory (jsonb), hasWorkout, hasRun, hasHabitCompletion, inactiveRunLength, state (`WORKOUT`|`RUN`|`BOTH`|`REST`|`MISSED`|`EMPTY`). One row per user per day; the heatmap reads only this table.
- **quote** — text, author, source, verifiedAt, active. Seeded with a curated, attribution-checked set (see §8.1).

---

## 7. API surface

REST, JSON, `/api/v1`. All list endpoints are paginated (`?page=&size=`) and all date params are user-local `YYYY-MM-DD`.

```
POST   /auth/signup                       { email, password, displayName, timeZone }
POST   /auth/login
POST   /auth/refresh
POST   /auth/logout
GET    /me                                → profile + settings
PATCH  /me/settings

GET    /home/summary?date=                → quote, score, today/week/month points by category,
                                            job metrics, active goals, recent ledger
GET    /insights/heatmap?from=&to=        → daily_summary rows
GET    /insights/day/{date}               → activities + points grouped by category
GET    /points/ledger?from=&to=&category= → paginated activity log
GET    /quotes/today

GET    /exercises?q=                      → catalog + user's own, for autocomplete
POST   /exercises
PATCH  /exercises/{id}
DELETE /exercises/{id}

GET    /workout-plans
POST   /workout-plans                     { name, dayCount }
PATCH  /workout-plans/{id}
DELETE /workout-plans/{id}
POST   /workout-plans/{id}/days/{dayIndex}/exercises      { exerciseId, targetSets, targetReps }
PATCH  /workout-plans/{id}/days/{dayIndex}/exercises/order { orderedIds[] }
POST   /workout-plans/{id}/exercises/{exerciseId}/move    { toDayIndex }   # swap
POST   /workout-plans/{id}/elite                          { exerciseId, dayIndexes[] }

GET    /workouts/session?date=&planId=&dayIndex=   → session + per-exercise state, PRs, last log
POST   /workouts/sets                              { date, exerciseId, enteredWeight, weightMode,
                                                     addedWeight, reps }
PATCH  /workouts/sets/{id}
DELETE /workouts/sets/{id}                         → reverses points
GET    /workouts/exercises/{id}/history?range=     → chart series + PR timeline

GET    /habits
POST   /habits
PATCH  /habits/{id}
DELETE /habits/{id}
PATCH  /habits/order
GET    /habits/board?date=                → habits + state + current/best streak +
                                            "2 days from a bonus" hints
POST   /habits/{id}/logs                  { date }        → returns celebrations
DELETE /habits/{id}/logs/{date}           → reverses points

GET    /activities
POST   /activities                        { name, polarity, points }
POST   /activities/{id}/logs              { date, count }
DELETE /activity-logs/{id}

GET    /runs?from=&to=
POST   /runs                              { date, distanceMeters, durationSeconds, type }
PATCH  /runs/{id}
DELETE /runs/{id}
GET    /runs/records                      → top 3 per bracket + overall

GET    /goals?status=
POST   /goals
PATCH  /goals/{id}
POST   /goals/{id}/complete
DELETE /goals/{id}

GET    /jobs?status=
POST   /jobs
PATCH  /jobs/{id}
POST   /jobs/{id}/transition              { toStatus, roundNumber, note, occurredOn }
GET    /jobs/metrics                      → counts by status, response rate, follow-ups due

GET    /body-metrics?from=&to=
POST   /body-metrics
DELETE /body-metrics/{id}

GET    /rewards
POST   /rewards
POST   /rewards/{id}/redeem
DELETE /reward-redemptions/{id}           → refunds points
```

Every mutating endpoint returns the affected resource **plus** `{ "points": { "delta": 32, "newTotal": 4821, "celebrations": [...] } }` so the UI can animate without a second round trip.

---

## 8. Page specifications

Shared shell:
- **Header:** app wordmark left, user name / Sign in right, hamburger far right (full page list, per the plan).
- **Bottom navigation on mobile [ADD]:** Home · Workout · Habits · Run · More. A hamburger alone means two taps for the most-used screens, which fights the "log fast" goal. Keep the hamburger for the full list including Goals, Jobs, Body, Rewards, Settings.
- **Score pill** in the header, always visible, animating on change (§9.4).
- A global **undo toast** after every points-earning action: "Logged. +12 points. Undo". 6-second window, calls the delete endpoint.

### 8.1 Home
Order on mobile, top to bottom:
1. **Quote card** — one deterministic quote per user per day (`hash(userId, date) % quotePoolSize`), so it does not change on refresh. Author name shown; only verified attributions in the pool. Seed ~200 quotes with a `source` field; reject anything that cannot be traced to a real speaker or work. Do not fetch quotes from a third-party API at runtime — misattribution is the norm in those datasets.
2. **Score card** — total score, plus today / this week (Monday start) / this month, with a category breakdown (segmented bar + legend, tappable to filter the ledger).
3. **Heatmap** — 12 months scrollable, current month in view. See §8.1.1.
4. **Goal progress** — bars for active goals; hidden when none.
5. **Job metrics** — counts by stage; **hidden entirely when there are no applications**, per the plan.
6. **Activity log** — the last 20 ledger entries, grouped by day, each line showing description and signed amount. Reversed entries are shown struck through.

#### 8.1.1 Heatmap rules
Cell state comes from `daily_summary.state`:
- `BOTH` — workout and run logged.
- `WORKOUT` / `RUN` — one of the two.
- `REST` — no workout and no run, and this is the 1st or 2nd consecutive such day.
- `MISSED` — no workout and no run, and this is the 3rd or later consecutive such day.
- `EMPTY` — future date, or before the account started.

`inactiveRunLength` is computed over the **continuous history**, so the run carries across month boundaries exactly as the plan requires: the 1st of a month looks back at the last days of the previous month.

Cell colour encodes **points earned** (5 intensity steps). The emoji/glyph in the cell corner encodes **state**: dumbbell, runner, sleeping face, cross. When both workout and run are present, show a small combined badge, not two glyphs.

Tapping a cell opens a day sheet: every activity for that day grouped by category, with points per group and the day total.

Accessibility: the heatmap must be navigable by keyboard, each cell has an `aria-label` like "12 March, 45 points, workout and run", and the glyphs must never be the only carrier of meaning.

### 8.2 Planner
Two tabs: **Workouts** and **Habits**.

**Workout planner**
- Create plan: name + number of days. Days are then listed as cards ("Day 1", editable label).
- Add exercise to a day: a search field that suggests from the shared catalog as the user types, with "Create «bench press»" always available as the last option. Never block a user from typing a name that is not in the catalog.
- Exercise fields: name, muscle groups (multi), kind (strength/cardio), equipment type when strength, target sets/reps (optional).
- Reordering within a day: CDK drag-drop, long-press on touch, with keyboard alternative (move up/down in the overflow menu).
- Edit and delete exercises. Deleting an exercise that has logs archives it instead and warns: "You have 47 logs for this. It will be hidden from plans but your history and PRs stay."
- **Elite exercises:** marked on the exercise, then assigned to multiple days via a day-picker chip row. They render on each selected day but track once per date.
- **Optional bucket:** a special "Extras" lane per plan holding exercises not assigned to a day, used as the source and target for swaps.
- Exercises are shared across plans: adding an existing exercise to a new plan reuses the same record, so PRs and history follow it.

**Habit planner**
- Create habit: name, icon, points, type (normal/strict), penalty points (strict only), base bonus, bonus multiplier, schedule days.
- A live **preview panel** showing what the first four consistency bonuses will pay ("7 days → 20, 14 days → 30, 21 days → 45, 28 days → 68"). Multipliers are unintuitive; show the consequence at creation time.
- Commitment bonus is asked **once**, at the first habit's creation, stored on settings, editable in Settings.
- Habits are reorderable, editable, deletable (archive if logs exist).

### 8.3 Workout tracker
1. **Date picker** — ‹ prev · today · next ›, plus a tap-to-open calendar.
2. **Collapsed calendar strip** — same heatmap component, collapsed by default, showing workouts/runs per day.
3. **Plan picker** → **day picker** (day chips with labels). Remember the last used plan and suggest the next day in rotation **[ADD]**.
4. Exercises grouped by muscle group, each as a **log card**:
   - Header: exercise name, kind, and a colour-coded equipment chip (§9.2).
   - Recent PR and Lifetime PR, both as `weight × reps`.
   - Primary **Log** button → bottom sheet with weight / reps / sets, prefilled from the most recent log of that exercise. Weight mode toggle (per-hand vs combined) with the computed total shown live: "22.5 × 2 = 45 kg".
   - Logged sets listed under the card, each editable and deletable inline.
   - **History** button → chart sheet (weight over time, volume over time, PR markers).
   - **Swap** button → move this exercise to another day in the plan, or to the Extras bucket.
   - On completion the card turns to the "done" treatment and **animates to the bottom of the list**, so the next unfinished exercise rises to the top.
5. **PR celebration:** confetti/spark burst scoped to the card, plus haptic, plus the points delta flying into the header score pill. One celebration per PR, queued if several land at once.
6. **Session complete:** full-screen celebration, 0 points, per the plan. Summary of sets, volume, and points earned this session.
7. **Rest timer [ADD]:** starts automatically after a set is logged, configurable default, dismissible. This is the single most requested feature in any lifting app and it costs very little here.

### 8.4 Habit tracker
1. Date picker and collapsed calendar strip, as above.
2. **Habit cards** as a checklist: tap anywhere on the card to toggle. Each card shows current streak and best streak, and its points value.
3. States: done (green treatment), missed (red treatment, past days only), pending (neutral), planned (future days, dashed outline).
4. Only **today and yesterday** are editable. Other past days render locked with an explanatory tooltip. Future days are loggable as planned.
5. **Bonus hint strip** pinned at the bottom: "Ticking Read and Cold shower today reaches a 14-day streak: +30 each." Computed server-side in `/habits/board`.
6. Celebrations for consistency bonuses and for the commitment bonus (all habits done).
7. **Second tab: Activities** — user-created one-off positive and negative activities, each with points. Tap to log for the selected date, with a count stepper for repeatable ones. Reorderable, editable, deletable. Negative activities use the penalty treatment, not the same red as "missed", so the two do not blur.

### 8.5 Run tracker
1. **Run score header** — a separate running-only total (sum of ledger entries where `category = RUN`), shown alongside its contribution to the main score.
2. **Log card** — distance (km, decimal), duration (mm:ss picker), type. Type auto-selects `LONG` at ≥ 10 km and locks; below 10 km the user must choose interval or tempo. Derived pace shown live as the user types.
3. **PR section** — top 3 overall by distance and by pace.
4. **Bracket sections** — top 3 in each of 5 K, 10 K, 15 K, 21 K, 25 K, 42 K, 50 K; collapsed by default. A run counts toward a bracket if its distance is ≥ that bracket.
5. **All runs** — reverse chronological list, each row showing distance, duration, pace, type, points; swipe or menu to edit/delete.
6. Milestone celebrations fire on crossing a milestone, with a stronger treatment for a first-ever milestone.

### 8.6 Goals
- Create a goal: title, kind, period (week / month / target date), targets, reward points.
- Target kinds: habit adherence ("Read on 20 days"), exercise target ("Bench 100 kg"), run distance ("60 km this month"), body metric ("75 kg"), or a custom manual checkbox.
- Progress bars update automatically from the relevant logs; custom goals are marked complete manually.
- Completing a goal awards `rewardPoints` with a celebration. Reopening reverses it.
- Expired incomplete goals move to `FAILED` at rollover with no penalty, and offer "Extend" or "Archive".

### 8.7 Jobs
- List with a status filter and a compact pipeline summary at the top.
- Application fields: company, role, role ID, city, job URL, resume version, source (applied / referral requested / referred / recruiter), referrer name, note, applied date, next follow-up date.
- Status transitions via a stage stepper: Applied → Assessment → Interview (round n) → Offer, with Rejected / Withdrawn / Ghosted available at any point. Every transition writes a `job_event`, so the timeline is complete.
- Follow-up reminders: applications with `nextFollowUpOn <= today` surface in a "Needs follow-up" section and on Home.
- Metrics: counts by stage, response rate, average days to first response, interviews per application.

### 8.8 Body metrics
- Log weight (and optional body fat) with a date; height comes from settings.
- Current BMI band shown as a labelled scale, not a bare number: underweight / normal / overweight / obese, with the band boundaries visible.
- Delta since last log and since 30 days ago, with direction and colour that does **not** moralise (no red for gain — a bulking user is not failing).
- Weight chart with a 7-day moving average line, because daily weight is noisy.
- No points. **[ADD]** Optional: a weekly logging streak indicator, no points attached.

### 8.9 Rewards **[ADD — closes the loop promised in the plan's first line]**
- User creates rewards: name, cost in points, icon, repeatable or one-off, optional stock.
- Redeem button is disabled with a clear reason when the score is short: "Costs 500. You have 340."
- Redemption writes a negative ledger entry and appears in the activity log. Undo within the same day refunds.

### 8.10 Settings
Theme (system/light/dark), units, time zone, week start (fixed Monday), commitment bonus, habit reminder time, data export (JSON) **[ADD]**, delete account.

### 8.11 First-launch guide
A 5-step coach-mark tour, skippable, replayable from Settings:
1. Everything you log becomes points.
2. Build a plan first (link to Planner).
3. Log in two taps from the bottom bar.
4. Streaks pay bonuses — here is how they grow.
5. Undo is always available; points come back with it.
Store completion in `user_settings.onboardingCompletedAt`. For anonymous users store it locally so it does not repeat.

---

## 9. Design system

The client brief asks for fun, animated, mobile-first, dark and light. The direction below is the one to build; do not substitute a generic dashboard look.

### 9.1 Concept

**Cold steel, earned heat.** The interface is a quiet workshop: cool greys, precise rules, tabular numbers, nothing shouting. Colour temperature is the reward mechanic — warm colour appears *only* where the user has earned something. An untouched day is grey. A logged day glows. A PR is the hottest thing on the screen. This makes the heatmap and the score read instantly, and it means the celebration moments have somewhere to go, because the resting state is restrained.

Practical rule for every component: **if it is not showing an earned quantity, it is not warm.**

### 9.2 Tokens

```css
/* Light — "cold steel" */
--surface:        #F1F3F5;   /* cool paper, not cream */
--surface-raised: #FFFFFF;
--surface-sunken: #E4E8EB;
--ink:            #14171A;
--ink-muted:      #5A626B;
--line:           #D6DBE0;

/* Dark — "night shift" */
--surface:        #0F1216;
--surface-raised: #171B21;
--surface-sunken: #0A0C0F;
--ink:            #E9ECEF;
--ink-muted:      #949CA6;
--line:           #262C34;

/* Earned heat — the only warm colours in the app */
--heat-1: #FFC24A;   /* spark    — small gains, low heatmap steps */
--heat-2: #F5811F;   /* ember    — normal gains */
--heat-3: #E0521B;   /* forge    — big gains, PRs */
--heat-4: #B32D12;   /* crucible — top heatmap step, milestones */

/* Semantics */
--done:    #2E9E6B;  /* habit completed */
--penalty: #A8324A;  /* deliberately a cool-shifted red so it never reads as "heat" */
--info:    #3A6EA5;
--focus:   #4C9AFF;
```

Equipment chips (colour-coded per the plan) use desaturated cool hues so they do not compete with heat: barbell `#4A5A78`, dumbbell `#4F6F63`, machine `#6B5A78`, bodyweight `#5E6B72`, cardio `#3F6C7A`.

Radii: `4px` inputs and chips, `10px` cards, `20px` sheets. Do not put the same radius on everything. Elevation is a 1px `--line` border plus a tight shadow only on floating layers (sheets, popovers, the score pill) — never on every card.

### 9.3 Type

- **Display / metrics:** `Archivo Expanded` 600–700. Wide, industrial, built for numbers. Used for the score, PR values, and section headings.
- **Body / UI:** `IBM Plex Sans` 400/500/600. Engineering-adjacent, excellent tabular figures, not the default UI face.
- All numeric readouts use `font-variant-numeric: tabular-nums` so values do not jitter while animating.
- Scale (mobile): 12 / 14 / 16 / 20 / 26 / 34 / 46. Line height 1.5 body, 1.15 display.
- Sentence case everywhere. No all-caps labels, no eyebrow labels above headings.

### 9.4 Motion

One orchestrated moment, then restraint.

**The signature moment** is the points award: the delta appears at the source (the card the user just tapped), travels to the score pill in the header, and the pill counts up while a brief heat bloom passes through it. This is the app's memorable interaction, so it is worth polishing. Everything else is action-response: sheets slide, cards reorder with FLIP, checkmarks draw.

- PR celebration: spark burst scoped to the card + haptic + score animation. Not full-screen.
- Session complete and milestone: full-screen, ≤ 1.8 s, dismissible on tap.
- Consistency and commitment bonuses: a banded sweep across the habit list.
- **No entrance animations on page load.** No hover transitions on every card.
- `prefers-reduced-motion: reduce` must remove all non-essential motion and replace celebrations with a static state change plus the number update. Test this; it is not optional.
- Target 60 fps: animate `transform` and `opacity` only. Celebrations use CSS/canvas, not DOM churn.

### 9.5 Layout

Mobile-first, single column, content max-width 560px, 16px gutters. At ≥ 1024px, a two-column layout on Home (score + heatmap left, goals + log right) and a persistent left nav replacing the bottom bar. Tap targets ≥ 44px. Primary actions sit in the lower third of the screen, within thumb reach.

### 9.6 Quality floor
Keyboard focus visible everywhere. Colour never the sole carrier of meaning. Contrast ≥ 4.5:1 for text in both themes. Live regions announce points changes. Every empty state proposes an action ("No plans yet. Build your first one.") and every error says what to do next.

---

## 10. Non-functional requirements

**Testing**
- Points engine: unit tests per rule, plus property-based reconciliation tests (§5.3). Target 90%+ coverage on the `points` module. This module is where correctness actually matters.
- Backend: `@SpringBootTest` slice tests per module, H2 in-memory, one seeded fixture user.
- Frontend: Vitest for stores and pure logic; Playwright for four end-to-end journeys — sign up → create plan → log a set → see points; habit 7-day streak bonus; run milestone; undo restores the score.
- Every bug fix ships with the test that would have caught it.

**Performance**
- Home must render from a single `/home/summary` call. No N+1 waterfalls.
- Heatmap reads only `daily_summary`, never recomputes on request.
- Lazy-load every route except Home. Initial JS budget 250 KB gzipped; fail the build if exceeded.

**Security**
- BCrypt (cost 12), JWT signed with a rotating secret from the environment, refresh tokens hashed at rest.
- Every query filtered by `user_id` at the repository layer. Add an integration test that asserts user A cannot read user B's rows for every resource.
- Validate all input server-side; the sanity limits in §5.7 are enforced in the domain layer, not just the form.
- Rate-limit auth endpoints.

**Observability**
- Structured JSON logs with a request ID. Log every ledger write at INFO with rule code and amount.
- `/actuator/health` and a `/actuator/metrics` counter per rule code.

**Data portability**
- JSON export of everything the user owns. Import is out of scope for v1 but keep the export schema versioned.

---

## 11. Build order

Each milestone is independently runnable. Do not start the next one until the current one is demoable and its tests pass.

**M0 — Foundations (1 slice)**
Monorepo, Gradle + Angular scaffolds, Flyway baseline, Docker Compose for Postgres (unused in dev but ready), CI running both test suites, `CLAUDE.md`, design tokens implemented as CSS custom properties with theme switching, and the ~15 UI primitives (button, card, chip, sheet, date stepper, stepper input, toast, empty state, skeleton).

**M1 — Identity and shell**
Signup, login, refresh, logout, settings. App shell with header, hamburger, bottom nav, theme toggle, score pill (static). Anonymous browsing with the seeded demo dataset. Auth-required interception with action replay.

**M2 — Points engine**
Ledger, rule config, award/reverse/snapshot, idempotency, score cache, reconciliation framework, day rollover job, full test suite. No UI beyond a debug page. **This milestone is the project's spine — do not rush it.**

**M3 — Habits**
Habit planner, habit board, logging, streaks, consistency bonus, strict penalties, commitment bonus, edit-window locking, bonus hint strip, celebrations. First real end-to-end points flow.

**M4 — Workouts**
Exercise catalog and autocomplete, plan builder with drag-drop, elite and optional buckets, session tracker, set logging with weight modes, PR calculation and bonuses, swap, history charts, card reordering on completion, rest timer.

**M5 — Runs**
Run logging, type rules, distance points, milestones, records by bracket, run-only score header.

**M6 — Home**
Daily summary rollup, heatmap with state and emoji rules, quote pool and daily selection, score breakdown, activity log, goal bars, job metrics slot.

**M7 — Goals, Jobs, Body**
Goal kinds and auto-progress, job pipeline with events and metrics, body metrics with BMI bands and moving average.

**M8 — Rewards, onboarding, polish**
Rewards and redemption, first-launch guide, empty states, reduced-motion pass, accessibility audit, celebration polish.

**M9 — Android and hardening**
Capacitor wrap, safe areas, haptics, local notifications, back-button handling, offline queue, Play Store build, Postgres switch, load-test the ledger with 100k entries.

---

## 12. Working agreement for Claude Code

- Read `CLAUDE.md` before every session.
- Work one milestone at a time; open one PR per milestone slice with a short description of what changed and which open decisions were touched.
- Never award points outside `PointsService`. Never compute a score in the frontend.
- Never delete a ledger row. Never edit `amount`.
- No `LocalDate.now()` outside `DayService`. No `new Date()` for logical dates in the frontend.
- No magic numbers for points — everything comes from `points_rule_config`.
- When the spec is ambiguous, pick the documented default in §13, note it in the PR, and keep going. Do not stall.
- Prefer boring, readable code. This app will be maintained by one person.

---

## 13. Open decisions (defaults chosen, confirm when convenient)

1. **Milestone distances** — "21" and "42" are read as 21.1 km and 42.2 km (half and full marathon). *Default: yes.*
2. **Points for logging a set** — the plan only specifies PR bonuses. *Default: 1 point per set, plus PR bonuses, configurable to 0.*
3. **Milestone stacking** — a 25 km run could pay 10 K + 15 K + 21 K + 25 K. *Default: highest milestone only, doubled if it is a first-ever.*
4. **Consistency bonus cap** — uncapped geometric growth breaks the economy. *Default: cap the multiplier exponent at 12 (84 days).*
5. **Strict habit penalties on rest days** — a habit scheduled Mon–Fri should not be penalised on Sunday. *Default: added `scheduleDays`; penalties apply only on scheduled days.*
6. **Workout edit window** — unspecified. *Default: 7 days.*
7. **Units** — *Default: metric (kg, km), with an imperial toggle in settings deferred to M9.*
8. **Rewards** — implied by the plan's first line, absent from the page list. *Default: build a minimal version at M8.*
9. **Job points** — *Default: 0, config-enabled, since job activity is easy to inflate.*
10. **Quotes** — *Default: a curated, attribution-verified local pool of ~200, not a third-party API.*
11. **Multiple active plans** — *Default: several plans may exist; one is "active" and preselected in the tracker.*
12. **Body-weight PRs** — *Default: added weight only counts toward PR ordering, per the plan; total including body weight is displayed but not ranked.*

---

## 14. What was added beyond the original plan, and why

| Addition | Reason |
|---|---|
| Rewards screen | The plan's own premise is spending points; without a sink the loop is open. |
| Bottom navigation | Hamburger-only navigation costs two taps on the screens used daily. |
| Offline logging queue | Gyms have no signal; a failed log loses the point *and* the habit of logging. |
| Idempotency keys | Retries on mobile networks would otherwise double-award points. |
| Consistency bonus cap and daily caps | Prevents the points economy from collapsing. |
| Habit schedule days | Weekday-only habits currently fail every weekend. |
| Rest timer | Standard in lifting apps and nearly free given the session UI already exists. |
| Demo dataset for anonymous users | The plan allows browsing without login, but every screen would be empty. |
| Job event history | Makes the pipeline auditable and the metrics computable. |
| Weight moving average | Daily weight is noisy enough to be misleading without it. |
| Data export | Cheap insurance for a single-user-owned dataset. |
| Reduced-motion path | The app is animation-heavy by design; this keeps it usable for everyone. |
