# 部署指南

适用于单机或小规模生产部署：Linux + MySQL 8 + Redis 6+ + Nginx，前后端同域。

## 1. 前置条件

- Linux x86_64 或 arm64，具备 systemd
- MySQL `8.0+`，字符集 `utf8mb4`
- Redis `6+`
- Nginx（可选，但生产建议用于 TLS 与静态资源）
- 已解析到服务器的域名与证书

## 2. 准备数据库

为应用创建独立账号，不要使用 `root`：

```sql
CREATE DATABASE bugpk_api DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'bugpk'@'127.0.0.1' IDENTIFIED BY '<强密码>';
GRANT ALL PRIVILEGES ON bugpk_api.* TO 'bugpk'@'127.0.0.1';
FLUSH PRIVILEGES;
```

初始化 SQL 带空库守卫：目标库非空时脚本会主动报错中止。**不要**把它导入已有业务数据的库。

## 3. 目录布局

生产建议平铺部署，二进制、配置、迁移与前端产物在同一目录：

```text
/www/wwwroot/bugpk_app/
├── bugpk-api              # 后端二进制
├── config.yaml            # 非敏感配置
├── .env                   # 密钥与环境变量（权限 600）
├── migrations/            # 迁移 SQL（升级时使用）
├── sql/full/              # 全量初始化 SQL（仅新装）
└── frontend/dist/         # 前端构建产物
```

## 4. 构建后端

```bash
git clone https://github.com/jiuhunwl/bugpkapi-new-go.git
cd bugpkapi-new-go

make build-linux-amd64    # 产物：bugpk-api-linux-amd64
make build-linux-arm64    # ARM 服务器用这个
```

把二进制改名为 `bugpk-api` 并上传到部署目录，同时上传 `migrations/` 与 `sql/`。

## 5. 初始化前端

在构建机（Node.js `^22.18.0 || ^24.0.0`，pnpm `>= 11`）：

```bash
git clone https://github.com/jiuhunwl/bugpkapi-new-vue.git
cd bugpkapi-new-vue
pnpm install
pnpm build                # 产物：playground/dist
```

同域部署时保持 `VITE_GLOB_API_URL=/system-api`（默认值），不要改成后端绝对地址，否则会引入跨域与刷新令牌 Cookie 问题。把 `playground/dist` 上传到 `frontend/dist`。

## 6. 配置

```bash
cp configs/config.yaml.example /www/wwwroot/bugpk_app/config.yaml
cp deploy/.env.example /www/wwwroot/bugpk_app/.env
chmod 600 /www/wwwroot/bugpk_app/.env
```

`.env` 至少补齐：

```dotenv
APP_ENV=production
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=bugpk_api
DB_USER=bugpk
DB_PASSWORD=<强密码>
DB_TIMEZONE_DATA_CONFIRMATION=mysql-civil-v1:<按配置参考计算>

JWT_SECRET=<openssl rand -hex 32>
JWT_COOKIE_SECURE=true
INITIAL_ADMIN_PASSWORD=<8-64 位强口令>
PAYMENT_CONFIG_SECRET=<openssl rand -hex 32>
SYSTEM_CONFIG_SECRET=<openssl rand -hex 32>

FRONTEND_SERVE_ENABLED=true
FRONTEND_DIST_PATH=frontend/dist
MIGRATION_ENABLED=false
SERVER_TRUSTED_PROXIES=127.0.0.1
```

`DB_TIMEZONE_DATA_CONFIRMATION` 的计算方法见 [配置参考](./configuration.md#数据库时区确认dbtimezone_data_confirmation)。这一步不能跳过，新库也必须确认。

## 7. 导入全量 SQL

```bash
mysql --default-character-set=utf8mb4 -u bugpk -p bugpk_api \
  < /www/wwwroot/bugpk_app/sql/full/bugpk_api_full_v1.2.113.sql
```

导入后确认 `MIGRATION_ENABLED=false`，避免应用启动时重复执行迁移。

## 8. 首次启动与管理员

```bash
cd /www/wwwroot/bugpk_app
set -a; source .env; set +a
./bugpk-api
```

首次启动会用 `INITIAL_ADMIN_PASSWORD` 创建账号 `admin`。登录成功后建议：

1. 立即修改密码并开启两步验证；
2. 从 `.env` 中移除 `INITIAL_ADMIN_PASSWORD`；
3. 重启服务确认仍能正常启动。

健康检查：

```bash
curl -fsS http://127.0.0.1:8080/healthz
curl -fsS http://127.0.0.1:8080/readyz
```

## 9. systemd

`/etc/systemd/system/bugpk-api.service`：

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

## 10. Nginx（同域，前端独立托管）

两种模式任选其一：

- **模式 A：Go 托管前端**：`FRONTEND_SERVE_ENABLED=true`，Nginx 把整个站点反代到 `127.0.0.1:8080`。
- **模式 B：Nginx 托管前端**：`FRONTEND_SERVE_ENABLED=false`，Nginx 提供静态文件，只把 `/system-api`、`/api`、`/open-api` 反代到后端。

下面以模式 B 为例：

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
    http2 on;                      # nginx < 1.25.1 改为 listen 443 ssl http2;
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

`/etc/nginx/snippets/bugpk-proxy.conf`：

```nginx
proxy_http_version 1.1;
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header Connection        "";   # 普通 API 不要带 upgrade，否则 keep-alive 失效
proxy_connect_timeout 20s;
proxy_read_timeout    300s;
proxy_send_timeout    60s;
```

要点：

- `proxy_pass` 不带尾斜杠，保留 `/system-api`、`/api/{apiCode}`、`/open-api` 前缀。
- `/api/{apiCode}` 必须用 `^~`，否则会被静态资源或 SPA 回退接管。
- 后续若增加 WebSocket，单独加 `location ^~ /ws/` 并设置 `Upgrade` / `Connection "upgrade"`。

`/healthz`、`/readyz` 若需从外部探活，可另外放行或交给负载均衡内网访问。

仓库 `deploy/宝塔Nginx配置.conf` 提供了宝塔面板的完整版本，可作为对照。

## 11. 升级

```bash
# 1. 备份
mysqldump --single-transaction -u bugpk -p bugpk_api > bugpk_api_$(date +%F).sql

# 2. 替换二进制，保留 migrations/
systemctl stop bugpk-api

# 3. 方案 A：让程序执行迁移
sed -i 's/^MIGRATION_ENABLED=.*/MIGRATION_ENABLED=true/' .env
systemctl start bugpk-api
# 确认日志出现迁移完成后，改回 false 并再次重启

# 方案 B：手工执行新增迁移
mysql --default-character-set=utf8mb4 -u bugpk -p bugpk_api < migrations/0000xx_xxx.up.sql
```

`sql/full` 只用于全新安装，不要用于升级已有库。

## 12. 验收清单

- [ ] `curl /healthz` 与 `/readyz` 均返回成功
- [ ] 浏览器访问域名可打开前端，刷新子路由不 404
- [ ] `admin` 可登录，菜单按角色渲染
- [ ] 管理端能保存站点配置（验证 `SYSTEM_CONFIG_SECRET`）
- [ ] 配置一条测试接口后，`/api/{apiCode}` 可正常转发
- [ ] 客户端 IP 显示为真实 IP（验证 `SERVER_TRUSTED_PROXIES`）
- [ ] 关闭 `.env` 中 `INITIAL_ADMIN_PASSWORD` 后仍能正常启动

## 13. 常见启动失败

| 现象 | 原因与处理 |
| --- | --- |
| `database civil-time baseline is not confirmed` | `DB_TIMEZONE_DATA_CONFIRMATION` 缺失或与当前库不匹配，重新计算 |
| `MySQL session time_zone is "+00:00"` | 中间件或 `init_connect` 改写了会话时区，需保持 `+08:00` |
| `full SQL ... requires an empty database` | 目标库已有表，换空库或先确认数据可清理 |
| `JWT_COOKIE_SECURE must be true in production` | 生产强制 HTTPS Cookie，设置 `JWT_COOKIE_SECURE=true` 并启用 TLS |
| `JWT_SECRET must be at least 32 bytes` | 生成 32 字节以上随机串 |
| `PAYMENT_CONFIG_SECRET must be at least 32 bytes` | 补齐支付与系统配置加密密钥 |
| 登录后立刻掉线 | 检查是否跨域访问后端、`JWT_COOKIE_SECURE` 与站点协议是否一致 |
| 客户端 IP 全是 `127.0.0.1` | 未设置 `SERVER_TRUSTED_PROXIES`，或 Nginx 未传 `X-Forwarded-For` |
