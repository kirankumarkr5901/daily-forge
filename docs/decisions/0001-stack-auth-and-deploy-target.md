# ADR 0001 — Stack, auth, and deployment target

**Date:** 2026-09-06
**Status:** Accepted

## Context

The spec (`DAILYFORGE_SPEC.md` v1.0) commits to a Java/Spring Boot backend, but the owner mentioned holding a Firebase account and asked for Google OAuth and a step-by-step deployment plan. Those needed reconciling before any code was written, because they determine the data model, the identity module, and the deployment pipeline.

## Decisions

1. **Backend stays Spring Boot, as specced.** A Firebase-native design (Firestore + Cloud Functions) was considered and rejected: the append-only ledger, the recompute-and-reconcile pass (§5.3), the property-based reconciliation tests, and the idempotent daily rollover job all assume one transactional SQL database with window functions. Rebuilding those on Firestore would trade a week of setup for a permanent correctness tax on the one module where correctness actually matters.

2. **Google OAuth *and* email/password, both in v1.** One user record may carry either or both. The frontend obtains a Google ID token; the backend verifies it server-side and issues DailyForge's own JWT, so a Google token is never a session token. Google's `sub` is stored so an email change does not orphan an account, and a verified Google email matching an existing password account links rather than duplicates.

3. **Firebase is not a dependency.** Google sign-in is implemented against Google Identity Services directly, which works identically on the chosen free-tier hosts. The Firebase account remains an available fallback for identity or hosting if preferred later. *This is an inference from the owner's "just in case" framing, not an explicit instruction — it is cheap to reverse.*

4. **Deployment target: free-tier friendly.** Netlify or Vercel for the Angular build, Render or Fly.io for the backend container, Neon or Supabase for PostgreSQL. The vendor inside each slot is deliberately left open because nothing in the design depends on a vendor-specific API.

5. **First deployable slice: spec milestones M0–M3** — foundations and design system, identity and shell, the points engine, then habits. Habits is the cheapest feature that exercises the entire points engine end to end, so it proves the spine before workouts are built on top of it.

## Consequences

- Local development uses **file-mode** H2, not in-memory, so streak testing survives restarts. Tests use in-memory H2.
- Flyway plain-SQL migrations must stay portable across H2 (`MODE=PostgreSQL`) and PostgreSQL 16. Any query that cannot be expressed portably goes behind a repository interface with two implementations, never a branch in a service.
- Free-tier backends sleep. The rollover job must catch up missed days rather than assuming it ran hourly — already required by §5.6's idempotency rule, but now load-bearing rather than theoretical.
- The deployment pipeline is built at Phase 1, before the app is feature-complete. See `docs/DEPLOYMENT.md`.
