# ADR-0007: Owner-only RLS default in VEYA 0.1 (no public profile reads)

Date: 2026-09-24
Status: Accepted
Release: VEYA 0.1
(Supersedes the original 0.1 draft plan's "public by default, private opt-in" proposal.)

## Decision
`profiles` and `user_privacy_settings` are readable and writable only by their owner in 0.1. No anonymous or cross-user SELECT policy exists. `is_private_account` is stored but has no RLS effect yet.

## Context / Problem
0.1 has no follows, blocking, reporting, or moderation, and no age-category system. The original draft plan proposed public-by-default profiles with an opt-in private flag — a reasonable default for a mature social app, but not for a foundation release with zero protective systems built yet.

## Alternatives Considered
- Public-by-default with `is_private_account` opt-out (the original draft): the friendlier UX default, but exposes every user's username/display name/avatar/bio to any caller, authenticated or not, with no discovery feature in 0.1 that actually needs it, and no way yet to know which accounts might belong to a minor (age categories arrive in 0.7).
- Public-by-default with `is_private_account` opt-in, deferred moderation: rejected for the same exposure reasoning — deferring moderation doesn't defer the exposure itself.

## Reason for Decision
No 0.1 feature needs public reads (no search, no explore, no follow suggestions). Starting maximally restrictive and adding a scoped public-read policy in 0.2 — once follows/blocking/reporting exist together — is a strictly additive migration path. Starting public and tightening later is where real-world RLS gaps tend to get missed once real user data exists. This ordering also directly follows the master spec's own priority list: security and privacy rank above features.

## Consequences
0.2 inherits the responsibility of designing the first public-read policy correctly, ideally via a restricted view/column set rather than widening the base `profiles` table (see architecture review §2) and ideally co-designed with blocking/reporting rather than shipped ahead of them. No 0.1 feature is lost by this change — nothing in 0.1 currently reads another user's profile.
