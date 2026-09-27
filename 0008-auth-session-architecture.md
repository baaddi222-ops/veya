# ADR-0008: Auth/session architecture — single stream, centralized redirect

Date: 2026-09-24
Status: Accepted
Release: VEYA 0.1

## Decision
Auth state is exposed as a single Riverpod `StreamProvider<AuthState>` wrapping `supabase.auth.onAuthStateChange`. Session restoration, expiry, and sign-out all flow through this one stream, and `go_router`'s `redirect` is the only place that acts on it for navigation.

## Context / Problem
Session restoration, expiry, and manual sign-out are really the same underlying event from the app's point of view (auth state changed) and should not need three separate code paths, or per-screen checks that are easy to forget when adding new screens.

## Alternatives Considered
- Per-screen auth checks (each screen's `initState` checks `supabase.auth.currentSession` and navigates manually): duplicated logic, easy to miss on a new screen, and doesn't naturally handle expiry occurring while a screen is already open.
- A separate "session watcher" widget wrapping the whole app outside the router: workable, but `go_router`'s built-in `refreshListenable` + `redirect` already does exactly this with less custom code.

## Reason for Decision
One stream, one redirect function. The `AsyncLoading` state before the stream's first emission is treated explicitly as "don't redirect yet" (rather than assuming unauthenticated), which is what prevents a wrong-screen flash on cold start while a persisted session is still being restored.

## Consequences
Any future feature that needs to react to sign-out (e.g. clearing feature-specific cached state in 0.2+) can listen to the same `authStateProvider` rather than needing a new notification mechanism.
