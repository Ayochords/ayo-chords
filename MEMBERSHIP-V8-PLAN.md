# AYO CHORDS V8 — Membership, XP & Library Permissions

## Scope and release safety
- Implement only on `v8-development`.
- Do not modify `main` (the live V7 deployment) or change the GitHub Pages URL.
- Keep existing song, chord-image, Setlist, backup, and restore data compatible.

## Member identity and profile
Each authenticated Firebase user has a profile keyed by immutable Firebase Auth UID, never by IP address or device.
Suggested fields:
- `uid`, `displayName`, `photoURL`, `createdAt`
- `xpTotal`, derived `levelId`, `updatedAt`
- Keep email and private account data out of public member listings.

## Five membership levels (initial proposed thresholds)
| Level | XP threshold | Library delete permission |
|---|---:|---|
| MEMBER | 0 | Delete own songs only |
| MUSICIAN | 100 | Delete own songs only |
| PRO | 500 | Elevated delete permission, subject to confirmed scope below |
| PREMIUM | 1,000 | Elevated delete permission, subject to confirmed scope below |
| VIP | 1,500 | Elevated delete permission, subject to confirmed scope below |

Levels are derived from authoritative XP. The profile shows current XP, next threshold, and remaining XP. Thresholds are initial defaults and can be adjusted before release.

## XP activity policy (initial defaults)
| Activity | XP | Eligibility |
|---|---:|---|
| Add a song | +20 | Only after a genuinely new song is successfully saved to Cloud; no XP for edits, failed uploads, imports, or duplicates |
| Create a Setlist | +10 | Only after a genuinely new Setlist is successfully saved; edits do not award XP |
| Daily active use | +5 | At most once per UTC calendar day after authenticated use |

Award XP idempotently with a unique event key (UID + activity + target ID/day). Do not award duplicate points for retries or repeated taps. Do not backfill XP from existing content until a migration policy is approved.

## Permission policy
- MEMBER and MUSICIAN may delete only songs they own.
- PRO, PREMIUM, and VIP are proposed to receive broader song-deletion privileges.
- Before enabling broader deletion, confirm whether this means deleting songs belonging to any member in the shared library, or only any song in the signed-in member's own library.
- If cross-member deletion is approved, require trusted server-side authorization / protected Firestore rules, a confirmation showing the song and original owner, an audit record (actor UID, target song, owner UID, timestamp), and preferably soft-delete/trash with a recovery window.
- Never grant cross-member deletion based on IP address, client-side UI state, or a client-editable profile field.
- XP and level cannot themselves be edited by the member. Privileged role claims must be assigned through a trusted administrative process.

## UI requirements
- Add a member profile screen with avatar, display name, level badge, XP total, progress to next level, and activity history.
- Show level consistently without disrupting existing library and Setlist workflows.
- Provide loading, empty, offline, and sync-error states.
- Show clear permission messaging when a member attempts an action unavailable at their level.

## Test and release gates
1. Test registration and existing account sign-in.
2. Test XP for successful song creation, Setlist creation, and once-daily activity.
3. Verify retries, duplicate taps, edits, failed writes, imports, and offline actions cannot award duplicate XP.
4. Verify members cannot read or alter another member's private profile or XP ledger.
5. Verify all five level thresholds and permission boundaries.
6. Test deletion audit and recovery behavior, including unauthorized attempts.
7. Test iPhone Safari and Android Chrome.
8. Keep V8 on the development branch until all gates pass; only then prepare a reviewed release to the existing URL.
