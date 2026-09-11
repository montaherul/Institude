# 07 — Integrations

## 1. Event bus (internal)

- **Outbox pattern** — publisher writes `core.event_outbox` in the SAME DB transaction as the business change (no lost events).
- **Dispatcher job** (queue `events`, retry 3× exponential backoff) publishes `event_type + payload` to Redis channel `ev:{tenant_id}:{event_type}` and marks dispatched.
- **Listeners** subscribe via a central listener map (DI-registered handlers, e.g. `BusListenerRegistry`); each listener begins with an entitlement guard:

```csharp
bus.Register("school.StudentEnrolled", () => new UpdateTransportPool());
// UpdateTransportPool.Handle(payload) {
//   if (!HasEntitlement("transport")) return;   // no-op, module independent
//   … upsert transport.subscribers suggestion …
// }
```

- Keys: `event_key` unique → idempotent dispatch/dedupe.
- Failure handling: outbox row `failed` + attempts; Dead-letter table + admin retry button. No listener = event discarded silently.

## 2. Payments — Stripe (platform billing)

- `core.payment_gateways` (test/live); Stripe customer created per tenant at first checkout.
- Checkout: `PaymentIntent` → `subscribe` endpoint → webhook `checkout.session.completed` → **create `core.invoices`(paid) → extend `entitlements`** (or add module) → emit `EntitlementChanged`.
- Customer portal (`/billing/portal`) for card updates/voids; idempotency keys on webhooks; retry-safe handler.
- Future providers (bkash/nagad) behind same `core/payments` abstraction.

### Module-level payments (tenant collects from their customers)
- School fee collection, Rent invoice payment, Transport fares — **separate** from platform billing; each module stores its own `gateway`/`txn_id`/`receipt_no` columns + pays out to the tenant (see 05).

## 3. Stripe webhook events handled

`checkout.session.completed`, `invoice.payment_succeeded`, `invoice.payment_failed`, `customer.subscription.updated/deleted` → entitlement adjustments + notifications + audit rows.

## 4. Notifications

- Providers: **Email (SMTP provider per tenant or platform)** · **SMS (Bangladeshi gateway + Twilio)** · **Push (FCM/WebPush optional)**.
- Template engine: `{name}`, `{period}`, `{amount}`, `{url}` placeholders; per-channel body.
- One dispatch pipeline → `core.notifications` row (status/attempts) → retries → failure logs surfaced in admin.

## 5. SMS & WhatsApp

- Multi-gateway strategy side: for **end-customer comms** (rent overdue, trip reminder, fee due) use tenant-configured SMS; WhatsApp via Cloud API (templates + webhook statuses) as module-level channel.
- Consent captured at onboarding per module; unsubscribe link in body.

## 6. Maps & reports exports

- Google Maps embed for property addresses / route stop map; optional geocoding on stop create.
- Report seals: CSV export (all tables), PDF via the Application/Documents layer (e.g. QuestPDF) for receipts/agreements/report cards; barcode on subscriptions when using carded passes.

## 7. AI assistant (optional slice)

- Tenant settings flag `ai_enabled`; server-side call (Anthropic/OpenAI) for draft letters/notices in School module & maintenance summaries in Rent. Key masked; disabled on trial plans.

## 8. Storage & media

- `core.media` polymorphic; local `public/` default → S3 option in system settings; used for logos, property photos, student photos, notices.

## 9. Idempotency & failure conventions

- All webhooks: `Idempotency-Key` safe, handler upsert by `(event type, txn/event id)`.
- All external sends: recorded first (pending) → attempt → status; retries capped.
- Secrets rotation supported (masked UI, `last4` shown).