# VEYA — Testing Strategy

## Three explicit verification levels (used in every status report, no exceptions)

| Level | Meaning |
|---|---|
| **Written** | Test code/specification exists in the repo. Nothing executed. |
| **Statically verified** | Code/schema/policy read and reasoned about carefully without running it live. |
| **Executed / verified** | Actually ran against a real environment; result (pass or fail) reported honestly. |

A release-gate item is only PASS at **Executed/verified**. Written and statically-verified are real progress, never reported as PASS.

## Test categories

- **Unit** (`test/unit/`): domain logic, `Failure` construction — no Flutter/Supabase dependency.
- **Widget** (`test/widget/`): form validation and interaction on auth screens, using Riverpod provider overrides (fake repository) — no live backend needed once a Flutter SDK is available.
- **Integration** (`test/integration/`): full auth flow against a real Supabase test project — requires both Flutter SDK and network/Supabase access.
- **Security / RLS** (`supabase/tests/rls_security_tests.sql`): adversarial cross-user/anonymous access attempts run directly against Postgres — requires a live Supabase/Postgres project, independent of the Flutter app.

## Current status (VEYA 0.1, this environment)

This container has no Flutter/Dart SDK, no network access, and no connected Supabase project. Every test above is currently **WRITTEN**, not executed. See the final VEYA 0.1 status report for the file-by-file breakdown and exactly what needs to happen (and where) to move each to EXECUTED.
