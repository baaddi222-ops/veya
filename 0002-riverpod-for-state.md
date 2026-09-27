# ADR-0002: Riverpod for state management

Date: 2026-09-24
Status: Accepted
Release: VEYA 0.1

## Decision
Use `flutter_riverpod` project-wide for state management, including auth state, session state, and feature state.

## Context / Problem
Need one consistent, testable state-management approach that will still make sense after years of feature growth (posts, feed, video, messaging, Projects, ...), and that maps cleanly onto async backend calls (auth streams, data fetches).

## Alternatives Considered
- `provider`: older, more boilerplate, relies on `BuildContext` lookups that fail at runtime rather than compile time. No real advantage over Riverpod for a multi-year project.
- `bloc`: more ceremony (events/states/blocs per feature) than 0.1 needs; more appropriate for very large teams with strict enforced conventions. Not justified now; could be reconsidered at 1.0+ if the team grows, but that is a future decision, not a 0.1 one.
- `GetX`: encourages service-locator/global-state patterns that fight the chosen clean-architecture layering and make testing harder. Rejected.

## Reason for Decision
Riverpod is compile-safe (no runtime context-lookup failures), testable without a widget tree (critical for unit-testing controllers), and its `AsyncValue` maps directly onto Supabase's stream- and future-based APIs (auth state changes, data fetches) with built-in loading/error handling.

## Consequences
Every feature's `application/` layer is expected to follow the same Riverpod provider/controller shape, which keeps the codebase consistent as it grows. Widget tests can override providers with fakes (see `test/widget/sign_in_screen_test.dart`) without touching a real backend.
