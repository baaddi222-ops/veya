# VEYA — Architecture (as of 0.1)

## Layers

```
presentation  →  application  →  domain  →  data
   (widgets)     (Riverpod        (entities,   (repository
                  controllers)     interfaces)  implementations,
                                                 Supabase calls)
```

- `domain/` is pure Dart. No `package:flutter/*`, no `package:supabase_flutter/*` imports. This is enforced by convention and code review in 0.1 (no automated import-linter is configured yet — a candidate for a future release if this becomes hard to keep honest by review alone).
- `data/` implements domain-defined repository interfaces. Only `data/` files import `supabase_flutter`.
- `application/` holds Riverpod providers/controllers that orchestrate domain + data for the UI.
- `presentation/` is Flutter widgets only.

## Folder layout

```
lib/
├── main.dart / app.dart
├── core/            # cross-cutting: config, router, theme, error, logging, constants
├── features/
│   ├── auth/        # domain / data / application / presentation
│   └── profile/     # domain / data / application / presentation
└── shared/          # dumb, reusable, feature-agnostic widgets
```

New features in 0.2+ (`posts/`, `feed/`, ...) are added as new folders under `features/` with the same four-layer internal shape — no top-level restructure anticipated.

## Why this shape (short form — full reasoning in docs/adr/)

- **Feature-first, not layer-first at the top level**: scales as VEYA grows toward a large roadmap without hunting across unrelated folders for one feature's code. (ADR-0004)
- **Riverpod**: compile-safe, testable without a widget tree, clean async support for auth streams. (ADR-0002)
- **Unified auth state (0.1-D)**: authentication is one flat, six-variant `AuthState` owned by a single `AuthNotifier`, not split across multiple providers — every consumer (router, screens) switches on exactly one thing, and every non-authenticated variant is treated identically (fail closed) for routing purposes. (ADR-0011)
- **go_router**: declarative, centralizes auth-gated redirects in one place instead of scattering `if` checks per screen. (ADR-0003)
- **Repository interfaces**: domain/application/presentation never depend on `supabase_flutter` directly — backend can evolve without rewriting the app. (ADR-0005)
- **Typed update models (0.1-E)**: mutable-field updates go through a typed value object (`ProfileUpdate`), not loose parameters or raw maps — the set of editable fields is closed and visible in one place, a second, independent layer on top of the actual security boundary (0.1-C's column-level database grants).
- **Shared validators (0.1-E)**: field validation (`ProfileValidators`) lives in one place per field, reused by every screen that edits that field, mirroring the database's own CHECK constraints rather than risking two screens silently drifting into different rules.

## What 0.1 does NOT contain

Posts, likes, comments, follows, feed, search, explore, recommendations, video, stories, messaging, notifications, collaborative posts, Projects, sources/context, verification, payments, creator tools, analytics, moderation, guardian/age system, country policy engine, AI administration. See the VEYA roadmap for when each is planned.
