# V8 Cloud Duplicate Safety Review

Date: 2026-11-23
Branch: `v8-development`

## Scope reviewed

Reviewed the current `index.html` implementation on `v8-development`, especially `saveCloudSong()` and the `public_songs` Firestore snapshot handler.

No Firebase database was accessed or modified during this review. No changes were made to `main` / the live V7 deployment.

## Confirmed behavior in the current branch

- Song identity is calculated from normalized title + artist.
- The Cloud save path still scans `public_songs` to detect legacy records, then uses a SHA-256 identity reservation and Firestore transaction to atomically reserve a normalized title + artist for a song ID.
- The snapshot handler hides duplicate records from the displayed library.
- Automatic Cloud deletion is restricted to duplicate records whose `owner_uid` matches the signed-in user and the selected canonical record's owner.
- Duplicates owned by different accounts are not deleted by this cleanup path.

## Risks that must be resolved before production

1. **Read cost and scale:** fetching the entire public collection on every song save remains increasingly expensive and slow as the library grows. A legacy backfill is needed before removing this scan.
2. **Security Rules:** the transaction prevents collisions among clients using this code, but app code alone is not a security boundary. Firestore Rules must require the matching reservation and song write in the same atomic operation; older clients and direct writes must be blocked.
3. **Ownership / image preservation:** cleanup must never delete a record until the canonical record, owner, chord image and Setlist references have been checked. The current same-owner guard is useful, but it is not a complete recovery or audit mechanism.
4. **Rules are not in this repository path:** no `firestore.rules`, `firebase.json`, or `firestore.indexes.json` was found at the repository root on this branch. Firestore authorization therefore cannot be verified from this checkout.

## Safe implementation sequence

1. Review and deploy Firestore Security Rules that require the matching `dedupe_lock_id`, reservation ownership, and song ownership in the same atomic write.
2. Add a legacy migration / audit mode that reports duplicate groups without deleting any documents.
3. Produce a dry-run report showing canonical candidate, duplicate document IDs, owners, image presence and Setlist references.
4. Backfill reservation records for verified canonical songs, then remove the full-collection legacy scan only after migration verification.
5. Only after backup and explicit review, perform controlled cleanup with a recovery path and audit log.
6. Run the test matrix in `V8-FIRESTORE-TRANSACTION-TEST-PLAN.md` against a Firebase test project and the deployed Security Rules.

## Release gate

Do not run destructive duplicate cleanup and do not publish V8 until the uniqueness mechanism, Security Rules, legacy audit, image preservation, and recovery tests have passed against a Firebase test project.
