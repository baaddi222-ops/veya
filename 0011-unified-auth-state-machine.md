# ADR-0011: Unified, fail-closed AuthState (replacing the two-provider split)

Date: 2026-09-26
Status: Accepted
Release: VEYA 0.1-D
(Supersedes the state-management half of ADR-0008, which described the original two-provider design.)

## Decision
Authentication state is a single sealed `AuthState` with six variants (`AuthInitializing`, `AuthUnauthenticated`, `AuthAuthenticating`, `AuthAuthenticated`, `AuthSigningOut`, `AuthenticationError`), owned by one `AuthNotifier`. The router treats every variant except `AuthAuthenticated` as not-authenticated.

## Context / Problem
0.1-C's design used two providers: `authStateProvider` (a `StreamProvider<AuthState>` with only `Authenticated`/`Unauthenticated` variants, wrapped in Riverpod's own `AsyncValue` for loading/error) and `authControllerProvider` (an `AsyncNotifier<void>` tracking in-flight sign-in/up/out actions separately). This worked, but required unwrapping two layers (`AsyncValue.when` around a 2-variant enum) to answer "what is auth doing right now?", and — more importantly — treated a stream error as "make no redirect decision," which could leave a user stuck on a splash screen indefinitely if the session stream ever errored (e.g. a transient network failure while resolving a persisted session).

## Alternatives Considered
- Keep the two-provider design and just add error handling to the `StreamProvider`'s `error` branch in the router: smaller diff, but still leaves two separate state sources that can drift out of sync (e.g. `authControllerProvider` shows `isLoading: true` for an in-flight sign-in while `authStateProvider` still reports the old `Unauthenticated` value from before the attempt — technically fine today, but a growing surface for inconsistency as more auth-dependent UI is added in later releases).
- A full `bloc`-style event/state machine: more ceremony than justified for 0.1's scope; would also be a bigger, less justified rewrite than the master spec's "only modify what is necessary" instruction allows for.

## Reason for Decision
A single flat state, matching the explicit vocabulary VEYA 0.1-D's own specification asked for, gives every consumer (router, both auth screens, the sign-out button) one thing to switch on. Treating every non-`Authenticated` state as not-authenticated for routing is a fail-closed default: an in-flight action, an error, or an unresolved stream can never be mistaken for a confirmed session. This directly closes a real gap found during the 0.1-D audit (a stream error previously produced an indefinite splash with no recovery path).

## Consequences
`AuthNotifier.signIn()`/`signUp()` read `repository.currentUser` immediately after a successful call rather than only waiting for the stream to independently emit — this avoids a timing gap where the UI would sit in `AuthAuthenticating` a moment longer than necessary, at the cost of one extra (synchronous, in-memory) read per successful call. `signOut()` unconditionally forces `AuthUnauthenticated` in a `finally` block even if the remote call throws, which means a user's local "signed out" experience is honored immediately regardless of network trouble — consistent with 0.1-D §8's explicit requirement, and consciously prioritizing the user's stated intent (leave the account) over strict consistency with the remote call's outcome.
