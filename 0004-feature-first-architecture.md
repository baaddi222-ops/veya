# ADR-0004: Feature-first top-level organization

Date: 2026-09-24
Status: Accepted
Release: VEYA 0.1

## Decision
Top-level `lib/` folders are organized by feature (`features/auth/`, `features/profile/`, and future `features/posts/`, `features/feed/`, ...), each with its own `domain/data/application/presentation` layers inside.

## Context / Problem
VEYA's roadmap spans a very large number of eventual features (posts, feed, video, messaging, Projects, guardian system, ...). The top-level organization needs to still make sense at that scale, not just at 0.1's two features.

## Alternatives Considered
- Layer-first at the top level (`lib/widgets/`, `lib/repositories/`, `lib/controllers/` each containing every feature's files): works for small apps, but past a few features requires hunting across several unrelated folders to change one thing — doesn't scale toward VEYA's roadmap size.

## Reason for Decision
Feature-first folders map directly onto the release-based roadmap itself (each 0.x release largely adds one or more `features/*` folders), keeping the connection between "what shipped in which release" and "where the code lives" obvious years later.

## Consequences
Adding a new feature in 0.2+ means adding a new `features/<name>/` folder with the same four-layer internal shape — no top-level restructure anticipated through at least 1.0. Cross-feature shared code lives in `shared/` (dumb, feature-agnostic widgets only) or `core/` (infrastructure), never duplicated per feature.
