# 体能约课助手

少儿体能机构约课与课时管理系统：微信小程序（家长/教练）+ Web 管理后台 + Go 后端。

需求与架构见 [docs/](docs/)，开发约定见 [CLAUDE.md](CLAUDE.md)。

## 快速开始

```bash
# 后端本地运行（需 Go 1.25+）
cd backend && make run
curl http://localhost:8080/healthz

# 服务器部署（先替换 deploy/nginx.conf 中的 TODO_DOMAIN，配置 deploy/.env）
cd deploy && cp .env.example .env && docker compose up -d --build
```

## 目录

| 目录 | 内容 |
| --- | --- |
| backend/ | Go + Gin + GORM + PostgreSQL 单体后端 |
| miniprogram/ | Taro 4 + React 小程序（待初始化，见目录内 README） |
| admin/ | Ant Design Pro 管理后台（待初始化，见目录内 README） |
| deploy/ | Docker Compose / Nginx 部署配置 |
| docs/ | PRD、技术方案、调研结论 |
