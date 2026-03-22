# Plan: 优化 easycode-java-orm 表单 UI（frontend-design）

## Context
用户要求使用 frontend-design 插件优化 easycode-java-orm 表单的整体 UI，使页面更美观。当前表单 UI 由 `form_capture_server.py` 中的 HTML/CSS/JS 模板直接渲染，需要在不破坏现有交互与自动化测试的前提下提升视觉设计与信息层级。

## Goals
- 提升表单视觉品质：布局、留白、字体、配色、阴影与组件层级更清晰。
- 保持所有字段语义、label 文案、id/name、datalist 与按钮文本不变，避免破坏 Playwright 测试和表单提交。
- 不改变现有交互流程（项目切换、刷新候选、提交行为）。

## Plan
1. 复用现有表单结构
   - 仅调整 `HTML_TEMPLATE` 的样式与局部布局（容器、卡片、标题区、按钮、输入框、提示区）。
   - 保持 input/textarea/button 的 id、name、label 文案、按钮文本与 datalist id 不变。

2. 使用 frontend-design 生成改进方案
   - 将现有 HTML/CSS 作为基础输入，让插件输出优化后的样式与结构（如果需要微调结构，仅允许不影响标识符的布局变动）。
   - 重点优化：标题区、信息提示区、表单分组、选项卡片、提示文案与状态区可读性。

3. 手工对齐与兼容性检查
   - 检查修改后是否仍满足交互脚本使用的 DOM id（例如 `projectName-input`、`bizPackageName-refresh` 等）。
   - 确保响应式规则仍适配窄屏。

## Critical Files
- plugins/sds_backend_plugin/skills/easycode-java-orm/scripts/form_capture_server.py (HTML_TEMPLATE 内联样式)
- plugins/sds_backend_plugin/skills/easycode-java-orm/tests/easycode-form.spec.ts (保持测试可用，不修改)

## Verification
- 启动 `form_capture_server.py` 打开页面，检查视觉层级、输入/按钮状态、响应式布局。
- 运行现有 Playwright 测试，确认交互与提交无回归。
