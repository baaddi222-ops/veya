# ADR-0003: go_router for routing/navigation

Date: 2026-09-24
Status: Accepted
Release: VEYA 0.1

## Decision
Use `go_router` for all navigation, with auth-gated redirect logic centralized in one router configuration.

## Context / Problem
Auth-gating (unauthenticated → sign-in, authenticated → away from auth screens, expired session → sign-in) needs one reliable code path, not duplicated `if` checks per screen, which becomes unmaintainable as the number of screens grows across 0.2–1.0+.

## Alternatives Considered
- Imperative `Navigator` push/pop with manual auth checks in each screen's `initState`: error-prone, easy to miss a screen, and doesn't scale past a handful of routes.
- `auto_route`: code-generation based; adds a build_runner dependency for marginal benefit over `go_router`'s declarative API at 0.1's scale.

## Reason for Decision
`go_router`'s `redirect` + `refreshListenable` combination lets auth-state changes (including session-expiry) drive navigation from a single place, tied directly to the Riverpod auth stream. It is also URL-based, which keeps a future web build from being architecturally blocked, per the master spec's requirement not to unnecessarily prevent future desktop/web.

## Consequences
All new routes across future releases (0.2+) are added to the same `GoRouter` configuration in `core/router/app_router.dart`, keeping auth-gating consistent by construction rather than by convention alone.
