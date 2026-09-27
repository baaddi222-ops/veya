# ADR-0012: Encrypted session storage via a custom LocalStorage

Date: 2026-09-26
Status: Accepted
Release: VEYA 0.1-D (follow-up)

## Decision
Supabase session persistence (access/refresh tokens) uses a custom `LocalStorage` implementation (`SecureSessionStorage`) backed by `flutter_secure_storage` — platform Keychain on iOS/macOS, Keystore-backed encrypted storage on Android — instead of `supabase_flutter`'s own default.

## Context / Problem
The 0.1-D audit found that `supabase_flutter` (v2.x, as pinned in this project) persists the session via plain `SharedPreferences` by default: sandboxed per-app by the OS, but not encrypted at rest. This was initially documented as a deliberate, un-fixed known limitation rather than auto-patched, specifically so the trade-off could be weighed rather than reflexively resolved with a new dependency. On review, the trade-off favors fixing it for VEYA specifically (see "Reason for Decision"), so this ADR formalizes that decision.

**In fairness to `supabase_flutter`'s own default**, a maintainer's stated rationale for not using encrypted storage as the SDK's default is legitimate and worth recording here rather than dismissed: the access token is equally accessible via `localStorage` on the web platform regardless of what native mobile does, so encrypted-by-default wouldn't be a genuine security improvement on every platform the SDK targets, and `SharedPreferences` is supported everywhere with one fewer dependency. This is a reasonable default for a general-purpose SDK targeting web and mobile uniformly.

## Alternatives Considered
- **Keep the SDK default (SharedPreferences), do nothing:** the original 0.1-D position. Reasonable for a foundation release, but VEYA is mobile-first (per the master spec) and is explicitly a multi-year project handling increasingly sensitive data (messaging, guardian/minor data in later releases) — the calculus is different for VEYA specifically than for a generic cross-platform SDK default.
- **`flutter_secure_storage` for the whole app's future local caching needs, not just the session:** rejected as scope creep — 0.1 has no other local-cache requirement yet: this ADR covers session storage only.

## Reason for Decision
VEYA's mobile-first, multi-year, increasingly-sensitive-data trajectory justifies encrypting the one piece of local state that would matter most if a device were compromised (the session token, which is bearer-equivalent to being signed in). The cost is one additional, narrowly-scoped dependency (`flutter_secure_storage`) used for exactly this purpose, not a general-purpose local-storage layer.

## Consequences
One new dependency added to `pubspec.yaml`. `SecureSessionStorage` implements exactly the four-function contract `LocalStorage` requires (`initialize`, `hasAccessToken`, `accessToken`, `persistSession`, `removePersistedSession`) and nothing else — no broader storage abstraction was introduced. This project's web-build aspiration (noted in ADR-0003) is not currently affected either way, since `flutter_secure_storage`'s web implementation falls back to browser storage with equivalent characteristics to the SDK default there — mobile is where this change actually matters, which matches VEYA's stated mobile-first priority.
