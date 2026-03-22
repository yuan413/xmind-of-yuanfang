# Plan: easycode-java-orm 由 Python 生成内容（LLM 仅传参）

## Context
当前 easycode-java-orm 由模型按规则文档生成 Java/Dao.xml 内容；仓库内的 Python 脚本只负责 discovery 与表单捕获，不生成实际文件内容（见 `discover_main_interaction_context.py:132`、`discover_main_interaction_context.py:143`、`discover_main_interaction_context.py:196` 与 `form_capture_server.py:586`）。目标是把“内容生成”改为 Python 脚本执行，模型只传递必要参数，以减少模型分析/生成成本并提升生成速度。

## Goals
- Python 脚本可根据参数与表结构生成 VO/Entity/Mapper/ServiceImpl/Dao.xml/Enum/Controller/Api。
- LLM 只负责参数收集/传递，不再拼装具体文件内容。
- 当本地无 Python 环境时：用 Bash 扫描 bizPackageName、dataSource；文件内容仍走现有 LLM 生成路径。
- AskUserQuestion 在任何文件生成前确认 Api/Controller/Enum 选择（不等待核心文件生成后）。
- 生成逻辑与 `references/*-rule.md` 的规范一致，覆盖核心文件强制生成与覆盖确认流程。

## Plan
1. **新增 Python 环境检测与 fallback 分支**
   - 在任何 python 脚本调用前检测 `python3` 是否可用；不可用时进入 Bash fallback 分支。
   - Python 可用时保持既有 discovery + form_capture 流程。
   - Python 不可用时：禁用 browser form，强制 AskUserQuestion 流程。

2. **Bash 扫描替代 discovery 的候选项产出**
   - 在无 Python 环境时，用 Bash 扫描：
     - `bizPackageName`：基于 `src/main/java/**/service/**` 目录生成候选与预览信息。
     - `dataSource`：解析 `config/productiondocker/application-druid.properties` 获取 datasource 列表。
   - 允许在 Bash 中调用系统工具（如 ruby/node）辅助解析，但不依赖 Python。
   - 生成的候选结构需与 discovery 脚本输出一致，供 AskUserQuestion/交互模板直接复用。

3. **生成前的 Api/Controller/Enum 二次确认**
   - 保持主交互一次性收集 `tableName`、`createTableSql`、`bizPackageName`、`dataSource`、`generateFiles`、`tableConflictStrategy`。
   - 在任何文件生成/内容渲染之前，追加 AskUserQuestion：明确确认 Api/Controller/Enum 是否生成（允许修改选择）。
   - 二次确认提示需展示默认勾选与原因（如 enum 检测到才默认勾选）。
   - 该确认需适配两种载体：browser form 提交后、以及对话内 AskUserQuestion 模式。

4. **Python 生成脚本继续落地**
   - 新增 `plugins/sds_backend_plugin/skills/easycode-java-orm/scripts/generate_java_orm.py`（stdlib-only）：
     - 解析输入参数（含 schema/SQL）。
     - 基于纯字符串模板渲染各文件内容（不引入外部依赖）。
     - 输出 JSON：生成清单、文件路径、枚举识别结果、验证结果、是否需要覆盖确认。
   - 复用 discovery 逻辑：
     - `discover_main_interaction_context.build_derived_values`（`discover_main_interaction_context.py:132`）
     - `discover_main_interaction_context.derive_core_paths`（`discover_main_interaction_context.py:143`）
     - `discover_main_interaction_context.build_core_file_checks`（`discover_main_interaction_context.py:196`）

5. **模板与规则对齐**
   - 将 `references/*-rule.md` 关键约束固化为 Python 模板与校验：
     - VO/Entity：Javadoc、注解、枚举 link 规则
     - Mapper/ServiceImpl：注解、父类/接口
     - Dao.xml：namespace 与 resultMap/column list
     - Controller/Api：注解与 path/FeignClient 规则
     - Enum：COMMENT 中 `code:value` 识别与类模板

6. **表结构来源策略**
   - 若提供 `createTableSql`：在脚本内解析列与注释，生成 schema。
   - 若未提供 SQL：允许 LLM 传入 schema JSON 或由脚本读取已有表结构（按你的选择确认后执行）。

7. **覆盖与校验流程落地**
   - 核心 5 文件缺失时强制生成（复用 `coreFileChecks`）。
   - 已存在且被选中时执行内容校验（按规则检查必需结构）。
   - 需要覆盖时通过 AskUserQuestion 二次确认（保持现有流程约束）。

8. **Skill 流程接入**
   - 在 `SKILL.md` 中新增“调用 Python 生成脚本”的步骤，明确参数传递与结果解析。
   - 增加 Python 不可用时的 Bash fallback 说明与交互路径。

9. **文档与版本策略**
   - 实际功能变更需要更新 `CHANGELOG.md` 与 `README.md`。
   - 版本号变更需先征询你的确认（CLAUDE.md 要求）。

## Critical Files
- 交互与规则入口：`plugins/sds_backend_plugin/skills/easycode-java-orm/SKILL.md`
- Discovery：`plugins/sds_backend_plugin/skills/easycode-java-orm/scripts/discover_main_interaction_context.py`
- 表单捕获（字段一致性参考）：`plugins/sds_backend_plugin/skills/easycode-java-orm/scripts/form_capture_server.py`
- 规则文档：`plugins/sds_backend_plugin/skills/easycode-java-orm/references/*-rule.md`
- 交互示例：`plugins/sds_backend_plugin/skills/easycode-java-orm/references/examples/interaction-patterns.md`
- 新增生成器：`plugins/sds_backend_plugin/skills/easycode-java-orm/scripts/generate_java_orm.py`（拟新增）

## Verification
- Python 可用场景：走 discovery + form_capture 流程，确认 Api/Controller/Enum 在生成前被二次确认。
- Python 不可用场景：走 Bash 扫描 + AskUserQuestion 流程，候选项与交互展示完整。
- 以固定 `createTableSql` 生成全量文件，确认输出清单与路径一致。
- 对比生成内容与 `references/examples/*` 中示例（若需，局部比对关键段落）。
- 运行最小自检项：核心 5 文件齐全、必需注解/继承/namespace/枚举 link 校验通过。
