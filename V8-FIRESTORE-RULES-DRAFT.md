# V8 Firestore Security Rules — Deployment Draft

This is a design draft, not a replacement for the project's deployed rules. The repository branch did not contain the current Firestore Rules, so the existing access model must be retrieved and reviewed before any deployment.

## Required invariants

For every create/update of `public_songs/{songId}`:
- The caller must be authenticated.
- `owner_uid` must equal the caller's UID.
- `dedupe_lock_id` must be present.
- The matching `song_uniqueness/{dedupe_lock_id}` document after the atomic operation must contain the same `song_id`, `owner_uid`, and `dedupe_key`.

For every create/update of `song_uniqueness/{lockId}`:
- The caller must be authenticated.
- `owner_uid` must equal the caller's UID.
- The song document after the atomic operation must point back to this `lockId` and match the reservation's owner and `dedupe_key`.

For deletion:
- A user may delete only a song they own.
- A reservation may be deleted only by its owner and only when the song after the atomic operation no longer points to that reservation.
- A duplicate document must never be allowed to remove a reservation that belongs to the canonical song.

## Draft rule logic (merge only after review)

```javascript
match /public_songs/{songId} {
  allow read: if true;

  allow create, update: if request.auth != null
    && request.resource.data.owner_uid == request.auth.uid
    && request.resource.data.dedupe_lock_id is string
    && getAfter(/databases/$(database)/documents/song_uniqueness/$(request.resource.data.dedupe_lock_id)).data.song_id == songId
    && getAfter(/databases/$(database)/documents/song_uniqueness/$(request.resource.data.dedupe_lock_id)).data.owner_uid == request.auth.uid
    && getAfter(/databases/$(database)/documents/song_uniqueness/$(request.resource.data.dedupe_lock_id)).data.dedupe_key == request.resource.data.dedupe_key;

  allow delete: if request.auth != null
    && resource.data.owner_uid == request.auth.uid
    && (!existsAfter(/databases/$(database)/documents/song_uniqueness/$(resource.data.dedupe_lock_id))
      || getAfter(/databases/$(database)/documents/song_uniqueness/$(resource.data.dedupe_lock_id)).data.song_id != songId);
}

match /song_uniqueness/{lockId} {
  allow read: if request.auth != null;

  allow create, update: if request.auth != null
    && request.resource.data.owner_uid == request.auth.uid
    && getAfter(/databases/$(database)/documents/public_songs/$(request.resource.data.song_id)).data.owner_uid == request.auth.uid
    && getAfter(/databases/$(database)/documents/public_songs/$(request.resource.data.song_id)).data.dedupe_lock_id == lockId
    && getAfter(/databases/$(database)/documents/public_songs/$(request.resource.data.song_id)).data.dedupe_key == request.resource.data.dedupe_key;

  allow delete: if request.auth != null
    && resource.data.owner_uid == request.auth.uid
    && (!existsAfter(/databases/$(database)/documents/public_songs/$(resource.data.song_id))
      || getAfter(/databases/$(database)/documents/public_songs/$(resource.data.song_id)).data.dedupe_lock_id != lockId);
}
```

## Before deployment

1. Merge these constraints with the actual current rules; do not replace unrelated rules for Setlists, private search history, backups, or other collections.
2. Validate the rules in the Firebase Emulator Suite or a disposable Firebase test project.
3. Confirm that a transaction creating/updating a song and reservation passes, while song-only writes, reservation-only writes, cross-user overwrites, and duplicate lock claims fail.
4. Confirm the existing app's public song reads and authenticated Setlist/search operations still work.
5. Deploy only after the full regression suite passes.

No rules from this draft have been deployed.
