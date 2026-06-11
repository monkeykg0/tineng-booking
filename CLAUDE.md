# 体能约课助手（tineng-booking）

少儿体能机构的约课与课时管理系统：微信小程序（家长 + 教练）+ Web 管理后台 + Go 单体后端。给真实机构使用的生产系统，单人开发维护。

## 必读文档（docs/，为唯一权威版本）

- `少儿体能约课小程序_PRD_v0.2.md`（v0.2.4）—— 需求、业务规则、状态机、验收用例。**写任何业务代码前先查 PRD 对应章节**，特别是第 3 章状态机、第 5 章业务规则、第 8 章边界场景。
- `少儿体能约课小程序_技术方案_v1.0.md`（v1.2）—— 架构与选型决策及理由。
- `实地调研清单.md` —— 机构调研结论。

## 技术栈（已定稿，不要更换或追加）

| 层 | 选型 |
| --- | --- |
| 后端 | Go 1.25 + Gin + GORM + PostgreSQL 16，单体单实例 |
| 小程序 | Taro 4 + React + TypeScript + NutUI React |
| 管理后台 | Ant Design Pro（umi 4 + ProComponents） |
| 部署 | Docker Compose（deploy/），阿里云 2c4g，Ubuntu 24.04 |

明确**不引入**：Redis、消息队列、微服务、K8s、读写分离（理由见技术方案 2.2 / 第 9 章）。如确有需要，先在技术方案中论证再动手。

## 硬性红线（违反即返工）

1. **所有业务规则只写在 backend/internal/service 层**。handler 只做参数绑定与响应；repository 只做数据访问。PRD 第 12 章的 GWT 验收用例要能直接落成 service 层表驱动测试。
2. **课时变动必须同事务写流水**：任何 remaining_credits 的增减（签到/撤销/调整/发卡）必须在同一事务内写 credit_transaction（含变动前后余额、操作者、原因），不允许任何代码绕过对应 service 函数直接改余额。
3. **防超卖不依赖 ORM 读改写**：名额扣减用原子 UPDATE（`SET booked_count = booked_count + 1 WHERE ... AND booked_count < capacity`，按影响行数判断），重复预约靠部分唯一索引兜底（技术方案 6.1/6.2）。
4. **数据权限在 service 层校验**：家长只能访问绑定学员、教练只能访问自己课次（含助教身份），不能只靠路由中间件。手机号对教练端展示必须脱敏。
5. **密钥不进 git**：微信 AppSecret、数据库密码、JWT secret 全部走环境变量；.env 已在 .gitignore。

## 工程约定

- API 响应统一 `{"code": 0, "message": "ok", "data": {}}`；业务错误码分段：1xxx 通用、2xxx 预约、3xxx 课时卡、4xxx 权限（技术方案 6.6）。错误提示必须"说明原因 + 下一步"，禁止裸的"操作失败"。
- 时间判断一律以服务端为准；状态流转必须符合 PRD 第 3 章状态机，禁止跳转未定义的状态边。
- 机构可配置规则（CFG-01~10）从 merchant_config 读取，不许硬编码默认值散落各处（集中一处定义默认值）。
- 迁移：MVP 期用 GORM AutoMigrate；上线前切 golang-migrate（migrations/ 目录已留）。
- 测试底线：预约与扣课 service 必须覆盖 PRD 第 12 章 12 条 GWT 用例 + 第 8 章 E-01~E-14 边界场景。
- admin 用 Pro 模板自带体系（umi request / useModel / access.ts），**不混搭** TanStack Query、Zustand、Axios。
- miniprogram 锁定 Taro 小版本；只投微信端，不为跨端做抽象；微信专属能力直接用微信 API。
- Git：main 保护，feature 分支开发，小步提交。

## 目录说明与待初始化项

```text
backend/      Go 后端，骨架已建好；cmd/api/main.go 当前为健康检查 hello-world（供部署链路验证）
miniprogram/  待用 Taro CLI 初始化（见该目录 README）
admin/        待用 Ant Design Pro 脚手架初始化（见该目录 README；初始化后先做减法：删国际化/mock/示例页）
deploy/       docker-compose.yml + nginx.conf 模板，域名占位符 TODO_DOMAIN 需替换
docs/         需求与方案文档（权威版本）
```

## 沟通约定

- 回复、思考、任务清单一律中文。
- 遵循 KISS：实现 PRD 标 P0 的内容，P1/P2 不提前实现；发现 PRD 与代码矛盾时先指出，不要默默改需求。
