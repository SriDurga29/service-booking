# Stillwell service booking

A mobile-friendly booking application with a Node.js HTTP API, SQLite persistence, and Stripe Checkout. The customer page never marks a booking paid: only the signed Stripe webhook, followed by a server-to-server Checkout Session retrieval, can make that transition.

## Run locally

Requires Node.js 22.13 or newer (Node 24 recommended). No npm packages are required.

1. Copy `.env.example` to `.env` and set `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, and an `ADMIN_API_KEY` for payment-enabled use.
2. Run `node src/server.js` and open `http://localhost:3000` (or `npm.cmd start` in PowerShell).

The app seeds three example services and appointment slots on first launch. Set Stripe's webhook endpoint to `https://<your-domain>/api/webhooks/stripe` and subscribe to `checkout.session.completed`, `checkout.session.async_payment_succeeded`, `checkout.session.async_payment_failed`, and `charge.refunded`. Use Stripe test credentials locally. Without credentials, browsing and booking holds work; checkout returns 503 and does not pretend that payment succeeded.

## Architecture and consistency

- `src/server.js` loads environment settings, starts the HTTP server, and closes the database cleanly.
- `src/service.js` owns schema initialization, validation, slot claiming, booking state transitions, checkout creation, access control, and webhook processing.
- `public/` contains the responsive service catalogue, slot picker, customer details form, payment redirect, lookup, status and refund request views.
- `test/booking.test.js` covers booking idempotency, malformed inputs, competing slot claims, private lookups, provider verification, webhook replay, inactive services, hold expiry, and refund state.

### Tables and invariants

`customers` has a unique email; `services` supplies the authoritative price and active flag; `slots` belongs to one service and uniquely identifies a service/start-time pair; `bookings` references customer, service and slot and stores separate booking/payment state, a unique client idempotency key, an expiry, and only a hash of its private access token; `payments` is one-to-one with a booking and has unique provider-session and idempotency references; `webhook_events` uses the provider event ID as its primary key.

Booking creation runs under `BEGIN IMMEDIATE`, rechecks service activity and inserts the live slot claim in one transaction. A partial unique index permits at most one `PENDING_PAYMENT`, `CONFIRMED` or `REFUND_REQUESTED` booking per slot. Pending claims expire after `HOLD_MINUTES` (15 by default); stale holds are cancelled before availability checks. The database constraint remains the final concurrency guard.

The browser supplies an idempotency key and a random 256-bit access token. The server stores only its HMAC digest; a replay using the same key and token returns the original booking, while key reuse with another token is rejected. The returned token authorizes access to that single booking and is kept in the browser's local storage so a customer returning from Stripe can look up the result. Treat it like a password; do not share it.

The customer refund endpoint records the request; an operator fulfills it through `POST /api/admin/bookings/:id/refund` using the admin key. Full-refund status changes only after Stripe's `charge.refunded` webhook. Partial refunds are not marked refunded. If payment arrives after its hold expires or its service becomes inactive, the app declines confirmation and requests a compensating Stripe refund with a stable idempotency key.

### Booking and payment states

`PENDING_PAYMENT` + `PENDING` → `CONFIRMED` + `SUCCESS` only after verified payment. A failed asynchronous payment becomes `CANCELLED` + `FAILED`; an abandoned hold expires to `CANCELLED`. A paid booking may move to `REFUND_REQUESTED`; it remains there until the provider's `charge.refunded` webhook confirms the full refund, at which point it becomes `REFUNDED` + `REFUNDED`. The customer refund endpoint records the request; fulfillment is an operator action through the payment provider. Partial refunds are not marked refunded.

The webhook signature is checked over the raw request body with Stripe's timestamp tolerance. Completed payment events are independently retrieved from Stripe and checked for paid/complete status, booking metadata, currency, amount, and Checkout Session ID before the booking is confirmed. Event IDs are unique, making successful deliveries idempotent. If provider verification fails transiently, the event record is removed before returning an error so Stripe can safely retry it. In production, use a durable webhook queue/outbox for stronger operational recovery and alert on webhook failures.

## HTTP API

- `GET /api/services`, `GET /api/services/:id`, `GET /api/services/:id/slots`
- `POST /api/bookings` — JSON customer/service/slot/accessToken, plus `Idempotency-Key`
- `POST /api/bookings/:id/checkout` — booking `Authorization: Bearer <accessToken>`
- `GET /api/bookings/:id` — same booking authorization
- `POST /api/webhooks/stripe` — Stripe signature required
- `POST /api/bookings/:id/refund` — records a refund request, same booking authorization
- `GET /api/admin/bookings` — `X-Admin-Api-Key` required
- `POST /api/admin/bookings/:id/refund` — initiates an idempotent Stripe refund for a requested refund, `X-Admin-Api-Key` required
- `GET /api/health`

Errors use `{ "error": { "code": "...", "message": "..." } }`. Booking creation returns 201, validation errors 422, unavailable/conflicting state 409, missing resources 404, and repeated identical booking requests 200.

## Tests

Run `node --test` (or `npm.cmd test` in PowerShell). The built-in Node test runner uses an in-memory SQLite database and a controlled Stripe API response; it never creates a fake production payment success path.

## Deployment notes

SQLite with WAL is intended for one application host with a persistent local disk. For multiple instances, move the same uniqueness invariant and transactional claim to PostgreSQL (for example a partial unique index and transaction with row locking). Configure TLS at the edge, `PUBLIC_BASE_URL` to the externally reachable HTTPS origin, Stripe live credentials in a secret manager, a strong unique admin API key, database backups, and log/alert collection. Do not expose database files or `.env`. Restrict admin API access at the network layer as well. Payment secrets and raw payment credentials are never sent to or stored by the browser/application.

### Environment variables

| Variable | Purpose | Default |
| --- | --- | --- |
| `PORT` | HTTP listen port | `3000` |
| `DATABASE_PATH` | SQLite database file | `./data/bookings.sqlite` |
| `PUBLIC_BASE_URL` | Customer return URLs | `http://localhost:3000` |
| `STRIPE_SECRET_KEY` | Server-only Stripe API key | unset (payments disabled) |
| `STRIPE_WEBHOOK_SECRET` | Stripe signing secret | unset (webhooks rejected) |
| `ADMIN_API_KEY` | Admin API bearer secret | unset (admin disabled) |
| `HOLD_MINUTES` | Pending appointment hold duration | `15` (minimum 5) |
