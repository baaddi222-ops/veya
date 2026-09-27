# ADR-0005: Repository interfaces (domain defines, data implements)

Date: 2026-09-24
Status: Accepted
Release: VEYA 0.1

## Decision
Every backend interaction is accessed through a domain-defined abstract interface (`AuthRepository`, `ProfileRepository`), implemented in `data/` against Supabase. `application/` and `presentation/` never import `supabase_flutter` directly.

## Context / Problem
The master spec ties VEYA to Supabase/Postgres now, but a 3–6 year project should not be structurally unable to adapt if a concrete technical reason emerges later (e.g. adding a server-side layer in front of Supabase, or swapping a specific backend piece).

## Alternatives Considered
- Calling `supabase_flutter` directly from Riverpod controllers or widgets: less code short-term, but spreads backend-specific exception types and query syntax throughout the app, making both testing (no clean seam for fakes) and any future backend change far more invasive.

## Reason for Decision
A single, well-defined seam (`data/`) is where all Supabase-specific code and exception handling lives. Everything above it only depends on plain Dart interfaces and domain types, which is both easier to unit-test (fake implementations, see `test/widget/sign_in_screen_test.dart`) and keeps ADR-0001's stated backend-swap mitigation real rather than aspirational.

## Consequences
Every new feature repeats this pattern: domain interface first, then a Supabase-backed implementation in `data/`. Slightly more files per feature than calling Supabase directly, traded for testability and a genuine architectural seam.
