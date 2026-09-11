# 12 — Deployment & DevOps

## 1. Environment topology

```
[Browser] → CDN/WAF → Load Balancer → Reverse proxy (Nginx) → Kestrel (ASP.NET Core, n replicas)
                                        │  shared storage (uploads)
                                        ├─ Redis (optional cache/session)
                                        ├─ SQL Server (single shared schema) — primary DB
                                        └─ Background workers (IHostedService inside each replica)
```

## 2. Requirements

- .NET 8 SDK/runtime (self-contained publish per environment).
- Node ≥ 20 for the frontend bundle build (`npm ci && npm run build`); committed `wwwroot` bundles optional.
- SQL Server (2022+; Azure SQL option) — connection via `ConnectionStrings:Default`.
- Reverse proxy: Nginx (config below), HTTPS via Let's Encrypt; Windows option: IIS reverse proxy/ARR.

## 3. Build & artifacts

- `dotnet publish MightySchool.Web -c Release -o ./out` (self-contained, single-file optional) + `npm ci && npm run build` (frontend into `wwwroot`).
- EF migrations are part of the build (`dotnet ef migrations bundle` optional for offline apply).
- Optional docker image: tag with `GIT_SHA`; rolling deploy (health-check based) for zero downtime.
- Migrations run in a job step before worker cutover.

## 4. Nginx vhost (per tenant custom-domain)

```nginx
server {
  listen 443 ssl http2;
  server_name +.institute.bdboibazer.com;            # subdomain layout
  ssl_certificate     /etc/letsencrypt/live/domain/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/domain/privkey.pem;

  location / {
    proxy_pass http://127.0.0.1:5000;                # Kestrel
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";           # SignalR/WebSockets if needed
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
  location ~* \.(js|css|png|jpg|jpeg|gif|svg|ico|woff2)$ { expires 30d; add_header Cache-Control "public, immutable"; }
  location = /favicon.ico { access_log off; log_not_found off; }
}
```
Wildcard cert or per-domain; custom-domain tenants get their own vhost/DNS record.

## 5. Workers & scheduler

- Background jobs run inside each ASP.NET Core replica (`IHostedService`); no separate queue daemon for v1.
- Optional Hangfire dashboard/service for job observability.
- Health endpoint (`/healthz`) used by the load balancer; systemd/Windows-Service `Restart=always` on each replica.

## 6. Environment config

`appsettings.Production.json` (secrets via environment variables / secret store, never committed); `appsettings.example.json` documents the vars:

```json
{
  "ConnectionStrings": { "Default": "Server=...;Database=mighty_school;..." },
  "Cache": { "Redis": "" },
  "Storage": { "Provider": "local|s3" },
  "Mail": { "Host": "...", "Port": 587, "Username": "...", "Password": "..." },
  "Sms": { "Gateway": "...", "ApiKey": "..." },
  "WhatsApp": { "AccessKey": "..." },
  "Payment": { "Gateway": "...", "Key": "...", "Secret": "..." },
  "Anthropic": { "ApiKey": "...", "Model": "..." },
  "GoogleMeet": { "ClientId": "...", "ClientSecret": "..." },
  "App": { "Timezone": "Asia/Dhaka" }
}
```

## 7. Release runbook

1. Enable maintenance (offline page via `UseStatusCodePages`/`app_offline.htm`) with allowlisted IPs → pull release → apply `dotnet ef database update` → rebuild caches → restart workers → back online.
2. Zero-downtime: run migrations first, then cut workers, last swap web.
3. Cache invalidation for tenant (`/institute-cache-clear`) after settings updates.

## 8. Monitoring & alerts

- Serilog (file daily rotation) to shared sink; **Sentry** (`Sentry.AspNetCore`) or Application Insights for exceptions; alerts on worker health, backup failure, disk space.
- Dashboard: request rate (per tenant), JS errors, SQL slow queries (`Query Insights`), SMS/WhatsApp success rate.
- Scheduled job success beacon; dead-worker restart (`Restart=always`).

## 9. Backup/recovery

- DB: `sqlcmd`/`BACKUP DATABASE` (or SQL Server managed backups) daily, encrypted (or TDE), to S3, retain 30 days + monthly archive.
- Uploads: `rclone sync wwwroot/uploads` daily.
- DR: restore to staging → smoke suite (migrate + seed check) → failover.
- RPO ≤ 24 h, RTO ≤ 4 h.

## 10. Compliance ops

- Security headers on all responses (HSTS, X-Content-Type-Options, X-Frame-Options DENY, Referrer-Policy) via middleware.
- Rate limit on login/compose/AI (see 10).
- Dependency registry: `dotnet list package --vulnerable` (NuGet audit) + `npm audit` in CI.
- Dependency update policy: monthly; emergency hotfix pipeline documented.