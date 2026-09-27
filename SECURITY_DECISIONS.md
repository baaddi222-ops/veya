# VEYA — Security Decisions (0.1)

## RLS strategy: owner-only default

Every table has RLS enabled from its creation migration — never added "later." In 0.1, both `profiles` and `user_privacy_settings` are readable and writable **only by their owner** (`auth.uid()` match via the `public.is_owner()` helper function). No anonymous or cross-user read policy exists.

**Why:** 0.1 has no follows, blocking, reporting, or moderation, and no age-category system. A public-by-default profile directory would have real exposure cost and zero present product benefit (nothing in 0.1 browses other users' profiles). Starting owner-only and adding a public-read policy additively in 0.2 — once follows/blocking/reporting exist together — is a strictly safer migration path than starting public and tightening later. Full reasoning: architecture review §2, ADR-0007.

## Authorization model per table

### `profiles`
- **SELECT**: owner only.
- **INSERT**: none via client — only the `handle_new_user()` trigger creates rows.
- **UPDATE**: owner only, and only while `account_status = 'active'`.
- **DELETE**: none via client — account deletion is a future orchestrated server-side process, not a bare `DELETE` (architecture review §6).

### `user_privacy_settings`
- **SELECT / UPDATE**: owner only.
- **INSERT / DELETE**: none via client — same trigger-owns-lifecycle reasoning.

## Signup trigger security properties

`handle_new_user()` (migration 0003) is `SECURITY DEFINER` because the `auth.users` insert context has no RLS-authenticated session with insert rights on `public` tables. This requires explicit mitigation of the well-known Postgres `SECURITY DEFINER` search-path-hijack vector:

1. `set search_path = public, pg_temp` pinned on the function itself.
2. Every referenced object schema-qualified (`public.profiles`, never bare `profiles`).
3. Minimal scope — exactly two inserts, nothing else, no dynamic SQL.

Both inserts happen in the same transaction as the `auth.users` insert; a failure in either rolls back the whole signup, so an authenticated user without a profile row is architecturally impossible. Full reasoning: `docs/adr/0009-profile-creation-trigger-design.md`.

## Secrets

- Supabase **anon/public key** and project URL are the only Supabase values present in the Flutter client, injected via `--dart-define-from-file` (see `core/config/env.dart`).
- The Supabase **service-role key** is never read, referenced, or present anywhere in this repository's Flutter code.
- `.env` is git-ignored; `.env.example` ships with placeholders only.

## Error handling / information leakage

Raw `PostgrestException`/`AuthException` text never reaches the UI. `SupabaseAuthRepository._mapAuthMessage()` and `SupabaseProfileRepository` map known error codes to user-safe strings; anything unmapped falls back to a generic message — never the backend's raw error text.

## Logging

`AppLogger` never receives tokens, passwords, or full PII. Error logs include exception objects/stack traces for debugging, not request payloads.

## VEYA 0.1-E — Profile & Account Foundation

0.1-E's security-relevant work was entirely about closing gaps in the *client-side expression* of the security boundary, not the boundary itself (which 0.1-C already established and 0.1-E's regression pass re-confirmed unchanged).

**Real gap: `display_name` had no length constraint.** `bio` has had a 280-char CHECK constraint since 0.1-B; `display_name` had none — confirmed live before fixing (a 51-char value was accepted with no constraint present). Fixed in migration `0007` (50-char CHECK, Unicode-codepoint-aware via `char_length()`), confirmed live afterward (51 chars rejected, a 34-codepoint mixed Arabic/Latin/emoji string accepted correctly).

**Real gap: no typed boundary on the Dart side for updatable fields.** Before 0.1-E, `updateMyProfile` accepted four loose parameters — functionally safe today (0.1-C's column-level grants are the actual enforcement), but structurally nothing stopped a future change from adding a fifth, unsafe one to that same pattern. `ProfileUpdate` (a closed value object) is a second, independent layer on top of the database grants — belt-and-braces, not a replacement for them.

**Confirmed unchanged (re-verified live in 0.1-E's regression pass):** RLS owner-only reads/updates, column-level grants blocking `created_at`/`account_status` modification, anonymous access fully denied, duplicate-username rejection, `user_privacy_settings` ownership boundary.

**New finding, out of 0.1-E's scope, not silently dropped:** Supabase's security advisor flagged `auth_leaked_password_protection` (HaveIBeenPwned checking) as disabled — a project-level Auth configuration setting, not a schema/RLS/grant matter. No tool in this session's Supabase MCP toolset could toggle it (only SQL execution and migration tools were available, not Auth-config management). Documented as **BLOCKED**, requiring either dashboard access or different tooling in a future session.

## VEYA 0.1-D — Authentication Foundation Audit

0.1-D's objective was to establish a secure, maintainable session/auth foundation on the Flutter side — no database changes. The audit re-read the actual current auth repository, router, and profile relationship from scratch, confirmed the following were already correct (unchanged), and fixed two real gaps.

**Confirmed already correct, unchanged:**
- Identity is always derived from the Supabase session (`repository.currentUser` / `authStateChanges()`), never from a caller-supplied id. Neither `AuthRepository` nor `ProfileRepository` accepts an ownership id parameter anywhere in their interfaces — this is a compile-time guarantee, not just a convention (0.1-D §2, §14).
- No manual profile-row creation exists in Flutter — the `handle_new_user()` trigger (0.1-B/C) remains the sole creator, confirmed by a full repo search for insert-into-`profiles` logic (0.1-D §15).
- `main.dart` awaits `Supabase.initialize()` before `runApp()`, so the widget tree and router never build before Supabase itself is ready — there is no window for navigation-before-initialization (0.1-D §13).
- Exactly one listener on the auth stream for the app's lifetime — verified directly in `test/unit/auth_notifier_test.dart` (TEST 12) against a fake repository's `StreamController`, asserting `hasListener` is `true` after the provider is read and `false` after the container is disposed.

**Real gap 1 — no unified, fail-closed auth state model.** 0.1-C's design split state across two providers (`authStateProvider: StreamProvider<AuthState>` for identity, `authControllerProvider: AsyncNotifier<void>` for in-flight actions), and treated a stream error as "no redirect decision" — which meant a network error resolving the session could leave a user stuck on a splash screen indefinitely, with no fail-closed fallback. Fixed by consolidating into one `AuthNotifier` exposing six explicit states (`AuthInitializing`, `AuthUnauthenticated`, `AuthAuthenticating`, `AuthAuthenticated`, `AuthSigningOut`, `AuthenticationError`), with the router treating everything except `AuthAuthenticated` as not-authenticated — including a stream error, which now resolves to `AuthUnauthenticated` rather than an indefinite hang. See ADR-0011.

**Real gap 2 — session storage encryption — found, then fixed.** Checked precisely (not assumed): `supabase_flutter` v2.x (pinned in this project) persists the session — including the access and refresh token — via `SharedPreferences` by default, not an encrypted store. `SharedPreferences` is sandboxed per-app by the OS but is **not encrypted at rest**. Initially documented as a deliberate, un-fixed known limitation so the trade-off could be weighed rather than reflexively patched. On review, fixed: `lib/core/config/secure_session_storage.dart` implements a custom `LocalStorage` backed by `flutter_secure_storage` (platform Keychain/Keystore), wired in via `Supabase.initialize(authOptions: FlutterAuthClientOptions(localStorage: SecureSessionStorage()))` in `main.dart`. See ADR-0012 for the full reasoning, including the SDK maintainer's legitimate counter-argument for the plain default (web-platform parity, one fewer dependency) — recorded rather than dismissed, since VEYA's mobile-first, multi-year trajectory is what tips the trade-off, not a claim that the SDK's own default is simply wrong.

**Security audit items confirmed (0.1-D §17), all by static review — no Flutter runtime available to execute against:**
- No token/password ever passed to `AppLogger` — every log call site reviewed; only exception objects and internal messages are logged, never raw credentials.
- No `service_role` key, database password, or other privileged secret anywhere in the Flutter client (re-scanned the full repo; same two documentation-comment matches as in every prior scan, confirmed not actual secrets).
- `--dart-define` (not `flutter_dotenv`) remains the config strategy, unchanged — no new environment mechanism introduced.

## VEYA 0.1-C — Security Foundation Audit (executed against `veya-dev`)

0.1-C's objective was to audit and harden the security boundary established in 0.1-B — not to add features. The audit re-read every migration, repository, and trigger from scratch (not assumed correct from prior reports), then tested live. It found **two real, previously-undetected gaps**, fixed both, and re-verified.

### Finding 1 — Ownership field protection gap (RLS §8 of the audit spec)

**The gap:** the UPDATE RLS policy's `with check (public.is_owner(id))` verifies row ownership but not which columns changed. Live test confirmed an authenticated owner could set their own `created_at` to an arbitrary date and their own `account_status` to `'suspended'` via a normal UPDATE — both should be system/audit-owned fields, not client-settable.

**The fix (migration `0006_harden_table_and_column_privileges.sql`):** removed `authenticated`'s blanket table-level UPDATE grant and replaced it with column-level grants — `UPDATE (username, display_name, avatar_url, bio)` on `profiles`, `UPDATE (is_private_account)` on `user_privacy_settings`. This is checked by Postgres *before* RLS is even evaluated, and is authoritative regardless of how a request reaches the database (Flutter app, a direct REST call with the anon key, or any future client) — see ADR-0010.

**Re-verified live after the fix:** attempting to set `created_at`/`account_status` now fails with `permission denied for table profiles` (42501) — closed. A legitimate update to an allowed column (`username`) still succeeds, and the `updated_at` trigger still fires correctly (confirmed the `BEFORE UPDATE` trigger's internal column writes are not subject to the invoking role's column-grant restrictions — a genuine, live-tested distinction, not assumed).

### Finding 2 — `anon` held unnecessary table-level grants (Database Privileges §4)

**The gap:** `information_schema.role_table_grants`, queried live, showed `anon` held table-level `INSERT`/`UPDATE`/`DELETE`/`SELECT` on both tables (Supabase's broad-by-default convention). RLS already blocked `anon` completely (zero policies for `anon` = deny-all), but the table-level grant meant RLS enablement was the *only* thing standing between `anon` and full access — a single accidentally-dropped policy would have silently reopened both tables.

**The fix:** migration 0006 also revokes all table-level grants from `anon` entirely on both tables. Behavior change (documented, not hidden): an anonymous SELECT attempt now returns a `permission denied` error instead of silently returning zero rows — both are correctly "denied," but the error now surfaces one layer earlier. No current app flow depends on the old zero-rows behavior.

### Additional audit results (all executed live against `veya-dev`)

- **SECURITY DEFINER audit:** confirmed via `pg_proc` that `handle_new_user` is the only `SECURITY DEFINER` function (owner `postgres`, `search_path=public, pg_temp` pinned); `is_owner`/`set_updated_at` are correctly `SECURITY INVOKER`. Confirmed via `information_schema.routine_privileges` that only `service_role` can `EXECUTE handle_new_user`/`set_updated_at`, and only `authenticated`+`service_role` can `EXECUTE is_owner` (needed for its own RLS policies) — `anon` has none of these.
- **Trigger malicious-input test:** signed up a real test user with a SQL-injection-shaped string and a large unicode payload in `raw_user_meta_data`. The resulting profile was completely unaffected (`display_name`/`bio` null, normal placeholder username) — the trigger never reads `raw_user_meta_data` at all, so hostile metadata has zero effect by construction.
- **Direct RPC call to `handle_new_user()`** by an authenticated session: `permission denied for function handle_new_user` — confirmed still blocked after 0.1-B's fix.
- **Username constraint matrix:** empty string, 25-character string, uppercase, and a SQL-injection-shaped string were each attempted via UPDATE and each rejected by the appropriate `CHECK` constraint; a valid username was accepted.
- **Full regression pass:** all 10 original 0.1-B RLS scenarios (owner read/update, cross-user denial, anonymous denial, client INSERT/DELETE denial, suspended-account denial, privacy-settings owner-only, duplicate-username rejection, trigger row creation) were re-run after migration 0006 and still pass.
- **Supabase's live security and performance advisors** both returned zero findings after the fix (were already zero before this audit started, post-0.1-B's fix — confirming no regression).
- **Concurrent username collision:** not independently executable via true parallel connections in this environment (tool limitation — one statement stream at a time). Documented rather than faked: protection is a Postgres unique b-tree index, atomic by construction regardless of concurrency, which is exactly why the architecture never relied on an application-level "SELECT first, then write" check.

Full evidence, and the WRITTEN/STATICALLY VERIFIED/EXECUTED/BLOCKED classification of every item, is in `docs/RELEASE_STATUS.md` and `supabase/tests/rls_security_tests.sql`.

## Live security verification (executed against the `veya-dev` Supabase project)

All 10 scenarios in `supabase/tests/rls_security_tests.sql` were run live and passed — see `docs/RELEASE_STATUS.md` for the full evidence table. Two additional real findings surfaced only by running this against a live project (not by static review):

1. **Supabase's own security advisor** flagged that `handle_new_user()` — despite being `SECURITY DEFINER` and intended only for the `auth.users` trigger — was directly callable via PostgREST's automatic RPC exposure (`/rest/v1/rpc/handle_new_user`) by both `anon` and `authenticated`. Fixed in migration `0005_harden_trigger_function_privileges.sql` by revoking `EXECUTE` on `handle_new_user()`, `is_owner()`, and `set_updated_at()` from `anon`/`authenticated`/`public`. Re-running the advisor afterward returned zero findings.
2. Cascade delete (`ON DELETE CASCADE` from `auth.users` → `profiles` → `user_privacy_settings`) was confirmed to work correctly when cleaning up test data — deleting the `auth.users` row removed both dependent rows automatically, with no orphaned data.

This is a concrete example of why "statically verified" and "executed/verified" are kept as distinct states in this project: both the RPC-exposure issue above and the 0.1-C ownership-field gap were invisible to code review alone and only surfaced once live tests ran against a real database.

## Pre-implementation checklist (from the approved architecture review, §9) — updated after 0.1-C

- [x] Authentication only via Supabase Auth SDK — implemented; Flutter-side execution still not run (no SDK in this container — unrelated to 0.1-C's scope).
- [x] Authorization reasoned about for SELECT/INSERT/UPDATE/DELETE on both tables — implemented and **EXECUTED/VERIFIED live** (migrations 0004, 0006).
- [x] RLS enabled on every table — **EXECUTED/VERIFIED live**.
- [x] Cross-user access tested — **EXECUTED/VERIFIED live** (TEST 2, 4, 7 in `rls_security_tests.sql`).
- [x] Anonymous access tested — **EXECUTED/VERIFIED live** (TEST 3).
- [x] Profile ownership: no client path creates a row for another user's id — **EXECUTED/VERIFIED live** (TEST 5); ownership *field* protection (created_at/account_status) also now **EXECUTED/VERIFIED live** (TEST 11, 0.1-C finding).
- [x] Privacy settings owner-only, no exceptions — **EXECUTED/VERIFIED live**.
- [x] Trigger privileges: pinned `search_path`, full schema qualification, minimal scope — **EXECUTED/VERIFIED live** via `pg_proc`/`routine_privileges` inspection (0.1-C).
- [x] No service-role key in the client — verified by inspection (`grep` over the full repo, re-run in 0.1-C, still clean).
- [ ] Session handling verified end-to-end — **BLOCKED**, needs a Flutter SDK (unchanged, outside 0.1-C's database-security scope).
- [x] No raw exception text shown to users — implemented in repository error mapping (statically reviewed, unchanged in 0.1-C).
- [x] No sensitive data in logs — implemented (statically reviewed, unchanged in 0.1-C).
