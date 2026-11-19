# Kiflet mobile

**The driver app for Kiflet**: shipment tracking and proof of delivery for
Ethiopian trucking companies. This repository will hold the React Native
app (built with Expo) that drivers use to see their loads, report
progress, record deliveries with a signature and photos, and send their
GPS trail, with or without signal.

Kiflet is three repositories:

| Repository | What |
|---|---|
| [Kiflet/backend](https://github.com/Kiflet/backend) | The Go backend, its APIs, workers and infrastructure. Today it also serves the driver app as a web app (PWA), which this repository will replace. |
| [Kiflet/web](https://github.com/Kiflet/web) | The Next.js web app |
| **Kiflet/mobile** (this one) | The driver app |

## Status

Planning. There is no app code yet: drivers use the PWA served by
Kiflet/backend at `/driver/`.

- [`docs/architecture.md`](docs/architecture.md): how the app works
  offline, syncs, records GPS in the background and records deliveries.
- [`docs/migration.md`](docs/migration.md): the plan, step by step, from
  the PWA to this app, with the parity checklist and the pilot.
- The whole system: [Kiflet/backend `docs/architecture.md`](https://github.com/Kiflet/backend/blob/main/docs/architecture.md)
  and [`docs/migration.md`](https://github.com/Kiflet/backend/blob/main/docs/migration.md).

## Running it

Not yet. Step 2 of `docs/migration.md` adds the app, and this section with
it: a development build (`eas build --profile development`), a backend from
Kiflet/backend with the demo data, and `npx expo start`.
