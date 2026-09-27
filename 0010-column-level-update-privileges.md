# ADR-0010: Column-level UPDATE grants instead of table-wide UPDATE

Date: 2026-09-25
Status: Accepted
Release: VEYA 0.1-C

## Decision
`authenticated` no longer holds a blanket table-level `UPDATE` grant on `profiles` or `user_privacy_settings`. Instead it holds `UPDATE` only on the specific columns an owner may legitimately change: `username, display_name, avatar_url, bio` on `profiles`, and `is_private_account` on `user_privacy_settings`. `id`, `created_at`, `updated_at`, and `account_status` are excluded.

## Context / Problem
A live 0.1-C security audit found that the existing UPDATE RLS policy (`with check (public.is_owner(id))`) only verifies *row* ownership, not *which columns* changed. This was tested directly: an authenticated owner could successfully set their own `created_at` to an arbitrary date and their own `account_status` to `suspended` via a normal UPDATE statement — RLS has no native concept of "this column is off-limits," only "this row is or isn't visible/writable."

## Alternatives Considered
- A `BEFORE UPDATE` trigger that resets `NEW.created_at := OLD.created_at` and `NEW.account_status := OLD.account_status` unconditionally: works, but is a second, redundant protection layer implemented in application logic (plpgsql) for something Postgres's own grant system already does natively and more legibly.
- Continuing to rely on the Flutter repository only exposing safe fields (`ProfileRepository.updateMyProfile` already only accepts `username`/`displayName`/`avatarUrl`/`bio`): this was already true and is good practice, but it is a client-side convention, not a security boundary — the master spec is explicit that a malicious or bypassed client must not be able to do more than the UI allows, and this table-level gap meant a direct REST/RPC call with the anon key and a valid session JWT could bypass it entirely.

## Reason for Decision
Column-level `GRANT` is checked by Postgres *before* RLS is even evaluated, and is authoritative regardless of how the request reaches the database (Flutter app, direct REST call, another future client). It requires no additional application logic to maintain and cannot be silently bypassed by a future RLS policy change that forgets about column scope, since it's an independent layer.

## Consequences
Any future field added to `profiles` or `user_privacy_settings` that should be user-editable must be explicitly added to the column-level `GRANT UPDATE (...)` list, or it will be silently non-editable by owners (a safe failure direction — noticed immediately as "can't save," not a silent security gap). `account_status` transitions (suspend, restore, soft-delete) now structurally require an elevated/service-role path — which is the correct direction anyway, since 0.1's architecture already intended account status changes to eventually go through moderation/admin systems (future releases), not self-service.
