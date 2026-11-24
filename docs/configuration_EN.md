# Configuration Reference

[简体中文](configuration.md) · **English**

The backend configuration is merged from three layers, with later layers overriding earlier ones:

1. Built-in code defaults (`internal/config`)
2. The YAML configuration file (default `config.yaml`)
3. Environment variables (including `.env` in the same directory; existing process environment variables are not overridden by `.env`)

## Configuration File Location

Resolved in the following order; `CONFIG_FILE` can be used to force a specific file:

1. The file pointed to by the `CONFIG_FILE` environment variable
2. `config.yaml` in the same directory as the executable (flat production deployment)
3. `configs/config.yaml` in the current working directory (local `go run` development)

`configs/config.yaml.example` is a complete, commented template; copy it to `config.yaml` and modify as needed. The binary automatically loads `.env` in the working directory on startup, so for production it is recommended to **put non-sensitive items in YAML and secrets in `.env`**.

> Never commit `.env`, `config.yaml`, or certificate private keys to the repository.

## Production Minimum Configuration

When `APP_ENV=production` (`staging` is validated in the same way as production), the following items being missing or non-compliant causes startup to be refused outright:

| Configuration | Requirement |
| --- | --- |
| `DB_USER` / `DB_PASSWORD` | Must not use `root`; the password must not be empty |
| `DB_TIMEZONE_DATA_CONFIRMATION` | Must equal the confirmation value for this database; see the next section |
| `JWT_SECRET` | ≥ 32 bytes, and must not be the built-in development placeholder |
| `JWT_COOKIE_SECURE` | Must be `true` |
| `JWT_COOKIE_SAME_SITE` | `Lax` or `Strict` (`None` is rejected in production) |
| `PAYMENT_CONFIG_SECRET` | ≥ 32 bytes |
| `SYSTEM_CONFIG_SECRET` | ≥ 32 bytes (falls back to `PAYMENT_CONFIG_SECRET` when empty) |
| `INITIAL_ADMIN_PASSWORD` | Required when creating or rotating the first administrator |
| `SERVER_TRUSTED_PROXIES` | When behind a reverse proxy, the proxy addresses must be set; otherwise all client IPs will be the proxy IP |

Generate a secret:

```bash
openssl rand -hex 32
```

## Database Timezone Confirmation (`DB_TIMEZONE_DATA_CONFIRMATION`)

The backend forces `time_zone='+08:00'` in the connection string and validates it before startup, migration, writing initial data, and registering routes:

- `@@session.time_zone` must equal `+08:00`;
- `DB_TIMEZONE_DATA_CONFIRMATION` must equal the confirmation value computed from the **target database identity**.

Confirmation value algorithm:

```text
"mysql-civil-v1:" + sha256( "bugpk/mysql-civil/v1" + "\0" + @@server_uuid + "\0" + database() + "\0" + "+08:00" ).hex()
```

It binds "this MySQL instance + this database" to the convention that "business dates are interpreted as UTC+08:00", preventing a confirmation value from the configuration from being copied to another database. **A newly created database also requires confirmation**; it is not required only when migrating existing data.

Calculation example (start with a read-only query for `@@server_uuid`):

```bash
SERVER_UUID=$(mysql -N -B -u bugpk -p -e "select @@server_uuid")
DB_NAME=bugpk_api

printf 'bugpk/mysql-civil/v1\000%s\000%s\000+08:00' "$SERVER_UUID" "$DB_NAME" \
  | sha256sum | awk '{print "mysql-civil-v1:" $1}'
```

Windows PowerShell:

```powershell
$serverUuid = "<result of select @@server_uuid>"
$dbName     = "bugpk_api"
$text       = "bugpk/mysql-civil/v1`0$serverUuid`0$dbName`0+08:00"
$bytes      = [Text.Encoding]::UTF8.GetBytes($text)
$hash       = [Security.Cryptography.SHA256]::Create().ComputeHash($bytes)
"mysql-civil-v1:" + ([BitConverter]::ToString($hash) -replace '-', '').ToLower()
```

Node.js:

```bash
node -e 'const c=require("crypto");const [u,d]=process.argv.slice(1);
console.log("mysql-civil-v1:"+c.createHash("sha256")
  .update(["bugpk/mysql-civil/v1",u,d,"+08:00"].join("\0")).digest("hex"))' \
  "$SERVER_UUID" "$DB_NAME"
```

Write the result into the environment variable:

```bash
DB_TIMEZONE_DATA_CONFIRMATION=mysql-civil-v1:<64-digit hexadecimal>
```

Notes:

- This confirmation does **not** convert any existing data; before switching, all old processes that write in UTC must be stopped.
- After changing the database (migrating to a new instance, renaming, or recreating the database), the confirmation value changes and must be recomputed.
- Explicitly setting it to an empty string revokes the confirmation, and the service then immediately fails to start.

## Variable Groups

### Application and Server

| Variable | Default | Description |
| --- | --- | --- |
| `APP_ENV` | `development` | `development` / `test` / `staging` / `production`; accepts the aliases `dev`/`test`/`stage`/`prod` |
| `SERVER_HOST` | `0.0.0.0` | Listen address |
| `SERVER_PORT` | `8080` | Listen port |
| `SERVER_READ_TIMEOUT_SECONDS` | `10` | Read timeout |
| `SERVER_WRITE_TIMEOUT_SECONDS` | `30` | Write timeout |
| `SERVER_IDLE_TIMEOUT_SECONDS` | `120` | Idle connection timeout |
| `SERVER_SHUTDOWN_TIMEOUT_SECONDS` | `30` | Graceful shutdown wait time |
| `SERVER_MAX_REQUEST_BODY_MB` | `4` | Maximum request body (MiB); 3–64 allowed |
| `SERVER_TRUSTED_PROXIES` | empty | Comma-separated proxy IPs/CIDRs; when empty, `X-Forwarded-For` is ignored and the TCP peer address is used |
| `SYSTEM_API_PREFIX` | `/system-api` | System API prefix; must not be `/` and must not fall under `/api/*` |
| `CORS_ALLOWED_ORIGINS` | empty | Comma-separated full Origins; empty means same-origin only and CORS is not registered |

### Frontend Hosting

| Variable | Default | Description |
| --- | --- | --- |
| `FRONTEND_SERVE_ENABLED` | `true` | Serve the frontend build output directly from the Go service |
| `FRONTEND_DIST_PATH` | `frontend/dist` | Frontend build output directory |

Set to `false` when using a same-origin reverse proxy plus a separate static site.

### Database

| Variable | Default | Description |
| --- | --- | --- |
| `DB_HOST` / `DB_PORT` | `localhost` / `3306` | MySQL address |
| `DB_NAME` | `bugpk_api` | Database name |
| `DB_USER` / `DB_PASSWORD` | — | Account; root/empty password is forbidden in production |
| `DB_SSL` | `false` | Whether to enable TLS |
| `DB_TIMEZONE_DATA_CONFIRMATION` | empty | See the previous section; required |
| `DB_MAX_OPEN_CONNS` | `100` | 1–2000 |
| `DB_MAX_IDLE_CONNS` | `50` | Must not exceed the maximum connection count |
| `DB_CONN_MAX_LIFETIME_MINUTES` | `30` | Maximum connection lifetime |
| `DB_CONN_MAX_IDLE_MINUTES` | `5` | Idle connection recycling |
| `DB_DIAL_TIMEOUT_SECONDS` | `5` | Connection establishment timeout |
| `DB_READ_TIMEOUT_SECONDS` | `15` | Read timeout |
| `DB_WRITE_TIMEOUT_SECONDS` | `15` | Write timeout |

### Migration

| Variable | Default | Description |
| --- | --- | --- |
| `MIGRATION_ENABLED` | `true` | Automatically run `migrations/` on startup |
| `MIGRATIONS_PATH` | `migrations` | Migration directory |

After manually importing `sql/full`, it must be set to `false`; otherwise the table creation statements will be executed again.

### Authentication and Sessions

| Variable | Default | Description |
| --- | --- | --- |
| `JWT_ISSUER` | `BugPk-Api` | Issuer |
| `JWT_SECRET` | development placeholder | ≥ 32 bytes; changing it invalidates all sessions |
| `JWT_ACCESS_TOKEN_MINUTES` | `30` | Access token validity (minutes) |
| `JWT_REFRESH_TOKEN_DAYS` | `7` | Refresh token validity (days) |
| `JWT_REFRESH_ROTATION_GRACE_SECONDS` | `60` | Grace period for concurrent refresh across multiple tabs; 0–300 |
| `JWT_REFRESH_COOKIE_NAME` | `REFRESH_TOKEN` | Refresh token cookie name |
| `JWT_COOKIE_SECURE` | `false` | Must be `true` in production |
| `JWT_COOKIE_SAME_SITE` | `Lax` | `Lax` / `Strict` / `None` (`None` requires Secure to be enabled and is not supported in production) |
| `TWOFA_CHALLENGE_SECONDS` | `300` | 2FA challenge token validity |
| `TWOFA_PENDING_SECONDS` | `600` | Validity of the temporarily stored secret during 2FA binding |
| `PASSWORD_RESET_ACCOUNT_DAY_LIMIT` | `5` | Daily limit of password reset emails per account |
| `INITIAL_ADMIN_PASSWORD` | empty | Password for the first administrator; see the next section |

Rules for `INITIAL_ADMIN_PASSWORD`:

- Required when creating an administrator, or when an account is detected to still use a historical seed hash / a publicly known default password;
- 8–64 characters and no more than 72 bytes;
- Must not reuse publicly known default passwords or placeholder text;
- Accounts that already have a custom password are not overwritten; the variable can be removed after successful initialization.

### Redis

| Variable | Default | Description |
| --- | --- | --- |
| `REDIS_HOST` / `REDIS_PORT` | `localhost` / `6379` | Caching, rate limiting, CAPTCHA, and session assistance |
| `REDIS_PASSWORD` | empty | Password |
| `REDIS_DATABASE` | `0` | 0–1023 |
| `REDIS_POOL_SIZE` | `256` | 1–5000 |
| `REDIS_MIN_IDLE_CONNS` | `32` | Must not exceed the pool size |
| `REDIS_MAX_RETRIES` | `2` | -1–10 |
| `REDIS_POOL_TIMEOUT_MS` | `2000` | 100–60000 |

### Analytics and Risk Control Aggregation

| Variable | Default | Description |
| --- | --- | --- |
| `ANALYTICS_ENABLED` | `true` | Incremental call log aggregation |
| `ANALYTICS_BATCH_SIZE` | `5000` | Rows processed per batch (100–20000) |
| `ANALYTICS_POLL_INTERVAL_MS` | `1000` | Incremental polling interval |
| `ANALYTICS_OVERVIEW_REFRESH_INTERVAL_MINUTES` | `5` | Overview snapshot refresh |
| `ANALYTICS_OVERVIEW_SNAPSHOT_WINDOW_MINUTES` | `60` | Snapshot window |
| `ANALYTICS_IP_HASH_SECRET` | empty | Secret for de-identifying IPs; once set it cannot be removed, otherwise aggregation grouping keys will drift |

### Encryption Secrets

| Variable | Description |
| --- | --- |
| `PAYMENT_CONFIG_SECRET` | Encrypts configurations such as payment channel private keys; ≥ 32 bytes in production |
| `SYSTEM_CONFIG_SECRET` | Encrypts system configurations such as the SMTP authorization code and SMS secrets; falls back to the payment secret when empty |

After either of these is changed, the fields already encrypted in the database cannot be decrypted with the new secret, and the corresponding configurations must be re-entered. **Backing up the secrets is as important as backing up the database.**

### Email and SMS

| Variable | Description |
| --- | --- |
| `EMAIL_PROVIDER` | Defaults to `smtp` |
| `EMAIL_ENABLED` | Whether email is enabled |
| `SMTP_HOST` / `SMTP_PORT` | SMTP service |
| `SMTP_USERNAME` / `SMTP_PASSWORD` | SMTP credentials |
| `SMTP_FROM` / `SMTP_FROM_NAME` | Sender address and display name |
| `SMTP_TLS` | Whether to enable TLS |
| `EMAIL_CODE_SUBJECT` / `EMAIL_CODE_TEMPLATE` | Default (Chinese) verification code template |
| `EMAIL_CODE_SUBJECT_EN` / `EMAIL_CODE_TEMPLATE_EN` | English verification code template; falls back to the default when empty |
| `EMAIL_RESET_SUBJECT` / `EMAIL_RESET_TEMPLATE` | Default password reset template |
| `EMAIL_RESET_SUBJECT_EN` / `EMAIL_RESET_TEMPLATE_EN` | English password reset template |
| `EMAIL_SEND_GLOBAL_MINUTE_LIMIT` / `EMAIL_SEND_GLOBAL_HOUR_LIMIT` | Global sending circuit breaker; 0 uses the built-in 30/minute and 300/hour |
| `SMS_PROVIDER` / `SMS_ENABLED` | SMS channel |

Template placeholders: `{{code}}`, `{{minutes}}`, `{{siteName}}`.

### Gateway Egress and IP Region

| Variable | Default | Description |
| --- | --- | --- |
| `UPSTREAM_ALLOW_PRIVATE_NETWORKS` | `false` | Enable only when the upstreams are truly on a trusted private network; otherwise this poses an SSRF risk |
| `PROXY_TEST_URL` | `https://api.ipify.org?format=json` | Egress proxy validation target; only public HTTP(S) is allowed |
| `IP_REGION_ENABLED` | `true` | Whether to resolve the caller's IP location |
| `IP_REGION_PROVIDER` | `ip-api` | `ip-api` / `bilibili` / `tencent-map` / `amap` / `ip2region` |
| `IP_REGION_TIMEOUT_MS` | `5000` | 100–30000 |
| `IP_REGION_CACHE_TTL_MINUTES` | `1440` | 0–43200 |
| `IP2REGION_V4_PATH` / `IP2REGION_V6_PATH` | `data/ip2region_v*.xdb` | Local database path |
| `IP_API_ENDPOINT` / `BILIBILI_IP_REGION_ENDPOINT` | public API | Can be replaced with a self-hosted source |
| `TENCENT_MAP_KEY` / `AMAP_KEY` | empty | Required when using the corresponding provider |

### Other

| Variable | Description |
| --- | --- |
| `OAUTH_ALLOW_PRIVATE_ENDPOINTS` | Allows third-party login to access private networks only for local integration testing; must be `false` in production |
| `OAUTH_HTTP_TIMEOUT_SECONDS` | Outbound timeout for third-party login; 3–60 |
| `CAPTCHA_PROVIDER` | Default CAPTCHA provider when no scenario configuration exists |
| `CAPTCHA_POW_DIFFICULTY` | Proof-of-work difficulty |
| `SEO_SITE_URL` | Public site address, used for absolute URLs in the sitemap; falls back to the request Host when empty |

## Troubleshooting

When startup fails, locate the cause by the exact error message. Common causes:

| Error keyword | Resolution |
| --- | --- |
| `database civil-time baseline is not confirmed` | Recompute and set `DB_TIMEZONE_DATA_CONFIRMATION` as described above |
| `MySQL session time_zone is ...` | The time zone was rewritten by an intermediary; check the proxy or `init_connect` |
| `JWT_SECRET must be at least 32 bytes` | Regenerate with `openssl rand -hex 32` |
| `JWT_COOKIE_SECURE must be true in production` | Production must use HTTPS and set `JWT_COOKIE_SECURE=true` |
| `INITIAL_ADMIN_PASSWORD is required ...` | This variable must be provided on first startup or when rotating a historical default password |
| `production database credentials must not use root/root` | Create a dedicated database account for the application |
| `PAYMENT_CONFIG_SECRET must be at least 32 bytes in production` | Complete both configuration encryption secrets |