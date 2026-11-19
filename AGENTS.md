# AGENTS.md

## Project

Kiflet mobile: the driver app for Kiflet, offline-first shipment tracking
and proof of delivery for Ethiopian trucking companies. React Native with
Expo, TypeScript, Android first.

- `docs/architecture.md`: how the app is built (SQLite, the outbox and
  sync, background GPS, uploads, push)
- `docs/migration.md`: the plan for replacing the PWA, and the parity
  checklist
- `CHANGELOG.md`: what changed in each release
- The rest of the system is in [Kiflet/backend](https://github.com/Kiflet/backend):
  its `docs/architecture.md` (the whole platform), `docs/migration.md` (all
  phases), `docs/E2E_TEST_GUIDE.md` (every flow; sections 4 and 5 are the
  driver's) and the PWA this app replaces, `internal/web/static/driver`.

## State

No app code yet. Until it ships, the PWA in Kiflet/backend is what drivers
use, and it is the reference for how each screen must behave.

## Rules

- **Offline first.** Screens read only SQLite. Every action writes its new
  state and an outbox row in one SQLite transaction, then the sync engine
  sends it. Nothing a driver does waits on the network.
- **Every item sent has a `client_id`** and may be sent more than once;
  the server ignores repeats. Delete an item only when the server confirms
  it.
- **No business rules here.** Statuses, what a driver may change them to,
  reasons and limits come from `/api/driver/config`. The server decides
  conflicts.
- **The API contract is `@kiflet/api`,** generated from Kiflet/backend's
  `api/openapi/driver.yaml`. A missing field is a change in the backend
  first.
- **The location task only writes to SQLite;** the sync engine does all
  networking.
- **The token lives in SecureStore,** never in SQLite or logs.
- **Every word a driver reads is in English and Amharic** (`src/i18n`,
  same keys; a test checks).
- **Built for cheap phones in bright sun:** large tap targets (48 px and
  up), high contrast, the Desk and Night colours, small photos, little
  storage.
- **Native changes need a store release;** JavaScript changes go out with
  EAS Update. Know which one a change is before merging it.
- Commits and pull requests carry no AI co-author or "Generated with"
  lines.
- Every change gets a line in `CHANGELOG.md`.
