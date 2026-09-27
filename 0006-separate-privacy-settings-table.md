# ADR-0006: Separate `user_privacy_settings` table from `profiles`

Date: 2026-09-24
Status: Accepted
Release: VEYA 0.1

## Decision
Privacy/policy-adjacent settings (starting with `is_private_account`) live in their own `user_privacy_settings` table, not as columns on `profiles`.

## Context / Problem
The master spec's roadmap anticipates substantial future growth in privacy/policy data: guardian rules, country-specific policy, consent records, age-category effects on visibility. If that data lives as columns on `profiles`, that table grows wide and its RLS policies become entangled with identity-data RLS.

## Alternatives Considered
- Columns directly on `profiles`: simpler for 0.1 alone, but repeats the same widening problem every time a new privacy/policy concern is added (age category in 0.7, country policy in 0.8, ...), and mixes identity RLS reasoning with privacy RLS reasoning in the same policies.

## Reason for Decision
Splitting identity (`profiles`) from privacy/policy (`user_privacy_settings`, and future tables like a `user_policy_context`) keeps each table's purpose singular and its RLS independently reasoned-about — directly serving the master spec's "every table should have a clear purpose" rule and its explicit warning against future-bloat on core tables.

## Consequences
One extra join/query when both identity and privacy data are needed together (negligible at 0.1's scale — both are PK-joined 1:1). Future privacy/policy tables (age category, country, consent) can be added without ever widening `profiles` again.
