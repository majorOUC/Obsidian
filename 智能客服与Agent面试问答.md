# 第一部分：梦莹智能客服

---

## 项目介绍

### 一、整体架构与 Agent 设计

#### 你这个智能客服模块整体是怎么实现的？

**A：** 我这个智能客服模块采用 **Orchestrator-Workers 架构**，基于 LangGraph 状态图 + FastAPI 实现。整体分为六层：

**整体流程图：**

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 25, "rankSpacing": 35}, "themeVariables": {"fontSize": "13px"}}}%%
flowchart TD
    U[用户消息] --> GW[鉴权 + 限流]
    GW --> G["Guardrail 三层：黑名单 → 向量 → LLM 兜底"]
    G -->|拦截| RJ[拒绝话术]
    G -->|通过| O["Orchestrator：Mem0 画像注入<br/>结构化输出 TaskPlan"]
    O -->|无任务| S["Summary LLM 流式回复（SSE）"]
    O -->|有任务| E["TaskExecutor<br/>依赖分析，无依赖并行"]
    E --> W["四类 Worker 并行<br/>shop / voucher / reservation / knowledge"]
    W --> PA{待确认敏感操作？}
    PA -->|是| C["confirm<br/>interrupt() 暂停"]
    C -->|确认| EX["execute_sensitive<br/>幂等键 + 执行护栏"]
    C -->|取消| QX[回复已取消]
    PA -->|否| S
    EX --> S
    S --> H["Holdback 输出检查"]
    H --> SV["保存历史 + 事实<br/>Mem0 异步提取画像"]
```

**第一层：网关层**

- 身份鉴权：用户通过 memory_id 标识会话，Java 后端传递用户身份
- 速率限制：按用户维度的滑动窗口限流，防刷防滥用

**第二层：前置处理层**

- Guardrail 防护（输入侧）三层递进：关键词黑名单（AC 自动机 <1ms，命中即拒）→ 向量相似度（语义变体，score≥0.90 直接拒）→ LLM 兜底（越狱 + 相关性判断），过滤越狱攻击和无关问题

**第三层：编排层（Orchestrator）**

- 图入口节点 guardrail 统一重置跨请求残留的 state 字段
- Orchestrator（Primary Agent）用结构化输出（TaskPlan）理解用户意图，编排任务计划
- 统一检索用户画像（Mem0），注入系统提示词实现个性化
- 任务计划包含 understanding（意图理解）+ tasks（任务列表）+ depends_on（依赖关系）

**第四层：执行层（Executor + Workers）**

- TaskExecutor 按依赖关系分析后并行执行无依赖任务
- Shop Worker：查询商户信息、搜索区域商户
- Voucher Worker：查询/购买优惠券
- Reservation Worker：查询/创建预约
- Knowledge Worker：RAG 检索（向量检索 + BM25 + Rerank）
- 每个 Worker 独立熔断器（PyBreaker），故障隔离

**第五层：安全确认层**

- 敏感操作确认：子 Agent 遇到敏感工具不执行，动作透传到 state.pending_action
- 图级 confirm 节点调用 LangGraph interrupt() 暂停，等用户确认
- 恢复时只重放 confirm 节点（26ms），规划 LLM 完全不重跑

**第六层：输出层**

- Summary LLM 流式生成回复（SSE 格式）
- 输出安全检查（Holdback 缓冲 + 正则）：扣留最后 50 字符作为安全窗口，检查 system prompt 泄漏、技术信息泄露、PII 泄露
- 记忆层：Redis 存储会话历史（支持滑动窗口压缩），Mem0 异步提取用户偏好

**核心设计亮点：**

1. 纵深防御：输入 Guardrail + 输出 Guardrail 双层防护
2. 结构化输出：TaskPlan 的 understanding 字段实现隐式 CoT，先理解再编排
3. 状态持久化：AsyncRedisSaver 检查点 + interrupt 机制，支持断点续传
4. 工具容错：重试 + 熔断 + 错误分类，查询类可重试、操作类不重试
5. 流式体验：SSE 流式输出 + 执行状态实时推送，感知延迟低

---

#### 任务执行引擎的工作原理

**A：** 任务执行引擎（TaskExecutor，`app/agents/executor.py`）负责执行 Primary Agent 编排的任务计划。它是一个普通的 Python 组件，**不在 LangGraph 图内**——Orchestrator 节点内部直接 `await executor.execute(plan, user_id)`，这样并行调度用原生 asyncio 就够了，不需要额外引入图的复杂度。

**输入：结构化的任务计划**

Orchestrator 用结构化输出产出 TaskPlan（`plan_models.py`），包含：

- `understanding`：对用户请求的理解
- `tasks`：任务列表，每个 Task 包含 `agent`（shop/voucher/reservation/knowledge 四选一）、`goal`（任务目标，子 Agent 据 goal 决定调什么工具）、`depends_on`（前置任务索引，无依赖为 null）

**核心流程：依赖分析 + 分批并行**

`execute()` 用一个 while 循环做拓扑分批：

1. 每轮扫描所有未执行任务，收集"就绪任务"——没有依赖、或依赖任务已完成(enabled)的
2. `asyncio.gather` 并行执行这一批就绪任务
3. 重复直到全部执行完；如果某轮找不到任何就绪任务，说明存在循环依赖，告警后退出（防死循环）

例：用户问"推荐烧烤店，再看看我有没有券"→ 任务 0（knowledge）和任务 1（voucher）无依赖并行跑，任务 2 依赖 0 的结果就等下一批。

**单个任务的执行路径（execute_single_task）**

按顺序经过四道关卡：

1. **依赖检查与中间结果传递**：前置任务失败 → 当前任务直接跳过（标记 skipped，不浪费 LLM 调用）；前置成功 → 把前置结果拼进当前 goal（"前置任务结果：..."），实现子 Agent 间通过 results 列表传递中间结果；前置是待确认的敏感操作 → 不把标记原文喂给 LLM，替换成"需用户确认后才能执行"的提示语
2. **Agent 校验**：未知 agent 名直接失败返回
3. **熔断预检查**：通过 AGENT_BREAKER_MAP 取该子 Agent 的 PyBreaker（**与工具层共用同一组熔断器，共享失败计数**），若已 OPEN 直接短路返回"服务暂时不可用"——整个子 Agent 调用被拦下，LLM 调用都不发生
4. **执行**：voucher/reservation 这两个涉及用户身份的 Agent 额外传 user_id。**重试在工具层（retry_tool_call），Executor 不重复重试**，职责分离；异常统一交 error_handler 做错误分类，转成面向用户的话术，单任务失败不影响其他并行任务

**输出与敏感操作扫描**

返回 `task_details` 列表（task_index / agent / goal / result / success / pending_action / error_type）。Orchestrator 拿到后扫描 pending_action：敏感工具不真正执行、只返回待确认标记，第一个待确认操作交给图级 confirm 节点 interrupt 暂停等用户确认，同批其余敏感操作置为"请前一操作完成后重新发起"（v1 一次只确认一个）。

**设计要点**

1. **并行度由依赖 DAG 决定**：无依赖任务自动并行，LLM 调用时间从串行相加变并行取最大
2. **熔断下沉共享**：任务层预检查和工具层熔断用同一组熔断器，工具层的失败计数直接让任务层拦下后续调用，故障隔离到 Worker 粒度
3. **失败即降级**：任何任务失败都转成用户可读的话术放进 results，不会抛异常打断整轮对话
4. **可改进点**（主动说）：depends_on 目前是单个任务索引，不支持多依赖；循环依赖只告警不修复；多个敏感操作一次只确认一个
---
### 二、安全防护

#### 入口防护三层架构与 badcase 自演进（关键词黑名单 + 向量检测层）

**A：** 我的入口防护是**三层架构**：关键词黑名单 → 向量检测层 → LLM 兜底。业界调研过 Meta Llama Prompt Guard 2（DeBERTa 小分类器）、NVIDIA NeMo Guardrails（向量相似度启发式）、AWS/Azure 托管内容过滤，共识是**纯关键词可被对抗变体轻易绕过，但仍是确定性模式最便宜的第一道闸**；向量层负责语义变体，LLM 层同时承担越狱检测和相关性校验。

**第一层：关键词黑名单**

- 词表存 **MySQL（真相源）+ Redis Set（60s 热缓存）**，两级加载链，均失效时本层跳过（返回未判定），由后两层兜底
- 匹配引擎用 **AC 自动机（Aho-Corasick）**：Trie + 失败指针，单遍扫描同时匹配全部关键词，复杂度与词表规模无关；pyahocorasick 加载失败自动回退线性扫描

**第二层：向量检测层**

- 复用现有 Milvus，`guardrail_attacks` 集合存已知越狱/注入话术向量（40 条种子 + badcase 持续沉淀）
- 输入文本算 top-1 余弦相似度，**两档分流**：score ≥ 0.90 直接拒（高置信语义变体，省 LLM）；其余全部进 LLM 兜底

**第三层：LLM 检测（越狱 + 相关性双判断）**

- 模型选型：使用 **qwen-turbo**（非推理模型，~200-400ms，成本降 10 倍+）。guardrail 只需返回两个 boolean + 一句话原因，不需要推理能力
- 同时检查两项：`is_safe`（越狱检测）和 `is_relevant`（是否与本地生活服务相关），单次调用完成
- 前两层把明确攻击拦掉后，LLM 只处理真正的灰色地带

**被拦截请求不污染上下文**

Guardrail 拒绝的请求**不写入 Redis 会话历史**、不触发 mem0 偏好分析——用户消息和助手回复在图执行前不预存，仅在 guardrail 通过后才保存。避免攻击内容进入后续对话上下文或被提取为用户画像。

**badcase 自演进闭环**

只有 LLM 拦截的越狱攻击写入 `guardrail_log`（无关请求不记录，关键词/向量层拒绝不记录——已知模式无需沉淀）。挖掘脚本自动完成"提取 → 沉淀 → 合入 → 清理"四步：

1. **提取**：从 `guardrail_log` 拉取越狱攻击样本（`llm_reason LIKE '不安全:%'`），调 LLM 提取 ≤6 字词根及其拆字/拼音变体，入库 `guardrail_keyword` 状态 `pending`
2. **向量沉淀**：同批样本向量化入库 Milvus `guardrail_attacks`——向量层持续学习新攻击变体
3. **合入**：pending 词按命中次数降序排序，前50个自动合入 `active` 生效（AC 自动机即时加载），其余删除
4. **清理**：已沉淀的日志条目从 `guardrail_log` 删除，下次挖掘不会重复处理

整个流程通过前端 Guardrail 管理页面一键触发（`POST /api/guardrail/mine`），也可手动执行 `python -m app.scripts.mine_blacklist mine && evaluate`。

这样形成"漏网攻击 → LLM 兜住 → 日志记录 → 挖掘沉淀 → 向量库/关键词库变大 → 以后向量层/关键词层直接拦"的闭环。关键词库只保留高命中词（top50），避免无限膨胀；向量库持续积累攻击语义向量，低分放行窗口随库壮大自然缩小。

---

#### 敏感操作确认机制（图级 confirm 节点 + LangGraph interrupt + RedisSaver）

**核心机制 = 四段链路**：工具层标记拦截 → 状态透传解析 → LangGraph `interrupt()` 图级暂停（Redis checkpoint）→ 下一条消息正则识别 `Command(resume)` 恢复。

**① 工具层：敏感工具不执行，返回"标记"**（`app/agents/sensitive.py`）

`purchase_voucher`/`create_reservation` 在子 Agent 的 `_execute_tool` 里被拦截，不真调用，返回 `__PENDING_CONFIRMATION__ + JSON{tool_name, args, description}`。description 本身就是给用户的完整确认问句（拦截时先查券名/价格构造，如"将为您购买「满100减20券」（售价¥75，抵扣¥100），确认购买吗？"）；缺参/查无此券/秒杀券在校验段直接短路返回文本，不进确认流程。

**② 标记沿 executor → primary 透传回图，`parse_pending_marker` 解析**

用 `json.JSONDecoder().raw_decode` 而不是 `json.loads`——子 Agent 可能把标记拼接在正常查询结果后面（前面还有"查到的券列表"文本），raw_decode 从标记位置起解析第一个完整 JSON 对象、忽略后续。解析结果写入图 state 的 `pending_action`。

**③ 暂停 = confirm 节点的 `interrupt()` + Redis checkpoint**（`app/graph/builder.py`）

条件边 `route_after_orchestrator` 发现有 `pending_action` → 路由到独立轻量 confirm 节点 → `interrupt(payload)` 挂起整图。AsyncRedisSaver（专用 Redis Stack :6380，TTL 60min）把挂起点完整状态持久化。Router 检测到 `__interrupt__` 后把确认问句以 SSE `event: confirm` 推给前端弹"确认/拒绝"按钮，**跳过 Summary LLM**（不为问句花 token）。

**④ 恢复 = 下一条消息的模式匹配**（`app/routers/chat.py`）

`_handle_chat` 先 `graph.aget_state()` 探测 `snapshot.next` 是否非空（图是否停在半路）：

- 命中 `CONFIRM_PATTERN`（整句短句白名单）→ `Command(resume=True)`，从 confirm 节点继续，进 `execute_sensitive` 真执行——**前面编排/子 Agent 的全部 LLM 调用零重跑**（实测 26ms）
- 命中 `CANCEL_PATTERN` → `resume=False`，图结束，回"已取消"
- **其他任何消息 = 隐式取消** pending 后当新对话正常处理（防止用户改口后旧操作还留着；TTL 60min 兜底）


**为什么 interrupt() 不直接埋在子 Agent 代码里（关键设计决策）**：

LangGraph 官方确认 `interrupt()` 可以在节点内任意深度的嵌套函数里调用（包括工具函数内，靠异常沿调用栈传播），所以技术上可行。但恢复语义是**整个节点从头重新执行**——子 Agent 代码属于 orchestrator 节点，恢复时会重跑规划 LLM（TaskPlan 可能变化）、重跑子 Agent，成本高且不确定。把 interrupt 放在独立的轻量 confirm 节点里，恢复时只重放"读 state + interrupt() 返回 resume 值"这几行，零 LLM 成本。

**四个加分点**：

1. 真执行点是**唯一**的 `execute_sensitive` 节点，且复用 `retry_tool_call`——熔断/重试/BusinessError（业务拒绝不计熔断）语义在全链路一致；它也是操作事实（facts store）唯一可靠采集点，因为子 Agent 阶段根本还没产生真实结果，交给 Summary LLM 复述可能失真、会话历史窗口外还会被压缩掉
2. 确认/取消意图用**整句正则白名单判定，不用 LLM**——确定性、零成本、不被注入话术骗过；"确认下订单状态"这类长句一律走隐式取消+正常处理
3. 挂起状态活在 **Redis checkpoint 而不是内存**——服务重启不丢、多实例共享；TTL 60min 自动过期，解决确认悬挂与检查点无限累积
4. `_session_lock(memory_id)` per-session asyncio.Lock 保证同会话串行，防并发消息与 pending 状态交错、防双 resume

这套与 LangGraph 官方 `human-in-the-loop` 模式（interrupt + checkpointer + Command resume）完全同构，面试可直接报这个术语。

---

#### LangGraph 的 checkpoint 里存什么？什么时候写？

**A：** 一条 checkpoint 两部分：

**① 本体（世界状态）**：

| 字段 | 内容 |
|------|------|
| `channel_values` | 每个 State 字段的当前值（messages、task_results、pending_action 等），即"图的完整世界状态" |
| `channel_versions` + `versions_seen` | 通道版本号 vs 各节点已见版本 → 节点该不该跑是纯版本脏检查，恢复不靠内存标志 |
| `id` / `ts` / `pending_sends` | 单调递增可排序 ID、时间戳、跨节点 Send 消息 |

**② metadata（来历）**：`source`（input/loop/update/fork）、`step`（superstep 序号）、`parents`（父指针，快照串成可回放树）、`writes`（本步各节点写了什么）。

三个易错点：**只存增量**——没变的通道存引用不重复序列化；**messages 只有当前轮**——历史在会话层自己管；**不存代码对象**——LLM/工具实例不在 state 里。

**写入时机**：每个 superstep 结束、下一步开始前原子落一条。四类触发——ainvoke 入口（source=input）；每步写完汇总应用（source=loop，节点跑一半不落盘，原子单位是整个 superstep）；外部 updateState（source=update）；并行任务各自完成时先写 pending write（`checkpoint_write:` 键），集齐再合并——所以 5 个并行任务死 1 个，恢复只补跑失败的。

**RedisSaver 的三类 key**（以 memory_id 为 thread_id）：

```
checkpoint:{thread}:{ns}:{id}          快照本体（Hash：checkpoint+metadata 两 field）
checkpoint_latest:{thread}:{ns}        最新快照指针 → 恢复时 O(1) 直取，不必排序查
checkpoint_write:{thread}:{ns}:{id}:{task}:{idx}   superstep 进行中的半成品写入
```

对项目的意义：confirm 流程的 interrupt 把 pending_action 定格在 checkpoint 里，跨请求、跨重启靠 `Command(resume)` 从 latest 指针续跑；代价是 **state 字段活过请求边界**——primary 所有返回路径必须显式把 pending_action 置 None，否则下一轮被残留值误路由。另注意这三类 key 默认无 TTL，checkpoint 流只增不减，需清理策略。

> 面试话术："checkpoint 存两半：channel_values 是 State 各字段的当前值，加版本脏检查；metadata 记来源、步号、父指针，把快照串成可回放的树。写入的原子单位是 superstep——并行节点的输出先以 pending write 单独落盘、集齐后合并，所以 interrupt 暂停、进程重启后都能从最新 checkpoint + 未合并 writes 精确续跑。我们代码里'pending_action 必须显式清 None'就是这套持久语义的代价。"

---

#### 服务执行到一半崩溃了怎么办？已完成的任务会记录吗、会回滚吗？

**A：** **所有敏感操作必经 confirm 节点**（子 Agent 遇到敏感工具不执行，只产出 `__PENDING_CONFIRMATION__` 标记；真实工具调用只存在于用户确认之后的 execute_sensitive 节点）。这个前提把崩溃风险切成两段——

**确认之前崩溃：没有幂等风险。** 操作类工具一次都没被调用过，不存在"半截下单"、不存在未授权动作。挂着的只是一条待确认的 pending_action，它随 checkpoint 持久化，用户确认时正常走 execute_sensitive，什么都不用补救。

**execute_sensitive 执行时崩溃：唯一的失忆窗口。** 此时 Java/DB 侧已经真实执行（扣款/落预约），但节点没跑完——结果没进 checkpoint、facts 没记录、单号永远回不来了。用户再次确认触发节点重放：Agent 侧对"上次执行过没有"一无所知，盲目重调就是**真实二次扣款**（普通券一人多单，服务端没有去重），查无此事就是失忆。

**解法 = UUID 幂等键 + 调 Java 前先 Redis SETNX**（键语义：**每次确认尝试唯一，同次重放稳定**）——

- **requestId 铸造**（`app/agents/primary.py`）：pending_action 晋升时生成 UUID + created_at，随 orchestrator 的 checkpoint 持久化——崩溃重放时 execute_sensitive 从 state 恢复出**同一个键**；两次合法购买在不同轮次各自铸造新键，不会互相误伤；
- **调 Java 前先 SETNX**（`app/services/execution_guard.py`）：购买前先抢 Redis 执行标记（独立于 checkpoint 的锚点，"本地先落库"思想），调用**正常返回即释放**——只有崩溃才会遗留标记，所以"标记存在"精确等价于"存在一次结果未知的执行尝试"。重放路径抢不到标记即知上次结果未知，**不盲调**（`builder.py::_recover_purchase_result`）：时间窗反查 [铸造时刻, 现在] 内该用户该券的订单（`voucher_repo.find_order_since`），查到则如实返回原单号（找回来的就是丢失那次执行的单），查不到（调用可能在途）返回"可能已在处理中，请核实"。原则：**宁可输出模糊态，不自动产生第二笔扣款**；
- **预约（已根治，全栈自治）**：requestId 同源透传，`reservation` 表加唯一索引（`scripts/sql_reservation_request_id.sql`），repo（`app/repositories/reservation_repo.py`）INSERT 命中冲突时取回原预约返回（幂等取回，非报错）——重放天然安全，零失忆零重复；
- **Java 契约（待办，根治购买）**：tb_voucher_order 加 request_id 唯一索引、冲突返回原单号。页面下单与 Agent 下单共用同一契约（两渠道各铸各的 UUID，键的生命周期是"一次尝试"而非"一个商品"），页面侧的双击/重试双扣款顺带治好。

验证：`scripts/verify_sensitive_replay.py` 18 条断言全过（护栏 SETNX 语义 / repo 幂等取回 / 反查命中与未命中 / execute_sensitive_node 重放路径端到端——工具零调用、结果找回、事实记录、标记释放）。边界：护栏 fail-open（Redis 故障放行，Redis 同时承载 checkpoint，不可用时图本就无法执行）与反查不中的在途不确定窗口（拒绝重调，提示用户核实），根治等 Java 契约落地。

> 面试话术："敏感操作必经 confirm 节点，所以确认前崩溃连操作类工具都没执行过，没有幂等风险；唯一的风险窗口是 execute_sensitive 执行中——外部已扣款、结果未落盘，重放既失忆又可能二次扣款。我的解法两件套：requestId 用 UUID 在确认尝试铸造、随 checkpoint 存活，保证'同尝试同键、异尝试异键'；调 Java 前先 Redis SETNX 执行标记，重放见标记就不盲调，反查时间窗内的订单找回原单号，查不到宁可提示用户核实也不重调——宁可输出模糊态，不自动二次扣款。"

---

#### 输出层安全防护（Holdback 缓冲 + 正则检查）

**改进原因**：面试问题"你的 Agent 输出端有没有做安全检查？"、"LLM 回复泄露了 system prompt 怎么办？"、"流式输出怎么做内容审查？"

**问题**：项目原有 Guardrail 只做输入侧检查，输出侧无防护。LLM 可能在回复中泄露 system prompt 片段、内部技术信息（Redis/MySQL 连接地址、API Key）、用户 PII（手机号、身份证号）。

**方案设计**：采用 Holdback 缓冲区方案——流式输出时扣留最后 50 个字符作为安全检查窗口，所有到达用户的字符都经过正则检查。

流式输出流程：

1. LLM 逐 chunk 生成内容，累积到 holdback_buffer
2. buffer 超过 50 字符时，取出前面的安全部分做正则检查
3. 检查通过 → 输出给用户，保留后 50 字符继续缓冲
4. 检查命中 → 停止输出，替换为安全提示
5. 流结束后对剩余 buffer 做最后一次检查

正则检查规则（`app/guardrails/output_checker.py`）：

| 规则类别           | 严重程度 | 示例                                     |
| ------------------ | -------- | ---------------------------------------- |
| System prompt 泄漏 | high     | "你是智能客服的编排者"、"可用的子 Agent" |
| 内部技术信息       | high     | redis://、localhost:、sk-xxx、Traceback  |
| PII 敏感信息       | medium   | 手机号、身份证号、银行卡号、邮箱         |

**面试话术**：

> "输出端我用的是 Holdback 缓冲区方案——流式输出时不直接发给用户，而是扣留最后 50 个字符作为安全检查窗口。每次 buffer 累积超过 50 字符，就对前面的安全部分做正则检查，检查通过才输出。这样所有到达用户的内容都经过了检查，同时只增加了最后 50 个字符的延迟，用户基本无感知。检查规则覆盖三类：system prompt 泄漏、内部技术信息泄露、PII 敏感信息泄露。选正则而不是 LLM 判断，是因为流式场景下需要亚毫秒级的检查速度，正则 <1ms 完全满足。这是纵深防御的最后一道防线——输入侧 Guardrail 拦截大部分恶意请求，输出侧 Guardrail 兜底防止 LLM 偶尔的泄漏。"

### 三、上下文管理

#### 会话历史压缩优化（滑动窗口 + 分布式锁）

**改进原因**：

- 前端需要显示所有对话历史
- LLM 上下文需要压缩（避免 token 过多）
- 原方案每次保存消息都触发压缩，导致重复压缩

**实现方案**：

- **滑动窗口**：窗口内消息数 = 当前消息数 - 上次压缩标记位置
- **触发条件**：窗口内消息数 > 20 时触发压缩
- **压缩范围**：最近 8 条消息之外的消息（压缩 12 条，保留 8 条）
- **压缩方式**：调用 LLM 生成摘要
- **分布式锁**：使用 Redis SET NX 防止重复压缩

**核心设计**：

| 组件                        | 说明                                 |
| --------------------------- | ------------------------------------ |
| `mark:{memory_id}`        | 压缩标记位置（压缩后最老的消息位置） |
| `summary:{memory_id}`     | 摘要                                 |
| `compressing:{memory_id}` | 分布式锁（30秒过期）                 |

**核心流程**：

```
消息数 = 25，标记位置 = 0
窗口内消息数 = 25 - 0 = 25 > 20，触发压缩

压缩：
- 压缩范围：第 1-17 条消息（最近 8 条之外）
- 保留：第 18-25 条消息（最近 8 条）
- 生成摘要：调用 LLM
- 标记位置 = 17

发送给 LLM 的上下文：摘要 + 最近 8 条消息
前端显示：所有对话历史（Redis 中不删除）
```

**关键代码**：

```python
# save_message 中的压缩触发逻辑
mark_position = int(await self.redis.get(f"mark:{memory_id}") or 0)
window_count = len(history) - mark_position

if window_count > 20:
    # 使用 Redis SET NX 实现分布式锁（原子操作）
    lock_acquired = await self.redis.set(
        f"compressing:{memory_id}", "1", nx=True, ex=30
    )
    if lock_acquired:
        asyncio.create_task(self._compress_with_lock(memory_id))

# compress_if_needed 中的压缩逻辑
compress_end = len(history) - 8
compress_messages = history[mark_position:compress_end]

# 增量压缩
existing_summary = await self.get_summary(memory_id)
new_summary = await generate_summary_fn(compress_messages)

if existing_summary:
    summary = f"{existing_summary}\n{new_summary}"
else:
    summary = new_summary

# 保存摘要和更新标记
await self.save_summary(memory_id, summary, len(compress_messages))
await self.redis.set(f"mark:{memory_id}", compress_end)
```

**面试话术**：

> "会话历史压缩采用滑动窗口 + 分布式锁方案。前端需要显示所有对话历史，但 LLM 上下文需要压缩。设计思路是：记录压缩标记位置，窗口内消息数 = 当前消息数 - 标记位置。当窗口内消息数超过 20 时触发压缩，压缩最近 8 条消息之外的消息，调用 LLM 生成摘要。使用 Redis SET NX 实现分布式锁，防止重复压缩。发送给 LLM 的上下文是摘要 + 最近 8 条消息，前端显示完整对话历史。"

---

#### 操作事实原样保留（窗口外独立区）

**改进原因**：

- 购买优惠券、创建预约的返回是**客观业务事实**（订单号、预约号、金额、时间），但在普通链路里会经历两次有损处理：
  ① 只有 Summary LLM 的「复述稿」进会话历史，原始工具返回值从未持久化，单号可能被改写或省略；
  ② 窗口外消息还会被再压缩成不超过 200 字的摘要（prompt 字数预算），单号细节在第二次有损压缩中必然丢
- 用户下一轮问"刚才那个订单号是多少"，系统已经答不出来

**实现方案**（`app/services/facts.py`）：

| 设计点    | 做法                                                                                                    | 动机                                                                                              |
| ------ | ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| 采集点    | `graph/builder.py` execute_sensitive_node，仅 ok=True 时记录                                               | 操作工具在子 Agent 里只产出 `__PENDING_CONFIRMATION__` 标记，真实结果只在这个节点存在过一次；错误文本是叙事不是事实                     |
| 存储结构   | 独立 Redis Hash `facts:{memory_id}`，每条事实一个 field，HSETNX 原子写                                             | summary JSON 是 read-modify-write 且压缩跑在 asyncio.create_task 里，塞进去会与采集重叠→后写者覆盖、静默丢失；单命令原子写从根上避开竞态 |
| 与窗口的关系 | 不进消息数组，不占 history 下标                                                                                  | 滑动窗口（>20 触发、保留 8 条）在结构上无法把它挤出去——"保留在上下文窗口中"= 给事实一个窗口管不着的独立区                                     |
| 收录判据   | 只收 ACTION_FACT_TOOLS（purchase_voucher / create_reservation），查询类不收                                     | 判据是"丢失后无法从剩余上下文重新推出"；查询类可重查，纳入只挤占配额                                                             |
| 幂等键    | 优先解析返回里的业务单号（`订单号：`→`purchase_voucher:xxx`），解析不到回退参数 SHA1                                             | 宁可多存一条，也不能因单号缺失（落到"未知"）把多笔订单误合并                                                                 |
| 注入     | `conversation.get_conversation_context()` 唯一出口，同时填 primary/summary 两个 prompt 的 {conversation_context} | 两个 Agent 自动获得事实区，无需改 state 或模板；表头显式约束"必须原样引用，不得改写或省略"                                           |
| 配额     | 超 FACTS_MAX_ENTRIES 按 ts 淘汰最旧；每次写入 EXPIRE 续期对齐会话生命周期                                                  | 补独立存储不同寿命的代价；HSETNX 保最早 ts，淘汰顺序稳定                                                               |

**存的是全文，不是结构化元数据**：payload 只有 `{tool, text, ts}` 三个字段，`text` 是工具原始返回**全文逐字保真**（换行仅缩进两格），`tool/ts` 才是薄信封（标来源、排序、淘汰）。不做结构化抽取（拆 order_id/amount 字段）是故意的——这条链路要解决的原始问题就是"LLM 转述导致失真"，再引入一层解析/LLM 加工等于换个姿势重犯同样的错。

**核心代码**：

```python
# facts.py — 写入：单命令原子，重复即跳过（保留最早 ts）
created = await self.redis.hsetnx(self._key(memory_id), field, payload)
await self.redis.expire(self._key(memory_id), settings.FACTS_TTL)

# 注入渲染：原文一字不改，表头防改写
lines = [RENDER_HEADER]  # "…订单号、预约号、金额、时间等标识必须原样引用，不得改写或省略"
for item in entries:
    text = str(item.get("text", "")).replace("\n", "\n  ")
    lines.append(f"- [{item.get('tool', 'unknown')}] {text}")
```

**面试话术**：

> "上下文压缩对有损记忆是故意的，但对订单号这类事实是有 bug 的。我把事实从消息流里抽出来单独放 Redis Hash：采集点在确认执行节点——真实结果唯一存在的地方；写入用 HSETNX 原子操作，避开与异步摘要压缩的 read-modify-write 竞态；不占消息数组下标，所以滑动窗口在结构上挤不掉它；注入走会话上下文唯一出口，自动进两个 Agent 的 prompt，带'原样引用不得改写'的约束。收不收录的判据一条：丢失后能否从剩余上下文重新推出。存的内容是原始全文不是结构化元数据——因为要防的就是加工失真，再加一层抽取等于重犯同样的错。"

---

### 四、长期记忆管理

#### 集成 Mem0 记忆管理框架

**完整存取流程（一轮对话的时间线）**：

```
【轮初 · 读】 Orchestrator 收到用户消息
   └─ search_memories(user_id, query=本轮消息)：to_thread(memory.search) 按需语义检索
      → threshold 过滤（无关话题轮零注入）→ 按 (类别,关键词) 分组做时间消解
        （同关键词多条取 updated_at/created_at 最新，新格式恒胜旧格式）
      → 注入主 Agent system prompt 的画像段 → 开始任务拆解
【会话中】 子 Agent 执行工具（RAG/券/预约），记忆层不参与
【轮末 · 触发】 回复输出完 → save_message 落库
   → create_task(analyze_chat_history)（异步，用户零等待）
【轮末提取 · 增量】
   ① claim 锁：SET mem0_lock:{user_id} <token> NX EX 120，抢不到 → 直接放弃
   ② 读游标：new = history[mem0_marker.last_count:]；不足 3 条本轮不分析
   ③ 防御：last_count > len(history)（清空/重建）→ 归零重扫
      过滤：早于 mem0_reset 时间戳的消息跳过
   ④ to_thread(memory.add(new))：Mem0 按自定义 prompt 单程提取"类别+极性+关键词"偏好
      → MD5 hash 精确去重 → 纯追加批量写入（新算法无写时 UPDATE/DELETE，冲突留给读侧）
   ⑤ 成功 → 推进游标；失败 → 不推进，下轮重试同一切片（at-least-once）
   ⑥ finally → Lua 比对 token 释放锁
【治理】 每类配额 5 条（_enforce_category_limit），超额按时间淘汰
【治理】 _enforce_profile_hygiene（提取成功后）：同 (类别,关键词) 归并只留最新 + 单用户 100 条封顶淘汰最旧；
         每条 add 带 expiration_date=+90d（mem0 原生 TTL，到期检索端自动隐藏、不占配额视野）
【下一轮开始】 读阶段即可见本轮提取——"这轮说不吃香菜，下轮推荐生效"
```

**存储内容**：

- 每条记忆是一句 LLM 提取的偏好陈述，**强制"类别+极性+关键词"三段式**：`[饮品] 喜欢:拿铁` / `[美食] 不喜欢:香菜` / `[店铺] 喜欢:瑞幸`。格式由 Mem0 的 custom_instructions 约束，提取端就产出可分组、可判冲突的结构；极性是硬要求——新算法纯追加不覆盖，"喜欢/不喜欢日料"两条并存，冲突表达全靠这个字段
- **库治理**（挂在写路提取成功后，`_enforce_profile_hygiene`）：同 (类别,关键词) 归并只留最新（旧偏好不再占位）+ 单用户 100 条封顶淘汰最旧；每条记忆 `add(expiration_date=+90d)` 带原生 TTL，到期被 mem0 检索端过滤、不占配额视野。检索端另有 (类别,关键词) 分组时间消解——读侧择新是冲突处理第一道闸，写侧归并是最终收敛，两层互为兜底
- **边界**：订单号/预约号这类业务凭证**不进 Mem0**——它们是"必须原样复述的事实"，走独立的操作事实通道（见上下文管理章节）；Mem0 只存"适合被概括的偏好语义"

**提取时机（写路径）**：每轮对话回复完成、助手消息落库后，`asyncio.create_task(analyze_chat_history)` **异步触发，不阻塞用户响应**；Mem0 是同步阻塞库，内部所有 SDK 调用（add/search）经 `asyncio.to_thread` 卸载到工作线程，事件循环在等待期间继续服务其他请求。

**写路径增量细节（对应流程 ①-⑥）**：

| 机制 | 实现 | 解决什么 |
| --- | --- | --- |
| 计数游标 | Redis `mem0_marker:{user_id}` 记录已分析条数，每轮只切 `history[last_count:]` | 不重复分析旧消息 |
| 游标推进时机 | **`memory.add` 成功后才推进** | add 失败/崩溃 → 下轮重试同一切片（at-least-once），重复由 hash 精确去重 + 读侧时间消解吸收——漏提比重复提糟糕 |
| 游标失效防御 | 历史数组 append-only（摘要压缩只推进自己的 position 游标，不删消息），失效源仅"清空历史/TTL 过期"两种 → `last_count > current` 时归零重扫，且 clearChatHistory 主动删 marker | 防残留游标让提取停摆 |
| 并发防重 | 任务开头 `SET mem0_lock:{user_id} token NX EX 120` **原子预占**，抢不到直接放弃（不排队）；finally 中 Lua 比对 token 才删锁 | 见下 |

**读细节（轮初，对应流程第一步）**：每轮请求开始，Orchestrator 统一调 `search_memories(user_id, query=本轮消息)`：`to_thread(memory.search)` **按需语义检索**（threshold 起步 0.30）→ 结果按 (类别,关键词) 分组做**时间消解**（同关键词多条取最新、新格式恒胜旧格式）→ 注入画像段。演进注记：早期是"全量 get_all + 配额截断"（小规模下完备性优先），记忆格式升级带极性后改为按需检索——画像从"常驻事实"变"检索命中"，换来无关话题轮零噪声注入；代价是弱相关偏好可能漏注入，因此阈值宁松勿严、任何检索异常一律静默降级为空注入（漏注入只是平淡，阻断对话是事故）。

**面试话术**：

> "长期记忆用 Mem0 做，按一轮对话的生命周期讲：轮初 Orchestrator 拿本轮用户消息做**按需语义检索**（threshold 过滤 + 同关键词时间消解取最新，异常静默降级空注入）画像注入 prompt；轮末回复落库后起异步任务做增量提取——Redis 游标切新消息、Mem0 单程 LLM 提取'类别+极性+关键词'三段式偏好、**纯追加入库**（新算法没有写时 UPDATE/DELETE，冲突表达靠极性、消解靠读侧择新）、**提取成功后游标才推进**（at-least-once，重复交给 hash 去重，漏提比重复提糟糕）；全程 Mem0 的同步调用用 to_thread 隔离，不冻结事件循环。并发踩过真实的坑：连发消息时多个任务读到同一游标基线重复分析（日志 8,6,6），修复是 SET NX EX 原子预占做 single-flight——抢不到锁直接放弃而不是排队，被弃的消息由游标切片天然并段；锁本身三道关：NX 防并发窗口、TTL 防进程崩溃、token 比对释放防锁过期后误删新主的锁。"

---

#### 长期记忆里的冲突记忆怎么处理？

**问题背景**：用户先说"喜欢日料"、两周后说"不喜欢日料"——画像里出现相反断言，注入哪条、留哪条、要不要合并，是长期记忆系统的必考题。

**读侧：应用层时间消解。** OSS 版检索融合分**不含时间信号**（官方宣称的 Temporal Reasoning 是 Platform 专有优化），所以按 (类别, 关键词) 分组、组内取 `updated_at/created_at` 最新一条注入——"喜欢→不喜欢"并存时用户看到的是最新态度；旧格式无极性条在新格式面前恒败（存量数据兼容）。"不喜欢:X"**照常注入**：禁忌信息比偏好更重要。

**配套的治理**：写路提取成功后跑画像维护——同键归并（旧极性条物理删除，不再占配额）+ 单用户 100 条封顶淘汰最旧；每条记忆 add 时带 `expiration_date=+90d`，mem0 原生 TTL 到期自动从检索中消失——**遗忘是功能不是缺陷**：偏好的真相随时间衰减，忘记是平淡，记错是事故。threshold 过滤让无关话题轮零注入。

> **面试话术**："冲突我们分两端处理：写端保证冲突**可表达**（类别+极性+关键词三段式，mem0 新算法纯追加、写侧零覆盖，格式是唯一的事实载体）；读端负责**消解**——按关键词分组取时间最新，因为 OSS 检索分数没有 temporal 信号，时间推理是平台版特性，我们用 created_at 自己补上。这个设计还闭环了一个真实 Bug：旧格式无'不喜欢'的表达位，负面偏好被 hash 去重静默吞掉，看起来像'情感分析错误'，其实是 schema 缺陷。"

---
### 五、错误处理与稳定性

#### 工具系统与容错机制

**改进原因**：面试高频问题"工具调用失败，你的 Agent 是直接报错，还是会做恢复和重试？"、"熔断器是怎么实现的？"

**实现方案**：

- `app/agents/retry.py`：工具调用重试 + PyBreaker 熔断器（工具层，不重新调 LLM）
- `app/agents/error_handler.py`：错误分类（Executor 层）
- `app/agents/executor.py`：TaskExecutor 接入熔断检查和依赖跳过
- `app/services/circuit_breaker.py`：PyBreaker 熔断器服务（支持半开状态、Redis 持久化、LangSmith 监听器）

**核心功能**：

| 功能             | 层级                 | 说明                                                   |
| ---------------- | -------------------- | ------------------------------------------------------ |
| 按工具类型重试   | 工具层（retry.py）   | 查询类工具可重试，操作类工具不重试                     |
| 按错误类型重试   | 工具层（retry.py）   | 只有 TIMEOUT/NETWORK 错误会重试                        |
| 指数退避         | 工具层               | 重试延迟 = 0.5s × 1.5^attempt，最大 10s               |
| PyBreaker 熔断器 | 工具层 + Executor 层 | 同一 Agent 连续失败 3 次 → 熔断 60s，支持半开状态探测 |
| 依赖任务跳过     | Executor 层          | 前置任务失败时，依赖它的后续任务自动跳过               |
| LangSmith 监听器 | 服务层               | 熔断状态变更自动记录到 LangSmith                       |

**为什么用指数退避（退避因子 1.5）**：

固定延迟重试的问题：如果服务故障，所有请求都在相同间隔后重试，会导致"重试风暴"，加重服务负担。

指数退避的解决方案：每次重试延迟增加 1.5 倍，给服务恢复时间。

```
固定延迟：1s → 1s → 1s → 1s（服务压力大）
指数退避：0.5s → 0.75s → 1.125s → 1.69s（服务压力小）
```

为什么选择 1.5 作为退避因子：

- 1.0 = 固定延迟（无退避）
- 1.5 = 温和退避（推荐）
- 2.0 = 激进退避（延迟增加太快）

1.5 平衡了重试速度和服务恢复时间。

**按工具类型选择性重试**：

| 工具类型         | 重试策略  | 原因                         | 示例                                 |
| ---------------- | --------- | ---------------------------- | ------------------------------------ |
| 查询类（QUERY）  | ✅ 可重试 | 幂等操作，重复执行无副作用   | find_shop, rag_search                |
| 操作类（ACTION） | ❌ 不重试 | 非幂等，重试可能导致重复执行 | purchase_voucher, create_reservation |

**错误类型判断逻辑**：

错误分类采用双重判断机制：先按异常类型判断，再按错误消息关键词判断。

```python
def classify_error(self, error: Exception) -> ErrorType:
    # 1. 按异常类型判断（优先）
    error_type = type(error)
    if error_type in self.RETRYABLE_ERRORS:
        return self.RETRYABLE_ERRORS[error_type]

    # 2. 按错误消息关键词判断（兜底）
    error_msg = str(error).lower()
    if "timeout" in error_msg:
        return ErrorType.TIMEOUT
    if "connection" in error_msg or "network" in error_msg:
        return ErrorType.NETWORK
    if "invalid" in error_msg or "parameter" in error_msg:
        return ErrorType.PARAMETER

    return ErrorType.UNKNOWN
```

**可重试错误类型**：

| 异常类型                   | 错误类型 | 说明             |
| -------------------------- | -------- | ---------------- |
| `asyncio.TimeoutError`   | TIMEOUT  | 操作超时         |
| `ConnectionError`        | NETWORK  | 连接失败         |
| `ConnectionResetError`   | NETWORK  | 连接被重置       |
| `ConnectionRefusedError` | NETWORK  | 连接被拒绝       |
| `OSError`                | NETWORK  | 操作系统网络错误 |

**不可重试错误类型**：

| 异常类型       | 错误类型  | 说明         |
| -------------- | --------- | ------------ |
| `ValueError` | PARAMETER | 参数值错误   |
| `KeyError`   | PARAMETER | 字典键错误   |
| `TypeError`  | PARAMETER | 参数类型错误 |

**为什么需要双重判断**：

- 异常类型判断更准确（Python 标准异常）
- 消息关键词判断作为兜底（有些库抛出通用 Exception，但消息中包含具体信息）

**为什么重试在工具层而不是 Executor 层**：

```
❌ 之前（Executor 层重试）：
工具异常 → 子 Agent 异常 → Executor 重试整个 Agent → 重新调 LLM + 工具
                                                    ↑ 浪费 LLM 调用

✅ 之后（工具层重试）：
工具异常 → retry_tool_call() 重试工具 → 不重新调 LLM
```

**PyBreaker 熔断器配置**：

| 配置项        | 值        | 说明               |
| ------------- | --------- | ------------------ |
| fail_max      | 3         | 触发熔断的失败次数 |
| reset_timeout | 60s       | 熔断恢复时间       |
| 状态持久化    | Redis     | 多实例共享熔断状态 |
| 监听器        | LangSmith | 状态变更自动记录   |

**PyBreaker 三种状态**：

```
CLOSED（关闭）──(失败达阈值)──▶ OPEN（打开）──(超时)──▶ HALF_OPEN（半开）
    ▲                                                        │
    │                                                        │
    └──────────────────(成功)────────────────────────────────┘
                                                    │
                                                    ▼
                                                (失败)
                                                    │
                                                    ▼
                                              OPEN（打开）
```

**执行流程**：

```
子 Agent.run(goal)
    │
    ├─ LLM 调用（LangChain 内置重试 max_retries=3）
    │
    └─ 工具调用
        └─ retry_tool_call(tool.ainvoke, args, tool_name)
            │
            ├─ 检查熔断器状态（breaker.current_state == OPEN?）
            │   └─ 熔断中 → 抛出 CircuitBreakerError
            │
            ├─ 操作类工具（purchase_voucher 等）→ 直接执行，不重试
            │
            └─ 查询类工具（find_shop 等）
                ├─ 成功 → breaker.state.on_success() → 返回结果
                ├─ TIMEOUT/NETWORK → breaker.state.on_failure() → 指数退避重试（最多 2 次）
                └─ 其他异常 → breaker.state.on_failure() → 直接抛出

Executor.execute_single_task(task)
    │
    ├─ 检查前置任务是否失败 → 失败则跳过
    ├─ 检查熔断器状态（breaker.current_state == OPEN?）
    │   └─ 熔断中 → 返回"服务暂时不可用"
    │
    └─ 执行 agent.run(goal)
        ├─ 成功 → 返回结果
        └─ 异常 → error_handler.handle_error() → 返回友好错误信息
```

**面试话术**：

> "工具容错分两层：工具层负责重试和熔断，按工具类型和错误类型双重判断——查询类工具（幂等）网络/超时错误时重试，操作类工具（非幂等）不重试避免重复执行。熔断器使用 PyBreaker 库实现，支持三种状态：关闭、打开、半开——连续失败 3 次触发熔断，60 秒后进入半开状态，允许一个请求探测是否恢复。状态通过 Redis 持久化，多实例部署时共享。同时集成了 LangSmith 监听器，熔断状态变更自动记录到 LangSmith，方便监控和排查。Executor 层负责依赖管理：前置任务失败时依赖它的后续任务自动跳过，防止级联失败。"

---

#### 全链路错误处理与降级机制

**改进原因**：面试高频问题"你的 Agent 如何处理工具调用失败？"、"每个环节失败了怎么办？"

**全链路错误处理架构**：

```
用户请求
    ↓
① Guardrail 安全检查
    ↓ 通过
② Primary Agent 任务编排
    ↓ TaskPlan
③ TaskExecutor 调度
    ↓ 并行/串行
④ 子 Agent LLM 调用
    ↓ tool_calls
⑤ 工具调用（retry）
    ↓ 返回结果
⑥ Summary LLM 汇总
    ↓
响应用户
```

**各阶段错误处理策略**：

| 阶段             | 失败场景            | 处理策略                      | 设计原则                       |
| ---------------- | ------------------- | ----------------------------- | ------------------------------ |
| ① Guardrail     | LLM 调用失败        | 默认放行（fail-open）         | 宁可漏检不误杀，保证服务可用   |
| ② Primary Agent | 结构化输出失败      | 重试 3 次，每次间隔 1s        | 结构化输出偶发失败，重试可恢复 |
| ② Primary Agent | 客户端断开          | 直接返回"请求已取消"          | 资源释放，不继续执行           |
| ③ TaskExecutor  | 未知 Agent          | 返回错误信息                  | 配置问题，直接暴露             |
| ③ TaskExecutor  | 熔断器打开          | 返回"服务暂时不可用"          | 保护下游服务                   |
| ③ TaskExecutor  | 前置任务失败        | 依赖任务自动跳过              | 避免无意义执行                 |
| ④ 子 Agent      | LLM 超时/异常       | 循环最多 3 轮后返回"处理超时" | 防止无限循环                   |
| ⑤ 工具调用      | 查询类工具网络/超时 | 指数退避重试（最多 2 次）     | 查询幂等，重试安全             |
| ⑤ 工具调用      | 操作类工具失败      | 不重试，直接返回错误          | 操作非幂等，避免重复执行       |
| ⑥ Summary LLM   | 调用失败            | 返回"服务暂时繁忙"            | 保证用户有响应                 |
| ⑥ Summary LLM   | 未初始化            | 直接拼接任务结果              | 格式不友好但有内容             |

**核心代码**：

```python
# retry.py - 工具层重试（按工具类型选择性重试）
QUERY_TOOLS = {"find_shop", "rag_search", "find_voucher_by_shop", ...}  # 幂等，可重试
ACTION_TOOLS = {"purchase_voucher", "create_reservation"}  # 非幂等，不重试

async def retry_tool_call(tool_func, args, tool_name, max_retries=2):
    if tool_name in ACTION_TOOLS:
        return await tool_func(args)  # 操作类不重试
    for attempt in range(max_retries + 1):
        try:
            return await tool_func(args)
        except RETRYABLE_EXCEPTIONS as e:  # TimeoutError, ConnectionError
            delay = min(0.5 * (1.5 ** attempt), 10.0)  # 指数退避
            await asyncio.sleep(delay)
    raise last_exception

# error_handler.py - Executor 层错误分类 + 熔断器
class WorkerErrorHandler:
    CIRCUIT_BREAKER_THRESHOLD = 5   # 连续失败 5 次
    CIRCUIT_BREAKER_TIMEOUT = 60    # 熔断 60 秒

    def handle_error(self, worker_name, error, attempt, max_retries):
        error_type = self.classify_error(error)  # TIMEOUT/NETWORK/PARAMETER/BUSINESS/UNKNOWN
        self.record_failure(worker_name)  # 累加失败计数，检查熔断
        should_retry = self.should_retry(error_type, worker_name) and attempt < max_retries
        user_message = self.get_user_message(error_type, worker_name)  # 友好提示
        return ErrorResult(should_retry, error_type, user_message)

# guardrail/checker.py - Guardrail 失败时默认放行
async def check_guardrails(message):
    # 关键词快速检测 → LLM 兜底
    try:
        response = await llm.ainvoke(prompt)
        # 解析 JSON 判断安全性...
    except Exception as e:
        print(f"Guardrail check error: {e}")
    # 出错时默认放行
    return GuardrailResult(passed=True, check_type="error")

# chat.py - Summary LLM 三层降级
if summary_llm:
    try:
        async for chunk in summary_llm.astream([...]):
            yield text
    except Exception:
        yield "抱歉，服务暂时繁忙，请稍后再试。"  # 降级 1：错误提示
else:
    yield "\n\n".join([r["result"] for r in task_results])  # 降级 2：拼接结果
```

**熔断器机制**：

```
连续失败 5 次 → 熔断 60 秒 → 期间直接返回"服务不可用" → 60 秒后自动恢复
```

**依赖任务跳过**：

```
任务 0: knowledge（推荐火锅店）→ 失败
任务 1: reservation（预约火锅店，depends_on: 0）→ 自动跳过，返回"前置任务失败"
```

**面试话术**：

> "我设计了全链路错误处理机制，每个环节都有对应的降级策略。Guardrail 失败时默认放行，不影响正常服务；Primary Agent 结构化输出失败时重试 3 次；工具调用层按工具类型区分——查询类工具幂等可重试（指数退避），操作类工具不重试避免重复执行；Executor 层有熔断器，连续失败 5 次自动熔断 60 秒；Summary LLM 有三层降级：流式输出 → 拼接结果 → 错误提示。整体设计原则是：宁可降级也不能中断服务。"

---

#### PyBreaker 熔断器改造（半开状态 + Redis 持久化 + LangSmith 集成）

**改进原因**：面试高频问题"你的熔断器是怎么实现的？"、"熔断器支持半开状态吗？"、"多实例部署时熔断状态怎么共享？"

**问题发现**：

- 原有熔断器是手动实现的（字典 + 时间戳），存在以下问题：
  1. 不支持半开状态：超时后直接关闭，没有探测机制
  2. 状态不持久化：进程重启后熔断状态丢失
  3. 多实例不共享：负载均衡部署时，实例 A 熔断了，实例 B 还在继续请求
  4. 没有监控告警：熔断状态变更无法追踪

**实现方案**：

- 使用 PyBreaker 库替换手动熔断逻辑
- 集成 Redis 实现状态持久化（多实例共享）
- 集成 LangSmith 监听器，记录熔断状态变更
- 统一配置：3 次失败触发熔断，60 秒后尝试恢复

**核心代码**：

```python
# app/services/circuit_breaker.py - PyBreaker + Redis + LangSmith
import pybreaker
from datetime import datetime, timedelta, timezone
from pybreaker import CircuitRedisStorage

class LangSmithListener(pybreaker.CircuitBreakerListener):
    """LangSmith 监听器 - 记录熔断状态变更
    注意：PyBreaker 回调的钩子名是 state_change/failure/success，
    不是 on_state_change/on_failure/on_success（on_* 不会被调用）
    """
    def state_change(self, breaker, old_state, new_state):
        rt = get_current_run_tree()  # LangSmith 当前 trace
        if rt:
            rt.add_metadata({"circuit_breaker": {
                "name": breaker.name,
                "old_state": getattr(old_state, "name", old_state),
                "new_state": getattr(new_state, "name", new_state),
            }})
            rt.add_event("circuit_state_change", {...})

# Redis 状态持久化（多实例共享，按熔断器名称隔离 namespace）
# 注意：PyBreaker 的 CircuitRedisStorage 需要同步 Redis 客户端
def _create_redis_storage(namespace):
    import redis  # 同步 Redis 客户端，不是 redis.asyncio
    redis_client = redis.from_url(settings.REDIS_URL, decode_responses=False)
    return CircuitRedisStorage(pybreaker.STATE_CLOSED, redis_client, namespace=namespace)

# 预定义的熔断器配置（所有工具统一：3次失败触发，60秒恢复）
BREAKER_CONFIGS = {
    "shop_worker": BreakerConfig(fail_max=3, reset_timeout=60),
    "voucher_worker": BreakerConfig(fail_max=3, reset_timeout=60),
    "knowledge_worker": BreakerConfig(fail_max=3, reset_timeout=60),
    "reservation_worker": BreakerConfig(fail_max=3, reset_timeout=60),
}
```

```python
# app/services/circuit_breaker.py - 手动驱动状态机的三个入口
# PyBreaker 不支持原生 async（call_async 是 Tornado 协程），工具是协程函数，
# 不能用 breaker.call() 包装，因此调用前预检、调用后手动记录：

def ensure_breaker_allowed(breaker):
    """调用前预检：熔断中抛 CircuitBreakerError；冷却期已过则转半开、放行本请求做探测"""
    state = breaker.state
    if state.name != pybreaker.STATE_OPEN:
        return
    opened_at = breaker._state_storage.opened_at
    if opened_at and datetime.now(timezone.utc) < opened_at + timedelta(seconds=breaker.reset_timeout):
        raise pybreaker.CircuitBreakerError(f"Circuit breaker {breaker.name} is open")
    breaker.half_open()  # 冷却期已过，转半开，当前请求作为探测调用

def record_success(breaker):
    # 必须走 _handle_success()：它才会重置失败计数、在半开状态下关闭熔断器
    breaker.state._handle_success()

def record_failure(breaker, exc):
    # 必须走 _handle_error()：失败计数累加和触发熔断的逻辑都在它里面；
    # 达到阈值时 pybreaker 会抛 CircuitBreakerError，忽略它以保留原始业务异常
    try:
        breaker.state._handle_error(exc, reraise=False)
    except pybreaker.CircuitBreakerError:
        pass
```

```python
# app/agents/retry.py - 集成 PyBreaker（手动驱动状态机）
from app.services.circuit_breaker import (
    ensure_breaker_allowed, get_breaker, record_failure, record_success,
)

async def retry_tool_call(tool_func, args, tool_name, ...):
    breaker = get_breaker(TOOL_BREAKER_MAP.get(tool_name, "default"))

    # 调用前预检（含 OPEN -> HALF_OPEN 恢复转换），熔断中直接抛 CircuitBreakerError
    ensure_breaker_allowed(breaker)

    try:
        result = await tool_func(args)
        record_success(breaker)      # 记录成功（内部走 _handle_success，会重置计数）
        return result
    except Exception as e:
        record_failure(breaker, e)   # 记录失败（内部走 _handle_error，会计数并触发熔断）
        raise
```

**PyBreaker 三种状态**：

```
CLOSED（关闭）──(失败达阈值)──▶ OPEN（打开）──(冷却期过后，下一次请求预检)──▶ HALF_OPEN（半开）
    ▲                                                        │
    │                                                        │
    └──────────────────(探测成功)────────────────────────────┘
                                                    │
                                                    ▼
                                                (探测失败)
                                                    │
                                                    ▼
                                              OPEN（重新打开）
```

| 状态      | 说明     | 请求处理                       |
| --------- | -------- | ------------------------------ |
| CLOSED    | 正常状态 | 允许所有请求通过               |
| OPEN      | 熔断状态 | 拒绝所有请求，返回"服务不可用" |
| HALF_OPEN | 探测状态 | 只允许一个请求通过进行探测     |

**改造过程中遇到的问题和解决措施**：

| 问题                                                    | 原因                                                                                                 | 解决措施                                                                                                                                                  |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ModuleNotFoundError: No module named 'pybreaker'`    | 未安装 pybreaker 依赖                                                                                | `pip install pybreaker>=1.0.1`                                                                                                                          |
| `'coroutine' object has no attribute 'decode'`        | PyBreaker 的`call_async` 使用 Tornado 协程语法，不兼容 Python 原生 `async/await`                 | 手动实现熔断逻辑，不使用 PyBreaker 包装异步函数                                                                                                           |
| `'CircuitBreaker' object has no attribute 'is_alive'` | PyBreaker 没有`is_alive` 方法                                                                      | 使用`breaker.current_state == pybreaker.STATE_OPEN` 判断状态                                                                                            |
| `'CircuitBreaker' object has no attribute 'failure'`  | PyBreaker 没有`success()` 和 `failure()` 方法                                                    | 先尝试了`breaker.state.on_success()` / `on_failure()`，二次排查发现**并不生效**（见下表），最终封装 `_handle_success()` / `_handle_error()` |
| 开启 Redis 存储后工具调用失败                           | PyBreaker 的`CircuitRedisStorage` 需要同步 Redis 客户端，我们传入的是 `redis.asyncio` 异步客户端 | 使用`import redis`（同步客户端）而不是 `import redis.asyncio`                                                                                         |

**二次排查（编写验证用例时发现 4 个隐藏 Bug）**：

上述"解决方案"只是让代码不报错，实际功能全是失效的。写验证脚本直接驱动熔断器才发现：

```python
# 验证脚本的核心思路：不依赖"不报错"，直接驱动到目标状态断言
for i in range(10):
    breaker.state.on_failure(exc)          # 旧代码的记录方式
print(breaker._state_storage.counter)      # 输出 0 —— 计数根本没动，熔断器永远不会打开
```

| Bug                               | 现象                                                                                             | 根因                                                                                                                                                                                                                                                 | 修复方式                                                                                                                                                                                           |
| --------------------------------- | ------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ① 熔断器永远不会打开（最严重）   | 失败 10 次后`current_state` 仍是 closed，失败计数始终为 0                                      | pybreaker 1.x 重构了状态机：失败计数的累加在`_handle_error()` 内部（`_inc_counter()`），`on_failure()` 只负责"检查阈值、打开熔断"；直接调 `state.on_failure()` 不计数。同理 `on_success()` 不重置计数器（重置在 `_handle_success()` 里） | 新增`record_success()` / `record_failure()` 封装 `_handle_success()` / `_handle_error(exc, reraise=False)`；熔断触发时 pybreaker 抛的 CircuitBreakerError 被吞掉，保留原始业务异常给调用方 |
| ② 所有 Worker 共享同一个熔断状态 | Redis 里只有一组`pybreaker:state` key；shop 连续失败会让 voucher、knowledge 等所有工具一起被拒 | `CircuitRedisStorage` 创建时不传 `namespace` 参数，所有熔断器实例读写同一组 key，失败计数和状态互相污染                                                                                                                                          | 创建时传`namespace=熔断器名`，key 变成 `shop_worker:pybreaker:state` 等，各 Worker 完全隔离                                                                                                    |
| ③ LangSmith 监听器从未被调用     | 熔断状态变更时控制台无任何打印，LangSmith 上也无事件                                             | pybreaker 回调的钩子名是`state_change` / `failure` / `success`，监听器里写的 `on_state_change` / `on_failure` / `on_success` 永远不会被匹配到（继承基类空实现，静默失败）                                                                | 监听器方法重命名为`state_change` / `failure` / `success`                                                                                                                                     |
| ④ 熔断打开后永远无法恢复         | 一旦打开，60 秒后依然拒绝请求，永远卡在 OPEN                                                     | Redis 存储的`state` 只返回存储的字符串，不会自己转半开；原生 OPEN→HALF_OPEN 转换发生在 `before_call()` 里、且会同步执行传入的函数（async 工具用不了）；绕开 `call()` 后没有任何代码触发 `half_open()`                                       | 新增`ensure_breaker_allowed()` 预检：OPEN 且冷却期未过则抛 CircuitBreakerError；冷却期已过则调 `breaker.half_open()` 放行当前请求作为探测                                                      |

**修复后的验证**（全部通过）：

- 失败 3 次触发熔断：closed → open，重试循环停止冲击故障服务，原始异常保留
- Worker 隔离：shop 熔断时 voucher 的 key 和状态不受影响，工具正常执行
- 完整恢复闭环：closed → open →（冷却期）→ half-open → 探测成功 → closed
- 成功调用重置失败计数；监听器正常打印状态转换（closed -> open、open -> half-open、half-open -> closed）

**教训**：

1. **绕过第三方库的上层 API 直接调内部方法时，必须读源码确认方法职责**——"方法存在且不报错"不等于"方法干了你想的事"（`on_failure` 存在，但它只做阈值检查不做计数）
2. **集成第三方库后要写"驱动到目标状态"的验证用例**：触发熔断、恢复、隔离都要断言到位，"接口能跑通"和"功能真的生效"是两回事；这 4 个 Bug 全是静默失效，不验证永远发现不了
3. 注意库的大版本重构：pybreaker 1.x 的状态机与旧版文档差异很大，网上教程（包括 AI 生成的示例）大多基于旧版

**面试追问：冷却期内服务就已经恢复了怎么办？**

**核心答案**：固定冷却期的熔断器感知不到恢复——它不主动探测后端，冷却期内所有请求照样拒绝。第一个发现服务恢复的机会，是冷却期结束后的第一个请求（探测请求）。

```
t=0s    第 3 次失败，熔断打开（opened_at 记入 Redis）
t=5s    服务实际恢复了（熔断器不知道）
t=10s   用户请求 → 拒绝 ❌（工具根本没执行，明明能成功）
t=30s   用户请求 → 拒绝 ❌
t=61s   第一个请求 → 转半开、放行探测 → 成功 → 熔断闭合 ✅
```

最坏情况下，服务恢复后还要"多冤枉"冷却期的剩余时间（上例为 55 秒）。**这是刻意的设计权衡**：`reset_timeout` 越短感知恢复越快，但服务未恢复时探测流量越频繁地打到底层；越长保护性越好但恢复感知越慢，没有两全的值，只能按业务特性调。

**缓解方案**（已列入 TODO，暂未实施）：

| 方案                      | 做法                                                                                                                                                                     | 解决什么                                                           |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| ① 按工具类型差异化冷却期 | 查询类工具（find_shop、rag_search 等）幂等无害，冷却期配短（15-30 秒）快速试探恢复；操作类工具（purchase_voucher、create_reservation）保持 60 秒以上，避免对下游反复冲击 | 在不牺牲操作类工具保护性的前提下，缩短查询类工具的恢复感知延迟     |
| ② 熔断状态管理接口       | 新增`/admin/circuit-breakers`：查看所有 Worker 熔断器状态（复用 `get_breaker_states()`）+ 手动恢复端点，运维确认依赖恢复后主动调 `breaker.close()`，立即放行流量   | 提供人工介入通道，绕过固定冷却期，应对"确认已恢复但还在冷却"的场景 |

**追问话术**：

> "固定冷却期的熔断确实存在'恢复感知延迟'——服务在冷却期内恢复了熔断器也不知道，请求照样拒绝，最坏要多冤枉整个冷却期。我认为这是保护性和恢复速度的刻意权衡：冷却期越短探测越频繁、对故障服务冲击越大。我的缓解规划是两个方向：一是按工具类型差异化冷却期，查询类工具幂等、可以配短一点快速试探恢复，操作类工具保持较长冷却期更稳妥；二是加一个熔断管理接口，除了查看各 Worker 的熔断状态，还支持手动恢复——运维确认依赖已经恢复后直接关闭熔断，不用干等冷却期结束。"

**技术细节**：

- **Redis 状态持久化**：使用 `CircuitRedisStorage` 将熔断状态存入 Redis，多实例共享（需要同步 Redis 客户端），并通过 `namespace=熔断器名` 实现 Worker 级隔离
- **LangSmith 集成**：通过监听器记录状态变更事件和元数据，可在 LangSmith 界面查看（注意钩子名是 `state_change/failure/success`）
- **统一配置**：所有工具使用相同的熔断阈值（3 次失败 / 60 秒恢复）
- **工具映射**：每个工具映射到对应的 Worker 熔断器，熔断时整个 Worker 暂停服务
- **手动驱动状态机**：由于 PyBreaker 的异步兼容问题（`call_async` 是 Tornado 协程、`call` 只接受同步函数），不能用它包装 async 工具，改为调用前 `ensure_breaker_allowed()` 预检（含 OPEN→HALF_OPEN 恢复转换）、调用后 `record_success()/record_failure()` 走 `_handle_success()/_handle_error()` 记录——这两个私有方法才是真正做计数、触发熔断、重置状态的地方

**改进效果**：

| 对比项     | 原方案（手动实现） | PyBreaker                               |
| ---------- | ------------------ | --------------------------------------- |
| 半开状态   | ❌ 不支持          | ✅ 支持（冷却期后下一次请求预检时探测） |
| 状态持久化 | ❌ 内存            | ✅ Redis（多实例共享，namespace 隔离）  |
| 监控告警   | ❌ 无              | ✅ LangSmith 监听器                     |
| 代码量     | ~50 行手动管理     | ~20 行（预检 + 记录，状态机由库管理）   |
| 线程安全   | ❌ 需自己处理      | ✅ 内置                                 |

**面试话术**：

> "熔断器我使用 PyBreaker 库实现，替换了原来手动管理的字典+时间戳方案。PyBreaker 支持三种状态：关闭、打开、半开——冷却期过后放行一个请求探测是否恢复，比原来的直接关闭更平滑。状态通过 Redis 持久化并按 Worker 用 namespace 隔离，多实例部署共享熔断状态的同时互不污染。同时集成了 LangSmith 监听器，熔断状态变更会自动记录到 LangSmith。配置是 3 次失败触发熔断，60 秒后尝试恢复。
>
> 这块印象最深的是二次排查：因为 PyBreaker 不兼容原生 async，我不能用它的 `call()` 包装，只能手动驱动状态机。最初查文档选了 `state.on_success()/on_failure()` 这对方法，代码不报错就以为没问题；后来写验证脚本直接驱动熔断器断言状态，发现失败 10 次计数器还是 0——读源码才知道这版 PyBreaker 把计数逻辑放在 `_handle_error()` 里，`on_failure()` 只做阈值检查。同一次排查还发现了三个连带问题：Redis 存储不传 namespace 会导致所有 Worker 共享一组 key、熔断状态互相污染；监听器钩子名写成了 `on_*` 导致 LangSmith 集成从来没生效过；还有 OPEN 状态永远不会转半开，熔断一开就恢复不了。最后封装了三个入口函数统一修复：预检（含恢复转换）、记成功、记失败，并用'触发熔断、Worker 隔离、完整恢复闭环'三组用例验证通过。这件事给我的教训是：绕过库的上层 API 直接调内部方法时必须读源码确认职责，集成完必须写驱动到目标状态的验证用例——静默失效的 Bug 不报错，只看'能跑通'是发现不了的。"

---

### 六、RAG 检索

#### 你的 RAG 检索完整流程是什么样的？

**A：** 分离线导入和在线检索两条链路。

**离线导入**（`scripts/import_knowledge.py`）：

```
knowledge_content/*.txt
   ↓ 混合分块 hybrid_chunk()，6 层
   ① 按【】切分文档  
   ② 按"一、二、三"切分章节  
   ③ 按数字编号/Q:A 切分条目
   ④ 超过 MAX_CHUNK_SIZE=300 字符 → 按 。；\n， 找自然断点再切（找不到则硬切）
   ⑤ 低于 MIN_CHUNK_SIZE=50 字符 → 向上合并到前一块
   ⑥ 相邻块加 OVERLAP_SIZE=50 字符重叠（把前块尾部拼到后块开头）
   ↓ 批量调本地 Embedding 服务（bge-large-zh-v1.5，Flask+waitress 多线程，
     normalize_embeddings=True → 向量已归一化，故余弦等价于点积）
   ↓ 1024 维
   ↓ 单次写入 Milvus collection hmdp_knowledge —— 同一行数据兼有两种表示：
       embedding 字段：HNSW 索引 + COSINE 度量（稠密路）
       content 字段：开 chinese analyzer 的 VARCHAR，供 BM25 派生
       sparse_bm25 字段：由服务端 BM25 Function 从 content 自动派生，写入时不提供（稀疏路）
```

分块参数依据：bge-large-zh-v1.5 输入窗口 512 token，中文约 1-1.5 token/字，300 字 ≈ 450 token，留出头尾余量。

> BM25 走 Milvus 服务端原生能力。收益：只有一份数据源，不存在内存索引与库内数据不一致；analyzer 是 collection 级永久设置，改动只能 drop 重建重灌，所以导入脚本改 schema 需要 `--force` 显式确认。

**在线检索**（`app/services/rag.py` 的 `RAGService.search()`）：

```
Knowledge Agent 收到 goal（LLM 自主决定调用 rag_search，可产生多个 tool_calls
                     → asyncio.gather 并行执行，例如"推荐烧烤/日料/饮品"三路并发）
   ↓
① 双路并行召回（asyncio.gather，各 RAG_TOP_K=30 条）
   稠密路 vector_search: query → Embedding → Milvus HNSW 检索（ef=128）
   稀疏路 bm25_search:  query 原文直接传给 Milvus 服务端 BM25，
                       服务端用与入库同一 analyzer 切查询串 → 稀疏向量打分降序
   两路查的是同一 collection 的同一批 chunk（同一行的两种表示）→ 可按主键 doc_id 直接归并
   ↓
② 加权 RRF 融合（merge_results，k=60，w_dense=w_bm25=1.0）
   score(d) = Σ wᵢ/(k + rankᵢ)，只用名次不碰原始分数
   → 规避余弦(0~1)与 BM25(无上界)的量纲不可比问题
   → 多路共同认可的文档自然靠前
   → 按 doc_id 去重归并；并列分按 doc_id 定序 → 同一 query 两次请求结果可复现
   ↓
③ Rerank 精排（rerank()，调阿里云百炼 text-rerank 云端接口）
   模型 qwen3.7-text-rerank；请求体 input{query, documents} + parameters{top_n, instruct}，
   Bearer DASHSCOPE_API_KEY 认证，超时 10s
   返回按 relevance_score(0~1) 降序的 {index, relevance_score} 列表
   原理：交叉编码器把 query 与每个候选拼成一条序列联合编码，
   self-attention 让两侧 token 逐层互相可见再打分，精度远高于 bi-encoder，
   但每个候选都要一次在线前向 → 只给几十条候选用
   （本地 bge-reranker-v2-m3 服务保留为兼容 DashScope 返回格式的自托管备选，切回只需改 RERANK_URL）
   ↓
④ 相关性自适应阈值过滤
   threshold = max(RAG_RERANK_SCORE_FLOOR=0.30, best_score × RAG_RERANK_SCORE_RATIO=0.5)
   百炼 relevance_score 不可跨请求比较 → 以相对项（best×0.5）为主，跨 query 自适应
   绝对底线 0.30 → 兜住"全部不相关"，此时清空结果
   ↓
⑤ kept[:top_k]（RAG_RERANK_TOP_K=5）→ 格式化 "1. xxx\n\n2. xxx" → 交回 Agent → LLM 生成
```

**为什么必须两阶段**：bi-encoder（向量检索）能把 doc 向量离线预存、支撑百万级召回，但 query 和 doc 从未同时进模型、缺交互，精度有上限；cross-encoder 建模 query-doc 逐 token 交互，精度高但每个候选都要一次在线前向，只能给几十条用。所以是"bi-encoder 负责捞够、cross-encoder 负责排准"。

**为什么用 RRF 而不是加权分数融合**：两路分数量纲完全不同（余弦 0~1、BM25 无上界），直接加权必须先归一化，而归一化方式本身会显著影响结果、且随语料变化失效。RRF 只看名次，天然绕开这个问题，实现也只要十几行。RRF 里的权重 wᵢ 是相对倍率（当前两路都是 1.0），为"某一路明显更可信"时预留调节位——它乘在名次贡献上，不破坏量纲无关性。

**降级与兜底**：

| 故障点                    | 行为                                                       |
| ---------------------- | -------------------------------------------------------- |
| Embedding 服务失败         | `vector_search` 返回 []，退化为 BM25 单路（RRF 对空列表自然不贡献分数，无需分支）  |
| 未配置`DASHSCOPE_API_KEY` | 直接跳过精排，返回`results[:top_k]`（RRF 序）                        |
| Rerank 云端调用失败/超时       | `except` 捕获后返回 `results[:top_k]`，即 RRF 融合序，**跳过精排和阈值过滤** |
| 阈值过滤后为空                | 工具返回"未找到相关知识库内容"，不进入 LLM，避免基于无关内容编造                      |
| Milvus 不可用             | 稠密、稀疏两路同时为空 → 检索无结果。                                     |

**关键配置**：

| 参数                                               | 值                  | 含义                                                                           |
| -------------------------------------------------- | ------------------- | ------------------------------------------------------------------------------ |
| `RAG_TOP_K`                                      | 30                  | 每路召回条数（候选池规模）                                                     |
| `RAG_RERANK_TOP_K`                               | 5                   | 最终输出条数                                                                   |
| `RAG_ENABLE_RERANK`                              | true                | 是否精排                                                                       |
| `RAG_RERANK_MODEL`                               | qwen3.7-text-rerank | 百炼精排模型（`RERANK_URL` 指向 dashscope text-rerank）                      |
| `RAG_RERANK_TIMEOUT`                             | 10s                 | 云端精排超时（云端通常数百毫秒，比原本地服务的 15s 收得更紧）                  |
| `RAG_RERANK_SCORE_FLOOR`                         | 0.30                | 相关性绝对底线                                                                 |
| `RAG_RERANK_SCORE_RATIO`                         | 0.5                 | 相对最优分比例                                                                 |
| `RAG_RRF_K` / `WEIGHT_DENSE` / `WEIGHT_BM25` | 60 / 1.0 / 1.0      | RRF 平滑常数与两路权重                                                         |
| `RAG_HNSW_EF`                                    | 128                 | 需显著大于每路召回深度；原值 32 贴着 RAG_TOP_K=30 下限，近似搜索自身就在漏召回 |
| `MAX/MIN/OVERLAP_CHUNK`                          | 300/50/50           | 分块字符数                                                                     |

> 一句话总结：我的 RAG 是"混合分块单次写入 Milvus（稠密向量 + 服务端 BM25 双表示）→ bi-encoder+BM25 双路各召回 Top-30 → 加权 RRF 按 doc_id 名次归并 → 百炼交叉编码器精排 → relevance_score 过自适应阈值（max(0.30, best×0.5)）→ 取 Top-5 交给 LLM"。召回阶段刻意放大、精排阶段收紧输出，是因为 rerank 的价值在于"从噪声里捞真正相关的"，候选池太窄它就无从发挥。

#### 数据库与知识库实时同步

**改进原因**：面试高频问题"你的 RAG 知识库怎么更新？"、"数据库和知识库数据不一致怎么办？"

**实现方案**：

- 当商家通过 Java 后端 `POST /shop` 上架时，直接调用本地 Embedding 服务生成向量，存入 Milvus
- 采用增量更新策略，只向量化新增/变更的数据，不需要全量重建索引
- 支持新增、更新、删除操作的双向同步
- 使用重试机制确保同步成功，失败时返回明确错误信息

**核心代码**：

```java
// Java 后端 - EmbeddingService.java
@Service
public class EmbeddingService {
    @Resource
    private EmbeddingModel embeddingModel;

    public float[] embed(String text) {
        EmbeddingRequest request = new EmbeddingRequest(List.of(text), null);
        EmbeddingResponse response = embeddingModel.call(request);
        return response.getResults().get(0).getOutput();
    }
}

// Java 后端 - MilvusService.java
@Service
public class MilvusService {
    @Value("${milvus.uri:http://localhost:19530}")
    private String milvusUri;

    @Value("${milvus.collection:hmdp_knowledge}")
    private String collectionName;

    public boolean upsert(String docId, String content, float[] embedding, String source) {
        // 先删除旧数据
        delete(docId);
        // 插入新数据
        List<InsertParam.Field> fields = new ArrayList<>();
        fields.add(new InsertParam.Field("id", Collections.singletonList(docId)));
        fields.add(new InsertParam.Field("content", Collections.singletonList(content)));
        fields.add(new InsertParam.Field("embedding", Collections.singletonList(embedding)));
        fields.add(new InsertParam.Field("source", Collections.singletonList(source)));

        InsertParam param = InsertParam.newBuilder()
            .withCollectionName(collectionName)
            .withFields(fields)
            .build();

        R<MutationResult> result = client.insert(param);
        return result.getStatus() == R.Status.Success.getCode();
    }
}

// Java 后端 - ShopController.java
@PostMapping
public Result saveShop(@RequestBody Shop shop) {
    // 写入数据库
    shopService.save(shop);

    // 同步到知识库（带重试）
    boolean syncSuccess = syncShopToKnowledgeWithRetry(shop, 3);

    if (syncSuccess) {
        return Result.ok(shop.getId());
    } else {
        return Result.fail("店铺已保存，但知识库同步失败，请稍后重试");
    }
}

private boolean syncShopToKnowledgeWithRetry(Shop shop, int maxRetries) {
    ShopType shopType = shopTypeService.getById(shop.getTypeId());
    String shopTypeName = shopType != null ? shopType.getName() : "其他";

    for (int i = 0; i < maxRetries; i++) {
        try {
            boolean success = knowledgeSyncService.syncShopToKnowledge(shop, shopTypeName);
            if (success) {
                return true;
            }
        } catch (Exception e) {
            System.err.println("店铺 " + shop.getName() + " 同步失败，第 " + (i + 1) + " 次重试: " + e.getMessage());
        }
        // 等待后重试
        try { Thread.sleep(1000); } catch (InterruptedException ignored) {}
    }
    return false;
}
```

**技术细节**：

- Java 后端直接调用本地 Embedding 服务（localhost:5001），不依赖 Python 后端
- 使用 Spring AI 的 EmbeddingModel 生成向量，保证与 Python 后端使用相同的模型
- Milvus 支持增量插入，不需要重建索引
- 使用 `doc_id = "shop_{shop_id}"` 作为唯一标识，支持 upsert 操作
- 重试机制：最多重试 3 次，每次间隔 1 秒
- 同步失败时返回明确错误信息给用户

**改进效果**：

- 数据库和知识库实时同步，解决数据不一致问题
- 增量更新，性能优于全量重建
- 不依赖 Python 后端，架构更简单
- 重试机制确保同步可靠性
- 支持新增、更新、删除操作的完整生命周期管理
- 异步执行，不影响主业务流程

**面试话术**：

> "我实现了数据库和知识库的实时同步机制。当商家上架时，Java 后端会异步调用 Python 后端的同步接口，将店铺信息向量化后存入 Milvus。使用增量更新策略，只处理新增/变更的数据，不需要全量重建索引。使用 shop_id 作为唯一标识，支持 upsert 操作，确保数据一致性。"

---

---

### 七、性能与体验优化

#### Summary LLM 流式输出优化（感知延迟降低）

**改进原因**：面试高频问题"你的 Agent 如何优化用户感知延迟？"、"流式输出是怎么实现的？"

**问题发现**：

- Summary LLM（汇总多任务结果）耗时较长，用户等待时间感知明显
- 尝试用 `graph.astream_events()` 捕获图内部 Summary LLM 的流式输出，但 LangGraph 会缓冲内部流事件，导致输出"先出几个字，停顿，剩余一起出现"，没有逐字流式效果
- 根因：`astream_events` 是图级别的事件流，图节点内部的 `astream()` 调用会被图的执行引擎缓冲

**实现方案**：

将 Summary LLM 从图内部移到 Router 层直接流式调用，绕过图的缓冲：

1. **Primary Agent 只做编排**：任务规划 + 子 Agent 执行，把 `task_results` 和 `summary_context` 存到 state 返回
2. **Router 直接流式调用**：从 state 取出结果后，直接调 `_summary_llm.astream()` 流式 yield 给前端
3. **流是直连的**：Summary LLM 的 token 直接通过 `StreamingResponse` 推给前端，不经过任何中间缓冲

**核心代码**：

```python
# app/agents/primary.py - 只做编排，不调用 Summary LLM
async def primary_agent(state: State) -> dict:
    # ... 任务编排和执行 ...

    # 存储结果到 state，由 router 流式调用 Summary LLM
    return {
        "task_results": task_results,
        "summary_context": {
            "user_message": user_message,
            "results_text": results_text,
            "conversation_context": context,
            "user_memories": user_memories,
            "current_date": current_date,
            "weekday": weekday,
        },
    }

# app/routers/chat.py - 直接流式调用 Summary LLM
async def generate():
    final_state = await graph.ainvoke(input_state, config=config)

    task_results = final_state.get("task_results")
    summary_context = final_state.get("summary_context")

    if task_results and summary_context:
        summary_prompt = load_prompt("summary_system.txt").format(**summary_context)
        summary_llm = get_summary_llm()

        # 直接流式调用（不经过图，无缓冲）
        async for chunk in summary_llm.astream([
            SystemMessage(content=summary_prompt),
            HumanMessage(content="请生成用户友好的回复。")
        ]):
            yield chunk.content  # 逐 token 推给前端
```

**架构对比**：

```
改造前（有缓冲）：
  用户请求 → graph.astream_events() → [图内部 Summary LLM astream] → 缓冲 → 前端
                                        ↑ 图引擎缓冲导致输出不流畅

改造后（直连流式）：
  用户请求 → graph.ainvoke()（编排+执行）→ 取出 task_results
           → summary_llm.astream()（直连）→ 前端
             ↑ token 直接推给前端，无缓冲
```

**技术细节**：

- **职责分离**：图负责任务编排和执行（非流式），Router 负责最终输出（流式）
- **直连流式**：Summary LLM 的 `astream()` 在 Router 层直接调用，token 通过 `StreamingResponse` 直推前端
- **类型兼容**：处理 `chunk.content` 可能是 `str` 或 `list`（Anthropic content blocks 格式）

**改进效果**：

- 用户感知延迟显著降低：第一个 token 几乎立即出现，后续逐字输出
- 流式效果流畅：不再有"停顿-批量"现象
- 架构更清晰：编排逻辑和输出逻辑分离，职责明确

**面试话术**：

> "Summary LLM 是用户等待时间最长的环节。最初我用 LangGraph 的 astream_events 捕获图内部的流式输出，但发现图引擎会缓冲内部流事件，导致输出不流畅。我的解决方案是把 Summary LLM 从图里拿出来，在 Router 层直接流式调用。图只负责任务编排和执行（ainvoke），执行完后把结果传给 Summary LLM，Summary 的 token 直接通过 StreamingResponse 推给前端。这样用户几乎立即看到第一个字，体验大幅提升。核心思路是：编排逻辑用同步执行保证正确性，最终输出用流式调用优化体验。"

---

#### Vue 响应式流式输出（Buffer + requestAnimationFrame）

**改进原因**：面试高频问题"前端怎么实现逐字流式输出？"、"Vue 的响应式机制和流式更新有什么坑？"

**问题发现**：

- Summary LLM 通过 `StreamingResponse` 逐 token 推送，后端流式正常
- 前端用 `fetch` + `ReadableStream` 接收 chunk，用 `requestAnimationFrame` 逐字渲染
- 但实际效果是"先出两三个字，停顿，剩余一起出现"
- 根因：Vue 响应式断链

**根因分析**：

```javascript
// ❌ 错误写法
aiMessage = { role: 'ai', content: '' }
messages.value.push(aiMessage)  // push 时 Vue 创建响应式副本
aiMessage.content += text       // aiMessage 仍指向原始非响应式对象！
                                 // 不会触发 Vue 的 DOM 更新
```

`messages.value.push(obj)` 时，Vue 会在数组内部创建 `obj` 的响应式代理（Proxy）。但 `aiMessage` 变量仍然指向原始的非响应式对象。后续 `aiMessage.content += text` 修改的是原始对象，Vue 的响应式系统检测不到变化，DOM 不更新。

```javascript
// ✅ 正确写法
const msg = { role: 'ai', content: '' }
messages.value.push(msg)
// 从数组中取出 Vue 创建的响应式代理
aiMessage = messages.value[messages.value.length - 1]
// 现在 aiMessage 是响应式代理，修改 content 会触发 DOM 更新
aiMessage.content += text
```

**完整实现方案**：

```javascript
// api/chat.js - Buffer + requestAnimationFrame
export function streamChat(message, memoryId, token, onMessage, onError, onComplete) {
  let buffer = ''
  let streamDone = false
  let animating = false

  function scheduleRender() {
    if (animating) return
    animating = true

    async function tick() {
      if (buffer.length > 0) {
        const char = buffer[0]
        buffer = buffer.slice(1)
        await onMessage?.(char)  // 等待 Vue nextTick 完成 DOM 更新
        await new Promise(r => requestAnimationFrame(r))  // 等浏览器刷新
        tick()
      } else if (streamDone) {
        animating = false
        onComplete?.()
      } else {
        animating = false  // buffer 空但流未结束，暂停等新 chunk
      }
    }

    requestAnimationFrame(tick)
  }

  // fetch + ReadableStream 读取
  fetch(url, { method: 'POST', body, signal }).then(async response => {
    const reader = response.body.getReader()
    const decoder = new TextDecoder()
    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      buffer += decoder.decode(value, { stream: true })
      scheduleRender()  // 有新数据就启动/继续渲染
    }
    streamDone = true
    scheduleRender()
  })
}
```

```vue
<!-- Chat.vue - 组件中的使用 -->
<script setup>
const messages = ref([])
let aiMessage = null

chatController = streamChat(
  message, memoryId, token,
  // onMessage: 每次接收 1 个字符
  async (text) => {
    if (isFirstChunk) {
      const msg = { role: 'ai', content: '' }
      messages.value.push(msg)
      // 关键：从数组取出响应式代理
      aiMessage = messages.value[messages.value.length - 1]
      isFirstChunk = false
      loading.value = false
    }
    aiMessage.content += text
    await nextTick()  // 强制 Vue 刷新 DOM
    scrollToBottom()
  },
  onError,
  onComplete
)
</script>
```

**技术细节**：

- **Buffer 模式**：后端 chunk 先存入 buffer，由 `requestAnimationFrame` 循环逐字取出
- **requestAnimationFrame**：在浏览器每次重绘前触发，保证 DOM 更新被渲染
- **await nextTick()**：强制 Vue 在下一个字符处理前完成 DOM 更新
- **响应式代理**：从 `messages.value[index]` 取出的对象才是 Vue 的响应式代理

**改进效果**：

- 真正的逐字流式输出，用户体验流畅
- 解决了 Vue 响应式断链的隐蔽 bug

**面试话术**：

> "前端流式输出的实现有几个关键点。第一，用 buffer + requestAnimationFrame 实现逐字渲染，每帧输出一个字符，保证浏览器在每个字符后都刷新画面。第二，遇到一个隐蔽的 Vue 响应式 bug：把对象 push 进 reactive 数组后，原始变量仍然指向非响应式对象，修改属性不会触发 DOM 更新。解决方案是 push 后从数组中重新取出响应式代理。第三，onMessage 回调要用 async + await nextTick()，确保 Vue 的 DOM 更新在下一个 requestAnimationFrame 之前完成。"

---

#### SSE 流式输出与执行状态展示

**改进原因**：

- 原方案使用 `text/plain` 纯文本流，前端只能看到最终输出
- 用户无法感知 Agent 内部执行过程（规划、执行阶段）
- SSE（Server-Sent Events）支持多事件类型，可展示执行状态

**实现方案**：

- **EventBus 事件总线**：轻量级异步事件队列，用于图执行过程中向前端发送事件
- **SSE 传输格式**：`text/event-stream`，支持 `step`、`message`、`done` 三种事件
- **执行状态展示**：前端显示"正在规划任务..."、"正在执行任务..."等步骤提示

**核心设计**：

| 组件                          | 说明                         |
| ----------------------------- | ---------------------------- |
| `app/services/event_bus.py` | 事件总线，基于 asyncio.Queue |
| `current_event_bus`         | ContextVar，绑定到请求上下文 |
| `step` 事件                 | 规划/执行阶段的状态提示      |
| `message` 事件              | Summary LLM 流式输出内容     |
| `done` 事件                 | 流式输出完成                 |

**事件流程**：

```
用户请求
    ↓
EventBus 创建，绑定到 ContextVar
    ↓
Graph 后台执行（asyncio.create_task）
    ↓
Primary Agent 发送 step:planning 事件
    ↓
Primary Agent 发送 step:executing 事件
    ↓
EventBus 监听，yield SSE 事件
    ↓
Summary LLM 流式输出 message 事件
    ↓
发送 done 事件，关闭 EventBus
```

**SSE 事件格式**：

```
event: step
data: {"stage": "planning"}

event: step
data: {"stage": "executing"}

event: message
data: {"content": "根据您的位置"}

event: done
data: {}
```

**关键代码**：

```python
# event_bus.py
class EventBus:
    def __init__(self):
        self._queue: asyncio.Queue = asyncio.Queue()

    async def emit(self, event_type: str, data: dict):
        await self._queue.put({"event": event_type, "data": data})

    async def listen(self) -> AsyncGenerator:
        while True:
            event = await self._queue.get()
            if event is None:
                break
            yield event

# primary.py - 发送事件
bus = current_event_bus.get(None)
if bus:
    await bus.emit("step", {"stage": "planning"})
    # ... 规划完成 ...
    await bus.emit("step", {"stage": "executing"})

# chat.py - SSE 响应
async def generate_sse():
    bus = EventBus()
    token = current_event_bus.set(bus)
    graph_task = asyncio.create_task(graph.ainvoke(initial_state, config=config))
    async for event in bus.listen():
        yield f"event: {event['event']}\ndata: {json.dumps(event['data'], ensure_ascii=False)}\n\n"
    final_state = await graph_task
    # Summary LLM 流式输出
    async for chunk in summary_llm.astream(...):
        yield f"event: message\ndata: {json.dumps({'content': text}, ensure_ascii=False)}\n\n"
    yield "event: done\ndata: {}\n\n"
```

**前端实现**：

```javascript
// chat.js - SSE 解析
function handleSSEEvent(event, data) {
  switch (event) {
    case 'step':
      onStep?.(data.stage)
      break
    case 'message':
      if (data.content) {
        buffer += data.content
        scheduleRender()
      }
      break
  }
}

// Chat.vue - 步骤提示
<div v-if="currentStep" class="step-indicator">
  <div class="step-spinner"></div>
  <span>{{ currentStep }}</span>
</div>
```

**与原方案对比**：

| 特性         | 原方案 (text/plain) | SSE (text/event-stream)    |
| ------------ | ------------------- | -------------------------- |
| 传输格式     | 原始文本            | `data: xxx\n\n` 协议格式 |
| 事件类型     | ❌                  | ✅ 支持 step/message/done  |
| 执行状态展示 | ❌                  | ✅ 前端显示规划/执行阶段   |
| 自动重连     | ❌                  | ✅ 浏览器内置              |

**为什么 Summary LLM 仍在图外**：

- SSE 是传输格式，不解决 LangGraph 内部缓冲问题
- `astream_events()` 是图级别事件流，节点内部的 `astream()` 会被缓冲
- Summary LLM 在图外才能实现真正的逐字流式输出

**面试话术**：

> "流式输出使用 SSE（Server-Sent Events）实现，支持三种事件类型：step（执行状态）、message（内容输出）、done（完成）。设计了一个轻量级 EventBus，基于 asyncio.Queue，在图执行过程中向前端发送事件。Primary Agent 在规划和执行阶段分别发送 step 事件，前端显示步骤提示。Summary LLM 在图外直接流式调用，发送 message 事件，前端逐字显示。这样用户可以实时看到 Agent 的执行过程，提升交互体验。"

---

### 八、评测与可观测

#### 自动化评测体系

**A：** 我基于 LangSmith 构建了三层评测体系，覆盖从检索质量到最终回答的全链路。

**评测指标分三层**：

| 层级 | 指标 | 评什么 |
|---|---|---|
| **RAG 质量** | Faithfulness / Context Precision / Context Recall | 检索结果是否准确、完整，回答是否忠于检索内容（RAGAS 框架） |
| **工具调用** | 工具选择准确率、参数提取准确率、券选择准确率、时间解析准确率 | 子 Agent 是否选对、参数是否解析正确 |
| **端到端** | 任务完成率、安全拦截率、确认触发率 | 最终回答是否正确、安全防护是否生效、敏感操作是否触发确认 |

**LangSmith 集成**：

- **链路追踪**：每个请求自动记录完整调用链——Guardrail 判定 → Primary Agent 规划 → 子 Agent 执行 → 工具调用 → Summary 生成，每一步记录耗时、Token 用量、输入输出
- **Dataset 评测**：自建16条标注数据集（3 RAG / 6 工具 / 7 端到端），每条包含输入、期望行为和结构化断言字段（期望工具、确认触发、安全拦截）；改 prompt 或换模型后一键跑评测，自动计算各指标分数，防退化
- **LLM-as-Judge**：配 evaluator 用 LLM 自动判断回答是否准确、是否安全，不依赖人工逐条检查
- **版本对比**：每次实验（experiment）记录在 LangSmith，同 dataset 多个 experiment 可视化对比指标趋势

**评测流程**：改代码/prompt → 跑 dataset → 看指标变化 → 决定是否上线。不靠"感觉能用"，靠数据说话。

#### 评测数据集怎么设计的？评测具体怎么跑？

**A：** 我在 LangSmith 上自建了 16 条标注数据集，配合绑定在数据集上的 LLM-as-Judge evaluator，形成"改完就测、自动打分、实验对比"的回归评测闭环。

**数据集按三层指标覆盖所有典型情况**：

| 层 | 条数 | 覆盖的情况 |
|---|---|---|
| RAG 质量 | 3 | 知识库可答（平台规则类）、券种对比可答、知识库不可答（防编造探针） |
| 工具调用 | 6 | 查店、区域搜店（参数提取）、按店查券、查我的券、实体不存在（防编造探针）、券比较（券选择准确率） |
| 端到端 | 7 | 买券确认触发、预约确认触发（含时间解析）、prompt 注入拦截、无关请求拦截、隐私越权拒绝、组合任务（多工具编排）、纯问候放行 |

每条样本的 outputs 不只是参考答案，是结构化的：

- **`expected_behavior`**：期望行为的自然语言描述，作为 LLM-as-Judge 的判据
- **`must_trigger_confirm` / `must_block`**：敏感操作确认触发、安全拦截的布尔断言
- **`expected_tools`**：期望调用的工具清单（对应工具选择准确率）
- **`category` / `layer`**：情况分类标签，便于按维度聚合分析

设计上有两个刻意为之的点：一是**攻击样本的期望是"礼貌拒绝"而不是"完成任务"**——注入、无关请求、隐私越权三条，正确拒绝就等于识别正确；二是**防编造探针**——知识库答不了的问题、不存在的商户，期望是"如实告知"而非硬答，这是 faithfulness 的底线测试。

**评测怎么跑（target 怎么接）**：

- 本地写了一个 runner 脚本作为评测 target：逐条把样本问题 POST 到本地 8000 服务的 chat 接口（SSE 流式），汇聚完整回答后提交为 LangSmith experiment
- **身份隔离**：每个样本用独立 memory_id（user_9001 起编号），会话间零串扰，也不污染真实用户的历史和画像
- **LLM-as-Judge 绑定数据集**：judge 从三个维度打分——意图识别准确率（含"对攻击性输入的礼貌拒绝=识别正确"）、回复准确性（事实正确、不编造、拒绝理由合理）、回复完整性（覆盖用户核心需求）；绑定在 dataset 上，**每个新 experiment 跑完自动打分**
- 流程：改代码/prompt → 跑一遍 16 条（约 2 分钟）→ 新旧 experiment 在 UI 并排对比 → 数据说话决定是否上线

**一个真实的校准案例（评测闭环的价值）**：第一轮跑分发现"购买优惠券触发确认"样本没触发确认——排查发现不是系统问题，是**样本与真实库脱节**：我按知识库推荐文件写店名，但商户库里没有那家店、全库只有一家店有券。评测样本必须对齐被测系统的真实数据，否则测的不是系统能力而是数据现状。按真实数据校准后重跑，确认触发率恢复正常。

**踩坑：跨进程 trace 盲区**：agent 跑在 8000 服务进程，评测 runner 是另一个进程——即使两边都开了 LangSmith tracing，服务端的工具子 run 也落在服务端自己的 project，不在 experiment 的 trace 链上。所以"工具选择准确率"无法在 experiment 里自动断言，只能看服务端 trace 人工核对。教训：要把断言做进 trace，target 必须与被测系统同进程，或让服务端把工具调用信息带进响应。

---
## 项目补充

### 为什么选择 LangGraph 而不是 Spring AI？

**A：** 选择 LangGraph 主要基于以下考虑：

1. **多 Agent 编排能力**：LangGraph 原生支持状态图、条件路由、子 Agent 协作，而 Spring AI 主要是单 Agent 的 Tool-Use Loop
2. **Python 生态**：AI/ML 领域 Python 生态更成熟（LangChain、transformers、sentence-transformers 等）
3. **灵活性**：可以自定义 Agent 循环、工具执行、状态管理，不受框架限制
4. **本地模型支持**：方便接入本地 Embedding（BGE-small-zh）和 Rerank（BGE-reranker）模型
5. **独立部署**：Python 服务独立于 Java 后端，便于扩展和维护

**架构对比**：

| 维度       | Spring AI              | LangGraph                   |
| ---------- | ---------------------- | --------------------------- |
| Agent 模式 | 单 Agent Tool-Use Loop | Orchestrator-Workers 状态图 |
| 路由方式   | LLM 自动选择工具       | Orchestrator 委托路由       |
| 工具调用   | 框架自动循环           | Worker 独立执行             |
| 状态管理   | 框架内部               | 自定义 State Graph          |
| 并行执行   | 不支持                 | 支持（多个 tool_calls）     |
| 记忆共享   | 框架内部               | Orchestrator 层面统一检索   |

---

### 为什么使用 LangGraph 而不是纯 LangChain？

**A：** LangChain 本身只提供了 Agent 的基础构建模块（LLM + Tools + Memory），并没有内置多 Agent 编排能力。使用 LangGraph 而不是纯 LangChain，主要基于以下考虑：

1. **多 Agent 状态编排**：LangGraph 基于状态图（StateGraph），原生支持节点间的条件路由、循环、并行执行。纯 LangChain 要实现类似功能，需要自己写大量循环和条件逻辑
2. **持久化与中断机制**：LangGraph 内置了 Checkpointer（MemorySaver / RedisSaver），支持图执行状态持久化和 interrupt() 机制，实现敏感操作确认、断点续传等功能。纯 LangChain 没有这些
3. **可观测性**：LangGraph 的每个节点都是一个独立的执行单元，天然适配 LangSmith 的 Trace 追踪，每个节点的输入输出、耗时、Token 消耗一目了然
4. **复杂流程控制**：Guardrail → Orchestrator → Workers 这种多层流程，用 LangGraph 的节点 + 条件边可以清晰表达；纯 LangChain 只有 AgentExecutor 的简单循环

**对比**：

| 维度       | 纯 LangChain           | LangGraph       |
| ---------- | ---------------------- | --------------- |
| Agent 模式 | 单 Agent Executor 循环 | 多节点状态图    |
| 状态管理   | 无内置支持             | State + Reducer |
| 持久化     | 无                     | Checkpointer    |
| 人机交互   | 无                     | interrupt()     |
| 条件路由   | 需自己实现             | 条件边原生支持  |
| 并行执行   | 不支持                 | 多节点并行      |
| 可观测性   | Callbacks              | LangSmith 集成  |

**面试话术**：

> "纯 LangChain 只提供了 Agent 的基础构建模块——LLM + Tools + Memory + AgentExecutor 循环。它没有内置多 Agent 编排、状态持久化、中断机制这些能力。LangGraph 基于状态图，原生支持节点间的条件路由、循环、并行执行，内置 Checkpointer 支持状态持久化和 interrupt() 实现人机交互。我的项目需要 Guardrail → Orchestrator → Workers 的多层流程，还需要敏感操作确认（interrupt），这些都是纯 LangChain 无法提供的。"

### 为什么任务编排的多任务并行不走 LangGraph 的 Send API？

**A：** LangGraph 的 `Send` 是**静态扇出**：在分叉点一次性枚举所有分支，每个分支拿分叉时刻的状态副本，执行中互不可见，结果在汇合点经 reducer 合并。而 TaskExecutor 的并行是"依赖传播"语义，Send 表达不了：

1. **后置任务的输入由执行结果决定**：任务 goal 要拼上前置的真实输出（`goal += 前置任务结果：prev_detail['result']`），Send 派发时刻这个值还不存在
2. **失败要级联跳过**：前置 `success=False` 时动态标记 skip、不调子 Agent 不花 token——Send 分支无法依据运行时结果取消彼此
3. **合并语义对不上**：结果必须是按任务序号索引的完整有序数组（含跳过行、熔断行、未执行兜底行），Send 汇合要自定义 reducer 按 task_index 归位；asyncio.gather 版本里共享内存几行赋值就解决
4. **横切控制流要全局视角**：熔断预检、多 pending_action 只放行第一个——都是对整体结果集的顺序性决策，对称并行分支里做不了

**适用边界**：Send 适合分支相互独立、无中间互读的 map-reduce（项目里 RAG 三路并行召回就是这么做的）；带依赖的 DAG 则选择"节点内拓扑波次 asyncio.gather、节点间才用图并行"——executor 是普通 Python 组件不在图内，图只负责路由和 confirm interrupt。

> 面试话术："Send 是静态扇出、分支隔离、reducer 汇合，表达不了'前置失败级联跳过、前置输出注入后置 goal'这类依赖传播。我把抽象层级划死了：图的并行原语留给节点级路由和中断恢复，任务 DAG 内部用 orchestrator 里的 asyncio.gather 波次调度——共享内存让跨任务读写是免费的。"

### 分块策略是什么？为什么这样选？

**A：** 采用**混合分块：结构分块为主 + 长度兜底 + 过短合并 + 重叠滑窗**，一共 6 层。

```python
# scripts/import_knowledge.py:25-27（三个参数全在这里，注释就写着依据）
MAX_CHUNK_SIZE = 300   # 单块最大字符数（中文约 1-1.5 token/字，300字 ≈ 450 tokens）
MIN_CHUNK_SIZE = 50    # 单块最小字符数，低于此值向上合并
OVERLAP_SIZE   = 50    # 相邻块重叠字符数（≈17%，保证跨边界关键词不丢失）
```

**① 结构分块（`split_by_pattern`，3 层）** 按知识库文档的天然层级正则切分，顺序切三层：

| 层 | 正则                          | 语义单位    |
| -- | ----------------------------- | ----------- |
| 1  | `【.+?】`                   | 文档        |
| 2  | `[一二三四五六七八九十]+、` | 章节        |
| 3  | `(?:\d+\.\s                   | Q\d+[：:])` |

关键实现细节：`keep_delimiter=True`。做法是 `re.split(f'({pattern})', text)` 用捕获组让分隔符也出现在结果里，再靠 `re.match(pattern, part)` 识别出哪一段是分隔符，把它**挪到下一个片段的开头**而不是丢掉。这样每个 chunk 都以 `【商家入驻流程】` 或 `三、` 这种标题打头，检索出来自带归属信息，不需要额外存 metadata 也能知道这段属于哪个文档。

**④ 长度兜底（`split_by_size`）** 结构切完仍然超过 300 字的块，在 `remaining[:max_size]` 这个窗口内对 `['。', '；', '\n', '，']` 四类分隔符各做一次 `rfind`，取**位置最靠后**的那个作为断点，`cut_pos + 1` 切下、剩余部分进下一轮。四类分隔符地位平等，不做优先级降级——目的是让每块尽量填满 300 字、减少碎片。窗口内一个自然断点都没有才 `cut_pos = max_size` 硬切。代价是偶尔会在逗号处断开而非更早的句号，但下一块从断点继续且带重叠，断裂有限。

**⑤ 过短合并（`merge_small_chunks`）** 双向检查：`if len(merged[-1]) < min_size` 或 `elif len(chunk) < min_size`，任一成立就用 `'\n'` 拼接合并。避免「欢迎光临」这种几十字的碎片单独成块——碎片块的向量几乎全是噪声，还会挤占 Top-K 名额。

**⑥ 重叠滑窗（`add_overlap`）** `prev_tail = chunks[i-1][-50:]`，然后 `prev_tail + chunks[i]` 前置拼接。保证恰好落在边界上的关键词（如「满减」跨块）在两块里都能被检索到。

**为什么这么选**：

- 知识库是人工整理的，天然有 `【】` / 一、二、三 / 1. 2. / Q&A 四级标记，结构信息比语义相似度更可靠且零成本；固定长度分块会把一个 QA 对从中间切断。
- 语义分块（embedding 聚类断点）要在线算向量、慢且不可解释，这里结构已经够清晰，属于过度设计。
- 300 字上限来自 bge-large-zh-v1.5 的 512 token 输入窗口，留出头尾特殊 token 余量；不是拍脑袋的小数。

> 一句话话术：「我的分块是混合策略：三层结构切分打文档、章节、条目，超 300 字的按窗口内最靠后的自然断点再切，低于 50 字的向上合并，最后给相邻块加 50 字重叠。300 这个数是按 bge-large-zh 的 512 token 窗口倒推的，重叠 50 字约 17%。分隔符是正则保留的，所以每个 chunk 开头都带自己所属的文档标题。」

### Mem0 记忆框架

**问题**：Mem0 是干什么的，都能做什么操作？

**A：**

**Mem0 是什么**：
Mem0 是一个专为 AI 应用设计的长期记忆管理框架，它能够从对话中自动提取用户偏好，并提供语义检索能力。在本项目中，Mem0 用于实现用户画像的持久化存储。

**核心功能**：

| 功能                 | 说明                             | 本项目应用                       |
| -------------------- | -------------------------------- | -------------------------------- |
| **记忆存储**   | 从对话中自动提取结构化信息       | 提取用户偏好（饮品、美食、店铺） |
| **语义检索**   | 基于向量相似度检索相关记忆       | 检索与当前查询相关的用户偏好     |
| **记忆管理**   | 支持增删改查操作                 | 支持清空用户画像、按类别限制数量 |
| **多用户隔离** | 不同用户记忆完全隔离             | 通过 user_id 隔离不同用户        |
| **向量存储**   | 支持 Milvus、Chroma 等向量数据库 | 使用 Milvus 存储用户画像向量     |

**支持的操作**：

```python
# 1. 添加记忆（自动提取）
memory.add(conversation, user_id="user_123")

# 2. 语义检索
memories = memory.search(query="用户喜欢什么", user_id="user_123")

# 3. 获取所有记忆
all_memories = memory.get_all(filters={"user_id": "user_123"})

# 4. 删除记忆
memory.delete(memory_id="mem_001")

# 5. 更新记忆
memory.update(memory_id="mem_001", text="新的内容")
```

**本项目中的集成方式**：

```python
# app/services/mem0_service.py
class Mem0Service:
    def __init__(self):
        # 使用 Milvus 作为向量存储
        config = MemoryConfig(
            vector_store={
                "provider": "milvus",
                "config": {"collection_name": "mem0_user_preferences"}
            },
            custom_instructions="从对话中提取用户偏好，按 [饮品]、[美食]、[店铺] 分类"
        )
        self._memory = Memory(config=config)
```

**面试话术**：

> "Mem0 是一个 AI 记忆管理框架，我用它来实现用户画像的持久化存储。它能够自动从对话中提取用户偏好（如喜欢喝什么、吃什么、去什么店），存储到 Milvus 向量数据库中。每次对话时，我会检索用户画像并注入到系统提示词中，实现个性化推荐。Mem0 支持语义检索、多用户隔离、记忆管理等操作，非常适合需要长期记忆的 AI 应用场景。"

---

**问题：除了 Mem0 外还有什么长期记忆框架？为什么选择 Mem0？相比自己实现长期记忆框架，Mem0 有什么好处？**

**A：**

**主流长期记忆框架对比**：

| 框架                     | 核心特点            | Token 消耗      | 检索延迟 | GitHub Stars | 适用场景            |
| ------------------------ | ------------------- | --------------- | -------- | ------------ | ------------------- |
| **Mem0**           | 自动提取 + 语义检索 | ~1,764 tokens   | 0.2s     | 55.7k+       | 通用场景，快速集成  |
| **Zep**            | 时序知识图谱        | 600,000+ tokens | 小时级   | 3k+          | 复杂关系推理        |
| **LangMem**        | LangChain 官方      | 中等            | 快       | 2k+          | LangGraph 生态      |
| **Letta (MemGPT)** | 操作系统级记忆      | 中等            | 快       | 15k+         | 复杂多轮对话        |
| **Cognee**         | 图记忆              | 中等            | 快       | 2k+          | 永久记忆 + 会话记忆 |

**为什么选择 Mem0**：

| 优势                       | 说明                                                  |
| -------------------------- | ----------------------------------------------------- |
| **Token 消耗最低**   | ~1,764 tokens/conversation，比 Zep 的 600,000+ 低 99% |
| **检索延迟最低**     | p95 延迟 0.200秒，适合实时应用                        |
| **准确率高**         | LoCoMo 基准 91.6%，LongMemEval 93.4%                  |
| **集成简单**         | 支持 LangChain、LangGraph、CrewAI 等主流框架          |
| **向量数据库支持广** | 支持 20+ 向量数据库（Milvus、Chroma、Qdrant 等）      |
| **社区活跃**         | GitHub 55.7k+ stars，持续维护                         |
| **有研究论文支持**   | ECAI 2025 论文（arXiv:2504.19413）验证性能            |

**相比自己实现的好处**：

| 方面                 | 自己实现                               | 使用 Mem0                         |
| -------------------- | -------------------------------------- | --------------------------------- |
| **开发成本**   | 需要设计记忆提取、存储、检索、更新逻辑 | 开箱即用，几行代码集成            |
| **记忆提取**   | 需要自己写 Prompt 提取用户偏好         | 自动从对话中提取，支持自定义指令  |
| **语义检索**   | 需要自己实现向量检索逻辑               | 内置语义检索，支持多信号融合      |
| **多用户隔离** | 需要自己设计隔离机制                   | 内置 user_id 隔离                 |
| **向量数据库** | 需要自己适配不同数据库                 | 支持 20+ 向量数据库，配置切换     |
| **性能优化**   | 需要自己优化检索延迟和 Token 消耗      | 已优化，0.2s 延迟，1,764 tokens   |
| **维护成本**   | 需要自己维护和更新                     | 社区维护，持续更新                |
| **基准测试**   | 需要自己设计评测体系                   | 有 LoCoMo、LongMemEval 等基准验证 |

**Mem0 的核心架构**：

```
对话输入
    ↓
① 记忆提取（单次 LLM 调用）
    - 自动提取用户偏好、事实、行为
    - 支持自定义提取指令
    ↓
② 记忆存储（向量化）
    - Embedding 后存入向量数据库
    - 支持 Milvus、Chroma、Qdrant 等
    ↓
③ 记忆检索（多信号融合）
    - 语义相似度
    - 关键词匹配
    - 实体关联
    ↓
④ 记忆注入（上下文）
    - 检索结果注入系统提示词
    - LLM 根据记忆生成个性化回复
```

**面试话术**：

> "长期记忆框架我选择 Mem0，主要基于三个考虑：第一是性能，Mem0 的 Token 消耗只有 1,764 tokens/conversation，比 Zep 的 600,000+ 低 99%，检索延迟只有 0.2秒，适合实时应用；第二是易用性，Mem0 开箱即用，几行代码就能集成，支持 20+ 向量数据库，与 LangChain/LangGraph 无缝集成；第三是可靠性，Mem0 有 ECAI 2025 研究论文支持，LoCoMo 基准准确率 91.6%，GitHub 55.7k+ stars，社区活跃。相比自己实现，Mem0 节省了大量开发成本，而且性能和可靠性都有保障。"

---

### 安全防护

**问题**：安全防护方面，如果用户在一大串请求中包含了某个越权操作，怎么能在这一大串请求中提取到这个越权的操作？或者对于上下文管理中，如果用户连续发送了大量的一长串信息来攻击 Agent，造成大量的 Token 消耗，怎么防护？

**A：**

**问题 1：如何在一大串请求中提取越权操作？**

**解决方案：多层防护架构**

```
用户输入
    ↓
① 关键词快速检测（<1ms）
    - 黑名单关键词匹配
    - 正则表达式匹配
    ↓
② LLM 语义分析（兜底）
    - 分析用户意图
    - 检测是否包含越权操作
    ↓
③ 工具执行前校验
    - 检查参数合法性
    - 检查用户权限
```

**本项目实现**：

```python
# app/guardrails/keyword_checker.py
class KeywordGuardrail:
    # 黑名单关键词
    BLACKLIST_PATTERNS = [
        r"(?i)(drop\s+table|delete\s+from|truncate)",  # SQL注入
        r"(?i)(system\s+prompt|指令|密码|token)",       # 提示词泄露
        r"(?i)(忽略|无视|忘记|override)",               # 越狱尝试
    ]
  
    def check(self, message: str) -> tuple[bool, str]:
        for pattern in self.BLACKLIST_PATTERNS:
            if re.search(pattern, message):
                return False, f"检测到危险模式: {pattern}"
        return True, "通过"
```

```python
# app/guardrails/checker.py - LLM 语义分析
async def check_guardrails(message: str) -> GuardrailResult:
    # 1. 关键词快速检测
    kw_passed, kw_reason = keyword_guardrail.check(message)
    if not kw_passed:
        return GuardrailResult(passed=False, reason=kw_reason)
  
    # 2. LLM 语义分析（兜底）
    prompt = f"""
    检测用户消息是否包含越权操作：
    - 是否试图访问其他用户数据
    - 是否试图执行管理员操作
    - 是否试图绕过安全限制
  
    用户消息：{message}
  
    返回 JSON：{{"is_safe": true/false, "reason": "..."}}
    """
    response = await llm.ainvoke(prompt)
    return parse_guardrail_result(response)
```

**问题 2：如何防护大量 token 消耗攻击？**

**解决方案：多层限流 + Token 预算**

| 防护层                 | 策略                      | 说明                 |
| ---------------------- | ------------------------- | -------------------- |
| **输入长度限制** | 限制单次输入最大长度      | 防止超长文本攻击     |
| **频率限制**     | 限制每分钟请求数          | 防止高频请求攻击     |
| **Token 预算**   | 限制每次请求的 token 消耗 | 防止单次请求消耗过多 |
| **对话历史压缩** | 滑动窗口 + LLM 摘要       | 防止历史消息累积过多 |

**面试话术**：

> "安全防护我采用了多层架构。第一层是关键词快速检测，用正则表达式匹配 SQL注入、提示词泄露等危险模式，延迟<1ms。第二层是 LLM 语义分析，检测是否包含越权操作或越狱尝试。第三层是工具执行前校验，检查参数合法性和用户权限。对于 Token 消耗攻击，我采用了输入长度限制、频率限制、Token 预算、对话历史压缩等多层防护。特别是对话历史压缩，使用滑动窗口 + LLM 摘要的方式，既保留了上下文，又控制了 Token 消耗。"

---

**问题 3：Agent 项目中怎么做鉴权？比如有的用户没有权限调用某些工具，应该怎么处理？**

**A：**

**方案概述**：
根据用户 token 进行角色分类，采用两层防线实现权限控制——动态工具注册（主防线）+ 运行时权限校验（安全网）。

**角色定义**：

| 角色     | 标识      | 典型场景                             |
| -------- | --------- | ------------------------------------ |
| 游客     | `guest` | 未登录，只能浏览商户和搜索知识库     |
| 普通用户 | `user`  | 已登录，可查询券和预约，不能执行操作 |
| VIP 用户 | `vip`   | 付费用户，拥有全部功能               |
| 管理员   | `admin` | 内部测试，拥有全部功能               |

**权限模型**：

采用 RBAC（基于角色的访问控制），权限精确到工具级别：

| 工具                   | 所属 Agent  | 权限级别 | guest | user | vip | admin |
| ---------------------- | ----------- | -------- | ----- | ---- | --- | ----- |
| `search_shops`       | shop        | 查询     | ✅    | ✅   | ✅  | ✅    |
| `get_shop_detail`    | shop        | 查询     | ✅    | ✅   | ✅  | ✅    |
| `search_knowledge`   | knowledge   | 查询     | ✅    | ✅   | ✅  | ✅    |
| `query_vouchers`     | voucher     | 查询     | ❌    | ✅   | ✅  | ✅    |
| `query_reservations` | reservation | 查询     | ❌    | ✅   | ✅  | ✅    |
| `purchase_voucher`   | voucher     | 操作     | ❌    | ❌   | ✅  | ✅    |
| `create_reservation` | reservation | 操作     | ❌    | ❌   | ✅  | ✅    |
| `cancel_reservation` | reservation | 操作     | ❌    | ❌   | ✅  | ✅    |

**两层防线架构**：

```
用户请求 (token → role)
  ↓
[第一层：动态工具注册]  ← 主防线，Manifest 层过滤
  根据角色只注册有权限的工具
  Orchestrator 只看到可用 Agent + few-shot
  子 Agent 只 bind 有权限的 tools
  LLM 物理上无法调用无权工具
  ↓
[第二层：运行时权限校验]  ← 安全网，Execution 层兜底
  工具执行前检查 user_role
  防御链式攻击、参数注入、LLM 幻觉
  ↓
返回用户
```

**为什么需要两层**：动态注册解决 95% 的问题——LLM 看不到无权工具，从源头不编排无权任务。运行时校验解决 5% 的边缘情况——链式攻击（Agent A 的输出触发 Agent B 调用无权工具）、参数注入（调用授权工具但传越权参数）、LLM 幻觉（极小概率但存在）。

**动态 Prompt 注入的关键**：不能只动态生成可用 Agent 列表，few-shot 示例和路由规则也要跟着角色变。比如 guest 看到的示例只有 shop 和 knowledge，vip 看到的才有 voucher 和 reservation。如果 few-shot 不变，LLM 看到示例里有"买券"但工具列表里没有，会困惑。

**权限传递机制**：

1. **路由层**：token 解析 → 角色识别（`resolve_role(token)`）→ 写入 state 的 `user_role` 字段
2. **编排层**：Primary Agent 读取 `user_role`，调用 `build_role_section(role)` 动态生成可用 Agent + 路由规则 + few-shot，替换 system_prompt 中的 `{role_section}` 占位符
3. **子 Agent 层**：各子 Agent 的 `run()` 接收 `user_role`，调用 `get_allowed_tools()` 过滤工具后 `bind_tools()`，LLM 只能看到有权限的工具
4. **工具层（兜底）**：`retry_tool_call()` 调用前执行 `check_tool_permission(tool_name, user_role)`，防御链式攻击和参数注入

**面试话术**：

> "权限控制我用两层防线。第一层是动态工具注册——根据用户角色，子 Agent 只 bind 有权限的工具，LLM 物理上看不到也调用不了无权工具。Orchestrator 层面的 system_prompt 也动态生成，可用 Agent、路由规则、few-shot 示例三者绑定角色一起注入。第二层是运行时权限校验——即使 LLM 幻觉尝试越权调用，工具执行前也会被确定性代码拒绝。选两层是因为动态注册解决 95% 的问题，运行时校验兜底 5% 的边缘情况，比如链式攻击和参数注入。"

---

# 第二部分：梦莹本地生活咨询平台

## 项目介绍

### 项目概述

#### 技术栈

SpringBoot + Redis + MyBatis-Plus + Redisson + Kafka + SpringAI + Milvus

#### 核心功能

- 商户查询与缓存优化
- 秒杀优惠券（防超卖、一人一单）
- 点赞、关注、Feed 流
- 智能客服（RAG + 大模型）

---

### Redis 缓存优化问题

#### 你提到使用逻辑过期防止缓存击穿，具体是怎么实现的？

**参考答案：**

```java
// 核心思路：不设置 TTL，而是在 value 中存储过期时间
public void saveWithLogicExpire(String key, Object data, long time, TimeUnit unit) {
    // 1. 计算逻辑过期时间
    long expireTime = System.currentTimeMillis() + unit.toMillis(time);
  
    // 2. 封装数据
    RedisData redisData = new RedisData();
    redisData.setData(data);
    redisData.setExpireTime(expireTime);
  
    // 3. 写入 Redis（不设 TTL）
    stringRedisTemplate.opsForValue().set(key, JSON.toJSONString(redisData));
}

// 查询时检查是否过期
public <T, ID> T queryWithLogicExpire(String key, Class<T> type, ID id, Function<ID, T> dbFallback) {
    String json = stringRedisTemplate.opsForValue().get(key);
    RedisData redisData = JSON.parseObject(json, RedisData.class);
  
    // 检查逻辑过期时间
    if (redisData.getExpireTime().isBefore(LocalDateTime.now())) {
        // 已过期，开启异步线程重建缓存
        CACHE_REBUILD_EXECUTOR.submit(() -> {
            try {
                T result = dbFallback.apply(id);
                saveWithLogicExpire(key, result, 10, TimeUnit.MINUTES);
            } catch (Exception e) {
                log.error("重建缓存失败", e);
            }
        });
    }
  
    // 未过期，直接返回
    return JSON.parseObject(redisData.getData(), type);
}
```

**追问：逻辑过期和 TTL 过期有什么区别？**

| 对比项               | TTL 过期                | 逻辑过期                     |
| -------------------- | ----------------------- | ---------------------------- |
| **实现复杂度** | 简单，设置 expire 即可  | 复杂，需要封装数据           |
| **缓存击穿**   | 过期瞬间大量请求打到 DB | 平滑过渡，第一个请求重建缓存 |
| **适用场景**   | 普通数据                | 热点数据                     |
| **数据一致性** | 过期后不一致            | 重建期间返回旧数据           |

---

#### 逻辑过期解决缓存击穿问题，Redis 中存满了怎么办？

**A：** 逻辑过期不设 Redis TTL，key 永远不过期，确实会撑满 Redis。解决方案是"逻辑过期 + 兜底 TTL"：value 里存逻辑过期时间（比如 30 分钟），用于判断是否需要异步刷新；Redis 层面设一个很长的 TTL（比如 24 小时）作为兜底。

两层过期的配合：

- **逻辑过期（30 分钟）**：热点数据到期后异步刷新，不阻塞请求
- **Redis TTL（24 小时）**：兜底清理，冷数据没人访问 24 小时后自动过期释放内存

热点数据在 30 分钟内会被持续访问和刷新，Redis TTL 永远不会触发；冷数据没人访问，24 小时后自动过期。

---

#### 在我的项目中哪些地方需要避免缓存击穿？

**A：** 项目中最需要防缓存击穿的有两个场景：

**1. 商户分类列表（ShopType）** — 最需要防击穿。首页流量最大，所有用户都会访问，且只有一个 key（`CACHE_SHOP_TYPE`），过期瞬间所有请求同时打到数据库。当前代码只做了简单的 set + TTL，没有防击穿措施。适合用互斥锁或逻辑过期。

**2. 商户详情（Shop）** — 秒杀场景下同一家商户会被大量并发访问。当前用 `queryWithPassThrough` 只防了缓存穿透，互斥锁和逻辑过期的防击穿方案已写好但是注释掉了。

**3. 秒杀库存** — 不需要防击穿，因为没有缓存层，直接用 Redis + Lua 操作库存，本身就是原子操作。

---

#### 缓存穿透你是怎么解决的？布隆过滤器和缓存空值各有什么优缺点？

**参考答案：**

```java
// 缓存空值方案
public <T, ID> T queryWithPassThrough(String key, Class<T> type, ID id, Function<ID, T> dbFallback) {
    String cacheValue = stringRedisTemplate.opsForValue().get(key);
  
    // 1. 命中缓存（包括空值）
    if (StrUtil.isNotBlank(cacheValue)) {
        return JSON.parseObject(cacheValue, type);
    }
  
    // 2. 缓存是空字符串（表示之前查询过，数据库中没有）
    if (cacheValue != null) {
        return null;
    }
  
    // 3. 查询数据库
    T dbResult = dbFallback.apply(id);
  
    // 4. 数据库也没有，写入空值（TTL 设短一点）
    if (dbResult == null) {
        stringRedisTemplate.opsForValue().set(key, "", 2, TimeUnit.MINUTES);
        return null;
    }
  
    // 5. 数据库有，写入缓存
    stringRedisTemplate.opsForValue().set(key, JSON.toJSONString(dbResult), 30, TimeUnit.MINUTES);
    return dbResult;
}
```

**优缺点对比：**

| 方案                 | 优点               | 缺点                         | 适用场景                     |
| -------------------- | ------------------ | ---------------------------- | ---------------------------- |
| **缓存空值**   | 实现简单，兼容性好 | 需要设置 TTL，可能短暂不一致 | 穿透不频繁，能容忍短暂不一致 |
| **布隆过滤器** | 内存占用小，查询快 | 有误判率，实现复杂           | 海量数据，穿透频繁           |

---

#### Redis 分布式锁的实现原理？Redisson 的 WatchDog 机制是什么？

**参考答案：**

```java
// Redisson 分布式锁
RLock lock = redisson.getLock("lock:seckill:" + voucherId);

// 尝试加锁
if (lock.tryLock(0, -1, TimeUnit.SECONDS)) {
    try {
        // 1. 检查库存
        int stock = Integer.parseInt(stringRedisTemplate.opsForValue().get("stock:" + voucherId));
        if (stock <= 0) {
            return Result.fail("库存不足");
        }
  
        // 2. 扣减库存（Lua 脚本保证原子性）
        String script = """
            if tonumber(redis.call('get', KEYS[1])) <= 0 then
                return 0
            end
            return redis.call('decr', KEYS[1])
        """;
        Long result = stringRedisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Arrays.asList("stock:" + voucherId)
        );
  
        if (result == 0) {
            return Result.fail("库存不足");
        }
  
        // 3. 创建订单
        createOrder(voucherId, userId);
  
        return Result.ok("下单成功");
    } finally {
        lock.unlock();
    }
} else {
    return Result.fail("不允许重复下单");
}
```

**WatchDog 机制：**

- **问题**：业务执行时间超过锁的 TTL，锁自动释放导致并发问题
- **解决**：Redisson 启动后台线程，每 10 秒检查一次，如果锁还持有，就续期
- **注意**：WatchDog 只在 `tryLock()` 不指定 leaseTime 时生效
- **续期时长**：默认锁 TTL 为 30 秒，每 10 秒续期一次，保持 30 秒

---

### 超卖和一人一单问题

#### 秒杀活动开始前什么时候把优惠券信息添加到redis中？设置ttl吗？

**A：**

**添加时机：**

优惠券库存信息是在**创建秒杀优惠券时**就添加到 Redis 中的，而不是在秒杀活动开始前才添加。

```java
// VoucherServiceImpl.java
@Override
@Transactional
public void addSeckillVoucher(Voucher voucher) {
    // 保存优惠券
    save(voucher);
    // 保存秒杀信息
    SeckillVoucher seckillVoucher = new SeckillVoucher();
    seckillVoucher.setVoucherId(voucher.getId());
    seckillVoucher.setStock(voucher.getStock());
    seckillVoucher.setBeginTime(voucher.getBeginTime());
    seckillVoucher.setEndTime(voucher.getEndTime());
    seckillVoucherService.save(seckillVoucher);
  
    // 保存秒杀库存到 Redis，设置过期时间
    long seconds = Duration.between(LocalDateTime.now(), voucher.getEndTime().plusHours(1))
            .getSeconds();
  
    if (seconds > 0) {
        stringRedisTemplate.opsForValue().set(
            RedisConstants.SECKILL_STOCK_KEY + voucher.getId(),
            voucher.getStock().toString(),
            seconds,
            TimeUnit.SECONDS
        );
    }
}
```

**TTL 设置策略：**

采用 **TTL + 定时任务兜底** 的方式：

| 策略               | 作用                                             | 触发时机       |
| ------------------ | ------------------------------------------------ | -------------- |
| **TTL 过期** | 主动清理，正常情况下的主要清理方式               | Redis 自动触发 |
| **定时任务** | 兜底清理，处理异常情况（Redis 重启、TTL 失效等） | 每天凌晨 2 点  |

**定时任务兜底代码：**

```java
@Component
public class SeckillCleanupTask {
  
    @Resource
    private StringRedisTemplate stringRedisTemplate;
  
    @Resource
    private ISeckillVoucherService seckillVoucherService;
  
    @Scheduled(cron = "0 0 2 * * ?")
    public void cleanupExpiredSeckillStock() {
        List<SeckillVoucher> expiredVouchers = seckillVoucherService.lambdaQuery()
                .lt(SeckillVoucher::getEndTime, LocalDateTime.now().minusHours(1))
                .list();
  
        for (SeckillVoucher voucher : expiredVouchers) {
            String key = RedisConstants.SECKILL_STOCK_KEY + voucher.getVoucherId();
            stringRedisTemplate.delete(key);
        }
    }
}
```

> **一句话总结**：创建秒杀优惠券时就写入 Redis 并设置 TTL（秒杀结束时间 + 1 小时），同时用定时任务兜底清理，防止 Redis 重启或 TTL 失效导致数据残留。

---

#### 如何防止超卖？Redis+Lua 脚本的作用是什么？

**参考答案：**

```lua
-- 超卖检查 + 库存扣减（原子操作）
local stock = tonumber(redis.call('get', KEYS[1]))
if stock <= 0 then
    return 0  -- 库存不足
end
return redis.call('decr', KEYS[1])  -- 扣减库存
```

**为什么用 Lua 脚本？**

1. **原子性**：Redis 执行 Lua 脚本时不会插入其他命令
2. **避免竞态条件**：检查库存和扣减库存不会被打断
3. **减少网络开销**：一次网络请求完成多个操作

**对比方案：**

| 方案                   | 优点     | 缺点                        |
| ---------------------- | -------- | --------------------------- |
| **数据库悲观锁** | 简单可靠 | 性能差，并发低              |
| **数据库乐观锁** | 性能好   | 失败需要重试                |
| **Redis+Lua**    | 性能最好 | 需要维护 Redis 和 DB 一致性 |

---

#### 一人一单怎么实现？

参考 [Kafka 异步优化中一人多单问题的三层校验](#一人多单问题的三层校验)

---

### 多级缓存

#### 为什么需要 Caffeine+Redis 二级缓存？怎么保证一致性？

**参考答案：**

```java
// 二级缓存封装
public <T, ID> T queryWithTwoLevelCache(String key, Class<T> type, ID id, Function<ID, T> dbFallback) {
    // 1. 查询本地缓存
    T localCache = caffeineCache.get(key);
    if (localCache != null) {
        return localCache;
    }
  
    // 2. 查询 Redis 缓存
    String redisJson = stringRedisTemplate.opsForValue().get(key);
    if (StrUtil.isNotBlank(redisJson)) {
        T redisCache = JSON.parseObject(redisJson, type);
        // 写入本地缓存
        caffeineCache.put(key, redisCache);
        return redisCache;
    }
  
    // 3. 查询数据库
    T dbResult = dbFallback.apply(id);
  
    if (dbResult != null) {
        // 4. 写入两级缓存
        stringRedisTemplate.opsForValue().set(key, JSON.toJSONString(dbResult), 30, TimeUnit.MINUTES);
        caffeineCache.put(key, dbResult);
    }
  
    return dbResult;
}
```

**一致性保证：**

1. **主动失效**：数据更新时，同时删除 Redis 和本地缓存
2. **设置短 TTL**：本地缓存 TTL 设得比 Redis 更短
3. **监听 Redis 失效**：通过 Redis Keyspace Notifications 感知失效

**性能提升：**

| 缓存层级          | 访问延迟 | QPS      |
| ----------------- | -------- | -------- |
| Caffeine 本地缓存 | < 1ms    | 100 万+  |
| Redis 远程缓存    | 1-5ms    | 10 万+   |
| 数据库            | 10-100ms | 1 万以下 |

---

#### **如何使用Redis Pub/Sub + 本地缓存实现多级缓存？**

**参考答案：**

**方案设计：**

使用 Redis Pub/Sub 模式 + 本地缓存（Caffeine）+ 10 分钟过期时间兜底的方案实现多级缓存。

**架构：**

```
用户请求 → 本地缓存（Caffeine）→ Redis → MySQL
                ↑
        Redis Pub/Sub 通知清除
```

**实现步骤：**

1. **查询流程（Cache Aside 模式）：**

```java
public Shop getShopById(Long id) {
    // 1. 查询本地缓存
    Shop shop = localCache.getIfPresent(id);
    if (shop != null) {
        return shop;
    }
  
    // 2. 查询Redis缓存
    String json = redisTemplate.opsForValue().get("cache:shop:" + id);
    if (json != null) {
        shop = JSON.parseObject(json, Shop.class);
        localCache.put(id, shop);  // 写入本地缓存
        return shop;
    }
  
    // 3. 查询数据库
    shop = shopMapper.selectById(id);
    if (shop != null) {
        redisTemplate.opsForValue().set("cache:shop:" + id, JSON.toJSONString(shop), 30, TimeUnit.MINUTES);
        localCache.put(id, shop);
    }
    return shop;
}
```

2. **更新流程（发送通知）：**

```java
public void updateShop(Shop shop) {
    // 1. 更新数据库
    shopMapper.updateById(shop);
  
    // 2. 删除Redis缓存
    redisTemplate.delete("cache:shop:" + shop.getId());
  
    // 3. 发送Pub/Sub通知所有实例清除本地缓存
    redisTemplate.convertAndSend("shop:cache:clear", shop.getId().toString());
}
```

3. **订阅消息（清除本地缓存）：**

```java
@Component
public class CacheMessageListener {
  
    @Autowired
    private Cache<Long, Shop> localCache;
  
    public void onMessage(String message) {
        Long shopId = Long.parseLong(message);
        localCache.invalidate(shopId);
    }
}
```

4. **配置本地缓存（10分钟过期兜底）：**

```java
@Bean
public Cache<Long, Shop> localCache() {
    return Caffeine.newBuilder()
        .expireAfterWrite(10, TimeUnit.MINUTES)  // 10分钟过期兜底
        .maximumSize(1000)
        .build();
}
```

**为什么选择这个方案：**

| 优点     | 说明                                      |
| -------- | ----------------------------------------- |
| 实时性高 | Pub/Sub 毫秒级通知                        |
| 实现简单 | 不需要引入额外的 MQ 组件                  |
| 可靠性   | 10 分钟过期时间兜底，消息丢失也能自动恢复 |
| 性能好   | 本地缓存访问速度极快（纳秒级）            |

**消息丢失场景及处理：**

- 服务实例重启期间：消息丢失，10 分钟后自动过期
- Redis 重启：消息丢失，10 分钟后自动过期
- 网络抖动：消息丢失，10 分钟后自动过期

**面试话术：**
"项目中使用 Redis Pub/Sub + Caffeine 本地缓存实现多级缓存。数据更新时，通过 Redis Pub/Sub 通知所有服务实例清除本地缓存，保证数据一致性。同时设置 10 分钟过期时间作为兜底，即使消息丢失，最多 10 分钟后本地缓存自动过期，从 Redis 或数据库加载最新数据。这个方案兼顾了实时性和可靠性，实现简单，不需要引入额外的 MQ 组件。"

---

#### Caffeine 的缓存淘汰策略是什么，为什么选这个策略？

**A：** Caffeine 用的是 **Window TinyLFU（W-TinyLFU）** 策略，结合了 LRU 和 LFU 的优点。

传统 LRU 的问题是突发流量会把热点数据挤掉，传统 LFU 的问题是历史频率高的数据占着不淘汰。W-TinyLFU 的做法是把缓存分为两部分：

- **窗口缓存（1%）**：用 LRU，新数据先进这里，给新数据一个"试用期"
- **主缓存（99%）**：用 LFU，保留真正频繁访问的热点数据

窗口满了要进入主缓存时，和主缓存中最不频繁的数据比较，频率高的留下，低的淘汰。用 Count-Min Sketch 算法统计频率，内存开销极小。

命中率对比：LRU ~80%、LFU ~85%、W-TinyLFU ~90%+，在大多数场景下接近理论最优。

---

### ⭐⭐Kafka 异步优化

#### 一人多单问题的三层校验

**问题背景：**

秒杀场景下如何防止同一用户重复下单（一人多单）？

**面试时回答：**

> 我对这个秒杀流程中的一人多单问题采用了三层纵深防御：
>
> 第一层是 Redis + Lua 脚本，在用户请求入口处通过 SISMEMBER 判断用户是否已经下过单，同时扣减库存，这一步是原子操作，能拦截住绝大多数的重复请求。
>
> 第二层是消息去重表，在消费者消费消息之前，先用 Kafka 消息的 partition + offset 组合作为主键插入一张消息去重表，如果插入成功说明是首次消费，执行后续业务逻辑；如果主键冲突说明已经消费过了，直接跳过。这一层解决的是 Kafka 重复投递、消费者 rebalance、崩溃重消费等场景。
>
> 第三层是数据库唯一索引，在秒杀订单表上对 (user_id, voucher_id) 建立唯一索引，插入订单时如果触发 DuplicateKeyException，就回滚库存。这是最终的兜底保障，即使前两层都失效，数据库层面也能保证不会出现重复订单。

**三层校验架构：**

```
用户请求
    │
    ▼
┌─────────────────────────────────────────┐
│ 第一层：Redis + Lua 脚本                 │
│ SISMEMBER 检查是否已下单 + 库存扣减      │ ← 原子操作，响应最快（微秒级）
│ 拦截绝大多数重复请求                      │
└─────────────────────────────────────────┘
    │ 通过
    ▼
┌─────────────────────────────────────────┐
│ Kafka 异步削峰                           │
│ 生产者发送消息到 seckill.order topic     │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 第二层：消息去重表                        │
│ 用 partition+offset 作为主键插入去重表    │ ← 消费前判重，避免无意义的业务执行
│ 成功 → 首次消费，继续处理                 │
│ 冲突 → 已消费过，直接跳过                 │
└─────────────────────────────────────────┘
    │ 首次消费
    ▼
┌─────────────────────────────────────────┐
│ 扣减库存 + 创建秒杀订单                  │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 第三层：数据库唯一索引                    │
│ UNIQUE KEY (user_id, voucher_id)        │ ← 最终兜底，任何遗漏都能拦住
│ DuplicateKeyException → 回滚库存         │
└─────────────────────────────────────────┘
```

**每一层解决的问题：**

| 防护层           | 位置         | 机制                                  | 解决的问题                                                                                                                                                                                                                 |
| ---------------- | ------------ | ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **第一层** | 应用入口     | Redis + Lua（SISMEMBER + 原子扣减）   | 用户重复点击、前端重复请求，响应最快                                                                                                                                                                                       |
| **第二层** | Kafka 消费者 | 消息去重表（partition + offset 主键） | Kafka 重复投递、消费者 rebalance、崩溃后重消费，在业务逻辑执行之前拦截                                                                                                                                                     |
| **第三层** | 数据库       | 唯一索引 + DuplicateKeyException      | 最终兜底：① 代码 Bug 导致去重逻辑失效（如消息去重表 insert 被跳过）；② 分布式锁失效（Redis 故障、锁过期）；③ Redis 数据不一致（Lua 脚本检查通过但用户实际已下单）。即使前两层都失效，数据库层面也能保证不会出现重复订单 |

**第一层关键点：为什么不用分布式锁？**

第一层的资格校验（SISMEMBER 判重 + 扣库存）不需要分布式锁，因为 Lua 脚本在 Redis 服务端是原子执行的，脚本本身就是临界区：

- 分布式锁防的是"读-改-写"竞态，而 Redis 单线程执行命令，一个 Lua 脚本一旦开始就不会被其他请求插入，校验和扣减之间不存在竞态窗口——**用原子性换掉了互斥**
- 性能差距的本质不是"绕过了串行"（两者最终都在 Redis 单线程上串行），而是**临界区大小**：锁方案的临界区跨越加锁+查+判+扣+解锁约 5 次网络往返，同券请求全部排队，吞吐被压到 1/临界区时长（约几百~2000 QPS）；Lua 的临界区只有服务端微秒级内存操作，客户端 1 次 RTT，吞吐可达 5~10 万 QPS+
- 秒杀的锁只能是券级全局锁（库存是共享资源），等于把高并发退化成串行，用锁做入口校验是反模式

Lua 的代价与边界：

- 脚本执行会阻塞整个 Redis 实例，所以脚本必须是 O(1)~O(logN) 的短操作
- Lua 只保证 Redis 内部原子性，跨存储（Redis→MySQL）的一致性仍需去重表、唯一索引兜底——这正是三层纵深防御存在的原因

**第二层关键点：为什么不用分布式锁？**

之前的方案在消费者端用 Redisson 分布式锁（key=userId）保证同一用户串行处理，但存在以下问题：

- Redisson 的 watchdog 续期机制在消费者线程发生 Full GC（Stop-The-World）时可能失效，导致锁提前过期
- 分布式锁依赖 Redis，如果 Redis 出现故障，锁机制就失效了
- 分布式锁只解决了并发问题，不解决 Kafka 重复投递问题

改用消息去重表后：

- 数据库主键冲突是原子操作，不依赖任何外部组件
- 在业务逻辑执行之前就拦截，连库存扣减都不会执行，避免无意义的数据库操作
- 不需要锁续期、没有 watchdog 问题

**第二层核心代码：**

```java
@Override
@Transactional
public void createSeckillOrder(SeckillVoucherOrder order, String msgKey) {
    // 第二层：消息去重表判重
    try {
        messageProcessedMapper.insert(msgKey);
    } catch (DuplicateKeyException e) {
        log.info("消息已处理过，跳过: {}", msgKey);
        return;
    }
  
    // 扣减库存
    boolean success = seckillVoucherService.update()
            .setSql("stock = stock - 1")
            .eq("voucher_id", order.getVoucherId())
            .gt("stock", 0)
            .update();
    if (!success) {
        log.error("库存不足");
        return;
    }
  
    // 第三层：创建订单（唯一索引兜底）
    try {
        save(order);
    } catch (DuplicateKeyException e) {
        seckillVoucherService.update()
                .setSql("stock = stock + 1")
                .eq("voucher_id", order.getVoucherId())
                .update();
        log.error("不能重复下单");
    }
}
```

**追问：如果消费者维护去重表，后续业务报错（如数据库连接池满）导致落库失败，去重表已经记录了消息，消费者无法再次消费怎么办？**

***去重表和业务操作在同一个数据库、同一个事务中。***分两种情况：

**1. 数据库完全不可用（连接池满了）：** 去重表的 insert 也执行不了，整个操作一起失败，Kafka 不提交 offset，自动重投。

**2. 业务逻辑失败（如库存不足）：** 去重表的插入在事务最前面，和其他业务操作绑在同一个 `@Transactional` 事务中。事务回滚时去重记录也会被回滚，Kafka 重投时可以重新消费。

不存在"去重表成功但业务失败"的不一致情况，因为两者在同一个事务中，要么一起成功，要么一起回滚。

> "对于一人多单问题，我采用了纵深防御的设计思想，三层校验各司其职：Lua 脚本在入口处快速拦截，消息去重表在消费前拦截避免无意义的业务执行，唯一索引在数据库层面做最终兜底。每一层只处理自己最擅长的场景，层层递进，确保数据一致性。"

---

#### Kafka 如何保证消息的有序性

**A：** Kafka 只保证**单分区有序**，不保证跨分区有序。

要保证全局有序只能用单分区，但吞吐量受限。实际场景中通常只需要部分有序——比如同一用户的订单消息有序就行。项目中用 userId 作为消息 key，Kafka 对 key hash 分配分区，同一个用户的消息一定进同一个分区，保证顺序。不同用户的消息分布在不同分区，可以并行消费。

**追问：一定能保证分区内有序吗？**

默认配置下不能 100% 保证。生产者重试可能导致乱序：msg1 发送失败，msg2 发送成功，msg1 重试成功后排在 msg2 后面。

解决方案是开启幂等性（`enable.idempotence=true`）。核心机制是 PID（Producer ID）+ 序列号。每条消息带一个按分区递增的序列号，Broker 维护每个生产者在每个分区的最大序列号。收到消息时，序列号不连续就拒绝写入（保序），序列号 ≤ 已有最大值就判定为重复直接丢弃（去重）。两件事用同一个序列号机制同时解决。

---

#### 为什么给生产者开启幂等可以保证分区内有序，还能保证消息不会重复消费

**A：** 核心机制是 PID（Producer ID）+ Sequence Number（序列号）。

**保证有序：** Broker 强制要求序列号连续。生产者发 msg2(seq=2) 失败，再发 msg3(seq=3) 时 Broker 发现 seq=2 没成功，拒绝 seq=3。生产者重试 seq=2 成功后，seq=3 才能写入。分区内的顺序由序列号强制保证。

**去重：** 生产者发 msg1(seq=1) 成功但没收到 ack，重试 msg1(seq=1)。Broker 发现 seq=1 已经存在，直接丢弃不重复写入。

一句话总结：Broker 通过序列号同时做了两件事——不连续就拒绝（保序），重复就丢弃（去重）。

---

#### 发送 Kafka 消息失败导致订单丢失的问题是怎么解决的？（Outbox 方案）

**A：** **解决的问题**：秒杀链路里 `kafkaTemplate.send()` 是 fire-and-forget，不等 broker ack——Kafka 故障时静默丢单（Redis 已扣库存、用户看到"成功"、DB 没有订单）。Outbox 把"投递失败"从**丢失变成延迟**：消息先落库建立"必然被投递"的锚点，投递动作全部移到响应之后。

**实现链路（纯轮询，无快车道）：**

```
用户关键路径（同步）：
Lua 原子校验/预扣
  → INSERT tb_outbox（id=预生成 orderId 主键，next_retry_time=now，payload 只 INSERT 不可变）
      └─ INSERT 失败 → rollback_seckill.lua 补偿 Redis（库存加回、SREM 用户）→ "系统繁忙请重试"
  → 返回 orderId（排队语义，前端轮询最终状态）

投递（响应之后，OutboxDispatcher @Scheduled fixedDelay=1s 独家执行）：
扫 status=0 且 next_retry_time<=now 的行（LIMIT 200）
  → CAS claim 抢占：UPDATE 推 next_retry_time=now+30s，影响行数 0 即跳过
  → send().get(2s) → 成功 markSent（status=2）
  → 失败 markRetry 指数退避 min(2^(n+1),60)s；>16 次置死信 status=3 + 告警
  → 每日归档清理 status=2
```

**四个关键设计**：① **落表先于返回**——outbox 行是"必然被投递"承诺的持久化锚点，先返回后异步补写存在崩溃窗口；② **状态单一所有权**——status 只归扫描器流转，表里没有"发送中"态，状态机单调 0→2/0→3 由所有 UPDATE 的 `WHERE status=0` 在 SQL 层强制；③ **claim 租约**——一条原子 UPDATE 同时解决多实例互斥（扫描器天生 multi-active，无主从无需选主，任一实例挂掉活被其他实例自然接管）与宕机自愈（发送中崩溃的行 30s 租约过期自动重回扫描，无需回收任务）；④ **重发即设计**——重发消息 partition+offset 变了、去重表拦不住，由订单主键唯一索引（三层校验第三层）消化，消费幂等是方案的前提而非巧合。

**为什么同步写 MySQL 扛得住——流量漏斗**：outbox 写入量 ≠ 请求量，上限是库存量——10 万人抢 100 张，第 101 个起被 Lua 原子拒绝，根本不写表；且避开的是热点行 update 单行锁排队，outbox 是每单独立新行的纯 INSERT（雪花 ID 单调递增、写最右叶页）。极端再上：攒批写 / 分库分表 / Redis 消息表变体（对账差集补发，代价是可靠性绑 Redis 持久化）。**代价也要讲**：响应时间从"仅依赖 Redis"变成耦合 MySQL 的 P99（抖动时秒杀变慢、MySQL 不可用时接口不可用——用可用性换零丢单）；排队时长多出扫描延迟（平均 ~0.5s，间隔已从 5s 调到 1s；曾实现过"写表后即时发一次"的快车道，投递可压到毫秒级，但双路径竞争需要 10s 保护期避让在途消息，权衡后按业务可接受的延迟回退纯轮询换简单）。

> **一句话话术**：秒杀架构不是"高并发写库"，而是用 Redis 原子校验把并发写整形为库存量的写——outbox 承接的是整形后的涓流；relay 有轮询和 binlog（Debezium）两种实现，我们按运维规模选了轮询，表上"只 INSERT、payload 不可变"的纪律保留了未来切 binlog 订阅的兼容性。业内同构：这个模式叫 Outbox Pattern，RocketMQ 事务消息是它的中间件化版本（半消息+定时回查，回查即兜底扫描；我们的重发竞争靠保护期/租约化解，RocketMQ 用"回查问状态代替盲目重发"从结构上消掉了它）。

#### 用户超时未支付的问题是怎么处理的？（超时关单 + 库存回补）

**A：** **解决的问题**：Lua 校验已扣 Redis 库存、`seckill:order` 已记用户，消费落单后订单停在 status=0 待支付——用户不支付，库存和购买资格就永久虚占；单纯关单不行（Redis 不回补，这份库存再也卖不出去，用户也永远不能再抢），必须**超时关单 + 补偿回补**成一条链路。

**实现链路（与 Outbox 同一套纪律）：**

```
OrderTimeoutTask @Scheduled fixedDelay=60s:
  SELECT id,voucher_id,user_id FROM 订单
    WHERE status=0 AND create_time < now()-5min LIMIT 100
    （新增索引 idx_status_create，常态扫空结果）
  逐单 @Transactional（①②原子）:
    ① CAS 关单: UPDATE SET status=3 WHERE id=? AND status=0
        └─ rowcount=0 → 已被支付/已被别实例关，跳过
    ② DB 回补: tb_seckill_voucher stock+1
    ③ INSERT tb_outbox(topic='seckill.rollback',
         payload={voucherId,userId,orderId}, id=新雪花)

SeckillRollbackConsumer（复用 Dispatcher→Kafka→消费管道）:
  SET NX seckill:rollback:{orderId} EX 1d    ← 幂等闸门
  抢到 → rollback_seckill.lua（incrby stock+1、srem order userId）→ ack
  没抢到（Kafka 重投的重复消息）→ 直接 ack
```

**与支付的竞态**由 CAS 天然仲裁：支付路径同样是 `UPDATE SET status=1 WHERE id=? AND status=0`，谁先提交谁赢——支付赢则关单 rowcount=0、回补消息根本不产生；关单赢则支付接口发现 status=3，返回"订单已取消请重新下单"。

**原子性边界**：关单 + DB 回补 + 回补承诺落表在同一 MySQL 库一个本地事务里搞定，无分布式事务；Redis 回补是异步承诺——③ 提交后 Dispatcher 保证必然投递，Kafka 重投由 SET NX 拦截（`incrby` 不幂等，没闸门重复消费会虚增库存）。

**为什么 rollback 用独立 topic 而不是复用 `seckill.order`**：

| | seckill.order | seckill.rollback |
|---|---|---|
| 生产者 | 下单链路（Lua 通过落 outbox） | 超时关单任务（CAS 取消时落 outbox） |
| 消费者 | SeckillOrderConsumer → 建订单 | SeckillRollbackConsumer → 执行 rollback Lua |
| 幂等机制 | 订单主键唯一索引天然幂等 | incrby 不幂等，靠 SET NX 闸门 |

混用一坑：消费语义相反，任一逻辑异常不提交 offset 时另一种消息被连坐重放；重试/幂等代码无法统一；积压告警含义不同（建单堆积=下单失败，回补堆积=库存虚占），独立 topic + 独立 group 才能分开控消费速率。tb_outbox 表本身共用（topic 列区分），Dispatcher 零改动。

**方向性原则**：所有故障都落在"少卖"安全侧——Kafka/Redis 短暂不可用时回补延迟，Redis 库存暂时偏小（最多少卖，无资损），恢复后消息自动到达；库存小时级全量刷新任务做最终对账。绝不超卖。

> **一句话话术**：关单不是一条 UPDATE，而是"订单状态 CAS 仲裁 + 本地事务包住 DB 回补和承诺落表 + Outbox 保证 Redis 回补最终执行"的三段状态流转；新增的只有扫描任务和回补消费者两个组件，中间全部复用既有 Outbox 管道，幂等靠 CAS 与 SET NX 两道闸门。

### 限流

#### 滑动窗口限流怎么实现？和固定窗口有什么区别？

**参考答案：**

```java
@Aspect
@Component
public class RateLimitAspect {
  
    @Around("execution(* com.hmdp.controller.*.*(..)) && @annotation(rateLimit)")
    public Object around(ProceedingJoinPoint joinPoint, RateLimit rateLimit) throws Throwable {
        // 获取用户 ID
        Long userId = UserHolder.getUser().getId();	//从ThreadLocal中获取用户id
        String key = "rate:limit:" + userId;
  
        // 滑动窗口：使用 ZSet
        long now = System.currentTimeMillis();
        long windowSize = rateLimit.windowSize(); // 窗口大小（毫秒）
        int maxCount = rateLimit.maxCount();      // 最大请求数
  
        // 1. 移除窗口外的数据
        stringRedisTemplate.opsForZSet().removeRangeByScore(key, 0, now - windowSize);
  
        // 2. 统计窗口内请求数
        Long count = stringRedisTemplate.opsForZSet().zCard(key);
  
        if (count != null && count >= maxCount) {
            return Result.fail("请求过于频繁，请稍后再试");
        }
  
        // 3. 添加当前请求
        stringRedisTemplate.opsForZSet().add(key, String.valueOf(now), now);
        stringRedisTemplate.expire(key, windowSize, TimeUnit.MILLISECONDS);
  
        return joinPoint.proceed();
    }
}
```

**滑动窗口 vs 固定窗口：**

| 对比项               | 固定窗口                   | 滑动窗口       |
| -------------------- | -------------------------- | -------------- |
| **实现复杂度** | 简单（incr + expire）      | 较复杂（ZSet） |
| **临界问题**   | 有（窗口切换瞬间流量翻倍） | 无             |
| **精度**       | 低                         | 高             |
| **性能**       | 好                         | 稍差           |
| **内存占用**   | 低                         | 高             |

**示例：**

```
固定窗口问题：
10:00:59 - 100 个请求（窗口 1）
10:01:01 - 100 个请求（窗口 2）
实际 2 秒内处理了 200 个请求，超出限流阈值

滑动窗口解决：
任意时刻往前推 1 秒，请求数都不超过 100
```

---

#### 你了解其他限流算法吗，固定窗口、漏桶、令牌桶，区别是什么？

**参考答案：**

我了解有==固定窗口、滑动窗口、漏桶、令牌桶==这四种限流算法。

**固定窗口算法：**

- **核心缺陷**：窗口切换时2倍流量冲击
- **适用场景**：低频管理后台

**漏桶算法：**

- **核心原理**：请求以固定的速率被处理，不管请求的突发性。它的工作方式类似于一个底部有孔的桶，水（请求）以任意速率流入桶中，但只能以恒定的速率流出。如果桶满了，多余的请求会被丢弃。
- **核心缺陷**：无法处理突发流量。即使系统有空闲资源，请求也只能以固定速率处理
- **适用场景**：数据库写入保护

**令牌桶算法：**

- **核心原理**：以固定速率生成令牌放入桶中，每个请求需要获取一个令牌才能被处理。如果桶中有令牌，请求可以立即被处理，允许一定程度的突发流量（因为桶中积累的令牌可以被一次性使用）。
- **优点**：令牌桶允许突发流量，这在某些场景下是优势
- **核心缺陷**：如果桶中积累了令牌，恶意用户可以利用这些令牌在瞬间发送大量请求，也就是所谓的脉冲攻击，导致其他用户无令牌可用
- **适用场景**：内部API调用

**滑动窗口算法：**

- **核心原理**：将时间窗口细分为多个小窗口，统计滑动窗口内的总请求数。随着时间推移，窗口向前滑动，旧的小窗口数据过期。
- **优点1**：可以提供精确的控制
- **优点2**：滑动窗口通过限制每个时间窗口内的请求数量，通常能够有效应对脉冲攻击。即使攻击者试图在短时间内发起大量请求，由于每个时间窗口内的请求数是有上限的，滑动窗口可以有效抑制这种突发流量，防止恶意脉冲攻击
- **核心缺陷**：内存占用稍高
- **适用场景**：秒杀/防刷/爬虫防护

**算法对比总结：**

| 算法               | 核心缺陷                   | 适用场景           |
| ------------------ | -------------------------- | ------------------ |
| **固定窗口** | 窗口切换时2倍流量冲击      | 低频管理后台       |
| **漏桶**     | 无法处理合法突发流量       | 数据库写入保护     |
| **令牌桶**   | 黑客利用桶容量发起脉冲攻击 | 内部API调用        |
| **滑动窗口** | 内存占用稍高               | 秒杀/防刷/爬虫防护 |

## 项目补充

### 项目中有使用线程池吗？

**参考答案：**

**项目中的使用场景：**

**1. 缓存重建：**

```java
// 逻辑过期缓存重建
private static final ExecutorService CACHE_REBUILD_EXECUTOR = 
    Executors.newFixedThreadPool(10);

public <T, ID> T queryWithLogicExpire(String key, Class<T> type, ID id, Function<ID, T> dbFallback) {
    // ...
    if (redisData.getExpireTime().isBefore(LocalDateTime.now())) {
        // 开启异步线程重建缓存
        CACHE_REBUILD_EXECUTOR.submit(() -> {
            try {
                T result = dbFallback.apply(id);
                saveWithLogicExpire(key, result, 10, TimeUnit.MINUTES);
            } catch (Exception e) {
                log.error("重建缓存失败", e);
            }
        });
    }
    // ...
}
```

**2. 异步下单（秒杀场景）：**

```java
// 使用线程池异步处理订单
ExecutorService orderExecutor = new ThreadPoolExecutor(
    5,                      // 核心线程数
    10,                     // 最大线程数
    60L, TimeUnit.SECONDS,  // 空闲线程存活时间
    new LinkedBlockingQueue<>(1000),  // 队列容量
    new ThreadPoolExecutor.CallerRunsPolicy()  // 拒绝策略
);

// 提交订单处理任务
orderExecutor.submit(() -> {
    handleVoucherOrder(voucherOrder);
});
```

**线程池参数说明：**

| 参数                      | 说明             | 推荐值                                             |
| ------------------------- | ---------------- | -------------------------------------------------- |
| **corePoolSize**    | 核心线程数       | CPU 密集型：CPU 核数 + 1；IO 密集型：CPU 核数 × 2 |
| **maximumPoolSize** | 最大线程数       | 根据业务峰值调整                                   |
| **keepAliveTime**   | 空闲线程存活时间 | 60 秒                                              |
| **workQueue**       | 任务队列         | LinkedBlockingQueue / ArrayBlockingQueue           |
| **rejectPolicy**    | 拒绝策略         | CallerRunsPolicy（调用者运行）                     |

**为什么不用 Executors 创建线程池？**

```java
// ❌ 不推荐：可能 OOM
Executors.newFixedThreadPool(10);  // 队列无界
Executors.newCachedThreadPool();   // 线程数无界

// ✅ 推荐：手动创建，明确参数
new ThreadPoolExecutor(
    corePoolSize,
    maximumPoolSize,
    keepAliveTime,
    TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000),  // 有界队列
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

---

### 项目中在哪里使用了单例模式？

**参考答案：**

**项目中使用了饿汉式单例（静态常量）：**

**1. 秒杀订单线程池（VoucherOrderServiceImpl）：**

```java
@Service
public class VoucherOrderServiceImpl extends ServiceImpl<...> {
  
    // 饿汉式单例：类加载时初始化，线程安全
    private static final ExecutorService SECKILL_ORDER_EXECUTOR = 
        Executors.newSingleThreadExecutor();
  
    @PostConstruct
    private void init() {
        // 启动订单处理线程
        SECKILL_ORDER_EXECUTOR.submit(new VoucherOrderHandler());
    }
}
```

**2. 缓存重建线程池（CacheClient）：**

```java
public class CacheClient {
  
    // 饿汉式单例：全局共享一个线程池
    private static final ExecutorService CACHE_REBUILD_EXECUTOR = 
        Executors.newFixedThreadPool(10);
  
    public <T, ID> T queryWithLogicExpire(...) {
        // 复用同一个线程池重建缓存
        CACHE_REBUILD_EXECUTOR.submit(() -> {
            // 异步重建缓存
        });
    }
}
```

**3. Lua 脚本对象（VoucherOrderServiceImpl）：**

```java
@Service
public class VoucherOrderServiceImpl {
  
    // 静态常量，类加载时初始化
    private static final DefaultRedisScript<Long> SECKILL_SCRIPT;
  
    static {
        SECKILL_SCRIPT = new DefaultRedisScript<>();
        SECKILL_SCRIPT.setLocation(new ClassPathResource("seckill.lua"));
        SECKILL_SCRIPT.setResultType(Long.class);
    }
}
```

**4. UserHolder 工具类（ThreadLocal 封装）：**

```java
public class UserHolder {
  
    // 静态常量，全局共享
    private static final ThreadLocal<UserDTO> tl = new ThreadLocal<>();
  
    public static void saveUser(UserDTO user) {
        tl.set(user);
    }
  
    public static UserDTO getUser() {
        return tl.get();
    }
  
    public static void removeUser() {
        tl.remove();
    }
}
```

**单例模式的特点：**

| 特点                 | 说明                                   |
| -------------------- | -------------------------------------- |
| **私有构造器** | 防止外部 new 对象                      |
| **静态实例**   | 类加载时创建（饿汉）或延迟创建（懒汉） |
| **全局访问点** | 通过静态方法获取实例                   |

**为什么这些场景适合用单例？**

| 场景                  | 原因                                     |
| --------------------- | ---------------------------------------- |
| **线程池**      | 全局只需一个，避免重复创建线程，节省资源 |
| **Lua 脚本**    | 脚本内容不变，复用减少内存开销           |
| **ThreadLocal** | 工具类，全局共享一个 ThreadLocal 实例    |

**饿汉式 vs 懒汉式：**

| 对比项               | 饿汉式                   | 懒汉式                     |
| -------------------- | ------------------------ | -------------------------- |
| **初始化时机** | 类加载时                 | 第一次使用时               |
| **线程安全**   | 天然安全                 | 需要双重检查锁             |
| **资源利用**   | 可能浪费（未使用也创建） | 延迟加载，节省资源         |
| **适用场景**   | 对象创建成本低，必定使用 | 对象创建成本高，可能不使用 |

**项目中使用的都是饿汉式，因为：**

- 线程池、脚本对象创建成本低
- 这些对象在系统运行期间必定会被使用
- 饿汉式代码简单，天然线程安全

---

### 秒杀活动期间Redis宕机了怎么办？

**参考答案：**

**1. 问题分析：秒杀场景下Redis宕机的影响**

```
用户请求 → UserHolder获取用户 → redisIdWorker生成订单ID → Lua脚本检查(库存+防重复) → 返回订单ID
                                                                                    ↓
                                                                    VoucherOrderHandler异步处理
                                                                                    ↓
                                                    从Stream队列读取 → Redisson分布式锁 → 创建订单
```

**Redis宕机后的影响链：**

| 影响项                 | 正常流程               | 宕机后果               |
| ---------------------- | ---------------------- | ---------------------- |
| **用户认证**     | Redis存储Token         | 无法获取当前登录用户   |
| **订单ID生成**   | Redis INCR自增         | 无法生成订单ID         |
| **秒杀资格校验** | Lua脚本检查库存+防重复 | 库存检查失效，可能超卖 |
| **分布式锁**     | Redisson获取锁         | 并发控制失效           |
| **Stream队列**   | 消息持久化在Redis      | 异步订单处理中断       |

**结论：秒杀系统完全瘫痪！**

**2. 解决方案：应用层降级策略**

```java
@Service
@Slf4j
public class VoucherOrderServiceImpl extends ServiceImpl<VoucherOrderMapper, VoucherOrder> 
    implements IVoucherOrderService {

    @Override
    public Result seckillVoucher(Long voucherId) {
        try {
            // 尝试 Redis 秒杀流程
            return doSeckillWithRedis(voucherId);
        } catch (RedisConnectionException e) {
            // Redis 不可用，降级到数据库
            log.error("Redis不可用，降级到数据库秒杀", e);
            return doSeckillWithDatabase(voucherId);
        }
    }
  
    // Redis 秒杀（高性能）
    private Result doSeckillWithRedis(Long voucherId) {
        Long userId = UserHolder.getUser().getId();
        long orderId = redisIdWorker.nextId("order");
  
        Long result = stringRedisTemplate.execute(
            SECKILL_SCRIPT,
            Collections.emptyList(),
            voucherId.toString(), userId.toString(), String.valueOf(orderId)
        );
  
        if (result.intValue() != 0) {
            return Result.fail(result == 1 ? "库存不足" : "不能重复下单");
        }
        return Result.ok(orderId);
    }
  
    // 数据库秒杀（降级 - 性能差但可用）
    @Transactional
    private Result doSeckillWithDatabase(Long voucherId) {
        // 1. 查询秒杀券
        SeckillVoucher voucher = seckillVoucherService.getById(voucherId);
        if (voucher == null) {
            return Result.fail("优惠券不存在");
        }
  
        // 2. 检查库存
        if (voucher.getStock() < 1) {
            return Result.fail("库存不足");
        }
  
        // 3. 检查重复下单（查询数据库）
        Long userId = UserHolder.getUser().getId();
        Long count = query()
            .eq("user_id", userId)
            .eq("voucher_id", voucherId)
            .count();
        if (count > 0) {
            return Result.fail("不能重复下单");
        }
  
        // 4. 生成订单ID（使用数据库自增或UUID）
        long orderId = generateOrderId();
  
        // 5. 扣减库存（乐观锁）
        boolean success = seckillVoucherService.update()
            .setSql("stock = stock - 1")
            .eq("voucher_id", voucherId)
            .gt("stock", 0)  // 乐观锁，防止超卖
            .update();
      
        if (!success) {
            return Result.fail("库存不足");
        }
  
        // 6. 创建订单
        VoucherOrder order = new VoucherOrder();
        order.setId(orderId);
        order.setUserId(userId);
        order.setVoucherId(voucherId);
        order.setCreateTime(LocalDateTime.now());
        save(order);
  
        return Result.ok(orderId);
    }
  
    // 生成订单ID的备选方案
    private long generateOrderId() {
        // 方案1：UUID
        // return UUID.randomUUID().toString().hashCode();
  
        // 方案2：数据库自增（需要单独表）
        // return dbIdGenerator.nextId();
  
        // 方案3：时间戳+随机数（不完美但可用）
        return System.currentTimeMillis() * 1000 + new Random().nextInt(999);
    }
}
```

**3. 完整的高可用架构**

```
┌─────────────────────────────────────────────────────────────┐
│                  秒杀系统降级架构                            │
│                                                             │
│  用户请求                                                   │
│      ↓                                                     │
│  ┌─────────────────┐                                       │
│  │  用户认证降级    │ → Redis宕机 → 使用Session/Token直接放行│
│  └────────┬────────┘                                       │
│           ↓                                                │
│  ┌─────────────────┐                                       │
│  │  秒杀资格校验    │                                       │
│  └────────┬────────┘                                       │
│           ↓                                                │
│      Redis正常？ ──否──→ 数据库降级流程                      │
│           │                                                │
│          是                                                │
│           ↓                                                │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │ Lua脚本检查     │  │ 库存扣减        │                  │
│  │ 库存+防重复    │  │ Redis原子操作   │                  │
│  └────────┬────────┘  └────────┬────────┘                  │
│           │                    │                            │
│           └────────┬───────────┘                            │
│                    ↓                                        │
│            返回订单ID                                        │
│                    ↓                                        │
│  ┌─────────────────┐                                       │
│  │ 异步订单创建    │                                       │
│  │ Stream队列消费  │                                       │
│  └─────────────────┘                                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                 数据库降级流程（Redis宕机时）              ││
│  │  1. 查询库存（SELECT）                                   ││
│  │  2. 检查重复（SELECT + COUNT）                           ││
│  │  3. 乐观锁扣库存（UPDATE WHERE stock > 0）               ││
│  │  4. 创建订单（INSERT）                                   ││
│  │  优点：可用                                             ││
│  │  缺点：性能差、无法抗住高并发                             ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

**4. Redis集群方案**

| 方案                    | 故障转移 | 数据分片 | 适用场景   | 复杂度 |
| ----------------------- | -------- | -------- | ---------- | ------ |
| **主从+哨兵**     | ✅ 自动  | ❌ 否    | 中小规模   | 低     |
| **Redis Cluster** | ✅ 自动  | ✅ 是    | 大规模生产 | 中     |
| **Codis**         | ✅ 自动  | ✅ 是    | 大规模生产 | 中     |

**集群配置示例：**

```yaml
# Redis Sentinel
spring:
  data:
    redis:
      sentinel:
        master: mymaster
        nodes: 192.168.1.10:26379,192.168.1.11:26379,192.168.1.12:26379

# Redis Cluster
spring:
  data:
    redis:
      cluster:
        nodes: 192.168.1.10:6379,192.168.1.11:6379,192.168.1.12:6379
        max-redirects: 3
```

**5. 总结：Redis宕机的应对策略**

| 层级                   | 正常流程       | 降级策略                 | 影响                     |
| ---------------------- | -------------- | ------------------------ | ------------------------ |
| **用户认证**     | Redis存储Token | 放行，业务层处理         | 部分功能受限             |
| **订单ID生成**   | Redis INCR     | 数据库序列/时间戳        | 订单ID格式改变           |
| **秒杀资格校验** | Lua脚本检查    | 数据库查询+乐观锁        | 性能下降，可能超时       |
| **库存扣减**     | Redis原子操作  | 数据库乐观锁             | 可能出现超卖（需监控）   |
| **重复下单检查** | Redis Set      | 数据库唯一索引           | 可能重复下单（事务保证） |
| **分布式锁**     | Redisson       | 数据库悲观锁（最后兜底） | 并发性能下降             |

**关键点：**

- ✅ Redis 宕机≠系统崩溃，只是降级到"慢速模式"
- ✅ 业务层必须做好 try-catch 异常处理
- ✅ 数据库层使用乐观锁防止超卖
- ✅ 监控告警及时发现 Redis 故障
- ✅ 降级期间限流保护数据库
- ✅ 集群可以缩短故障时间，但不能完全消除不可用（故障转移需要10-30秒）
