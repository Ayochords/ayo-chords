# V8 Firestore Transaction Test Plan

Date: 2026-11-23
Branch: `v8-development`

## Code change

`saveCloudSong()` now:
- Performs a legacy duplicate scan against `public_songs` before attempting a write.
- Computes a SHA-256 reservation ID from the normalized title + artist key.
- Uses a Firestore transaction to read the reservation and target song, then atomically write the reservation and song document.
- Rejects a reservation owned by a different song ID.
- Releases the previous reservation on an owned song rename only when that reservation still points to the same song and user.
- Stores `dedupe_lock_id` on the song document to support a future Security Rules check.
- Keeps the existing local image cache behavior after a successful Cloud write.

## Important limitation

The app code alone is not a security boundary. Firestore Security Rules must enforce that a song write and its matching `song_uniqueness` reservation are valid together. The current repository branch did not contain the deployed Firestore Rules or Firebase deployment configuration, so those rules have not been changed or verified. Older clients or direct Firestore writes could bypass the app-level reservation until rules are deployed.

The legacy scan is intentionally retained because older song documents may not have reservation records. This is a compatibility safety measure, but it still reads the whole collection and should be removed only after a verified migration/backfill.

## Required test matrix (Firebase test project only)

1. Two devices add the same normalized title + artist at the same time: exactly one succeeds.
2. Same title with a different artist: both succeed.
3. Different title with the same artist: both succeed.
4. Case, repeated spaces, non-breaking spaces, zero-width characters, and Unicode compatibility forms normalize consistently.
5. An existing legacy song without `dedupe_key` or a reservation blocks a new duplicate.
6. Editing an owned song without changing its title/artist preserves its reservation.
7. Renaming an owned song creates the new reservation and releases only its own old reservation.
8. A user cannot overwrite a song or reservation owned by another user.
9. A failed transaction leaves neither a partial song write nor a partial reservation.
10. Images remain intact after add/edit, and Setlist song IDs continue to resolve.
11. Network interruption and permission-denied errors show failure without reporting a successful upload.
12. Run the same tests against the deployed Security Rules, not only the local app code.

## Release gate

Do not publish V8, migrate legacy records destructively, or delete duplicate documents until the Security Rules and all tests above pass. Run the tests with a disposable Firebase test project and a backup of any data used for migration.
