# ADR-0014: No privacy-settings UI in 0.1-E

Date: 2026-09-26
Status: Accepted
Release: VEYA 0.1-E

## Decision
0.1-E adds no UI for `user_privacy_settings` (no "private account" toggle or privacy screen), even though the table and column (`is_private_account`) already exist (since 0.1-B).

## Context / Problem
0.1-E's spec explicitly frames this as conditional: "If the existing UI supports privacy settings, make sure [ownership boundaries hold]... Do not build the full future privacy center yet." No privacy UI exists today, so there was a real choice: add a minimal toggle now, or continue deferring.

## Alternatives Considered
- Add a simple `Switch` for `is_private_account` on the profile/edit screen: cheap to build, and the repository/RLS layer already safely supports reading and writing it (owner-only, per 0.1-C). Rejected anyway — see below.

## Reason for Decision
`is_private_account` is, by explicit 0.1-B/0.1-C design, **stored but currently inert**: no RLS policy anywhere reads it yet, because there is no public-read policy for it to gate (0.2 is expected to introduce both together). Shipping a toggle that visibly does nothing — flipping it to "private" would have zero actual effect on who can see the (already owner-only-visible) profile — would be actively misleading: a user could reasonably believe they'd restricted their visibility when nothing changed, since 0.1 has no public visibility to restrict in the first place. Building UI for a setting with no present effect is worse than not building it, not merely unnecessary.

## Consequences
The privacy-settings UI is deferred to whichever release actually introduces the first public-read policy (0.2, per the roadmap), so the toggle can be built and explained alongside the visibility model it actually controls, rather than shipped inert and re-explained later. `user_privacy_settings`' ownership boundary (owner-only read/write) was re-verified live during 0.1-E's regression pass regardless, since the table itself is in scope for security regression checking even without a UI.
