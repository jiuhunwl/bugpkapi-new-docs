# 配置参考

**简体中文** · [English](configuration_EN.md)

后端配置由三层合并而成，后者覆盖前者：

1. 代码内置默认值（`internal/config`）
2. YAML 配置文件（默认 `config.yaml`）
3. 环境变量（含同目录 `.env`，已存在的进程环境变量不会被 `.env` 覆盖）

## 配置文件位置

按以下顺序解析，可用 `CONFIG_FILE` 强制指定：

1. `CONFIG_FILE` 环境变量指向的文件
2. 可执行文件同目录的 `config.yaml`（生产平铺部署）
3. 当前工作目录的 `configs/config.yaml`（本地 `go run` 开发）

`configs/config.yaml.example` 是完整带注释的模板，复制为 `config.yaml` 后按需修改。二进制启动时会自动加载工作目录下的 `.env`，因此生产环境推荐 **YAML 放非敏感项、`.env` 放密钥**。

> `.env`、`config.yaml`、证书私钥一律不要提交到仓库。

## 生产环境最小配置

`APP_ENV=production`（`staging` 同样按生产校验）时，以下项目缺失或不合规会直接拒绝启动：

| 配置 | 要求 |
| --- | --- |
| `DB_USER` / `DB_PASSWORD` | 不得使用 `root`，密码不得为空 |
| `DB_TIMEZONE_DATA_CONFIRMATION` | 必须等于本库的确认值，见下节 |
| `JWT_SECRET` | ≥ 32 字节，且不能是内置开发占位值 |
| `JWT_COOKIE_SECURE` | 必须为 `true` |
| `JWT_COOKIE_SAME_SITE` | `Lax` 或 `Strict`（生产拒绝 `None`） |
| `PAYMENT_CONFIG_SECRET` | ≥ 32 字节 |
| `SYSTEM_CONFIG_SECRET` | ≥ 32 字节（留空时回退使用 `PAYMENT_CONFIG_SECRET`） |
| `INITIAL_ADMIN_PASSWORD` | 创建或轮换首个管理员时必须提供 |
| `SERVER_TRUSTED_PROXIES` | 前面有反向代理时必须填代理地址，否则客户端 IP 全部是代理 IP |

生成密钥：

```bash
openssl rand -hex 32
```

## 数据库时区确认（`DB_TIMEZONE_DATA_CONFIRMATION`）

后端在连接串中强制 `time_zone='+08:00'`，并在启动、迁移、写入初始数据、注册路由之前校验：

- `@@session.time_zone` 必须等于 `+08:00`；
- `DB_TIMEZONE_DATA_CONFIRMATION` 必须等于由 **目标库身份** 计算出的确认值。

确认值算法：

```text
"mysql-civil-v1:" + sha256( "bugpk/mysql-civil/v1" + "\0" + @@server_uuid + "\0" + database() + "\0" + "+08:00" ).hex()
```

它把「这台 MySQL 实例 + 这个库」绑定到「业务日期按 UTC+08:00 解释」这一约定上，防止把配置里的确认值复制到另一个库。**新建库同样需要确认**，不是只在存量数据迁移时要求。

计算示例（先只读查询 `@@server_uuid`）：

```bash
SERVER_UUID=$(mysql -N -B -u bugpk -p -e "select @@server_uuid")
DB_NAME=bugpk_api

printf 'bugpk/mysql-civil/v1\000%s\000%s\000+08:00' "$SERVER_UUID" "$DB_NAME" \
  | sha256sum | awk '{print "mysql-civil-v1:" $1}'
```

Windows PowerShell：

```powershell
$serverUuid = "<select @@server_uuid 的结果>"
$dbName     = "bugpk_api"
$text       = "bugpk/mysql-civil/v1`0$serverUuid`0$dbName`0+08:00"
$bytes      = [Text.Encoding]::UTF8.GetBytes($text)
$hash       = [Security.Cryptography.SHA256]::Create().ComputeHash($bytes)
"mysql-civil-v1:" + ([BitConverter]::ToString($hash) -replace '-', '').ToLower()
```

Node.js：

```bash
node -e 'const c=require("crypto");const [u,d]=process.argv.slice(1);
console.log("mysql-civil-v1:"+c.createHash("sha256")
  .update(["bugpk/mysql-civil/v1",u,d,"+08:00"].join("\0")).digest("hex"))' \
  "$SERVER_UUID" "$DB_NAME"
```

把结果写入环境变量：

```bash
DB_TIMEZONE_DATA_CONFIRMATION=mysql-civil-v1:<64 位十六进制>
```

注意：

- 该确认**不会**转换任何已有数据；切换前必须停掉所有按 UTC 写入的旧进程。
- 更换数据库（迁移到新实例、改名、重建库）后确认值会变，需要重新计算。
- 显式设置为空字符串会撤销确认，服务随即启动失败。

## 变量分组

### 应用与服务器

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `APP_ENV` | `development` | `development` / `test` / `staging` / `production`，接受 `dev`/`test`/`stage`/`prod` 别名 |
| `SERVER_HOST` | `0.0.0.0` | 监听地址 |
| `SERVER_PORT` | `8080` | 监听端口 |
| `SERVER_READ_TIMEOUT_SECONDS` | `10` | 读超时 |
| `SERVER_WRITE_TIMEOUT_SECONDS` | `30` | 写超时 |
| `SERVER_IDLE_TIMEOUT_SECONDS` | `120` | 空闲连接超时 |
| `SERVER_SHUTDOWN_TIMEOUT_SECONDS` | `30` | 优雅退出等待时间 |
| `SERVER_MAX_REQUEST_BODY_MB` | `4` | 请求体上限（MiB），允许 3–64 |
| `SERVER_TRUSTED_PROXIES` | 空 | 逗号分隔的代理 IP/CIDR；为空时忽略 `X-Forwarded-For`，使用 TCP 对端地址 |
| `SYSTEM_API_PREFIX` | `/system-api` | 系统接口前缀，不得为 `/`，不得落在 `/api/*` 下 |
| `CORS_ALLOWED_ORIGINS` | 空 | 逗号分隔的完整 Origin；留空表示仅同域，不注册 CORS |

### 前端托管

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `FRONTEND_SERVE_ENABLED` | `true` | 由 Go 服务直接托管前端产物 |
| `FRONTEND_DIST_PATH` | `frontend/dist` | 前端产物目录 |

同域反向代理 + 独立静态站点时设为 `false`。

### 数据库

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `DB_HOST` / `DB_PORT` | `localhost` / `3306` | MySQL 地址 |
| `DB_NAME` | `bugpk_api` | 库名 |
| `DB_USER` / `DB_PASSWORD` | — | 账号，生产禁止 root/空密码 |
| `DB_SSL` | `false` | 是否启用 TLS |
| `DB_TIMEZONE_DATA_CONFIRMATION` | 空 | 见上节，必填 |
| `DB_MAX_OPEN_CONNS` | `100` | 1–2000 |
| `DB_MAX_IDLE_CONNS` | `50` | 不得超过最大连接数 |
| `DB_CONN_MAX_LIFETIME_MINUTES` | `30` | 连接最长存活 |
| `DB_CONN_MAX_IDLE_MINUTES` | `5` | 空闲连接回收 |
| `DB_DIAL_TIMEOUT_SECONDS` | `5` | 建连超时 |
| `DB_READ_TIMEOUT_SECONDS` | `15` | 读超时 |
| `DB_WRITE_TIMEOUT_SECONDS` | `15` | 写超时 |

### 迁移

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `MIGRATION_ENABLED` | `true` | 启动时自动执行 `migrations/` |
| `MIGRATIONS_PATH` | `migrations` | 迁移目录 |

手工导入 `sql/full` 后必须设为 `false`，否则会重复执行建表语句。

### 认证与会话

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `JWT_ISSUER` | `BugPk-Api` | 签发者 |
| `JWT_SECRET` | 开发占位值 | ≥ 32 字节；更换后所有会话失效 |
| `JWT_ACCESS_TOKEN_MINUTES` | `30` | access token 有效期（分钟） |
| `JWT_REFRESH_TOKEN_DAYS` | `7` | refresh token 有效期（天） |
| `JWT_REFRESH_ROTATION_GRACE_SECONDS` | `60` | 多标签页并发续期宽限，0–300 |
| `JWT_REFRESH_COOKIE_NAME` | `REFRESH_TOKEN` | 刷新令牌 Cookie 名 |
| `JWT_COOKIE_SECURE` | `false` | 生产必须 `true` |
| `JWT_COOKIE_SAME_SITE` | `Lax` | `Lax` / `Strict` / `None`（`None` 需同时开启 Secure 且生产不支持） |
| `TWOFA_CHALLENGE_SECONDS` | `300` | 2FA 挑战 token 有效期 |
| `TWOFA_PENDING_SECONDS` | `600` | 2FA 绑定时密钥暂存有效期 |
| `PASSWORD_RESET_ACCOUNT_DAY_LIMIT` | `5` | 单账号每日找回密码邮件上限 |
| `INITIAL_ADMIN_PASSWORD` | 空 | 首个管理员口令，见下节 |

`INITIAL_ADMIN_PASSWORD` 规则：

- 创建管理员、或检测到账号仍是历史种子哈希 / 已公开默认口令时必填；
- 8–64 字符且不超过 72 字节；
- 不得复用已公开的默认口令与占位文案；
- 已有自定义密码的账号不会被覆盖，初始化成功后可以移除该变量。

### Redis

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `REDIS_HOST` / `REDIS_PORT` | `localhost` / `6379` | 缓存、限流、验证码与会话辅助 |
| `REDIS_PASSWORD` | 空 | 密码 |
| `REDIS_DATABASE` | `0` | 0–1023 |
| `REDIS_POOL_SIZE` | `256` | 1–5000 |
| `REDIS_MIN_IDLE_CONNS` | `32` | 不得超过池大小 |
| `REDIS_MAX_RETRIES` | `2` | -1–10 |
| `REDIS_POOL_TIMEOUT_MS` | `2000` | 100–60000 |

### 分析与风控汇总

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `ANALYTICS_ENABLED` | `true` | 调用日志增量汇总 |
| `ANALYTICS_BATCH_SIZE` | `5000` | 单批处理行数（100–20000） |
| `ANALYTICS_POLL_INTERVAL_MS` | `1000` | 增量轮询间隔 |
| `ANALYTICS_OVERVIEW_REFRESH_INTERVAL_MINUTES` | `5` | 概览快照刷新 |
| `ANALYTICS_OVERVIEW_SNAPSHOT_WINDOW_MINUTES` | `60` | 快照窗口 |
| `ANALYTICS_IP_HASH_SECRET` | 空 | IP 去标识化密钥；设置后不可再删除，否则汇总分组键漂移 |

### 加密密钥

| 变量 | 说明 |
| --- | --- |
| `PAYMENT_CONFIG_SECRET` | 加密支付渠道私钥等配置，生产 ≥ 32 字节 |
| `SYSTEM_CONFIG_SECRET` | 加密 SMTP 授权码、短信密钥等系统配置，留空回退支付密钥 |

两者变更后，数据库中已加密的字段无法用新密钥解密，需要重新录入对应配置。**备份密钥与备份数据库同等重要。**

### 邮件与短信

| 变量 | 说明 |
| --- | --- |
| `EMAIL_PROVIDER` | 默认 `smtp` |
| `EMAIL_ENABLED` | 是否启用邮件 |
| `SMTP_HOST` / `SMTP_PORT` | SMTP 服务 |
| `SMTP_USERNAME` / `SMTP_PASSWORD` | SMTP 凭据 |
| `SMTP_FROM` / `SMTP_FROM_NAME` | 发件人与显示名 |
| `SMTP_TLS` | 是否启用 TLS |
| `EMAIL_CODE_SUBJECT` / `EMAIL_CODE_TEMPLATE` | 默认（中文）验证码模板 |
| `EMAIL_CODE_SUBJECT_EN` / `EMAIL_CODE_TEMPLATE_EN` | 英文验证码模板，为空回退默认 |
| `EMAIL_RESET_SUBJECT` / `EMAIL_RESET_TEMPLATE` | 默认找回密码模板 |
| `EMAIL_RESET_SUBJECT_EN` / `EMAIL_RESET_TEMPLATE_EN` | 英文找回密码模板 |
| `EMAIL_SEND_GLOBAL_MINUTE_LIMIT` / `EMAIL_SEND_GLOBAL_HOUR_LIMIT` | 全站发信熔断，0 使用内置 30/分钟、300/小时 |
| `SMS_PROVIDER` / `SMS_ENABLED` | 短信渠道 |

模板占位符：`{{code}}`、`{{minutes}}`、`{{siteName}}`。

### 网关出站与 IP 地域

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `UPSTREAM_ALLOW_PRIVATE_NETWORKS` | `false` | 仅当上游确实位于可信内网时开启，否则构成 SSRF 风险 |
| `PROXY_TEST_URL` | `https://api.ipify.org?format=json` | 出口代理校验目标，仅允许公网 HTTP(S) |
| `IP_REGION_ENABLED` | `true` | 是否解析调用方 IP 归属地 |
| `IP_REGION_PROVIDER` | `ip-api` | `ip-api` / `bilibili` / `tencent-map` / `amap` / `ip2region` |
| `IP_REGION_TIMEOUT_MS` | `5000` | 100–30000 |
| `IP_REGION_CACHE_TTL_MINUTES` | `1440` | 0–43200 |
| `IP2REGION_V4_PATH` / `IP2REGION_V6_PATH` | `data/ip2region_v*.xdb` | 本地库路径 |
| `IP_API_ENDPOINT` / `BILIBILI_IP_REGION_ENDPOINT` | 公共接口 | 可替换为自建源 |
| `TENCENT_MAP_KEY` / `AMAP_KEY` | 空 | 使用对应 provider 时必填 |

### 其他

| 变量 | 说明 |
| --- | --- |
| `OAUTH_ALLOW_PRIVATE_ENDPOINTS` | 仅本地联调允许第三方登录访问私网，生产必须 `false` |
| `OAUTH_HTTP_TIMEOUT_SECONDS` | 第三方登录出站超时，3–60 |
| `CAPTCHA_PROVIDER` | 无场景配置时的默认验证码提供商 |
| `CAPTCHA_POW_DIFFICULTY` | 工作量证明难度 |
| `SEO_SITE_URL` | 站点对外地址，用于 sitemap 绝对 URL；留空回退请求 Host |

## 排错

启动失败时按错误原文定位，常见原因：

| 报错关键词 | 处理 |
| --- | --- |
| `database civil-time baseline is not confirmed` | 按上文重新计算并设置 `DB_TIMEZONE_DATA_CONFIRMATION` |
| `MySQL session time_zone is ...` | 连接被中间件改写了时区，检查代理或 `init_connect` |
| `JWT_SECRET must be at least 32 bytes` | 用 `openssl rand -hex 32` 重新生成 |
| `JWT_COOKIE_SECURE must be true in production` | 生产必须 HTTPS，并设置 `JWT_COOKIE_SECURE=true` |
| `INITIAL_ADMIN_PASSWORD is required ...` | 首次启动或轮换历史默认口令时必须提供该变量 |
| `production database credentials must not use root/root` | 为应用创建独立数据库账号 |
| `PAYMENT_CONFIG_SECRET must be at least 32 bytes in production` | 补齐两个配置加密密钥 |
