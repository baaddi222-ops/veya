# VEYA 0.1 — Release Status

Last updated: 2026-09-26 (VEYA 0.1-E — Profile & Account Foundation)

## VEYA 0.1-E — Profile & Account Foundation

**Objective:** a clean, secure, extensible profile/account foundation — not a social profile page. Audit-first, per spec: read the actual current profile repository, screen, tables, RLS, and tests before changing anything.

**Confirmed already correct (no change needed):** no `getProfile(userId)` method exists — both repository methods operate only on the current session's own user, with no id parameter anywhere; no duplicate profile-creation logic in Flutter; `user_privacy_settings` ownership boundary intact.

**Real gaps found and fixed:**
1. **No typed update model.** `updateMyProfile` took four loose `String?` parameters and built a raw map — nothing stopped a future change from adding an unintended field (e.g. `accountStatus`) to that same pattern. Fixed: `ProfileUpdate` value object (`lib/features/profile/domain/profile_update.dart`) declares a closed set of editable fields; the repository interface now takes one typed parameter instead of four loose ones.
2. **No edit-profile UI existed at all** — `ProfileScreen` was read-only. Built `EditProfileScreen` (username/display name/bio, no optimistic UI, save button disabled while a save is in flight, validation errors surfaced, server errors surfaced, only changed fields sent).
3. **`display_name` had no length constraint at the database level** — `bio` did (280 chars, since 0.1-B), `display_name` didn't, a real, confirmed-live inconsistency. Fixed: migration `0007` adds a 50-char CHECK constraint, confirmed live (rejects 51 chars, accepts a 34-codepoint mixed Arabic/Latin/emoji string correctly — Unicode-safe).
4. **Duplicated/risked-drifting validation logic.** `CompleteProfileScreen` had its own inline username regex; a new edit screen would have duplicated it. Fixed: `ProfileValidators` — one shared, pure-Dart validator per field, mirroring the database CHECK constraints exactly, used by both screens.
5. **Avatar field existed but was never displayed.** `avatar_url` has existed since 0.1-B with no UI rendering it anywhere. Added display-only avatar rendering to `ProfileScreen` (network image or initial-letter placeholder) — deliberately no upload UI (see ADR-0013): offering a raw "paste an avatar URL" field would look like a real feature without being one.

**Deliberately not built:** privacy-settings UI (`is_private_account` remains inert until 0.2 introduces the first public-read policy — shipping a toggle with zero present effect would be misleading; see ADR-0014). Avatar upload/storage pipeline (explicitly future scope).

**New unplanned finding (out of 0.1-E's own scope, surfaced during the live regression pass, not silently dropped):** Supabase's security advisor flagged `auth_leaked_password_protection` (HaveIBeenPwned checking) as disabled at the project level. This is a Supabase Auth *project configuration* setting, not something reachable via the SQL-execution MCP tools available in this session — no tool to toggle it was found. Documented here as **BLOCKED**, to be resolved via the Supabase dashboard directly (Authentication → Policies → Leaked Password Protection) or in a future release with the right tooling — not fixed, not ignored.

**Files created:** `lib/features/profile/domain/profile_update.dart`, `lib/features/profile/domain/profile_validators.dart`, `lib/features/profile/presentation/edit_profile_screen.dart`, `supabase/migrations/0007_add_display_name_length_constraint.sql`, `docs/adr/0013-avatar-field-without-upload-pipeline.md`, `docs/adr/0014-no-privacy-settings-ui-yet.md`, plus 5 new test files (`profile_update_test.dart`, `profile_validators_test.dart`, `profile_controller_test.dart`, `edit_profile_screen_test.dart`, and updates to `complete_profile_screen_test.dart`/`app_router_test.dart` for the new `ProfileUpdate` signature).

**Files modified:** `lib/features/profile/domain/profile_repository.dart`, `lib/features/profile/data/supabase_profile_repository.dart`, `lib/features/profile/application/profile_controller.dart`, `lib/features/profile/presentation/complete_profile_screen.dart`, `lib/features/profile/presentation/profile_screen.dart`, `lib/core/constants/app_constants.dart`.

**Live regression (executed against `veya-dev`):** 8-point regression battery re-run after migration 0007 — own read/update allowed, cross-user read/update denied, protected fields (`created_at`/`account_status`) still denied, anonymous access still denied, duplicate username still denied, privacy-settings ownership intact, new display_name constraint holds. All pass. Full 0.1-C/D database boundary confirmed unchanged and intact.

Full narrative: `docs/SECURITY_DECISIONS.md`, `docs/DATABASE_SCHEMA.md`, ADR-0013, ADR-0014.

## VEYA 0.1-D follow-up: encrypted session storage (closes known limitation #6)

The session-storage encryption gap flagged during 0.1-D (SharedPreferences by default, not encrypted at rest) has been closed. `lib/core/config/secure_session_storage.dart` — a custom `LocalStorage` backed by `flutter_secure_storage` — is now wired into `Supabase.initialize()` in `main.dart`. New dependency: `flutter_secure_storage: ^9.2.2`. New test: `test/unit/secure_session_storage_test.dart` (3 cases, mocking the platform channel directly — WRITTEN, not executed, no SDK). Full reasoning, including the SDK's own legitimate counter-argument for its plain default, in `docs/adr/0012-encrypted-session-storage.md`.

**Not executed** (unchanged constraint): no Flutter SDK in this container to run `flutter pub get` and confirm the new dependency resolves cleanly, or to run the new test against a real platform channel.

## VEYA 0.1-D — Authentication Foundation

**Objective:** a secure, maintainable auth/session state machine — no database changes, no new features. The audit re-read the actual current auth repository, router, profile relationship, and bootstrap sequence before changing anything.

**Confirmed already correct (no change needed):** identity always comes from the Supabase session, never a caller-supplied id (compile-time guarantee — neither repository interface has an id parameter); no manual profile creation in Flutter duplicating the trigger; `Supabase.initialize()` is awaited before `runApp()`, so there's no navigate-before-init race; exactly one listener on the auth stream for the app's lifetime.

**Real gap 1 — fixed:** 0.1-C's auth state was split across two providers and treated a session-stream error as "no redirect decision," which could leave a user on an indefinite splash screen if the stream ever errored. Replaced with one unified, fail-closed `AuthState` (six explicit variants: Initializing/Unauthenticated/Authenticating/Authenticated/SigningOut/AuthenticationError) — a stream error now resolves to `AuthUnauthenticated`, never an indefinite hang. See ADR-0011.

**Real gap 2 — found, documented, deliberately not auto-fixed:** `supabase_flutter` v2.x (pinned here) persists the session, including tokens, via `SharedPreferences` by default — sandboxed per-app, but not encrypted at rest. Not overridden with an encrypted `LocalStorage` in this release; flagged as a deliberate future hardening decision (would need a new dependency and its own ADR, not a reflexive addition) rather than silently left unmentioned.

**Files changed:** `lib/features/auth/application/auth_controller.dart` (rewritten — unified state machine), `lib/core/router/app_router.dart` (rewritten — fail-closed switch on the new state), `lib/features/auth/presentation/sign_in_screen.dart` + `sign_up_screen.dart` (updated to the new provider), `lib/features/profile/presentation/profile_screen.dart` (one-line update to the new provider name). `SupabaseAuthRepository`, `SupabaseProfileRepository`, and every migration/RLS/grant from 0.1-B/C are **unchanged**.

**New tests:** `test/unit/auth_notifier_test.dart` (12 cases covering spec Tests 1–6, 9, 12), `test/widget/app_router_test.dart` (3 cases covering spec Tests 7, 8, plus the startup-splash case), `test/widget/sign_in_screen_test.dart` (updated + 1 new error-state case). All **WRITTEN**, not executed (no Flutter SDK — unchanged limitation).

**Live regression check (executed against `veya-dev`):** confirmed 0.1-D made zero database changes (`list_migrations` still shows exactly the same 6 migrations as after 0.1-C) and the security advisor is still at zero findings. Ran one fresh signup + the 0.1-C ownership-field exploit attempt (still correctly denied with `permission denied`) + a legitimate column update (still succeeds) to directly confirm no regression, then cleaned up the test row.

Full narrative: `docs/SECURITY_DECISIONS.md` ("VEYA 0.1-D" section), `docs/AUTH_FLOW.md` (rewritten), `docs/adr/0011-unified-auth-state-machine.md`.

## VEYA 0.1-C — Security Foundation (executed against `veya-dev`)

**Objective:** audit and harden the security boundary established in 0.1-B — no new features. The audit began by re-reading the actual current files (migrations, RLS policies, grants, repositories) rather than assuming 0.1-B's report was correct, per the 0.1-C spec's explicit instruction.

**Two real security gaps were found and fixed, both confirmed live, neither previously caught by static review:**

1. **Ownership field protection gap.** An authenticated owner could change their own `created_at` and `account_status` directly via UPDATE — the RLS `WITH CHECK` only verified row ownership, not which columns changed. Reproduced live, then closed with column-level `GRANT UPDATE` restrictions (migration `0006`) so only `username`/`display_name`/`avatar_url`/`bio` (profiles) and `is_private_account` (privacy settings) are owner-writable. Re-tested live: the exploit now fails with `permission denied`, and legitimate updates to allowed columns still work correctly, including the `updated_at` trigger still firing.
2. **`anon` held unnecessary table-level grants** (INSERT/UPDATE/DELETE/SELECT) on both tables — RLS already blocked anon completely, but this meant RLS was the *only* layer preventing access. Revoked entirely in the same migration, so anonymous access is now denied at two independent layers.

**Also executed live in this audit:** full `SECURITY DEFINER`/grant audit via `pg_proc`/`routine_privileges` (all correct — see `docs/SECURITY_DECISIONS.md`), a trigger malicious-metadata test (SQL-injection-shaped string + unicode payload — trigger completely unaffected, since it never reads `raw_user_meta_data`), a direct-RPC-call test on `handle_new_user()` (still correctly denied), a username constraint matrix (empty/too-long/uppercase/injection-shaped all rejected, valid accepted), and a full regression pass of all 10 original 0.1-B RLS scenarios (all still pass). Supabase's live security and performance advisors both returned zero findings after the fix.

**Not executed:** true concurrent-connection username-collision testing (tool limitation — one statement stream at a time; documented as statically verified via the underlying unique-index guarantee instead, not faked as executed). Flutter-side execution (`analyze`/`test`/`build`, real SDK auth flow) remains blocked for the same pre-existing environment reason as 0.1-B — this was outside 0.1-C's database-security scope and no new attempt to install a Flutter SDK succeeded (verified directly, not assumed).

Full narrative: `docs/SECURITY_DECISIONS.md` (new "VEYA 0.1-C" section) and `docs/adr/0010-column-level-update-privileges.md`. Full evidence: `supabase/tests/rls_security_tests.sql` (TEST 11-15 added).

## Follow-up pass (after live DB verification): closing the known-limitation gap

Since the last update, the previously documented gap — no "choose your real username" screen — has been **closed**:

- `lib/features/profile/domain/profile.dart`: added `Profile.hasPlaceholderUsername`, matching migration 0003's exact placeholder format (`^user_[0-9a-f]{8}$`).
- `lib/features/profile/presentation/complete_profile_screen.dart`: new screen prompting for a real username, reusing the existing `ProfileRepository.updateMyProfile()` (unchanged) and the same `ValidationFailure` duplicate-username handling already proven live in TEST 10.
- `lib/core/router/app_router.dart`: new `_HomeGate` widget watches `myProfileProvider` for an authenticated user and shows `CompleteProfileScreen` while the placeholder is still in place, `ProfileScreen` once a real username is chosen — applied consistently to both `/` and `/profile`.
- New tests: `test/unit/profile_domain_test.dart` gained 3 cases for `hasPlaceholderUsername`; new `test/widget/complete_profile_screen_test.dart` (3 cases: validation, successful submit, duplicate-username error surfaced correctly). Both **WRITTEN**, not executed (no Flutter SDK — see below).
- Structural checks (layering `grep`, brace/paren/bracket balance) re-run across all 26 `.dart` files — still clean.
- Total project files: 53.

Known limitations list is now empty of Flutter-code gaps. The only remaining item is the environment limitation below.

## The one blocker that more effort inside this container cannot close

I checked directly rather than assuming: `which flutter dart` finds nothing, `apt` shows no cached Flutter/Dart packages, and a direct network probe returns `403` (egress blocked). **This container has no Flutter/Dart SDK and no path to install one.** This is a hard environment limit, not a matter of trying harder — I cannot run `flutter pub get`/`analyze`/`test`/`build`, and I cannot exercise the real Supabase Auth SDK signup/sign-in flow from the app itself, here.

Everything SQL/RLS/database-side is genuinely `EXECUTED/VERIFIED` against the live `veya-dev` project (see the Security section below and the previous update). Everything Flutter-side is now **complete as written code**, statically reviewed, and structurally checked — but not compiled, analyzed, or run.

**To actually finish VEYA 0.1's Build/Quality gate, the remaining path is unchanged from before:** an environment with a real Flutter/Dart SDK — your own machine, or Claude Code — pointed at this same project (`.env.json` already has the live `veya-dev` credentials). I did not want to claim this is done when it isn't; it is the one part of 0.1 that this specific environment genuinely cannot deliver.

## Live environment now connected

- **Supabase project:** `veya-dev` (ref `neqzigfaodwhfhjmubbj`), region `ap-southeast-1` (Singapore), organization `bwknuwhqymlcforcewsd`. Cost: $0/month (free tier).
- **Project URL:** `https://neqzigfaodwhfhjmubbj.supabase.co`
- **Client credential:** anon/publishable key only — see `.env.json` (gitignored, delivered directly to you). No service-role key was retrieved or used anywhere in this project.
- **Still not available in this container:** a Flutter/Dart SDK, so `flutter pub get` / `analyze` / `test` / `build` remain BLOCKED. Everything database/SQL-side is now EXECUTED, not just written.

## Definition of Done

Legend: NOT STARTED · WRITTEN · STATICALLY VERIFIED · EXECUTED/VERIFIED · BLOCKED

### Architecture
| Item | State |
|---|---|
| Folder structure | EXECUTED/VERIFIED |
| Layer boundaries (domain clean; only data/ touches Supabase) | STATICALLY VERIFIED via `grep` |
| ADR set (0000–0009) | WRITTEN |

### Authentication
| Item | State |
|---|---|
| Unified fail-closed AuthState (6 variants) implemented | **WRITTEN, statically reviewed (0.1-D)** |
| Session restoration / no navigate-before-init race | **STATICALLY VERIFIED (0.1-D)** — confirmed by reading `main.dart`'s actual bootstrap order (`await Supabase.initialize()` before `runApp()`) |
| Auth state listener disposal (no duplicate/leaked subscriptions) | **WRITTEN** — `test/unit/auth_notifier_test.dart` TEST 12 asserts this directly against a fake `StreamController`; not executed (no SDK) |
| Sign-out never leaves stale authenticated state | **WRITTEN, statically reviewed (0.1-D)** — `finally` block forces `AuthUnauthenticated` regardless of remote-call outcome |
| Session-stream error handled fail-closed (not an indefinite hang) | **WRITTEN, statically reviewed (0.1-D)** — real gap found and fixed, see `docs/SECURITY_DECISIONS.md` |
| Authentication errors mapped to user-safe messages | **WRITTEN, statically reviewed** (unchanged from 0.1-B/C) |
| Sign up / sign in / sign out (Flutter code, repository layer) | WRITTEN, statically reviewed (unchanged from 0.1-B/C) |
| Running against the live Supabase project via the actual Flutter/Supabase Auth SDK | BLOCKED — no Flutter SDK in this container |

### Routing
| Item | State |
|---|---|
| Protected routes enforced regardless of entry path (direct URL/deep link) | **WRITTEN** — `test/widget/app_router_test.dart` TEST 7 (unauthenticated → redirected off `/profile`), TEST 8 (authenticated → allowed); not executed (no SDK) |
| Redirect loop avoidance | **STATICALLY VERIFIED (0.1-D)** — redirect conditions are mutually exclusive by construction (`switch` over disjoint state variants), reasoned through by hand; not executed |
| Startup shows splash, not a premature screen, while `AuthInitializing` | **WRITTEN** — `test/widget/app_router_test.dart` third case; not executed |

### Database
| Item | State |
|---|---|
| `profiles` migration | **EXECUTED/VERIFIED** — applied to `veya-dev`, confirmed via `list_tables` (RLS on, correct columns/constraints/FKs) |
| `user_privacy_settings` migration | **EXECUTED/VERIFIED** — same |
| Signup trigger (`handle_new_user`) | **EXECUTED/VERIFIED** — fired correctly on 3 separate real test signups (see Security below) |
| Constraints (unique username, format, length, FKs, cascade delete) | **EXECUTED/VERIFIED** — unique-username violation reproduced live (error `23505`); cascade delete confirmed (deleting `auth.users` rows removed dependent `profiles`/`user_privacy_settings` rows automatically) |
| 0.1-D regression: no DB changes made, still exactly 6 migrations | **EXECUTED/VERIFIED (0.1-D)** — `list_migrations` re-checked live |
| 0.1-D regression: ownership-field exploit still denied, legitimate update still works | **EXECUTED/VERIFIED (0.1-D)** — fresh live test |

### Security
| Item | State | Evidence |
|---|---|---|
| RLS enabled on both tables | **EXECUTED/VERIFIED** | `list_tables` confirms `rls_enabled: true` on both |
| Owner can read/update own row | **EXECUTED/VERIFIED** | TEST 1, TEST 7 — 1 row returned each |
| Cross-user read blocked | **EXECUTED/VERIFIED** | TEST 2, TEST 7 — 0 rows returned |
| Anonymous read blocked | **EXECUTED/VERIFIED** | TEST 3 — 0 rows as `anon` role |
| Cross-user update blocked | **EXECUTED/VERIFIED** | TEST 4 — 0 rows changed |
| Client-side INSERT blocked | **EXECUTED/VERIFIED** | TEST 5 — `ERROR 42501: new row violates row-level security policy` |
| Client-side DELETE blocked | **EXECUTED/VERIFIED** | TEST 6 — row survived, 0 rows affected |
| Suspended account cannot update itself | **EXECUTED/VERIFIED** | TEST 8 — `bio` stayed `null` after attempted update |
| Duplicate username rejected | **EXECUTED/VERIFIED** | TEST 10 — `ERROR 23505: duplicate key value violates unique constraint` |
| Trigger creates exactly 1 profile + 1 privacy row, unique well-formed placeholder username | **EXECUTED/VERIFIED** | TEST 9 — confirmed on 3 real signups (2 initial + 1 post-hardening regression check) |
| No service-role key in client | STATICALLY VERIFIED via `grep` |
| **Supabase's own live security advisor** | **EXECUTED — found 1 real issue, now fixed** | `handle_new_user()` was callable directly via PostgREST RPC (`/rest/v1/rpc/handle_new_user`) by `anon`/`authenticated`, not just via the trigger. Fixed in migration `0005_harden_trigger_function_privileges.sql` (revoked EXECUTE from `anon`/`authenticated`/`public` on `handle_new_user`, `is_owner`, `set_updated_at`). Re-ran the advisor after the fix: **0 findings**, both security and performance. |
| Regression check after the 0005 hardening fix | **EXECUTED/VERIFIED** | Ran a fresh test signup post-fix — trigger and RLS both still function correctly |
| **Ownership field protection (created_at, account_status)** | **EXECUTED/VERIFIED — real gap found and fixed (0.1-C)** | Pre-fix: owner successfully changed own `created_at`/`account_status` via UPDATE (real vulnerability, reproduced live). Post-fix (migration 0006, column-level GRANT): `ERROR 42501: permission denied for table profiles`. TEST 11 |
| `anon` table-level grants removed (defense in depth beyond RLS) | **EXECUTED/VERIFIED (0.1-C)** | `information_schema.role_table_grants` showed anon held INSERT/UPDATE/DELETE/SELECT pre-fix; revoked in migration 0006; anon SELECT now returns `permission denied` instead of relying on RLS alone |
| SECURITY DEFINER audit (owner, search_path, EXECUTE grants) for all 3 functions | **EXECUTED/VERIFIED (0.1-C)** | Queried `pg_proc`/`information_schema.routine_privileges` directly — `handle_new_user` is the only definer function, correctly scoped; `is_owner`/`set_updated_at` correctly invoker |
| Direct RPC call to `handle_new_user()` denied | **EXECUTED/VERIFIED (0.1-C)** | TEST 12 — `permission denied for function handle_new_user` |
| Trigger immune to malicious/hostile signup metadata | **EXECUTED/VERIFIED (0.1-C)** | TEST 13 — SQL-injection-shaped string + unicode payload in `raw_user_meta_data` had zero effect (trigger never reads it) |
| Username constraint matrix (empty, too-long, uppercase, injection-shaped, valid) | **EXECUTED/VERIFIED (0.1-C)** | TEST 14 — all 4 invalid cases rejected by CHECK constraints, valid case accepted |
| Concurrent username collision safety | STATICALLY VERIFIED (0.1-C) — not independently executed under true concurrency (tool limitation: one statement stream at a time); protection is a Postgres unique b-tree index, atomic by construction | TEST 15 |
| Full regression of all 10 original 0.1-B RLS scenarios after 0.1-C's grant changes | **EXECUTED/VERIFIED (0.1-C)** | Re-ran TEST 1-10 equivalents post-migration-0006 — all still pass |
| Supabase advisors re-checked after 0.1-C changes | **EXECUTED/VERIFIED (0.1-C)** | Security: 0 findings. Performance: 0 findings |

### Privacy
| Item | State |
|---|---|
| Owner-only default | **EXECUTED/VERIFIED** — proven by the RLS tests above, re-confirmed in 0.1-E's regression pass |
| `is_private_account` stored, default `true`, inert (no policy reads it yet) | **EXECUTED/VERIFIED** — confirmed via TEST 9 output |
| No public profile exposure introduced in 0.1-E | **EXECUTED/VERIFIED (0.1-E)** — no new RLS policy added; cross-user read still denied in the regression pass |
| Deliberate no-privacy-UI decision documented | WRITTEN (0.1-E) — ADR-0014 |

### Profile (0.1-E)
| Item | State |
|---|---|
| Typed `ProfileUpdate` model — protected fields structurally inexpressible | **WRITTEN** — `test/unit/profile_update_test.dart` TEST 3; not executed (no SDK) |
| Database still rejects protected-field modification | **EXECUTED/VERIFIED** — re-confirmed live in 0.1-E's regression pass (REG4) |
| Edit-profile flow (loading/save/validation/error/no duplicate submissions) | **WRITTEN** — `test/widget/edit_profile_screen_test.dart` (3 cases); not executed (no SDK) |
| Display name length constraint (new) | **EXECUTED/VERIFIED** — migration 0007, tested live (51 chars rejected, Unicode 34-codepoint string accepted) |
| Username validation unchanged, shared validator extracted | **WRITTEN** — `test/unit/profile_validators_test.dart` (14 cases); DB-level equivalents already EXECUTED/VERIFIED in 0.1-C's `rls_security_tests.sql` TEST 14 |
| Avatar display (existing field, newly rendered) | **WRITTEN, statically reviewed** — no live/executed UI test possible without SDK |
| Repository ownership contract (no arbitrary id) | **WRITTEN** — `test/unit/profile_controller_test.dart` TEST 5; structurally enforced by the interface (no id parameter exists) |
| New finding: Auth leaked-password-protection disabled | **BLOCKED** — project-level Supabase Auth setting, no MCP tool available to toggle it in this session; needs dashboard access or different tooling |

### Testing
| Item | State |
|---|---|
| Unit tests (Dart) | WRITTEN — not executed (no Flutter SDK). Now also includes `profile_update_test.dart`, `profile_validators_test.dart`, `profile_controller_test.dart` (0.1-E, 21 new cases) alongside 0.1-A/B/D unit tests |
| Widget tests (Dart) | WRITTEN — not executed (no Flutter SDK). Now also includes `edit_profile_screen_test.dart` (0.1-E, 3 cases) and updated `complete_profile_screen_test.dart`/`app_router_test.dart` for the new `ProfileUpdate` signature |
| RLS/security tests | **EXECUTED/VERIFIED** — all 15 scenarios in `supabase/tests/rls_security_tests.sql` (10 from 0.1-B + 5 from 0.1-C) still pass; re-confirmed live during 0.1-D's and 0.1-E's regression checks, plus a new 8-point regression battery specific to 0.1-E's migration 0007 |
| Integration tests (full Flutter auth flow) | Specification WRITTEN — BLOCKED (needs Flutter SDK) |

### Documentation
| Item | State |
|---|---|
| ARCHITECTURE.md, DATABASE_SCHEMA.md, SECURITY_DECISIONS.md, AUTH_FLOW.md, TESTING_STRATEGY.md, RELEASE_STATUS.md, docs/adr/* | EXECUTED/VERIFIED — written, present, and now updated with live results |

### Build / Quality
| Item | State |
|---|---|
| `flutter pub get` / `analyze` / `test` / `build` | BLOCKED — no Flutter/Dart SDK in this container |
| Lightweight Dart structural checks (brace balance), SQL `$$`-delimiter balance, secret-leakage grep, layering grep | EXECUTED — not a substitute for real `analyze`/`test`/`build` |
| Live database migrations, RLS policies, trigger, security advisor | **EXECUTED/VERIFIED against a real Supabase project** |

## Known limitations (explicit, not hidden)

1. ~~No "choose your real username" screen~~ — **closed**, see the follow-up pass section above (`CompleteProfileScreen` + `_HomeGate` routing).
2. Email/password only — no social login, by design.
3. Flutter-side execution (`analyze`/`test`/`build`, and the real Supabase Auth SDK signup/sign-in flow from the app itself) is still BLOCKED — this container has no Flutter/Dart SDK and no path to install one (verified directly, not assumed). Everything SQL/RLS/database-side is genuinely executed and verified; the Flutter app code is written, statically reviewed, and structurally checked only.
4. Test data used to verify the trigger/RLS (emails under the reserved `.invalid` TLD, RFC 2606) was created and then deleted after verification — nothing fake was left resembling real user activity in the project, per the project's rule against fake production data.
5. Concurrent username-collision testing was not independently executed under true parallel connections — this environment's SQL tool runs one statement stream at a time. Documented as statically verified instead (the protection is a Postgres unique b-tree index, atomic by construction regardless of concurrency), not claimed as executed.
6. ~~Session storage encryption~~ — **closed** (0.1-D follow-up). `flutter_secure_storage`-backed `SecureSessionStorage` now persists the session via platform Keychain/Keystore instead of plain `SharedPreferences`. See ADR-0012 and `docs/SECURITY_DECISIONS.md`.

## What is still required to close remaining blockers

- A Flutter/Dart SDK environment (your machine, or Claude Code) to run `flutter pub get`, `flutter analyze`, `flutter test`, `flutter build`, and to actually exercise sign-up/sign-in/sign-out through the real Supabase Auth SDK end-to-end.
- `.env.json` (already generated with real `veya-dev` project values) is what that environment needs: run with `flutter run --dart-define-from-file=.env.json`.
