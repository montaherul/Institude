# 07 — Integrations

All keys stored via Institution Settings (per-tenant) or global settings (gateways). Secrets masked on read.

## 1. Google Meet

- **OAuth2:** `Client ID/Secret`, scopes calendar.readwrite, token storage + refresh (guarded service in shared package — one OAuth per institute supported at ref's level).
- Flow: create session → schedule Calendar event → returns `meet_link` stored on session → notify list.
- `test-connection` button validates credentials before save.
- Recurrence (repeat until) supported in session form.

## 2. SMS Gateway (multi-provider)

- Provider abstraction: `SMSProvider` interface (send(number,text)).
- Registered: **Twilio** (live/test creds, number) + **Bangladeshi gateway** (API key hit-rate/gateway credit style) under `sms_providers` table by institute.
- Per-request provider selection; count against package SMS balance.
- Sending: single/multi number compose (≤300 chars), template params (`{name}`, `{result}`), exam-result broadcast; attempts recorded in `sms_sent_logs` with status.

## 3. WhatsApp Cloud API

- Via Notification service (webhook/listener) creating message threads per recipient.
- Settings: Phone Number ID, WhatsApp Business ID, Access Key, preferred language, notification types.
- Templates (messages, with named/reusable template records) + automated status/logging to `whats_app_logs`; retry button resends.

## 4. Payment gateways

- Gateway manager supporting `bkash`, `nogod`, `sslcommerz`, `stripe`, `paystack`, `razorpay`, `phonepe`, `razorpayx` (each with key/secret/merchant).
- Test/live mode toggles baked into gateway config CRUD.
- Payment flow: init transaction → redirect → callback/verify → `transactions` (transaction_id, method, amount, from/from_type, details) → confirm collection.

## 5. Google Maps

- Contact page embed: static `<iframe>` using tenant address key; live map on map tab.

## 6. ID card / bar codes

- Student/staff barcode (code-128) generation; printed on ID cards and books (`books-ber-code-print`) — pure client-side lib, no external dependency.

## 7. AI Assistant (Anthropic)

- HTTP client wrapper; models configurable in `/ai/settings` (`anthropic_api_key`, model, enabled).
- RAG over institute data for chat/insights: embed relevant report rows (results, fees, attendance) per question; content writer for text generation.
- Sanitized prompts; never send PII beyond selected context; key masked; usable only while enabled.

## 8. File storage

- Local/public disk default; S3-compatible option in general settings (bucket, folder, region, credentials); URLs generated via storage link — used by syllabus files, notices, banners, images.

## 9. Email (SMTP)

- Mail chip via general settings (host/port/secure/from); used for password resets, results, contact-message notifications; error toasts on failure.

## 10. Mobile app sections

- CMS drives store URLs (App Store/Google Play) rendered on public site; (reference exposes section CRUD only — embed links as configured).

---

### Common contract (SMS/WhatsApp/Email)

```php
interface NotifiableChannel {
  // recipient, message/template, data[]  →  service id + status + log id
  sendLog(): Log  // status, sent_at, retries, error
}
```
Institution wide notifications configured through **Notifications table** (`notifiable` morphs, `channel`, `type`, `recipient`, `status`, `error`, counts). Admin dashboard widget shows pending/sent.