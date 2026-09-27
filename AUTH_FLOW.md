# VEYA — Auth Flow (as of 0.1-D)

## Authentication state model

Six explicit states (`lib/features/auth/application/auth_controller.dart`), not a boolean and not two separate providers layered on `AsyncValue` (that was 0.1-C's shape — replaced in 0.1-D because it didn't give a single, nameable answer to "what is auth doing right now?"):

| State | Meaning |
|---|---|
| `AuthInitializing` | Before the Supabase session stream has emitted its first value. Router shows a splash, makes no redirect decision. |
| `AuthUnauthenticated` | No valid session. |
| `AuthAuthenticating` | A sign-in/sign-up call is in flight. |
| `AuthAuthenticated(user)` | A valid session exists. The only state that grants access to protected routes. |
| `AuthSigningOut` | A sign-out call is in flight. |
| `AuthenticationError(failure)` | The last sign-in/sign-up attempt failed, or the session stream itself errored. |

`AuthNotifier` (a Riverpod `Notifier<AuthState>`) is the single source of truth, subscribing exactly once to `AuthRepository.authStateChanges()` in `build()` and cancelling that subscription in `ref.onDispose` — verified in `test/unit/auth_notifier_test.dart` (TEST 12) that there is exactly one listener on the underlying stream, and it's removed on dispose.

**Routing rule (fail-closed):** only `AuthAuthenticated` counts as authenticated for redirect purposes. Every other state — including `AuthAuthenticating` and `AuthSigningOut`, both of which represent identity that is *not yet confirmed* — is treated as not-authenticated. This means an in-flight sign-in can never grant protected access before it actually succeeds, and a sign-out revokes access to protected screens the instant it's requested, not only once the network call completes.

## Sign up
1. User submits email + password on `SignUpScreen`.
2. `AuthNotifier.signUp()` sets state to `AuthAuthenticating` immediately, then calls `SupabaseAuthRepository.signUp()` → `supabase.auth.signUp()`.
3. On success, Supabase creates the `auth.users` row, which fires `handle_new_user()` (migration 0003), atomically creating `profiles` and `user_privacy_settings` rows with a placeholder username.
4. `AuthNotifier` reads `repository.currentUser` right after the call resolves and sets `AuthAuthenticated(user)` explicitly — it does not wait for the session stream to independently catch up, so there is no window where the UI is stuck on `AuthAuthenticating` after a successful call.
5. `go_router`'s redirect sends the user to `/`. `_HomeGate` watches the user's own profile; since `Profile.hasPlaceholderUsername` is true right after signup, it shows `CompleteProfileScreen` instead of `ProfileScreen`, prompting a real username before continuing.
6. On failure, `AuthNotifier` sets `AuthenticationError(failure)` — the screen shows the user-safe message and the form is re-enabled (never stuck showing a spinner).

## Sign in
Same shape as sign up: `AuthAuthenticating` → `SupabaseAuthRepository.signIn()` → either `AuthAuthenticated(user)` or `AuthenticationError(failure)`.

## Sign out
1. `ProfileScreen`'s sign-out button → `AuthNotifier.signOut()`.
2. State is set to `AuthSigningOut` **immediately** — the router's fail-closed rule means protected screens stop being reachable at this instant, not after the network call finishes.
3. `supabase.auth.signOut()` is called. Whether it succeeds or throws, state is forced to `AuthUnauthenticated` in a `finally` block — sign-out never leaves stale authenticated state in the provider even if the remote call has trouble (0.1-D §8).

## Session restoration
`supabase_flutter` persists the session locally and restores it automatically on cold start (see "Session storage" in `SECURITY_DECISIONS.md` for what "locally" means and its trade-offs). Until the auth stream emits its first value, `AuthNotifier`'s state is `AuthInitializing`; the router makes no redirect decision in that state, and route `/` renders `_SplashScreen` instead of guessing. `main.dart` calls `Supabase.initialize()` and `await`s it *before* `runApp()`, so the widget tree (and the router) never builds before Supabase itself is ready — there is no window where navigation could occur before initialization, which was verified by reading the actual bootstrap sequence, not assumed (0.1-D §13).

## Session expiration
The Supabase SDK auto-refreshes tokens. On unrecoverable expiration, the auth stream emits `null` → `AuthNotifier` sets `AuthUnauthenticated` → the same fail-closed redirect path as an explicit sign-out sends the user to `/sign-in`. If the stream itself errors (e.g. a network failure resolving the session) rather than cleanly emitting `null`, `AuthNotifier`'s `onError` handler still sets `AuthUnauthenticated` — an error is never interpreted as "still authenticated" (fail closed, 0.1-D §7/§17).

## Verification status
The state machine, router integration, and error handling are **implemented and statically reviewed**, with unit and widget tests written (`test/unit/auth_notifier_test.dart`, `test/widget/app_router_test.dart`, updated `test/widget/sign_in_screen_test.dart`). These have **not been executed** — no Flutter SDK in this container (unchanged, pre-existing limitation — see `docs/RELEASE_STATUS.md`). The underlying database/RLS layer this flow depends on (trigger, RLS, grants) **was executed and re-verified live** as a regression check during 0.1-D, with no changes made to it.
