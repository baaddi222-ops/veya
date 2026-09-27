# ADR-0001: Flutter + Dart client, Supabase/PostgreSQL backend

Date: 2026-09-24
Status: Accepted
Release: VEYA 0.1

## Decision
VEYA's client is Flutter/Dart; the backend is Supabase (Postgres, Supabase Auth, Supabase Storage, Realtime when needed).

## Context / Problem
VEYA needs one codebase that can target mobile first and not architecturally block a future desktop/web build, plus a backend with real relational integrity, strong RLS-based authorization, and managed auth — appropriate for a 3–6 year, security/privacy-first social platform.

## Alternatives Considered
- Native iOS/Android separately: doubles engineering cost for a long-running solo/small-team project; rejected.
- A custom backend (Node/Django + hand-rolled auth): more control, but far more security surface to build and maintain correctly (auth, session handling, RLS-equivalent authorization) for a project explicitly prioritizing security first. Rejected for 0.1; the master spec allows revisiting only with a concrete technical reason.
- Firebase: weaker relational modeling and query flexibility than Postgres for a data model expected to grow into a large social graph; rejected.

## Reason for Decision
Flutter gives one codebase across platforms. Supabase gives real Postgres (relational integrity, RLS as a genuine security boundary) with managed auth, reducing the amount of custom security-critical code VEYA has to build and maintain itself.

## Consequences
Ties VEYA to Postgres semantics for authorization (a good fit for RLS-first security). Some vendor dependency on Supabase specifically for Auth/Storage/Realtime — mitigated by keeping all Supabase calls behind repository interfaces (ADR-0005), so the backend could be swapped with `data/`-layer changes only if ever necessary.
