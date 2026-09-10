# 12 — Deployment & DevOps

## 1. Model — one deployment

```
CDN/WAF → LB → Nginx + PHP-FPM (2 replicas) ──▶ PostgreSQL (1 instance, 3 schemas) + shared storage
             ├─ Redis (cache/session/queue, incl. event bus transport)
             ├─ Queue workers: events · notifications · billing · bookings
             └─ Scheduler (cron): daily fees, fines, subscriptions, backups
```

One app instance serves Core + all modules; modules activate per-tenant via entitlements. **This is the built-now topology.** Modular boundaries make later extraction mechanical (below).

## 2. Requirements

- PHP ≥ 8.3 (ext: bcmath, gd, intl, pdo_pgsql, zip, curl, fileinfo, mbstring) · Node ≥ 20 (Vite build).
- PostgreSQL ≥ 15 (schemas + gen_random_uuid) · Redis ≥ 7 (optionally managed).
- Reverse proxy w/ TLS; Stripe webhooks need public HTTPS endpoint.

## 3. Build & release

- `composer install --no-dev --optimize-autoloader && npm ci && npm run build`.
- Atomic release: `php artisan down --allow ip` → pull → migrate (additive) → refresh caches → restart workers → `php artisan up`.
- Feature/module flags shipped in config; entitlement flips never co-deploy with code (config-only).

## 4. Nginx (example)

```nginx
server {
  listen 443 ssl http2;
  server_name *.platform.example.com platform.example.com;
  root /var/www/multi-saas/public;
  index index.php;
  location / { try_files $uri $uri/ /index.php?$query_string; }
  location ~ \.php$ { include fastcgi_params; fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name; fastcgi_pass 127.0.0.1:9000; }
  location ~* \.(js|css|png|jpg|svg|woff2)$ { expires 30d; add_header Cache-Control "public"; }
  add_header Strict-Transport-Security "max-age=31536000" always;
}
```

## 5. Configuration inventory (`.env`)

`APP_URL`, `DB_*`, `CACHE_DRIVER=redis`, `SESSION_DRIVER=redis`, `QUEUE_CONNECTION=redis`, `STRIPE_KEY/SECRET`, `WEBHOOK_SECRET`, `SMS_*`, `ANTHROPIC_API_KEY`, `STORAGE_DISK`, `APP_TIMEZONE=Asia/Dhaka`. `.env.example` shipped; secrets via secret manager in prod.

## 6. Queue & scheduler

- Supervisor workers per queue: `events,high,default,notifications`.
- Cron: `* * * * * php artisan schedule:run` (tasks from 09 §4).

## 7. Backups & DR

- PostgreSQL `pg_dump` nightly, encrypted, S3 (30d + monthly). Media `rclone sync` nightly.
- Restore drill quarterly to staging → smoke suite (`migrate:fresh --seed` on copy, entropy checks).
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

- `composer audit`, npm audit (CI), dependency update monthly; emergency hotfix pipeline documented; headless security scan on scheduled tag builds.