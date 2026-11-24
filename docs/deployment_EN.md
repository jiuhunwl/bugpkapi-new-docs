# Deployment Guide

[简体中文](deployment.md) · **English**

Suitable for single-server or small-scale production deployments: Linux + MySQL 8 + Redis 6+ + Nginx, with the frontend and backend on the same origin.

## 1. Prerequisites

- Linux x86_64 or arm64 with systemd
- MySQL `8.0+`, character set `utf8mb4`
- Redis `6+`
- Nginx (optional, but recommended in production for TLS and static assets)
- A domain resolved to the server, plus a certificate

## 2. Prepare the database

Create a dedicated account for the application; do not use `root`:

```sql
CREATE DATABASE bugpk_api DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'bugpk'@'127.0.0.1' IDENTIFIED BY '<strong-password>';
GRANT ALL PRIVILEGES ON bugpk_api.* TO 'bugpk'@'127.0.0.1';
FLUSH PRIVILEGES;
```

The initialization SQL includes an empty-database guard: the script aborts with an error if the target database is not empty. **Do not** import it into a database that already contains business data.

## 3. Directory layout

In production a flat layout is recommended, with the binary, configuration, migrations and frontend build output in the same directory:

```text
/www/wwwroot/bugpk_app/
├── bugpk-api              # backend binary
├── config.yaml            # non-sensitive configuration
├── .env                   # secrets and environment variables (mode 600)
├── migrations/            # migration SQL (used for upgrades)
├── sql/full/              # full initialization SQL (fresh installs only)
└── frontend/dist/         # frontend build output
```

## 4. Build the backend

```bash
git clone https://github.com/jiuhunwl/bugpkapi-new-go.git
cd bugpkapi-new-go

make build-linux-amd64    # artifact: bugpk-api-linux-amd64
make build-linux-arm64    # use this one for ARM servers
```

Rename the binary to `bugpk-api` and upload it to the deployment directory, and upload `migrations/` and `sql/` as well.

## 5. Initialize the frontend

On the build machine (Node.js `^22.18.0 || ^24.0.0`, pnpm `>= 11`):

```bash
git clone https://github.com/jiuhunwl/bugpkapi-new-vue.git
cd bugpkapi-new-vue
pnpm install
pnpm build                # artifact: playground/dist
```

For same-origin deployment keep `VITE_GLOB_API_URL=/system-api` (the default); do not change it to an absolute backend address, otherwise you will introduce cross-origin and refresh-token cookie problems. Upload `playground/dist` to `frontend/dist`.

## 6. Configuration

```bash
cp configs/config.yaml.example /www/wwwroot/bugpk_app/config.yaml
cp deploy/.env.example /www/wwwroot/bugpk_app/.env
chmod 600 /www/wwwroot/bugpk_app/.env
```

Fill in at least the following in `.env`:

```dotenv
APP_ENV=production
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=bugpk_api
DB_USER=bugpk
DB_PASSWORD=<strong-password>
DB_TIMEZONE_DATA_CONFIRMATION=mysql-civil-v1:<computed per configuration reference>

JWT_SECRET=<openssl rand -hex 32>
JWT_COOKIE_SECURE=true
INITIAL_ADMIN_PASSWORD=<8-64 character strong password>
PAYMENT_CONFIG_SECRET=<openssl rand -hex 32>
SYSTEM_CONFIG_SECRET=<openssl rand -hex 32>

FRONTEND_SERVE_ENABLED=true
FRONTEND_DIST_PATH=frontend/dist
MIGRATION_ENABLED=false
SERVER_TRUSTED_PROXIES=127.0.0.1
```

See the [Configuration Reference](./configuration_EN.md#database-timezone-confirmation-dbtimezone_data_confirmation) for how to compute `DB_TIMEZONE_DATA_CONFIRMATION`. This step must not be skipped; even a new database must be confirmed.

## 7. Import the full SQL

```bash
mysql --default-character-set=utf8mb4 -u bugpk -p bugpk_api \
  < /www/wwwroot/bugpk_app/sql/full/bugpk_api_full_v1.2.113.sql
```

After importing, make sure `MIGRATION_ENABLED=false` so the application does not run migrations again on startup.

## 8. First startup and the administrator

```bash
cd /www/wwwroot/bugpk_app
set -a; source .env; set +a
./bugpk-api
```

The first startup creates the account `admin` using `INITIAL_ADMIN_PASSWORD`. After a successful login it is recommended to:

1. Change the password immediately and enable two-step verification;
2. Remove `INITIAL_ADMIN_PASSWORD` from `.env`;
3. Restart the service to confirm it still starts normally.

Health checks:

```bash
curl -fsS http://127.0.0.1:8080/healthz
curl -fsS http://127.0.0.1:8080/readyz
```

## 9. systemd

`/etc/systemd/system/bugpk-api.service`:

```ini
[Unit]
Description=BugPk-Api backend
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=bugpk
Group=bugpk
WorkingDirectory=/www/wwwroot/bugpk_app
EnvironmentFile=/www/wwwroot/bugpk_app/.env
ExecStart=/www/wwwroot/bugpk_app/bugpk-api
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```bash
sudo useradd -r -s /sbin/nologin bugpk
sudo chown -R bugpk:bugpk /www/wwwroot/bugpk_app
sudo systemctl daemon-reload
sudo systemctl enable --now bugpk-api
sudo journalctl -u bugpk-api -f
```

## 10. Nginx (same origin, frontend served separately)

Choose one of the two modes:

- **Mode A: Go serves the frontend**: `FRONTEND_SERVE_ENABLED=true`; Nginx reverse-proxies the entire site to `127.0.0.1:8080`.
- **Mode B: Nginx serves the frontend**: `FRONTEND_SERVE_ENABLED=false`; Nginx serves the static files and only reverse-proxies `/system-api`, `/api`, and `/open-api` to the backend.

Mode B is used as the example below:

```nginx
upstream bugpk_api_backend {
    server 127.0.0.1:8080;
    keepalive 64;
}

server {
    listen 80;
    server_name api.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    http2 on;                      # on nginx < 1.25.1 use listen 443 ssl http2;
    server_name api.example.com;

    root  /www/wwwroot/bugpk_app/frontend/dist;
    index index.html;

    ssl_certificate     /etc/nginx/ssl/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/api.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    add_header Strict-Transport-Security "max-age=31536000" always;

    client_max_body_size 16m;

    location = /system-api {
        proxy_pass http://bugpk_api_backend;
        include /etc/nginx/snippets/bugpk-proxy.conf;
    }
    location ^~ /system-api/ {
        proxy_pass http://bugpk_api_backend;
        include /etc/nginx/snippets/bugpk-proxy.conf;
    }
    location = /api {
        proxy_pass http://bugpk_api_backend;
        include /etc/nginx/snippets/bugpk-proxy.conf;
    }
    location ^~ /api/ {
        proxy_pass http://bugpk_api_backend;
        include /etc/nginx/snippets/bugpk-proxy.conf;
    }
    location = /open-api {
        proxy_pass http://bugpk_api_backend;
        include /etc/nginx/snippets/bugpk-proxy.conf;
    }
    location ^~ /open-api/ {
        proxy_pass http://bugpk_api_backend;
        include /etc/nginx/snippets/bugpk-proxy.conf;
    }

    location ~* \.(js|css|woff2?|ttf|svg|map)$ {
        expires 12h;
        add_header Cache-Control "public, max-age=43200";
    }
    location ~* \.(gif|jpg|jpeg|png|webp|ico)$ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000";
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

`/etc/nginx/snippets/bugpk-proxy.conf`:

```nginx
proxy_http_version 1.1;
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header Connection        "";   # do not send upgrade for regular APIs, otherwise keep-alive breaks
proxy_connect_timeout 20s;
proxy_read_timeout    300s;
proxy_send_timeout    60s;
```

Key points:

- `proxy_pass` has no trailing slash, preserving the `/system-api`, `/api/{apiCode}`, and `/open-api` prefixes.
- `/api/{apiCode}` must use `^~`, otherwise it will be taken over by static assets or the SPA fallback.
- If WebSocket is added later, add a separate `location ^~ /ws/` and set `Upgrade` / `Connection "upgrade"`.

If `/healthz` and `/readyz` need to be probed from outside, allow them separately or leave them to the load balancer on the internal network.

The repository's `deploy/宝塔Nginx配置.conf` provides a complete version for the BT Panel (BaoTa) for comparison.

## 11. Upgrading

```bash
# 1. Back up
mysqldump --single-transaction -u bugpk -p bugpk_api > bugpk_api_$(date +%F).sql

# 2. Replace the binary, keep migrations/
systemctl stop bugpk-api

# 3. Option A: let the application run migrations
sed -i 's/^MIGRATION_ENABLED=.*/MIGRATION_ENABLED=true/' .env
systemctl start bugpk-api
# after confirming the log shows migrations completed, set it back to false and restart again

# Option B: run the new migrations manually
mysql --default-character-set=utf8mb4 -u bugpk -p bugpk_api < migrations/0000xx_xxx.up.sql
```

`sql/full` is only for fresh installations; do not use it to upgrade an existing database.

## 12. Acceptance checklist

- [ ] `curl /healthz` and `/readyz` both return success
- [ ] The frontend opens in a browser via the domain, and refreshing a sub-route does not return 404
- [ ] `admin` can log in, and the menu renders according to the role
- [ ] The admin console can save site configuration (verifies `SYSTEM_CONFIG_SECRET`)
- [ ] After configuring a test API, `/api/{apiCode}` forwards correctly
- [ ] The client IP shows the real IP (verifies `SERVER_TRUSTED_PROXIES`)
- [ ] The service still starts normally after `INITIAL_ADMIN_PASSWORD` is removed from `.env`

## 13. Common startup failures

| Symptom | Cause and resolution |
| --- | --- |
| `database civil-time baseline is not confirmed` | `DB_TIMEZONE_DATA_CONFIRMATION` is missing or does not match the current database; recompute it |
| `MySQL session time_zone is "+00:00"` | Middleware or `init_connect` rewrote the session time zone; it must stay at `+08:00` |
| `full SQL ... requires an empty database` | The target database already has tables; switch to an empty database or first confirm the data can be cleared |
| `JWT_COOKIE_SECURE must be true in production` | Production enforces HTTPS cookies; set `JWT_COOKIE_SECURE=true` and enable TLS |
| `JWT_SECRET must be at least 32 bytes` | Generate a random string of at least 32 bytes |
| `PAYMENT_CONFIG_SECRET must be at least 32 bytes` | Provide the encryption keys for payment and system configuration |
| Logged out immediately after login | Check whether the backend is accessed cross-origin, and whether `JWT_COOKIE_SECURE` matches the site protocol |
| Client IP is always `127.0.0.1` | `SERVER_TRUSTED_PROXIES` is not set, or Nginx does not pass `X-Forwarded-For` |