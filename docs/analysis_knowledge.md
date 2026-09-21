# 互联网分析案例知识库

本功能使用本地关键词／标签检索，把相关案例的口径、前提和排查方法提供给 Supervisor 和 Workflow Planner。目标是帮助形成有依据的分析计划。它不是业务数据仓库，也不自动判定统计结论正确。

## 内容与代码

- `packages/backend/app/knowledge/cards.json`：运行时唯一知识源；当前 5 个企业案例、2 个方法条目。案例主要来自微软的同一篇论文，覆盖仍有限。
- `packages/backend/app/knowledge/retrieval.py`：结构校验、中英文检索、上下文预算、来源和版本摘要。
- `packages/backend/app/tasks/analysis_knowledge.py`：沿用现有 `tool_start/tool_end` 发布检索记录，并进入 AssistantMessage 的 steps。
- `packages/backend/app/tasks/agent_tasks.py`：根据当前用户问题检索，把同一份上下文传给 Supervisor 和规划器。
- `scripts/evaluate_analysis_knowledge.py`：可复现的小规模开发诊断，支持可选真实模型方法问答检查。

代码外的 `../data-agent-build-notes/knowledge-drafts/` 保留研究过程，不作为第二份运行时配置。

## 行为

1. 每次请求读取本地知识文件，按 NFKC／大小写归一后的关键词与标签匹配。英文使用词边界，中文按短语匹配；具体短语权重较高，同分按 ID 排序。
2. 最多选择 3 张卡，卡片上下文不超过 12000 字符；只添加完整卡片，不截断来源与方法条件。字符预算不等于精确 token 预算。
3. 匹配仅代表主题相关。字段含义、时间口径、随机化单位等适用条件由 Agent 结合用户输入和 schema 检查；缺少关键条件时提示澄清，不根据字段名擅自确定语义。
4. 通过 `planning_notes` 要求给出卡片 ID、适用或拒绝理由、数据缺口和分析步骤，再解释节点连接；这些仍是模型输出约束，不是确定性方法验证器。
5. “过程”中的 `analysis_knowledge` 记录卡片 ID、标题、命中词、来源、证据状态和目录 SHA-256。它说明检索结果，不声称模型已采用。实际采用理由在工作流工具输入的 `planning_notes` 中审阅。
6. 无匹配、知识文件缺失或格式错误时保留原有工作流路径，并明确没有可用参考；不编造引用。关闭开关则不增加上下文或检索事件。

知识条目只来自本地维护文件，不会在每次分析时自动联网搜文章，也不把用户数据写入知识库。检索日志不记录用户原问题或数据预览。

## 使用与开关

`ANALYSIS_KNOWLEDGE_ENABLED=true` 默认开启；关闭时在部署配置中设置为 `false`。

Compose 的环境变量通过 `env_file` 加载，修改 `.env` 后要重新创建服务才能更新环境：

```powershell
docker compose up -d --no-deps backend-api backend-worker
```

单纯更新已挂载的 Python 代码时，API 有 reload；Worker 需在没有运行中任务时重启：

```powershell
docker compose restart backend-worker
```

知识 JSON 每次检索重新加载；不需重启即可读取内容修订。

可试问（不需要上传数据）：

> 只讨论分析方法：某模块点击率下降，但点击次数增加，应该检查什么？请说明所需数据和参考依据。

> 只讨论方法：次日留存怎么算，需要先确认哪些口径？

具体数据分析仍需当前任务的数据源；知识库不提供当前业务事实。

## 验证命令

容器缺少测试工具时可安装仓库已声明的开发依赖；本功能没有增加生产依赖。

```powershell
docker compose exec -T backend-api uv pip install --python /app/.venv/bin/python 'pytest>=9.0.3' 'pytest-asyncio>=0.23.0' 'ruff>=0.15.12'
docker compose exec -T backend-api /app/.venv/bin/python -m pytest packages/backend/app/test/test_analysis_knowledge.py packages/backend/app/test/test_analysis_knowledge_integration.py packages/backend/app/test/test_workflow_prompts.py packages/backend/app/test/test_agent_prompts.py packages/backend/app/test/test_agent_orchestration.py -q
docker compose exec -T backend-api /app/.venv/bin/python /app/project/scripts/evaluate_analysis_knowledge.py --output /app/project/tmp/knowledge-evaluation.json
```

加 `--live` 会发起两次配置模型请求（无知识／有知识各一次），只发送合成的方法问题。输出包括模型、温度、usage、回答、知识版本；不要把单次问答当作统计显著的对比实验。

## 维护与限制

- 新卡需有 HTTPS 来源、精确章节定位、案例／方法类型、数据要求和结论边界；`evidence_status` 区分读过来源与执行过示例。
- 知识卡的修改改变 SHA-256，可追溯当时检索版本；不要把研究草稿与运行时卡分别维护成两套独立知识库。
- 关键词检索不理解否定句，也不能完整处理省略上下文的追问；可能漏检同义表达或召回不适用的案例。
- 无浏览器画面验收；后端事件和持久化消息结构已测试，展示复用既有通用工具步骤。
- 真模型仍可能过度推断：首次 smoke 中出现“人群变化必然导致两种 CTR 背离”的错误，已补充卡片边界，但不能以文字约束保证此类错误消失。
- 尚未完成独立留出题、真实数据执行质量和完整三组策略的统计评测；不能写成企业任务成功率或已量化准确率提升。
