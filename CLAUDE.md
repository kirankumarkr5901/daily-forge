# CLAUDE.md — DailyForge

Read this at the start of every session. The full specification is `docs/DAILYFORGE_SPEC.md`; this file is the short operating manual. Durable product truth and the decisions confirmed with the owner live in `PRODUCT.md`. The deployment phase plan is `docs/DEPLOYMENT.md`.

## What this is
A manual-logging fitness and habit tracker where every action converts to points, and points are the single currency across the app. Angular frontend, Spring Boot backend, Capacitor for Android.

## Commands
```bash
# backend
cd backend && ./gradlew bootRun          # http://localhost:8080
./gradlew test                            # unit + slice tests
./gradlew flywayInfo

# frontend
cd frontend && npm start                  # http://localhost:4200
npm test                                  # vitest
npm run e2e                               # playwright
npm run build -- --configuration=production

# android
npm run build && npx cap sync android && npx cap open android
```

## Non-negotiable invariants
1. **Points are awarded only by `PointsService`.** No other module writes to `points_entry`.
2. **The ledger is append-only.** Undo writes a compensating entry with `reverses_id` set. Never delete, never edit `amount`.
3. **Score is `SUM(amount)`.** `user_score_cache` is a cache; a recalculate endpoint must be able to rebuild it exactly.
4. **The frontend never calculates points.** It renders `points.delta`, `points.newTotal`, and `celebrations` returned by the API.
5. **All day logic uses the user's time zone**, via `DayService`. No `LocalDate.now()` anywhere else; no `new Date()` for logical dates in the frontend.
6. **No magic numbers.** Every point value, multiplier, and cap comes from `points_rule_config` or the user's own configuration.
7. **Every write endpoint is idempotent** via the `Idempotency-Key` header.
8. **Every query is scoped by `user_id`** at the repository layer.
9. **Migrations only.** `hibernate.ddl-auto=validate`. Schema changes go through Flyway.
10. **Editing the past triggers reconciliation.** Any change to habit logs, sets, or runs must recompute derived bonuses and reconcile the ledger in the same transaction.

## Module boundaries
`common`, `points`, `identity`, `workout`, `habit`, `activity`, `run`, `goal`, `job`, `body`, `reward`, `insight`.
Modules call each other through service interfaces only. Never inject another module's repository.

## Frontend conventions
- Standalone components, lazy routes, signals for state, RxJS for streams.
- **Every component is a folder** containing three files: `name.component.ts`, `name.component.html`, `name.component.scss`. No inline templates, no inline styles, no exceptions.
- Three shared signal stores: `SessionStore`, `PointsStore`, `SyncStore`.
- No Angular Material. Behaviour comes from `@angular/cdk`; visuals are our own primitives in `shared/ui`.
- Design tokens are CSS custom properties in `styles/tokens.css`. Never hardcode a hex value in a component.
- Every animation must have a `prefers-reduced-motion` path.
- No strings hardcoded in templates; use the `t()` lookup.

## Design in one line
Cold steel, earned heat: the interface is cool grey and quiet, and warm colour appears only where the user has earned something. See spec §9.

## Definition of done
- Tests pass, including the points reconciliation property tests.
- Works at 360px wide and with a keyboard alone.
- Both themes checked.
- Reduced-motion checked.
- Empty state and error state written, in plain language, proposing a next action.
- No new magic numbers, no new `LocalDate.now()`.

## When the spec is ambiguous
Use the default in spec §13, note the choice in the PR description, and keep moving. Do not stall on a decision that has a documented default.
