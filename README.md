# BugPk-Api 文档

BugPk-Api 是一套开源的**开放 API 服务平台**，包含后端网关与计费、管理后台、用户前台与配套文档。

| 仓库 | 说明 | 许可证 |
| --- | --- | --- |
| [bugpkapi-new-go](https://github.com/jiuhunwl/bugpkapi-new-go) | 后端服务：Go + Hertz + MySQL + Redis | AGPL-3.0 |
| [bugpkapi-new-vue](https://github.com/jiuhunwl/bugpkapi-new-vue) | 前端：Vue 3 + Vben Admin + Vite | MIT |
| **bugpkapi-new-docs**（本仓库） | 文档、架构说明与部署指南 | CC BY 4.0 |

## 从哪里开始

- 第一次接触项目：读 [架构概览](./docs/architecture.md)。
- 想把服务跑起来：读 [部署指南](./docs/deployment.md)。
- 要改配置或接支付：读 [配置参考](./docs/configuration.md)。
- 想参与开发：先看两个代码仓的 README、`CONTRIBUTING`（如有）与 `SECURITY.md`。

## 能力范围

| 模块 | 能力 |
| --- | --- |
| 开放 API 网关 | `/api/{apiCode}` 统一入口，按接口配置转发上游，参数校验、缓存与限流 |
| 接口与计费 | 接口管理、QPS 限制、套餐（全站 / 接口 / 点数）、按次计费、余额与流水 |
| 支付 | 支付宝、微信支付及可扩展渠道；回调幂等、订单对账、通知重放防护 |
| 用户与权限 | RBAC 角色 / 菜单 / 按钮权限、注册与找回密码、多场景验证码 |
| 风控与审计 | 调用日志指纹、风控快照、账号封禁、操作日志、IP 地域解析 |
| 内容运营 | 站点与品牌配置、SEO、公告、广告位、友链、捐赠、协议模板 |
| 数据分析 | 调用日志增量汇总（分钟 / 小时 / 天），看板与原始日志解耦 |
| 工单 | 用户工单与消息、管理端处理流、附件策略 |

## 最短可行路径

1. 准备一台 Linux 服务器，安装 MySQL 8 与 Redis 6+。
2. 创建空库并导入后端仓库 `sql/full` 下的初始化 SQL。
3. 用环境变量设置数据库口令、`JWT_SECRET`、`PAYMENT_CONFIG_SECRET` 与 `INITIAL_ADMIN_PASSWORD`，启动后端。
4. 构建前端产物，用 Nginx 同域托管并把 `/system-api`、`/api` 反代到后端。
5. 用 `admin` + 初始化口令登录后台，完成站点、接口、套餐与支付配置。

详细命令与反向代理示例见 [部署指南](./docs/deployment.md)。

## 安全

初始化 SQL 不包含任何用户种子，首个管理员由部署者的环境变量创建；生产环境的所有密钥都通过环境变量注入。漏洞报告方式见各代码仓的 `SECURITY.md`。

## 许可证

文档采用 CC BY 4.0；代码仓许可证以各自仓库 `LICENSE` 为准（后端 AGPL-3.0，前端 MIT）。