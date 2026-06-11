# Agno 框架运行时消息流与工具消息机制深度分析

> 基于 Agno 框架源码（`/workspace/libs/agno/agno/`）的完整逆向分析
> 核心文件: `agent/_run.py`, `agent/_response.py`, `agent/_default_tools.py`, `reasoning/manager.py`, `reasoning/helpers.py`, `reasoning/default.py`, `models/message.py`, `learn/machine.py`

---

## 一、完整运行时消息流全景图

一次Agent运行中，消息列表经历以下完整生命周期：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Agent Run 完整消息流生命周期                               │
│                    (_run / arun)                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Phase 1: 会话初始化                                                        │
│  ├── read_or_create_session()    → 加载/创建AgentSession                    │
│  ├── load_session_state()        → 从DB加载session_state                    │
│  ├── _initialize_session_state() → 注入current_user_id/session_id/run_id   │
│  └── resolve_run_dependencies()  → 解析callable依赖                        │
│                                                                             │
│  Phase 2: 预处理                                                            │
│  ├── execute_pre_hooks()         → 执行前置钩子                             │
│  └── determine_tools_for_model() → 确定可用工具列表                         │
│                                                                             │
│  Phase 3: 消息构建                                                          │
│  └── get_run_messages()          → 构建完整消息列表                         │
│      ├── [1] System Message                                                 │
│      ├── [2] Additional Input (few-shot)                                    │
│      ├── [3] History Messages                                               │
│      ├── [4] User Message (含knowledge+dependencies)                        │
│      └── [5] Input Messages (List[Message])                                 │
│                                                                             │
│  Phase 4: 后台任务启动                                                      │
│  ├── start_memory_future()       → 后台创建记忆                             │
│  ├── start_learning_future()     → 后台提取学习                             │
│  └── start_cultural_knowledge_future() → 后台更新文化知识                   │
│                                                                             │
│  Phase 5: 推理 (reasoning=True时)                                           │
│  └── handle_reasoning()          → 在消息列表中注入推理消息                 │
│      ├── Native推理: 注入reasoning_message到messages                        │
│      └── CoT推理: 注入assistant+reasoning_messages+assistant               │
│                                                                             │
│  Phase 6: 模型调用                                                          │
│  └── call_model_with_fallback()  → 发送messages给LLM                       │
│      ├── 内部while循环处理工具调用                                          │
│      ├── 每次工具调用 → assistant(tool_call) + tool(result)                 │
│      ├── CompressionManager在循环中压缩工具结果                             │
│      └── 返回最终ModelResponse                                              │
│                                                                             │
│  Phase 7: 后处理                                                            │
│  ├── generate_response_with_output_model() → 输出模型处理                   │
│  ├── parse_response_with_parser_model()     → 解析模型处理                  │
│  ├── update_run_response()                  → 更新运行输出                  │
│  ├── convert_response_to_structured_format()→ 结构化转换                    │
│  ├── generate_followups()                   → 生成后续建议                  │
│  └── execute_post_hooks()                   → 执行后置钩子                  │
│                                                                             │
│  Phase 8: 清理存储                                                          │
│  ├── wait_for_background_tasks() → 等待后台任务完成                         │
│  ├── create_session_summary()     → 创建会话摘要                            │
│  └── cleanup_and_store()          → 存储run和session                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、推理消息注入机制 (Phase 5)

### 2.1 两种推理模式的消息注入

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    推理消息注入到消息列表                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  原始消息列表:                                                              │
│  [system] [additional_input] [history...] [user]                            │
│                                                                             │
│  ──── Native推理模式 (DeepSeek/Anthropic/OpenAI/Gemini等) ────             │
│                                                                             │
│  推理Agent处理消息 → 返回reasoning_message                                  │
│  注入到消息列表:                                                            │
│  [system] [additional_input] [history...] [user]                            │
│  [reasoning_message (assistant角色, 含reasoning_content)]                   │
│                                                                             │
│  ──── CoT推理模式 (默认链式思维) ────                                      │
│                                                                             │
│  推理Agent多步处理 → 返回reasoning_messages                                 │
│  注入到消息列表:                                                            │
│  [system] [additional_input] [history...] [user]                            │
│  [assistant: "I have worked through this problem in-depth..."]              │
│  [reasoning_messages... (assistant/tool消息, add_to_agent_memory=False)]    │
│  [assistant: "Now I will summarize my reasoning and provide a final answer"]│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 CoT推理消息注入详解

**实现位置**: `reasoning/helpers.py` L42-L61

```python
def update_messages_with_reasoning(run_messages, reasoning_messages):
    # 注入"开始推理"标记消息
    run_messages.messages.append(
        Message(
            role="assistant",
            content="I have worked through this problem in-depth, running all necessary tools and have included my raw, step by step research. ",
            add_to_agent_memory=False,  # 不存入记忆
        )
    )
    # 注入推理过程消息（标记为不存入记忆）
    for message in reasoning_messages:
        message.add_to_agent_memory = False
    run_messages.messages.extend(reasoning_messages)
    # 注入"推理结束"标记消息
    run_messages.messages.append(
        Message(
            role="assistant",
            content="Now I will summarize my reasoning and provide a final answer. I will skip any tool calls already executed and steps that are not relevant to the final answer.",
            add_to_agent_memory=False,
        )
    )
```

**为什么这样实现**:
- **三明治结构**: 前后assistant消息形成"推理开始/结束"的语义边界，引导主模型理解中间内容是推理过程
- **add_to_agent_memory=False**: 推理过程消息不存入会话记忆，避免后续对话中出现冗长的推理历史
- **保留工具调用**: reasoning_messages中可能包含工具调用和结果，主模型可以看到推理过程中使用了哪些工具，避免重复调用
- **"skip tool calls already executed"**: 明确告诉主模型不要重复执行推理中已完成的工具调用

### 2.3 Native推理 vs CoT推理对比

```
┌──────────────────────┬───────────────────────────┬───────────────────────────┐
│  维度                │  Native推理               │  CoT推理                  │
├──────────────────────┼───────────────────────────┼───────────────────────────┤
│  触发条件            │  reasoning=True +         │  reasoning=True +         │
│                      │  模型原生支持推理          │  模型不支持原生推理        │
│  推理Agent           │  简单Agent(无instructions)│  复杂Agent(6步推理指令)    │
│  消息注入            │  1条reasoning_message     │  3部分(前标记+过程+后标记) │
│  工具使用            │  不使用工具               │  可以使用工具              │
│  输出格式            │  自由文本                 │  ReasoningSteps结构化输出  │
│  步骤控制            │  模型自行决定             │  min_steps/max_steps控制   │
│  Token消耗           │  较低(一次推理)           │  较高(多步推理+工具调用)   │
│  适用模型            │  DeepSeek/Anthropic/      │  任何模型                  │
│                      │  OpenAI/Gemini/Groq等     │                           │
└──────────────────────┴───────────────────────────┴───────────────────────────┘
```

### 2.4 CoT推理Agent的6步指令

**实现位置**: `reasoning/default.py` L13-L94

```
Step 1 - Problem Analysis:    重述问题，确定所需信息和工具
Step 2 - Decompose:           分解子任务，制定至少两种策略
Step 3 - Intent & Planning:   明确意图，选择策略，制定行动计划
Step 4 - Execute:             执行计划，每步记录Title/Action/Result/Reasoning/NextAction/Confidence
Step 5 - Validation:          交叉验证，使用替代方法确认
Step 6 - Final Answer:        提供最终答案
```

**关键设计**:
- `output_schema=ReasoningSteps`: 强制结构化输出，每步包含title/action/result/reasoning/next_action/confidence
- `NextAction`枚举: `CONTINUE`/`VALIDATE`/`FINAL_ANSWER`/`RESET`，控制推理循环
- `tools=agent.tools`: 推理Agent可以使用主Agent的所有工具
- `add_to_agent_memory=False`: 推理过程不污染主Agent的记忆

---

## 三、模型调用循环中的消息演化 (Phase 6)

### 3.1 工具调用循环

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    模型调用内部循环 (Model.process_response)                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  初始消息列表:                                                              │
│  [system] [additional_input] [history...] [user]                           │
│  (可能有reasoning消息)                                                      │
│                                                                             │
│  ──── 第1轮模型调用 ────                                                   │
│                                                                             │
│  LLM返回: assistant消息 (含tool_calls)                                      │
│  消息列表变为:                                                              │
│  [system] [additional_input] [history...] [user]                           │
│  [assistant: content + tool_calls: [{name:"search_knowledge", args:...}]]   │
│                                                                             │
│  执行工具 → 返回结果                                                        │
│  消息列表变为:                                                              │
│  [system] [additional_input] [history...] [user]                           │
│  [assistant: content + tool_calls]                                          │
│  [tool: tool_call_id=xxx, content="搜索结果JSON..."]                        │
│                                                                             │
│  ──── 第2轮模型调用 ────                                                   │
│                                                                             │
│  LLM看到完整消息列表，基于工具结果继续生成                                   │
│  可能返回: 新的assistant消息 (含更多tool_calls 或 最终回答)                  │
│                                                                             │
│  ──── 循环继续直到 ────                                                    │
│  1. LLM不再调用工具 (返回纯文本回答)                                        │
│  2. 达到tool_call_limit限制                                                 │
│  3. 工具调用标记stop_after_tool_call=True                                   │
│                                                                             │
│  ──── CompressionManager介入 ────                                          │
│                                                                             │
│  每轮工具调用后检查:                                                        │
│  should_compress() → 检查token数/工具结果数是否超阈值                       │
│  如果超阈值 → compress() → 将工具结果压缩为compressed_content               │
│                                                                             │
│  压缩后消息:                                                                │
│  [tool: content=原始内容, compressed_content="压缩后的关键信息"]            │
│  发送给LLM时使用compressed_content替代原始content                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Message对象的关键字段

**实现位置**: `models/message.py` L55-L121

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Message 对象关键字段                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  发送给LLM的字段:                                                           │
│  ├── role: str              # system/user/assistant/tool                    │
│  ├── content: str|list      # 消息内容                                     │
│  ├── name: str              # 参与者名称(区分同角色不同参与者)               │
│  ├── tool_call_id: str      # 工具调用ID(tool消息必须)                      │
│  ├── tool_calls: list       # 模型生成的工具调用(assistant消息)              │
│  ├── images/audio/videos/files  # 多媒体内容                               │
│  └── provider_data: dict    # 模型提供商特定数据                            │
│                                                                             │
│  不发送给LLM的元数据:                                                       │
│  ├── compressed_content: str    # 压缩后的工具结果内容                     │
│  ├── reasoning_content: str     # 推理内容(来自原生推理模型)                │
│  ├── redacted_reasoning_content # 脱敏推理内容                              │
│  ├── tool_name: str             # 工具名称                                 │
│  ├── tool_args: any             # 工具参数                                 │
│  ├── tool_call_error: bool      # 工具调用是否出错                         │
│  ├── stop_after_tool_call: bool # 调用后是否停止                           │
│  ├── add_to_agent_memory: bool  # 是否存入Agent记忆(默认True)              │
│  ├── from_history: bool         # 是否来自历史消息                         │
│  ├── temporary: bool            # 是否为临时消息(不持久化)                  │
│  ├── references: MessageReferences  # RAG引用信息                          │
│  ├── citations: Citations        # 模型返回的引用                          │
│  └── metrics: MessageMetrics     # 消息级别指标                            │
│                                                                             │
│  关键方法:                                                                  │
│  ├── get_content(use_compressed_content=False)                              │
│  │   → 如果use_compressed_content=True且有compressed_content,              │
│  │     返回compressed_content而非原始content                               │
│  └── get_content_string() → 始终返回content的字符串形式                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 四、默认工具消息机制深度分析

### 4.1 工具消息全景

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    默认工具及其返回消息                                      │
├──────────────────┬──────────────────────────────────────┬──────────────────┤
│  工具            │  返回给LLM的消息内容                  │  触发条件        │
├──────────────────┼──────────────────────────────────────┼──────────────────┤
│ update_user_     │ "Memory updated: ..." 或             │ enable_agentic_  │
│ memory           │ 具体的任务执行结果                    │ memory=True      │
├──────────────────┼──────────────────────────────────────┼──────────────────┤
│ search_knowledge │ JSON/YAML格式的文档列表               │ search_knowledge │
│ _base            │ 或 "No documents found"               │ =True            │
├──────────────────┼──────────────────────────────────────┼──────────────────┤
│ add_to_knowledge │ "Successfully added to knowledge     │ update_knowledge │
│                  │ base" 或 "Knowledge not available"    │ =True            │
├──────────────────┼──────────────────────────────────────┼──────────────────┤
│ get_chat_history │ JSON格式的聊天历史列表                │ read_chat_       │
│                  │                                      │ history=True     │
├──────────────────┼──────────────────────────────────────┼──────────────────┤
│ get_tool_call_   │ JSON格式的工具调用历史                │ read_tool_call_  │
│ history          │                                      │ history=True     │
├──────────────────┼──────────────────────────────────────┼──────────────────┤
│ search_past_     │ JSON格式的会话预览列表                │ search_past_     │
│ sessions         │ (session_id, created_at, runs)       │ sessions=True    │
├──────────────────┼──────────────────────────────────────┼──────────────────┤
│ read_past_       │ JSON格式的完整会话消息                │ search_past_     │
│ session          │                                      │ sessions=True    │
├──────────────────┼──────────────────────────────────────┼──────────────────┤
│ update_session_  │ "Updated session state: {state}"     │ enable_agentic_  │
│ state            │                                      │ state=True       │
├──────────────────┼──────────────────────────────────────┼──────────────────┤
│ create_or_update │ 文化知识更新结果                      │ enable_agentic_  │
│ _cultural_       │                                      │ culture=True     │
│ knowledge        │                                      │                  │
└──────────────────┴──────────────────────────────────────┴──────────────────┘
```

### 4.2 update_user_memory 深度分析

**作用**: 让Agent主动管理用户记忆（增删改查）。

**实现位置**: `agent/_default_tools.py` L38-L75

```python
def get_update_user_memory_function(agent, user_id=None, async_mode=False):
    def update_user_memory(task: str) -> str:
        """Use this function to submit a task to modify the Agent's memory.
        Describe the task in detail and be specific.
        The task can include adding a memory, updating a memory, deleting a memory, or clearing all memories.
        """
        agent.memory_manager = cast(MemoryManager, agent.memory_manager)
        response = agent.memory_manager.update_memory_task(task=task, user_id=user_id)
        return response
```

**返回消息**: 字符串，描述任务执行状态（如"Memory updated successfully"）

**为什么这样实现**:
- **自然语言接口**: `task`参数是自然语言描述，不是结构化操作。MemoryManager内部用LLM解析task并执行具体操作
- **单一入口**: 增删改查统一通过一个工具入口，减少工具数量，简化LLM的决策空间
- **user_id绑定**: 记忆与用户绑定，确保多用户场景下的隔离

### 4.3 search_knowledge_base 深度分析

**作用**: Agent主动搜索知识库（Agentic RAG）。

**实现位置**: `agent/_default_tools.py` L103-L282

```python
def create_knowledge_search_tool(agent, run_response, run_context, knowledge_filters, ...):
    def search_knowledge_base(query: str) -> str:
        """Use this function to search the knowledge base for information about a query."""
        docs = _messages.get_relevant_docs_from_knowledge(
            agent, query=query, filters=knowledge_filters, run_context=run_context
        )
        return _format_results(docs)  # JSON或YAML格式
```

**返回消息**: JSON/YAML格式的文档列表，或"No documents found"

**与add_knowledge_to_context的对比**:
```
┌──────────────────────┬───────────────────────────┬───────────────────────────┐
│  维度                │  add_knowledge_to_context │  search_knowledge_base    │
│                      │  (自动RAG)                │  (Agentic RAG)            │
├──────────────────────┼───────────────────────────┼───────────────────────────┤
│  触发方式            │  每次run自动执行          │  LLM主动调用工具          │
│  注入位置            │  用户消息(追加到末尾)     │  工具结果消息             │
│  检索时机            │  构建消息时(模型调用前)    │  模型调用后(工具执行时)   │
│  检索次数            │  1次(基于用户输入)        │  多次(基于LLM判断)        │
│  检索查询            │  用户原始输入             │  LLM构造的查询            │
│  Token效率           │  每次都消耗(可能不相关)   │  按需消耗(更精准)         │
│  适用场景            │  简单查询，知识库稳定     │  复杂推理，需多轮检索     │
│  结果追踪            │  run_response.references  │  run_response.references  │
│  过滤器支持          │  knowledge_filters        │  knowledge_filters +      │
│                      │                           │  agentic_filters          │
└──────────────────────┴───────────────────────────┴───────────────────────────┘
```

**Agentic Filters机制**:
当`enable_agentic_knowledge_filters=True`时，工具接受`filters`参数，LLM可以自主构造搜索过滤器：
```python
def search_knowledge_base_with_filters(query: str, filters: Optional[List[KnowledgeFilter]] = None) -> str:
    # LLM可以传入自定义过滤器
    docs = _messages.get_relevant_docs_from_knowledge(
        agent, query=query, filters=_resolve_filters(filters), ...
    )
```

### 4.4 search_past_sessions / read_past_session 深度分析

**作用**: 让Agent搜索和阅读历史会话，实现跨会话记忆检索。

**实现位置**: `agent/_default_tools.py` L459-L609

**search_past_sessions返回消息**:
```json
[
  {
    "session_id": "abc123",
    "created_at": "1700000000",
    "runs": [
      {"user": "How do I deploy...", "assistant": "You can deploy by..."},
      {"user": "What about Docker?", "assistant": "For Docker..."}
    ]
  }
]
```

**read_past_session返回消息**: 完整的会话消息JSON

**为什么这样实现**:
- **两步检索**: 先搜索会话列表（轻量），再读取具体会话（重量），避免一次性加载过多数据
- **预览截断**: `_truncate(text, limit=200)` 限制预览长度，每个run只显示前200字符
- **排除当前会话**: `current_session_id`过滤，避免搜索到自身

### 4.5 update_session_state 深度分析

**作用**: 让Agent主动更新session_state，实现有状态的对话。

**实现位置**: `agent/_default_tools.py` L347-L380

```python
def update_session_state_tool(agent, run_context, session_state_updates):
    if run_context.session_state is None:
        run_context.session_state = {}
    session_state = run_context.session_state
    for key, value in session_state_updates.items():
        session_state[key] = value
    return f"Updated session state: {session_state}"
```

**返回消息**: `"Updated session state: {完整session_state}"`

**为什么这样实现**:
- **直接合并**: `session_state_updates`中的key-value直接合并到现有session_state
- **返回完整状态**: 返回更新后的完整session_state，让LLM看到当前所有状态
- **与resolve_in_context联动**: 更新后的session_state会在后续消息中通过`{变量名}`被替换
- **条件触发**: `enable_agentic_state=True`时才注册此工具

---

## 五、LearningMachine工具消息机制

### 5.1 LearningMachine工具架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LearningMachine 工具体系                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LearningMachine.get_tools() → 聚合所有启用的store的工具                    │
│                                                                             │
│  ┌─ user_profile store ──────────────────────────────────────────────┐     │
│  │  工具: update_profile                                              │     │
│  │  返回: "Profile updated: {field_name} = {value}"                   │     │
│  │  作用: 更新结构化用户数据(姓名/偏好/设置等)                        │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                             │
│  ┌─ user_memory store ───────────────────────────────────────────────┐     │
│  │  工具: update_user_memory                                          │     │
│  │  返回: "Memory updated: ..." 或具体操作结果                        │     │
│  │  作用: 更新非结构化用户记忆(观察/偏好/事实等)                      │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                             │
│  ┌─ entity_memory store ─────────────────────────────────────────────┐     │
│  │  工具: search_entities, create_entity, update_entity,             │     │
│  │        add_fact, remove_fact, ...                                  │     │
│  │  返回: JSON格式的实体/事实数据                                     │     │
│  │  作用: 管理外部实体知识(项目/产品/组织等)                          │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                             │
│  ┌─ session_context store ───────────────────────────────────────────┐     │
│  │  工具: save_session_context                                       │     │
│  │  返回: 确认消息                                                    │     │
│  │  作用: 保存当前会话上下文供后续使用                                │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                             │
│  ┌─ learned_knowledge store ─────────────────────────────────────────┐     │
│  │  工具: search_learnings, save_learning                            │     │
│  │  返回: JSON格式的学习结果                                         │     │
│  │  作用: 保存和检索归纳性知识(见解/经验/最佳实践)                    │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 LearningMachine与MemoryManager的区别

```
┌──────────────────────┬───────────────────────────┬───────────────────────────┐
│  维度                │  MemoryManager            │  LearningMachine          │
├──────────────────────┼───────────────────────────┼───────────────────────────┤
│  定位                │  简单记忆管理              │  统一学习平台              │
│  存储类型            │  单一(user_memories)       │  多种(profile/memory/     │
│                      │                           │  entity/knowledge/context)│
│  工具数量            │  1个(update_user_memory)   │  多个(每个store独立工具)   │
│  注入系统提示词      │  是(add_memories_to_       │  是(add_learnings_to_     │
│                      │  context)                  │  context)                 │
│  后台处理            │  是(start_memory_future)   │  是(start_learning_future)│
│  初始化条件          │  enable_agentic_memory/    │  learning=True/           │
│                      │  update_memory_on_run      │  LearningMachine(...)     │
│  数据库要求          │  需要db                    │  需要db                   │
│  适用场景            │  简单用户偏好记忆          │  复杂知识管理和学习        │
└──────────────────────┴───────────────────────────┴───────────────────────────┘
```

---

## 六、ReasoningTools / KnowledgeTools / MemoryTools / WorkflowTools 深度分析

### 6.1 四大Toolkit统一模式

所有四个Toolkit都遵循 **Think → Act → Analyze** 三阶段循环模式：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    四大Toolkit统一模式                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─ ReasoningTools ──────────────────────────────────────────────────┐     │
│  │  think(topic)      → 思考：对主题进行推理                         │     │
│  │  analyze(topic)    → 分析：深入分析主题                           │     │
│  │  (无Act阶段)                                                     │     │
│  │  结果存储: session_state["reasoning_thoughts"]                    │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                             │
│  ┌─ KnowledgeTools ──────────────────────────────────────────────────┐     │
│  │  think(topic)      → 思考：确定搜索策略                           │     │
│  │  search_knowledge  → 行动：搜索知识库                             │     │
│  │   (query)           → 返回文档列表                                │     │
│  │  analyze(topic)    → 分析：综合分析搜索结果                       │     │
│  │  结果存储: session_state["knowledge_thoughts"]                    │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                             │
│  ┌─ MemoryTools ─────────────────────────────────────────────────────┐     │
│  │  think(topic)      → 思考：确定记忆操作策略                       │     │
│  │  add_memory/       → 行动：增删改查记忆                           │     │
│  │  search_memories/  → 返回操作结果                                 │     │
│  │  update_memory/                                                    │     │
│  │  delete_memory                                                     │     │
│  │  analyze(topic)    → 分析：分析记忆内容                           │     │
│  │  结果存储: session_state["memory_thoughts"]                       │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                             │
│  ┌─ WorkflowTools ───────────────────────────────────────────────────┐     │
│  │  think(topic)      → 思考：确定工作流执行策略                     │     │
│  │  run_workflow      → 行动：执行工作流                             │     │
│  │   (workflow_name,   → 返回工作流执行结果                          │     │
│  │    input)                                                          │     │
│  │  analyze(topic)    → 分析：分析工作流结果                         │     │
│  │  结果存储: session_state["workflow_thoughts"]                      │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Think工具的内部实现

**实现位置**: `tools/reasoning.py` L51-L115

```python
def think(self, topic: str) -> str:
    """对主题进行推理，存储思考结果到session_state"""
    # 1. 构建推理请求
    prompt = f"Think deeply about: {topic}"
    
    # 2. 调用LLM推理
    response = self.think_model.response(
        messages=[Message(role="user", content=prompt)]
    )
    
    # 3. 存储到session_state
    if "reasoning_thoughts" not in session_state:
        session_state["reasoning_thoughts"] = []
    session_state["reasoning_thoughts"].append({
        "topic": topic,
        "thought": response.content
    })
    
    # 4. 返回给LLM
    return response.content
```

**为什么这样实现**:
- **独立模型**: `think_model`可以是与主Agent不同的模型（更便宜/更快），降低推理成本
- **session_state存储**: 思考结果存储在session_state中，通过`add_session_state_to_context`在后续请求中自动注入到系统提示词
- **不暴露给用户**: think/analyze的结果只存在于session_state和工具返回消息中，不会直接展示给用户
- **可组合**: Think → Act → Analyze可以自由组合，LLM决定是否需要每个阶段

### 6.3 Toolkit.instructions注入机制

每个Toolkit通过`instructions`属性向系统提示词注入工具使用指南：

```python
class ReasoningTools(Toolkit):
    instructions: str = dedent("""\
    Use `think(topic)` to reason about a topic before acting.
    Use `analyze(topic)` to analyze results after acting.
    Always think before you act, and analyze after you act.
    """)
```

**注入路径**:
```
Toolkit.instructions → agent._tool_instructions → 系统提示词位置⑤ (tool_instructions)
```

---

## 七、后台任务与消息的异步关系

### 7.1 后台任务不直接修改消息

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    后台任务与消息的关系                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  主线程:                                                                    │
│  构建消息 → 推理 → 模型调用 → 处理响应 → 返回结果                          │
│                                                                             │
│  后台线程(并行执行):                                                        │
│  ├── start_memory_future()                                                 │
│  │   → 从当前消息中提取记忆，存入MemoryManager                             │
│  │   → 不修改消息列表，只更新DB中的记忆数据                                │
│  │   → 下次run时通过add_memories_to_context加载到系统提示词                │
│  │                                                                         │
│  ├── start_learning_future()                                               │
│  │   → 从当前消息中提取学习，存入LearningMachine                           │
│  │   → 不修改消息列表，只更新DB中的学习数据                                │
│  │   → 下次run时通过add_learnings_to_context加载到系统提示词               │
│  │                                                                         │
│  └── start_cultural_knowledge_future()                                     │
│      → 从当前消息中提取文化知识，存入CultureManager                        │
│      → 不修改消息列表，只更新DB中的文化知识                                │
│      → 下次run时通过add_culture_to_context加载到系统提示词                 │
│                                                                             │
│  关键设计: 后台任务的结果在"当前run"中不可见，在"下次run"中生效            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**为什么这样设计**:
- **性能**: 记忆/学习/文化知识的提取是耗时操作，不阻塞主流程
- **一致性**: 消息列表在模型调用前已确定，运行中不会被后台任务修改
- **延迟生效**: 记忆和学习是"经验积累"，在下次交互中体现更合理

---

## 八、完整消息列表在运行中的演化时间线

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    消息列表演化时间线                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  T0: 初始构建 (get_run_messages)                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ [system] [additional_input...] [history...] [user+knowledge+deps]  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  T1: 推理注入 (handle_reasoning, 仅reasoning=True时)                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ [system] [additional_input...] [history...] [user+knowledge+deps]  │    │
│  │ [assistant: "I have worked through..."]                             │    │
│  │ [reasoning_messages... (add_to_agent_memory=False)]                 │    │
│  │ [assistant: "Now I will summarize..."]                              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  T2: 第1轮模型调用 → 返回assistant+tool_calls                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ ... (同上)                                                         │    │
│  │ [assistant: content + tool_calls: [{name, args}]]                   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  T3: 工具执行 → 返回tool结果                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ ... (同上)                                                         │    │
│  │ [assistant: content + tool_calls]                                   │    │
│  │ [tool: content="搜索结果...", tool_call_id=xxx]                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  T4: 压缩检查 (CompressionManager)                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ 如果超阈值 → tool消息的compressed_content被设置                     │    │
│  │ 后续发送给LLM时使用compressed_content替代原始content                │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  T5: 第2轮模型调用 → 可能继续调用工具或返回最终回答                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ ... (同上)                                                         │    │
│  │ [assistant: content + tool_calls]                                   │    │
│  │ [tool: content/compressed_content]                                  │    │
│  │ [assistant: content + tool_calls: [{name, args}]]  ← 第2轮         │    │
│  │ [tool: content="结果..."]                          ← 第2轮结果     │    │
│  │ ... (循环直到LLM不再调用工具)                                       │    │
│  │ [assistant: "最终回答"]                             ← 最终回答      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  T6: 后处理 (output_model / parser_model / followups)                      │
│  → 不修改消息列表，只处理最终输出                                          │
│                                                                             │
│  T7: 后台任务完成                                                           │
│  → 记忆/学习/文化知识写入DB                                                │
│  → session_state持久化                                                     │
│  → session_summary生成                                                     │
│  → 下次run时这些数据将通过系统提示词/用户消息注入                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 九、关键源码索引

| 机制 | 核心文件 | 关键行号 |
|------|---------|---------|
| 运行主循环 | `agent/_run.py` | L334-L640 |
| 推理调度 | `agent/_response.py` | L74-L165 |
| 推理消息注入 | `reasoning/helpers.py` | L42-L61 |
| CoT推理Agent | `reasoning/default.py` | L13-L94 |
| Native推理管理 | `reasoning/manager.py` | L106-L900 |
| Message类 | `models/message.py` | L55-L121 |
| update_user_memory | `agent/_default_tools.py` | L38-L75 |
| search_knowledge_base | `agent/_default_tools.py` | L103-L282 |
| get_chat_history | `agent/_default_tools.py` | L285-L319 |
| get_tool_call_history | `agent/_default_tools.py` | L322-L344 |
| update_session_state | `agent/_default_tools.py` | L347-L380 |
| add_to_knowledge | `agent/_default_tools.py` | L383-L408 |
| search_past_sessions | `agent/_default_tools.py` | L459-L502 |
| read_past_session | `agent/_default_tools.py` | L560-L609 |
| update_cultural_knowledge | `agent/_default_tools.py` | L78-L100 |
| LearningMachine.get_tools | `learn/machine.py` | L437-L482 |
| ReasoningTools | `tools/reasoning.py` | L51-L115 |
| KnowledgeTools | `tools/knowledge.py` | L95-L115 |
| MemoryTools | `tools/memory.py` | L126-L183 |
| WorkflowTools | `tools/workflow.py` | - |
| CompressionManager | `compression/manager.py` | L52-L280 |
| 后台任务启动 | `agent/_run.py` | L489-L513 |
