# Kiflet mobile: migration

How this repository replaces today's driver app, a PWA in Kiflet/backend's
`internal/web/static/driver`, with the Expo app in `docs/architecture.md`,
without leaving any driver unable to work on the way.

It is the mobile part of the whole plan,
[Kiflet/backend `docs/migration.md`](https://github.com/Kiflet/backend/blob/main/docs/migration.md),
where it is phase 5.

Contents:

1. [Why, and what stays the same](#1-why-and-what-stays-the-same)
2. [What this repository needs from the backend first](#2-what-this-repository-needs-from-the-backend-first)
3. [Step 1: accounts and identifiers](#step-1-accounts-and-identifiers)
4. [Step 2: the empty app](#step-2-the-empty-app)
5. [Step 3: the database and the sync engine](#step-3-the-database-and-the-sync-engine)
6. [Step 4: screens](#step-4-screens)
7. [Step 5: GPS in the background](#step-5-gps-in-the-background)
8. [Step 6: delivery, uploads, push](#step-6-delivery-uploads-push)
9. [Step 7: tests and the pilot](#step-7-tests-and-the-pilot)
10. [Step 8: moving every driver](#step-8-moving-every-driver)
11. [The PWA, module by module](#the-pwa-module-by-module)
12. [Parity checklist](#parity-checklist)
13. [Rolling back](#rolling-back)
14. [Risks](#risks)
15. [Done](#done)

---

## 1. Why, and what stays the same

The PWA works offline and has been tested end to end, but a web app can't
record GPS with the screen off: browsers stop it. A truck's trail then has
gaps whenever the driver locks the phone, which is most of a trip. A
native app with a foreground location service doesn't have that limit,
and SQLite is sturdier than the browser's storage, which a phone may clear
when space runs low.

What stays the same, so the backend changes little:

- **The sync protocol:** one request with every waiting item, idempotent
  by `client_id`, a cursor (`internal/service/sync.go`).
- **The rules from the server:** `/api/driver/config` (statuses,
  transitions, reasons, photo limit, tile server).
- **The GPS intervals, battery saver and parked logic** (`gps.js`).
- **What a driver sees and does:** the same screens and words, in English
  and Amharic, and the same look (Desk and Night at large sizes).

The PWA keeps working, served by the backend, until no active driver uses
it.

---

## 2. What this repository needs from the backend first

From Kiflet/backend (its migration plan, phases 2 and 5):

| Change | Why | Can the app start without it? |
|---|---|---|
| `api/openapi/driver.yaml` and `@kiflet/api` with the driver API | types and client | Start with the current `/api/v1/driver/*` and types written to match, replace when the package lands |
| `/api/driver/*` paths (aliases of `/api/v1/driver/*` at first) | the new app's versioned API | yes, use `/api/v1/driver/*` until then |
| `POST /api/driver/uploads` (presigned URLs) and storage keys in `SyncPOD` | photos without inline base64 | yes: send photos inline, as the PWA does, until then |
| `POST /api/driver/push-token`, pushes for new loads and messages | waking the app | yes: sync on a timer only |
| `X-App-Version`, `min_app_version` in config, `426` | forcing updates | yes, but needed before production |
| A device's app and version on the dashboard's driver page | knowing who moved | needed for step 8 |
| Positions through RabbitMQ (backend phase 2) | burst capacity when many phones reconnect | yes; needed before the app is on most phones |

So the app can start as soon as the repository exists, against the API
that serves the PWA today, and adopt each backend change when it lands.

---

## Step 1: accounts and identifiers

Decide once, because some can't change after the first store release:

- [ ] **Android package and iOS bundle ID:** `et.kiflet.driver` if Kiflet
      registers `kiflet.et`, else `com.kiflet.driver`. Permanent once on
      Google Play.
- [ ] **App name on the phone:** "Kiflet Driver" (as the PWA's manifest).
- [ ] **Expo account and project** (owner: the Kiflet organization), EAS
      plan.
- [ ] **Google Play Console** developer account (organization), app
      created, upload key by EAS.
- [ ] **Firebase project** for FCM (Android push), its key in EAS.
- [ ] **Apple Developer Program** only when iOS is wanted (yearly fee).
- [ ] **Sentry** project, or decide to go without at first.
- [ ] **Icons:** the dark mark from the backend's `static/driver/icon.svg`
      and `icon-maskable.svg` (Desk's ink and paper colours), adaptive
      icon for Android, splash in `--bg`.

## Step 2: the empty app

1. `npx create-expo-app` with the TypeScript template, expo-router;
   `strict`; ESLint and Prettier.
2. `app.config.ts`: name, slug, package, version, permissions (location
   foreground and background, camera, notifications, foreground service of
   type location), plugins (expo-location, expo-camera, expo-notifications,
   expo-sqlite, sentry), `updates` and `runtimeVersion` for EAS Update.
3. `eas.json`: `development` (dev client), `preview` (internal testers,
   APK), `production` (AAB for Play).
4. CI (`.github/workflows/ci.yml`): install, lint, type-check, Jest;
   on `main`, an EAS Update to the preview channel; on a tag, an EAS
   production build. Dependabot for npm and Actions.
5. The design tokens and fonts (`src/ui`), copied from the backend's
   `static/driver/app.css`.

Done when: a development build installs on a phone and opens an empty
screen in both themes.

## Step 3: the database and the sync engine

The heart of the app; build and test it before most screens.

1. Drizzle schema and first migration (`docs/architecture.md`, section 4).
2. Sign-in (`POST /api/driver/login` with phone and PIN), token in
   SecureStore, PIN change and reset.
3. The outbox, `effective()`, and the actions (depart, arrive, deliver,
   delay, problem, message, setting change) as functions over SQLite, no
   React.
4. The sync engine: request building, gzip, applying the response in one
   transaction, the cursor, `more`, backoff, the triggers (after an action,
   timer, reconnect, background).
5. Unit tests against a fake API: lost responses, a refused item, a load
   removed while offline, `more`, 401, 426, a full day of positions.
6. An integration test against a backend running from Kiflet/backend's
   compose files with the demo data.

Done when: the tests pass and a debug screen can sign in as `0912345601`
(PIN `2580` in the demo data), list loads from SQLite and send a status
change made offline.

## Step 4: screens

Port each PWA screen (table below) in this order: My loads, a load, report,
messages, menu, sign-in polish, delivery (step 6). Same words, same order of
buttons, same large sizes. Check each against the PWA side by side on the
same phone.

## Step 5: GPS in the background

1. The location task (`TaskManager.defineTask`) writing to SQLite.
2. Starting and stopping with the active load and the sharing setting.
3. The intervals, battery saver and parked checks from `gps.js`.
4. The Android foreground service notification, in both languages.
5. The two-step permission flow with an explanation screen first.
6. A field test: a day's drive with the phone locked; compare the trail
   on the dashboard's shipment page with the PWA's on the same trip.

## Step 6: delivery, uploads, push

1. Record delivery: recipient, photos (resized and compressed), signature,
   the location check and reason.
2. Uploads: presigned when the backend has them, inline before that.
3. Push: register the token, handle a new load, a change and a message;
   open the right screen on tap.

## Step 7: tests and the pilot

1. Maestro flows (`docs/architecture.md`, section 15) running in CI on an
   emulator against staging, nightly.
2. The parity checklist below, by hand, on a cheap Android phone.
3. **Internal testing** (Play's internal track, plus the APK): the Kiflet
   team and a driver or two, for a week of real trips.
4. **Closed testing** with the pilot company's drivers: weak signal,
   screen locked, a day without data, the phones they own. The backend's
   ROADMAP calls for this session on a cheap phone; it happens here.
5. Fix what the pilot shows before production.

## Step 8: moving every driver

1. Production release on Google Play and the APK on the website.
2. The dashboard shows each driver's app (PWA or the new app) and version
   (backend change, section 2).
3. An SMS to drivers still on the PWA with the install link; a banner in
   the PWA asking them to install the app. A driver signs in to the new
   app with the same phone and PIN.
4. Anything waiting in the PWA's outbox must be sent before the driver
   switches: the PWA's banner says so, and shows "All sent" when it's safe.
5. When no active driver has used the PWA for 30 days, Kiflet/backend's
   phase 6 removes it and the `/api/v1/driver/*` paths.

---

## The PWA, module by module

| PWA (`internal/web/static/driver/`) | Here | Notes |
|---|---|---|
| `app.js` | `app/_layout.tsx`, start-up in `src/` | migrations, config, session, start sync and location |
| `router.js` | expo-router | |
| `state.js` (`S`) | Zustand stores | session, sync status, connectivity, battery |
| `store.js` (IndexedDB) | `src/db` (SQLite, Drizzle) | |
| `sync.js` | `src/sync` | `enqueue`, `effective`, `scheduleSync`, `sync`, `fixForEvent`, `freshFix` |
| `gps.js` | `src/location` | the same intervals; foreground service instead of `watchPosition` and wake lock |
| `api.js` | `src/api` | `@kiflet/api`, version header, gzip |
| `settings.js` | `src/settings` | `changeSetting`, `adoptSettings` (settings follow the driver to a new phone) |
| `signature.js` | react-native-signature-canvas | |
| `text.js` | `src/i18n/en.json`, `am.json` | same keys where possible |
| `dom.js` | React Native components | |
| `screens/login.js` | `app/(auth)/login.tsx` | |
| `screens/pin.js` | `app/(auth)/pin.tsx` | |
| `screens/list.js` | `app/(app)/index.tsx` | |
| `screens/detail.js` | `app/(app)/shipments/[id].tsx` | the mini-map with theme colours |
| `screens/deliver.js` | `app/(app)/shipments/[id]/deliver.tsx` | |
| `screens/report.js` | `app/(app)/shipments/[id]/report.tsx` | |
| `screens/message.js` | `app/(app)/messages.tsx` | |
| `screens/menu.js` | `app/(app)/menu.tsx` | |
| `sw.js`, `/driver/build.json` | EAS Update | no service worker |
| `manifest.webmanifest`, icons | `app.config.ts`, `assets/` | |
| `app.css` | `src/ui` tokens and components | |

---

## Parity checklist

From the backend's `docs/E2E_TEST_GUIDE.md`, sections 4 and 5, on a real
phone. Every item must pass before production.

- [ ] Sign in with phone and PIN; choose a new PIN when asked; forgot PIN by SMS code.
- [ ] Loads appear under Active; finished ones under "Finished this week".
- [ ] Start trip, I've arrived, report a delay and a problem with reasons.
- [ ] Offline: the pill says No signal; actions still work and show "waiting to send".
- [ ] Record delivery offline with name, signature and photos; it shows as delivered.
- [ ] Back online: everything sends within seconds; the dashboard shows the delivery, its signature, photos and "Location verified" when within range.
- [ ] A delivery far from the drop-off warns and asks for a reason.
- [ ] Conflicts: a load cancelled or reassigned while the phone was offline is handled as the guide's section 5 says.
- [ ] Messages from the office arrive (with a push) and replies reach the dashboard.
- [ ] GPS keeps recording with the screen locked for an hour of driving; the trail has no gaps the dashboard marks.
- [ ] Battery saver (chosen, and under 20% not charging) changes the interval to 15 minutes.
- [ ] Settings changed on one phone follow the driver to another.
- [ ] Amharic throughout; light and dark.
- [ ] An app below the minimum version shows the update screen and keeps its outbox.
- [ ] Sign out, and sign in as another driver: no data from the first.

---

## Rolling back

- A bad JavaScript release: republish the previous EAS Update to the
  channel; phones take it on the next start.
- A bad native release: halt the Play rollout and publish the previous
  build with a higher version code.
- A driver with a problem: sign in to the PWA, which keeps working until
  phase 6, after sending anything waiting in the app (the menu shows it).

---

## Risks

| Risk | What reduces it |
|---|---|
| Phone makers killing the location service | foreground service; per-maker guidance; the pilot on the phones drivers use; gaps shown on the dashboard |
| Play's background location review delays release | prepare the declaration, video and disclosure early (step 1); internal testing doesn't need the review |
| Data lost when a driver switches from the PWA | the PWA's banner waits for "All sent"; the dashboard shows each driver's waiting items |
| The sync engine behaving differently from the PWA's | port its tests and the guide's conflict cases first (step 3) |
| Expo SDK upgrades every few months | upgrade on a schedule, one SDK at a time, through a preview build |
| Drivers without Google Play or with little storage | the APK on the website; a small app (no heavy map SDK if MapLibre or Google Maps tiles suffice); storage shown in the menu |

---

## Done

When every active driver uses this app, the PWA has been removed from
Kiflet/backend, and positions keep coming with screens locked. Then
delete this file's finished parts.
