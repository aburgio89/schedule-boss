@AGENTS.md

# Schedule Boss — Project Context

## What this is

A cross-platform mobile app (React Native + Expo) that solves the "schedule boss"
problem: finding shared free time across a group (originally motivated by D&D
groups struggling to schedule sessions). This is a portfolio project for an
entry-level SWE resume, built by a recent BSSWE grad.

## Working style — IMPORTANT

The developer is concerned about using AI and wants to be involved in decision making. Default to:

- Explaining what to do and why, in plain steps
- Asking before generating large blocks of code
- Pointing out structural/architectural decisions and tradeoffs rather than
  just producing a working answer
  Do not scaffold entire features unprompted. If a fix requires touching several
  files, explain the plan first.

## Tech stack (decided, don't relitigate without reason)

- **Expo + Expo Router** (file-based routing under `src/app/`) — NOT a manually
  built React Navigation stack. Routes are defined by file/folder structure
  (e.g. `src/app/groups/[groupId].js`), navigation via `expo-router`'s `router`
  object or `<Link>`, dynamic params via `useLocalSearchParams()`.
- **Firebase JS SDK** (Firestore + Auth) — plain `firebase` package, not
  `@react-native-firebase` (avoiding native ejection for as long as possible).
  Config lives in `src/firebase/config.js`, pulling secrets from `.env` via
  `react-native-dotenv` (`@env` imports). Real Firebase keys are NOT committed;
  `.env.example` documents required vars.
- **react-native-dotenv** for env vars, configured via `babel.config.js`
  (`babel-preset-expo` + the dotenv plugin).
- No native calendar module yet (`react-native-calendar-events` planned for
  later — requires EAS dev build, not compatible with Expo Go).

## Data model (Firestore)

- `users/{userId}` — profile info + `connectedCalendars` status flags
- `groups/{groupId}` — name, ownerId, `memberIds` array, `inviteCode`
- `events/{eventId}` — belongs to a group; title, date range, `isRecurring`,
  simple `recurrenceRule` (WEEKLY/BIWEEKLY enum, not full RFC 5545), status
- `events/{eventId}/availability/{userId}` — subcollection, one doc per user
  per event: `slots: [{start, end, weight}]` where weight = preferred(2) /
  possible(1) / unavailable(0), plus `source` (manual/google/device)

Subcollection structure for availability is deliberate: enables a single
real-time listener per event, and makes "has everyone responded" a simple
doc-count check against `memberIds.length`.

## Feature priorities

**Building now / high priority:**

- Weighted availability (not just free/busy)
- Recurring session support (simple enum-based recurrence)
- Calendar sync — both Google Calendar (OAuth + Calendar API, cloud-based,
  cross-platform) AND device calendar (`react-native-calendar-events`,
  wraps EventKit on iOS / Calendar Provider on Android) — planned as two
  parallel sync paths feeding the same availability data shape

**Backburner / explicitly deprioritized (don't build unless asked):**

- "Missing person" nudge (ping non-responders)
- Fairness tracking (flag if the group always defers to one person's schedule)

## Overlap computation

Client-side for now (not a Cloud Function) — simple at D&D-group scale
(4–6 people). Approach: flatten all users' slots, bucket the date range into
fixed intervals (15–30 min), sum weights per bucket, sort descending. This is
intentional — moving to server-side is a documented "future improvement," not
a current requirement.

## Build order (roughly where we are / where we're headed)

1. ✅ Blank Expo app running via Expo Go
2. ✅ Firebase connected, `.env` + `.gitignore` set up
3. 🔄 Navigation — mid-correction from a manual React Navigation stack to
   proper Expo Router file-based structure (screens moving from `src/screens/`
   into `src/app/`)
4. ⬜ Create Group screen (Firestore write)
5. ⬜ Join Group flow (query + `arrayUnion`)
6. ⬜ Event creation screen
7. ⬜ Availability picker UI (build data layer before the visual grid)
8. ⬜ Real-time listener (`onSnapshot`) + overlap calculation
9. ⬜ Calendar sync (Google Calendar OAuth first, device calendar later)

## Notes on current file structure

`src/app/` is pre-scaffolded with `components/`, `constants/`, `hooks/`,
`utils/` from the starter template — check what's already there before
creating duplicates. There's also a `global.css`, suggesting NativeWind
(Tailwind for RN) may already be wired in — verify before assuming plain
StyleSheet styling.

Security rules: Firestore is currently in test/open mode for early
development. Real security rules need to be written before this is
resume-ready — don't treat open rules as a final state.
