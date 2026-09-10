# 12 — Deployment & DevOps

## 1. Environment topology

```
[Browser] → CDN/WAF → Load Balancer → Web (Nginx + PHP-FPM, n replicas)
                                          │  shared storage (uploads)
                                          ├─ Redis (cache/session/queue)
                                          └─ PostgreSQL or MySQL (single shared schema)
                                       Queue worker (notifications, fine, backups)
                                       Scheduler (cron container/magento-like schedule)
```

## 2. Requirements

- PHP ≥ 8.2, ext: bcmath, gd, intl, pdo_pgsql/mysql, zip, curl, fileinfo, mbstring, imagick optional.
- Node ≥ 20 for Vite build artifacts (committed dist optional).
- Composer 2; Redis (session/cache/queue if enabled).
- Reverse proxy: Nginx (config below), HTTPS via Let's Encrypt.

## 3. Build & artifacts

- `composer install --no-dev --optimize-autoloader && npm ci && npm run build`.
- Optional docker image: tag with `GIT_SHA`; rollout zero-downtime (rolling deploy).
- Migrations run in a job step before workers cut.

## 4. Nginx vhost (per tenant custom-domain)

```nginx
server {
  listen 443 ssl http2;
  server_name +.institute.bdboibazer.com;            # subdomain layout
  ssl_certificate     /etc/letsencrypt/live/domain/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/domain/privkey.pem;
  root /var/www/mighty-school/public;
  index index.php;
  location / { try_files $uri $uri/ /index.php?$query_string; }
  location ~ \.php$ {
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass 127.0.0.1:9000;
  }
  location ~* \.(js|css|png|jpg|jpeg|gif|svg|ico|woff2)$ { expires 30d; add_header Cache-Control "public, immutable"; }
  location = /favicon.ico { access_log off; log_not_found off; }
}
```
Wildcard cert or per-domain; custom-domain tenants get their own vhost/DNS record.

## 5. Queue & scheduler

- Supervisor: `php artisan queue:work --queue=high,notification,default`.
- Cron: `* * * * * cd /var/www/mighty-school && php artisan schedule:run >> /dev/null 2>&1`.

## 6. Environment config

```env
APP_NAME="Mighty School"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://institute.bdboibazer.com
LOG_CHANNEL=stack
DB_CONNECTION=pgsql|mysql
DB_HOST=...; DB_DATABASE=...; DB_USERNAME=...; DB_PASSWORD=...
CACHE_DRIVER=redis; SESSION_DRIVER=redis; QUEUE_CONNECTION=redis
REDIS_HOST=...; REDIS_PASSWORD=...
MAIL_* (SMTP creds)
ANTHROPIC_API_KEY=...
SMS_* / payment gateways secrets (or via env override)
STORAGE_LINK_PUBLIC=1
APP_TIMEZONE=Asia/Dhaka
```

## 7. Release runbook

1. `php artisan down` (maintenance, allowlisted IPs) → pull release → migrate → optimized cache/router/views config → restart workers → `php artisan up`.
2. Zero-downtime: run migrations first, then cut workers, last swap web.
3. Cache invalidation for tenant (`/institute-cache-clear`) after settings updates.

## 8. Monitoring & alerts

- UPtime + Laravel logs (daily rotation), Sentry for exceptions; alerts on queue length, backup failure, disk space.
- Dashboard: request rate (per tenant), JS errors, DB slow queries, SMS/WhatsApp success rate.
- Scheduled job success beacon; dead-worker restart (systemd restart=always).

## 9. Backup/recovery

- DB: `pg_dump`/`mysqldump` daily, encrypted (age) → S3, retain 30 days + monthly archive.
- Uploads: `rclone sync public/uploads` daily.
- DR: restore to staging → smoke suite (migrate+fresh seed check) → failover.
- RPO ≤ 24 h, RTO ≤ 4 h.

## 10. Compliance ops

- Security headers on all responses (HSTS, X-Content-Type-Options, X-Frame-Options DENY, Referrer-Policy).
- Rate limit on login/compose/AI (see 10).
- Vendor registry (composer audit, `npm audit`) in CI.
- Dependency update policy: monthly; emergency hotfix pipeline documented.