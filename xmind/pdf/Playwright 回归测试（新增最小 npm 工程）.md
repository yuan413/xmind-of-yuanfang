# Plan: easycode-java-orm Playwright 回归测试（新增最小 npm 工程）

## Context
用户希望在 easycode-java-orm skill 目录内新增最小 npm 工程，用 Playwright 验证浏览器表单：可切换项目加载候选并提交 payload。当前仓库与 sds-schedule-dispatch-center 均无 Playwright/npm 配置，因此需要在 skill 目录建立独立测试脚手架。

## Goals
- 在 easycode-java-orm 目录新增最小 npm 工程，支持 `npm test` 运行 Playwright。
- Playwright 测试覆盖：打开表单、切换 projectName、等待 `/candidates` 返回、填写 bizPackageName/dataSource、提交并校验输出 JSON。
- 假设运行时位于 sds-schedule-dispatch-center，用其作为 projectRoot 以获取表单初始化候选。

## Plan
1. 新增最小 npm 工程（skill 目录内）
   - 新增 package.json、playwright.config.*（或在测试中直接用 `@playwright/test` 默认配置）。
   - 依赖：`@playwright/test`（并安装浏览器）。
   - npm 脚本：`test` 运行 Playwright 测试。

2. 新增 Playwright 测试脚本
   - 放置在 `plugins/sds_backend_plugin/skills/easycode-java-orm/tests/` 或 `scripts/` 子目录（与 package.json 相邻）。
   - 启动 `form_capture_server.py` 子进程（project-root 指向 sds-schedule-dispatch-center），读取输出的 `FORM_CAPTURE_URL`。
   - Playwright 打开 URL，切换 projectName，等待 `/candidates` 返回成功，选择/填写 bizPackageName 与 dataSource，提交表单。
   - 读取输出 JSON 文件（/tmp/easycode-java-orm/form-capture-result.json），断言包含 projectName、bizPackageName、dataSource 且与 UI 输入一致。

3. 测试稳定性处理
   - 通过等待网络响应（/candidates）与页面状态文字，避免脆弱的固定 sleep。
   - 测试结束后关闭服务器子进程（或等待其 one-shot 自动退出）。

## Critical Files
- plugins/sds_backend_plugin/skills/easycode-java-orm/package.json (new)
- plugins/sds_backend_plugin/skills/easycode-java-orm/playwright.config.* (new, if needed)
- plugins/sds_backend_plugin/skills/easycode-java-orm/tests/easycode-form.spec.ts (new)
- plugins/sds_backend_plugin/skills/easycode-java-orm/scripts/form_capture_server.py (used by test)

## Verification
- 运行 `npm test`（在 skill 目录），应自动启动表单、完成切换与提交并通过断言。
- 输出 JSON 包含 projectName、bizPackageName、dataSource，且与测试输入一致。
