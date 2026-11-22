# AYO CHORDS V8 — Membership & Level System

## Scope and release safety
- Implement only on `v8-development`.
- Do not modify `main` (the live V7 deployment) or change the GitHub Pages URL.
- Keep existing song, chord-image, Setlist, backup, and restore data compatible.

## Member profile
Each authenticated Firebase user should have a profile stored under `users/{uid}/profile/membership` (or an equivalent single profile document), keyed by the immutable Firebase Auth UID, never by IP address or device.
Suggested fields:
- `uid`, `displayName`, `photoURL`, `createdAt`
- `xpTotal`, `levelId`, `updatedAt`
- optional public profile fields only; email and private account data must not be exposed in public member listings.

## Proposed levels (initial defaults; configurable later)
| Level | XP threshold |
|---|---:|
| MEMBER | 0 |
| MUSICIAN | 100 |
| PRO | 500 |
| VIP | 1,500 |

The displayed progress bar should show current XP, next level threshold, and remaining XP. Level is derived from the authoritative XP total, not manually selected by the client.

## XP activity policy (initial defaults)
| Activity | XP | Eligibility |
|---|---:|---|
| Add a song | +20 | Only after a new song is successfully saved to Cloud; no XP for edits, failed uploads, imports, or duplicates |
| Create a Setlist | +10 | Only after a new Setlist is successfully saved; edits do not award XP |
| Daily active use | +5 | At most once per UTC calendar day, after authenticated use |

## Abuse prevention and data integrity
- Award XP idempotently with a unique event key (UID + activity + target ID/day) so retries and repeated taps cannot award twice.
- Do not trust XP amounts, level names, owner roles, or activity claims supplied by the browser.
- Enforce award eligibility and privileged role checks in trusted server-side code or equivalent protected Firestore rules. Client-side checks alone are not authorization.
- Do not use IP address as a member identity or authorization credential.
- Keep a membership activity ledger so support/admin can audit XP awards and corrections.
- Profile reads/writes must be scoped to the signed-in UID; privileged administration must use an explicit trusted role claim, not a client-editable profile field.
- Do not backfill XP from existing songs or Setlists until a migration policy is reviewed; this avoids accidental duplicate or unfair awards.

## UI requirements
- Add a member profile entry point and profile screen showing avatar, display name, level badge, XP total, progress to next level, and activity history.
- Show the member's level consistently without changing the existing song library or Setlist workflows.
- Provide clear loading, empty, offline, and sync-error states.
- A user must not be able to edit their own XP, level, or privileged role from the profile UI.

## Test and release gates
1. Test new registration and existing account sign-in.
2. Test XP for successful song creation, Setlist creation, and once-daily activity.
3. Verify duplicate taps, retries, edits, failed writes, imports, and offline actions award no duplicate XP.
4. Verify users cannot read or alter another user's private profile or XP ledger.
5. Verify level progression and progress-bar boundaries.
6. Test iPhone Safari and Android Chrome.
7. Keep V8 on the development branch until all gates pass; only then prepare a reviewed release to the existing URL.
