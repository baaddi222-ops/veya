# VEYA — Database Schema (0.1)

Two tables. Every table has a clear, singular purpose — no speculative tables.

## `profiles`

Identity-only data for an authenticated user.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid, PK | FK → `auth.users.id`, `ON DELETE CASCADE` |
| `username` | text, unique, not null | 3–24 chars, `^[a-z0-9_]+$`, placeholder-generated at signup (see trigger) |
| `display_name` | text, nullable | ≤ 50 chars (migration 0007, 0.1-E — mirrors the `bio` pattern; `char_length()` is Unicode-codepoint-aware, confirmed live) |
| `avatar_url` | text, nullable | |
| `bio` | text, nullable | ≤ 280 chars |
| `account_status` | text, not null, default `active` | one of `active` / `suspended` / `deleted` |
| `created_at` | timestamptz, default `now()` | |
| `updated_at` | timestamptz, trigger-maintained | |

**Not included on purpose:** age, country, guardian links, verification status, consent records. These are future policy/compliance concerns kept structurally separate from identity data — see ADR-0006 and architecture review §4.

## `user_privacy_settings`

| Column | Type | Notes |
|---|---|---|
| `user_id` | uuid, PK | FK → `profiles.id`, `ON DELETE CASCADE` |
| `is_private_account` | boolean, default `true` | **Stored but inert in 0.1** — no RLS policy reads it yet; all data is owner-only regardless. Becomes meaningful in 0.2. |
| `created_at` / `updated_at` | timestamptz | |

Kept separate from `profiles` so privacy/policy data (expected to grow substantially — guardian rules, country policy, consent) doesn't repeatedly widen the identity table, and so its RLS can be reasoned about independently. See ADR-0006.

## Relationships

```
auth.users (Supabase-managed)
   │ 1:1, cascade delete
   ▼
profiles
   │ 1:1, cascade delete
   ▼
user_privacy_settings
```

Both cascades are correct *for these two tables specifically*, because nothing else references them yet. This is explicitly not a pattern to copy onto future user-generated-content tables (posts, comments, messages) once other users' data can reference them — see architecture review §6.

## Privilege model (as of 0.1-C)

RLS is not the only access layer — Postgres checks table/column-level `GRANT`s *before* RLS is even evaluated. As of migration `0006` (0.1-C audit):

- **`anon`** holds no table-level grant on either table at all — not SELECT, not INSERT, not UPDATE, not DELETE. Combined with RLS having zero `anon` policies, this is two independent layers of denial for anonymous access.
- **`authenticated`** holds `SELECT` on both tables (required for the SELECT RLS policies to be evaluated at all) and `UPDATE` restricted to specific columns only:
  - `profiles`: `username, display_name, avatar_url, bio` — **not** `id`, `created_at`, `updated_at`, `account_status`.
  - `user_privacy_settings`: `is_private_account` — **not** `user_id`, `created_at`, `updated_at`.
  - `authenticated` holds no `INSERT`, `DELETE`, `TRUNCATE`, `REFERENCES`, or `TRIGGER` grant on either table — row creation is exclusively the signup trigger's job (which runs `SECURITY DEFINER`, so it doesn't depend on `authenticated`'s own grants).

This closes a real gap found during the 0.1-C audit: RLS's `WITH CHECK` on the UPDATE policy only verifies row ownership, not which columns changed, so without this column-level restriction an owner could self-modify `created_at` or `account_status` directly. See ADR-0010 and `docs/SECURITY_DECISIONS.md`.

## Row creation

Both rows are created together, atomically, by the `handle_new_user()` trigger on `auth.users` insert (migration 0003) — never by a client-side INSERT. See `docs/adr/0009-profile-creation-trigger-design.md`.

## Indexes

- Unique index on `profiles.username` (required for the uniqueness constraint; also the only lookup pattern 0.1 needs). No other indexes added — premature indexing on tables with no query load yet is noise, not scalability, per the architecture review.

## Migration history

`0001` profiles, `0002` user_privacy_settings, `0003` signup trigger, `0004` RLS policies, `0005` trigger-function privilege hardening (0.1-C), `0006` table/column-level privilege hardening (0.1-C), `0007` `display_name` length constraint (0.1-E — closed a real inconsistency where `bio` had a length CHECK and `display_name` did not). All applied live to `veya-dev` and confirmed via `list_migrations`.
