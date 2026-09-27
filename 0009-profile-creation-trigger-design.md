# ADR-0009: Profile creation via a `SECURITY DEFINER` trigger

Date: 2026-09-24
Status: Accepted
Release: VEYA 0.1

## Decision
A single `AFTER INSERT` trigger on `auth.users`, `handle_new_user()`, atomically creates one `profiles` row and one `user_privacy_settings` row per new user, using a `SECURITY DEFINER` function with pinned `search_path` and fully schema-qualified object references.

## Context / Problem
RLS forbids client-side INSERT on `profiles`/`user_privacy_settings` (ADR-0007's ownership model requires this — a client-creatable profile row would let a user claim someone else's `auth.uid()`, or leave orphaned/duplicate rows). Something server-side must create these rows on signup, reliably and atomically.

## Alternatives Considered
- Client calls an explicit "create my profile" endpoint/RPC right after signup: adds a window where a user is authenticated but has no profile row yet (race condition if the app crashes or the call is skipped/retried oddly), which the app would then need to handle as a special case everywhere.
- A Supabase Edge Function triggered on signup webhook: more moving parts and an extra network hop for something a plain Postgres trigger does atomically and synchronously, in the same transaction as the signup itself.

## Reason for Decision
A database trigger inside the same transaction as the `auth.users` insert is the only approach that makes "every authenticated user has exactly one profile row" a true invariant rather than a best-effort convention — if either insert inside the trigger fails, the whole transaction (including the `auth.users` row) rolls back.

`SECURITY DEFINER` is required because the trigger's execution context isn't an RLS-authenticated session with insert rights. This requires two specific, non-negotiable mitigations against Postgres's known `SECURITY DEFINER` search-path-hijack vector: `set search_path = public, pg_temp` pinned on the function, and every referenced object schema-qualified. The function's scope is kept to exactly two inserts — no dynamic SQL, no broader privilege exercised than required.

A placeholder username (`user_` + 8 hex chars of the new UUID) is generated inside the trigger, rather than derived from the user's email, specifically to remove unique-constraint collisions as a realistic trigger-failure cause — email-derived usernames can collide; UUID-derived ones cannot.

## Consequences
Signup fails atomically (and loudly, via a generic Supabase Auth error) if profile creation fails for any reason, rather than silently leaving a user without a profile. The app includes a post-signup "choose your real username" step — `CompleteProfileScreen`, gated in `core/router/app_router.dart` via `Profile.hasPlaceholderUsername` — so users aren't stuck with the placeholder. (Initially shipped as a documented known limitation with the repository method ready but no UI calling it; closed in a follow-up pass — see `docs/RELEASE_STATUS.md`.)
