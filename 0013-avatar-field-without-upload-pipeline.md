# ADR-0013: Avatar field/display without an upload pipeline

Date: 2026-09-26
Status: Accepted
Release: VEYA 0.1-E

## Decision
`profiles.avatar_url` and the repository's ability to set it already existed (since 0.1-B) and are unchanged. In 0.1-E, `ProfileScreen` now *displays* whatever `avatar_url` holds (a `CircleAvatar` with a network image, or an initial-letter placeholder if null). No upload UI, image picker, image processing, or storage integration is added — `EditProfileScreen` deliberately has no avatar field.

## Context / Problem
0.1-E's spec explicitly asks to "prepare the profile architecture for avatar support" without building "a complete media-upload system," and warns specifically against trusting "a client-provided arbitrary storage path to another user's private data." The audit found `avatar_url` already exists at the schema/repository level (nothing to build there) but was never rendered anywhere in the UI.

## Alternatives Considered
- Add a text field letting the user paste an arbitrary image URL into `avatarUrl`: technically already possible via the existing repository method with zero new code, but this would ship a confusing, half-real "avatar feature" (no upload, no validation that the URL is even an image, no ownership/storage-path verification) that looks like a finished feature to a user but isn't one — worse for trust than not offering it at all. Rejected.
- Build a real upload pipeline now (image picker, Supabase Storage bucket, path scoping per user): explicitly out of 0.1-E's scope per the spec ("do not add image processing pipelines... those belong to later releases").

## Reason for Decision
Displaying an already-existing, already-safe field costs nothing and is a genuine (if small) improvement — the field existed but was invisible. Not adding upload avoids shipping a feature that looks complete but silently invites users to paste content they don't control the security of. The real abstraction future avatar upload needs (a storage bucket with per-user path scoping, e.g. `avatars/{user_id}/...` with matching Storage RLS policies) is a 0.2+-scale decision requiring its own migration and security review — nothing in 0.1-E's schema or repository interface blocks that.

## Consequences
When avatar upload is eventually built, `ProfileUpdate.avatarUrl` and `SupabaseProfileRepository`'s handling of it need no interface change — only a new upload step (picker + Storage call) that produces the URL passed into the existing `ProfileUpdate`. `ProfileScreen`'s display logic also needs no change at that point.
