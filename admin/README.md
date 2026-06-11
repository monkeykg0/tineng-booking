# 管理后台 —— 待初始化

技术栈：Ant Design Pro（umi 4 + ProComponents）（定稿，见 docs/技术方案 3.2）。

## 初始化步骤

```bash
# 在仓库根目录执行（本目录需为空，先删除本 README）
pnpm dlx create-umi@latest admin
# 选择 Ant Design Pro 模板 + pnpm
```

初始化后第一件事是做减法：

1. 删除国际化（src/locales 及相关配置）。
2. 删除 mock/ 目录，request 直连后端 `https://api.<域名>`（本地连 localhost:8080）。
3. 删除示例页面，按 PRD 第 13.3 节页面清单重建菜单。
4. `access.ts` 按 admin/staff 两角色配置，对接 JWT 的 role。
5. 锁定 @umijs/max 小版本。

纪律（CLAUDE.md 同步约定）：用模板自带的 umi request / useModel / access 体系，不引入 TanStack Query、Zustand、Axios。
