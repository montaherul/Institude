# 12 — Deployment & DevOps

## 1. Model — one deployment

```
CDN/WAF → LB → Kestrel (ASP.NET Core, 2 replicas, behind reverse proxy) ──▶ SQL Server (1 instance, 3 schemas) + shared storage
             ├─ Redis (cache/session/queue, incl. event bus transport)
             ├─ Background workers: events · notifications · billing · bookings
             └─ Scheduler (hosted services): daily fees, fines, subscriptions, backups
```

One app instance serves Core + all modules; modules activate per-tenant via entitlements. **This is the built-now topology.** Modular boundaries make later extraction mechanical (below).

## 2. Requirements

- .NET ≥ 8 (LTS; current target .NET 10) · Node ≥ 20 (Vite build).
- SQL Server 2019+ (schemas + NEWID()) · Redis ≥ 7 (optionally managed).
- Reverse proxy w/ TLS; Stripe webhooks need public HTTPS endpoint.

## 3. Build & release

- `dotnet publish -c Release -o ./artifacts && npm ci && npm run build`.
- Atomic release: maintenance mode (status page / reverse-proxy flag) → pull → `dotnet ef database update` (additive) → refresh caches → restart workers → serve.
- Feature/module flags shipped in config; entitlement flips never co-deploy with code (config-only).

## 4. Reverse proxy (Kestrel behind Nginx — example)

```nginx
server {
  listen 443 ssl http2;
  server_name *.platform.example.com platform.example.com;
  location / {
    proxy_pass http://127.0.0.1:5000;                       # Kestrel (dotnet run / systemd)
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection keep-alive;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
  location ~* \.(js|css|png|jpg|svg|woff2)$ { expires 30d; add_header Cache-Control "public"; }
  add_header Strict-Transport-Security "max-age=31536000" always;
}
```

## 5. Configuration inventory

App configuration lives in `appsettings.json` (base) with per-environment overrides. Environment-specific values (connection string `ConnectionStrings:Default`, Redis, Stripe keys, SMS, AI key, storage) come from environment variables / user-secrets (dev); secrets via a secret manager in prod.

## 6. Queue & scheduler

- Background workers per queue: `events, high, default, notifications` (hosted `BackgroundService`/Hangfire).
- Scheduled jobs run inside the worker process (hosted services) — tasks from 09 §4.

## 7. Backups & DR

- SQL Server `BACKUP DATABASE` (full) nightly, encrypted, S3 (30d + monthly). Media `rclone sync` nightly.
- Restore drill quarterly to staging → smoke suite (re-run EF migrations + seed on the copy, entropy checks).
- RPO ≤ 24 h · RTO ≤ 4 h. Secrets backed in vault, not DB.

## 8. Scale path (when to split — follow spec §6)

Precondition flags (any): single-module heavy load (GPS/trips telemetry), separate scaling need, independent deploy cadence, or compliance residency.

- First split: **Transport** (live trip data + GPS) → its own service via existing schema boundary + `core.event_outbox` as public API (HTTP/Redis bridge).
- Second: **Rent** if heavy invoicing batch; **School** stays monolith.
- Every step keeps Core + remaining modules monolithic — do NOT plan microservices now.

## 9. Monitoring & alerting

- Errors → Sentry (+ tenant context); logs daily rotation.
- Metrics: outbox depth, notification failures, webhook error rate, queue length, P95 API latency, DB bloat, backups age.
- Alerts: queue stuck, backup failure, Stripe webhook anomalies, disk/load thresholds. Healthcheck endpoint for LB; status page public.

## 10. Security ops

- `dotnet list package --vulnerable`, npm audit (CI), dependency update monthly; emergency hotfix pipeline documented; headless security scan on scheduled tag builds.