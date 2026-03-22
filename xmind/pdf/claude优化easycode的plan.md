# Context

需要把 `browser-form-submit-capture` 和 `easycode-java-orm` 两个 skill 的实际执行速度尽可能压到最低，并且优先在真实 SDS 项目 `/Users/01428674/IdeaProjects/sds-schedule-dispatch-center` 中验证，而不是只做静态文档优化。当前瓶颈既包括模型侧的 prompt/context 负担，也包括脚本启动、扫描、校验、浏览器往返等运行期开销。

已确认的现状：
- 本地热更新链路已存在：`/Users/01428674/IdeaProjects/sds-marketplace/.claude/commands/test-plugin.md`
- 真实目标项目存在：`/Users/01428674/IdeaProjects/sds-schedule-dispatch-center`
- `browser-form-submit-capture` 的主要热点在 `scripts/form_capture_server.py`
- `easycode-java-orm` 已有渐进式规则加载设计，但 `SKILL.md` 和 `references/code-examples.md` 仍然过大且有重复
- `skill-creator` 的本地参考已验证可用，可直接借用其“精简 SKILL、把细节下沉到 references、把重复确定性工作下沉到 scripts”的原则
- `find-skills` 本地定义当前未定位到，因此不把它作为第一阶段优化前提；若首轮收益不足，再把外部辅助 skill 发现作为可选补充路径

# Recommended Approach

按阶段优化，先做依赖型基础 skill，再做上层复杂 skill：
1. 先优化 `browser-form-submit-capture`
2. 在 `sds-schedule-dispatch-center` 做基线与回归测速
3. 再优化 `easycode-java-orm`
4. 最后做联动回归，确认 `easycode-java-orm` 通过更轻量的 browser-form 通路获得真实收益

这样做的原因：
- `browser-form-submit-capture` 同时是独立目标，也是 `easycode-java-orm` 的关键承载方式之一
- 其改动集中、收益可单独测量，适合作为第一阶段
- `easycode-java-orm` 的问题更偏 prompt 架构和步骤编排，适合在基础 skill 收敛后再处理
- 分阶段更容易识别“哪一项优化真正带来了速度收益”

# Plan

## Phase 1: Establish baseline measurements

目标：先得到 before 数据，避免“感觉变快”。

执行内容：
- 使用 `/test-plugin` 将 marketplace 当前插件热更新到已安装路径
- 在 `/Users/01428674/IdeaProjects/sds-schedule-dispatch-center` 中执行两类基线场景：
  - `browser-form-submit-capture`：最小 schema、富预览 schema、easycode-like schema
  - `easycode-java-orm`：AskUserQuestion 路径、browser-form 路径、已有核心文件路径
- 记录以下指标：
  - URL ready latency
  - submit-to-result latency
  - end-to-end wall clock
  - 交互轮次
  - 工具调用数/读取数
  - 结果 JSON 大小
  - 近似上下文载荷（读取的 skill/reference 体量）

复用现有能力：
- `/Users/01428674/IdeaProjects/sds-marketplace/.claude/commands/test-plugin.md`
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/browser-form-submit-capture/scripts/form_capture_server.py`

## Phase 2: Optimize browser-form-submit-capture

目标：先拿到高收益、低风险优化。

优先改动：
1. 精简 `SKILL.md`
   - 仅保留触发条件、最小输入契约、默认输出契约、异常边界
   - 把大段 schema 示例和重复说明移出热路径
2. 让 `form_capture_server.py` 支持按需输出
   - 当前热点函数：
     - `load_fields_schema()` `plugins/sds_backend_plugin/skills/browser-form-submit-capture/scripts/form_capture_server.py:304`
     - `build_form_html()` `plugins/sds_backend_plugin/skills/browser-form-submit-capture/scripts/form_capture_server.py:616`
     - `parse_form_body()` `plugins/sds_backend_plugin/skills/browser-form-submit-capture/scripts/form_capture_server.py:820`
     - `resolve_field_answer()` `plugins/sds_backend_plugin/skills/browser-form-submit-capture/scripts/form_capture_server.py:849`
     - `build_resolved_answers()` `plugins/sds_backend_plugin/skills/browser-form-submit-capture/scripts/form_capture_server.py:915`
     - `build_rendered_option_order()` `plugins/sds_backend_plugin/skills/browser-form-submit-capture/scripts/form_capture_server.py:920`
   - 默认先保持兼容；新增“精简输出模式”作为可选能力，避免一开始就破坏旧调用方
3. 优先复用现有 `--fields-json`
   - 该能力已存在于 `plugins/sds_backend_plugin/skills/browser-form-submit-capture/scripts/form_capture_server.py:1019`
   - 对短 schema 避免额外临时文件读写
4. 若 Phase 2 前三项完成后收益仍不够，再评估常驻 daemon/session 化
   - 该项收益高，但实现风险和回归面更大，放到第二优先层

关键文件：
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/browser-form-submit-capture/SKILL.md`
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/browser-form-submit-capture/scripts/form_capture_server.py`
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/browser-form-submit-capture/references/fields-schema.example.json`

## Phase 3: Optimize easycode-java-orm prompt architecture

目标：显著降低 skill 触发后的上下文载荷与无效读取。

优先改动：
1. 把 `SKILL.md` 改成薄编排层
   - 保留：触发条件、输入项、主流程、风险确认规则
   - 移除或下沉：大段交互文案、完整 browser schema、重复的渐进式加载说明、冗长自检说明
2. 拆分 `references/code-examples.md`
   - 当前文件混合了代码示例、交互示例、目录结构和 schema 示例，容易被整块过读
   - 拆成按职责分离的小 reference，使每次只加载当前步骤所需内容
3. 让每个 rule 文件尽量引用更小的专用示例，而不是共享巨型总文件

热点位置：
- `plugins/sds_backend_plugin/skills/easycode-java-orm/SKILL.md:113`（承载方式前置选择）
- `plugins/sds_backend_plugin/skills/easycode-java-orm/SKILL.md:544`（最小验证方案）
- `plugins/sds_backend_plugin/skills/easycode-java-orm/SKILL.md:587`（重复的渐进式规则加载说明）
- `plugins/sds_backend_plugin/skills/easycode-java-orm/references/code-examples.md`

关键文件：
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/easycode-java-orm/SKILL.md`
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/easycode-java-orm/references/code-examples.md`
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/easycode-java-orm/references/vo-rule.md`
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/easycode-java-orm/references/entity-rule.md`
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/easycode-java-orm/references/mapper-rule.md`
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/easycode-java-orm/references/serviceimpl-rule.md`
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/easycode-java-orm/references/dao-xml-rule.md`
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/easycode-java-orm/references/enum-rule.md`
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/easycode-java-orm/references/controller-rule.md`
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/easycode-java-orm/references/api-rule.md`

## Phase 4: Optimize easycode-java-orm runtime/tool flow

目标：减少真实执行中的扫描、读取和额外交互轮次。

优先改动：
1. 合并 discovery pass
   - 一次性完成 bizPackageName 扫描、dataSource 解析、目标路径推导、核心文件存在性探测
   - 避免在承载方式选择后重复扫描
2. 两段式 existing-file validation
   - 第一层只判断存在性
   - 第二层只对真正进入最终生成清单、且需要判断是否覆盖的文件做内容读取与校验
3. 视实现成本决定是否引入 deterministic helper script
   - 用脚本产出紧凑 JSON，总结扫描结果与派生值
   - 遵循 `skill-creator` 中“脚本负责重复且确定性强的工作，SKILL.md 保持精简”的原则
4. 对已明确指定承载方式的请求，先不改变默认交互契约
   - 本轮保持现有前置选择流程不变
   - 若第一轮完全兼容优化收益不足，再单独评估“用户已明确指定承载方式时减少冗余往返”这一增强项

关键文件：
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/easycode-java-orm/SKILL.md`
- `/Users/01428674/IdeaProjects/sds-marketplace/plugins/sds_backend_plugin/skills/easycode-java-orm/evals/evals.json`
- 可能新增的 deterministic helper script（放在 `easycode-java-orm/scripts/`）

## Phase 5: Integrated verification

目标：验证单 skill 优化和联动收益都成立。

执行内容：
- 再次通过 `/test-plugin` 热更新插件
- 在 `sds-schedule-dispatch-center` 中重跑基线场景
- 输出 before/after 对比表：
  - URL ready latency
  - submit-to-result latency
  - end-to-end wall clock
  - 交互轮次
  - 工具调用数
  - reference 读取量
  - 结果载荷大小
- 对 `easycode-java-orm` 做额外联动验证：
  - AskUserQuestion 路径未退化
  - browser-form 路径确实更快
  - 核心文件强制生成与覆盖确认规则未被破坏

# Reuse / Existing Patterns To Preserve

- `browser-form-submit-capture` 已有 `--fields-json` 能力：`plugins/sds_backend_plugin/skills/browser-form-submit-capture/scripts/form_capture_server.py:1019`
- `easycode-java-orm` 已有渐进式规则文档拆分，不推翻，只继续把热路径变薄
- `quick-diff-master` 已展示“轻 SKILL + 特定上下文策略”的模式：`plugins/sds_backend_plugin/skills/quick-diff-master/SKILL.md`
- `skill-creator` 的关键原则需贯彻：`SKILL.md` 保持 lean，详细信息放 `references/`，重复且确定性的工作下沉到 `scripts/`

# Verification Plan

1. 在 marketplace 仓库中完成每一小阶段修改后，用 `/test-plugin` 热更新到已安装插件目录
2. 在 `/Users/01428674/IdeaProjects/sds-schedule-dispatch-center` 中执行真实场景
3. 每个场景至少重复 3-5 次，记录中位数
4. 重点对比：
   - skill 触发后的首轮响应时间
   - browser-form URL 就绪时间
   - 提交后结果产出时间
   - 需要的交互轮次
   - 读取的文档/参考文件数量
5. 若引入可选的“精简输出模式”，需要补充兼容性回归，确保旧调用方仍可继续使用现有行为

# Decision

用户已确认本轮按“完全兼容”边界推进。

执行约束：
- 不改变默认输出契约
- 不改变默认交互契约
- 先做 prompt 精简、重复读取收敛、扫描/校验分层、脚本热路径减负
- 所有可能改变默认行为的优化（如默认精简输出、默认跳过冗余确认、激进的常驻服务语义变化）均延后，只有在第一轮完全兼容优化收益不足时再单独提案
