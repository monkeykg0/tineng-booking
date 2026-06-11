# 小程序（家长端 + 教练端）—— 待初始化

技术栈：Taro 4 + React + TypeScript + NutUI React（定稿，见 docs/技术方案 3.1）。

## 初始化步骤

```bash
npm i -g @tarojs/cli
# 在仓库根目录执行（本目录需为空，先删除本 README）
taro init miniprogram
# 模板选择：React + TypeScript + Sass + NutUI
```

初始化后：

1. 锁定 Taro 小版本（package.json 不用 ^ 范围，提交 lockfile）。
2. 建 `src/utils/request.ts` 统一封装 Taro.request（拼 JWT、统一错误码处理、401 跳登录）。
3. 页面清单按 PRD 第 13.1/13.2 节创建。

纪律（CLAUDE.md 同步约定）：只投微信端，不做跨端抽象；订阅消息、手机号快速验证等微信专属能力直接用微信 API。
