# V8 Cloud Duplicate Safety Review

Date: 2026-11-23
Branch: `v8-development`

## Scope reviewed

Reviewed the current `index.html` implementation on `v8-development`, especially `saveCloudSong()` and the `public_songs` Firestore snapshot handler.

No Firebase database was accessed or modified during this review. No changes were made to `main` / the live V7 deployment.

## Confirmed behavior in the current branch

- Song identity is calculated from normalized title + artist.
- The Cloud save path currently reads the entire `public_songs` collection and scans it client-side to detect duplicates, including legacy records without `dedupe_key`.
- The snapshot handler hides duplicate records from the displayed library.
- Automatic Cloud deletion is restricted to duplicate records whose `owner_uid` matches the signed-in user and the selected canonical record's owner.
- Duplicates owned by different accounts are not deleted by this cleanup path.

## Risks that must be resolved before production

1. **Read cost and scale:** fetching the entire public collection on every song save becomes increasingly expensive and slow as the library grows.
2. **Concurrent writes:** two clients can both scan before either writes, then both save the same title/artist. A client-side scan alone is not an atomic uniqueness guarantee.
3. **Ownership / image preservation:** cleanup must never delete a record until the canonical record, owner, chord image and Setlist references have been checked. The current same-owner guard is useful, but it is not a complete recovery or audit mechanism.
4. **Rules are not in this repository path:** no `firestore.rules`, `firebase.json`, or `firestore.indexes.json` was found at the repository root on this branch. Firestore authorization therefore cannot be verified from this checkout.

## Safe implementation sequence

1. Add a server-enforced uniqueness reservation keyed by a stable hash of normalized title + artist. Use a Firestore transaction so reservation and song creation succeed or fail together.
2. Review and update Firestore Security Rules so a client cannot bypass or overwrite another user's reservation, and so ownership checks are enforced server-side.
3. Add a legacy migration / audit mode that reports duplicate groups without deleting any documents.
4. Produce a dry-run report showing canonical candidate, duplicate document IDs, owners, image presence and Setlist references.
5. Only after backup and explicit review, perform controlled cleanup with a recovery path and audit log.
6. Add tests for concurrent attempts, legacy records, Unicode/spacing normalization, different owners, missing images, and Setlist references.

## Release gate

Do not run destructive duplicate cleanup and do not publish V8 until the uniqueness mechanism, Security Rules, legacy audit, image preservation, and recovery tests have passed against a Firebase test project.
