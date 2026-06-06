# Agno 框架系统提示词构建机制深度分析

> 基于 Agno 框架源码（`/workspace/libs/agno/agno/`）的完整逆向分析
> 核心文件: `agent/_messages.py`, `agent/agent.py`, `tools/toolkit.py`, `reasoning/manager.py`, `memory/manager.py`

---

## 一、系统提示词(System Prompt)构成全景图

### 1.1 构建流程示意图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Agno Agent 系统提示词构建流程                              │
│                   (get_system_message / aget_system_message)                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─ 前置判断 ──────────────────────────────────────────────────────────┐    │
│  │ 1. system_message 是否提供? → 是: 直接返回自定义system_message      │    │
│  │ 2. build_context == False? → 是: 返回 None (不构建上下文)          │    │
│  │ 3. 否: 进入默认构建流程 ↓                                          │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─ 默认构建流程 (按顺序拼接) ─────────────────────────────────────────┐    │
│  │                                                                     │    │
│  │  ① description          [Agent描述]                                │    │
│  │     └─ 条件: agent.description is not None                         │    │
│  │     └─ 大小: 用户自定义，无限制                                     │    │
│  │                                                                     │    │
│  │  ② role                 [Agent角色]                                │    │
│  │     └─ 条件: agent.role is not None                                │    │
│  │     └─ 格式: <your_role>\n{role}\n</your_role>                    │    │
│  │     └─ 大小: 用户自定义，无限制                                     │    │
│  │                                                                     │    │
│  │  ③ instructions         [Agent指令]                                │    │
│  │     ├─ 条件: agent.instructions is not None                        │    │
│  │     ├─ 来源: str | List[str] | Callable → 动态生成                 │    │
│  │     ├─ 追加: model.get_instructions_for_model(tools)               │    │
│  │     └─ 格式: <instructions>...</instructions> (use_instruction_tags)│    │
│  │                                                                     │    │
│  │  ④ additional_information [附加信息块]                              │    │
│  │     ├─ 4.1 markdown格式指令                                        │    │
│  │     │   └─ 条件: agent.markdown=True AND output_schema is None     │    │
│  │     │   └─ 内容: "Use markdown to format your answers."            │    │
│  │     │                                                              │    │
│  │     ├─ 4.2 当前时间 (add_datetime_to_context)                      │    │
│  │     │   └─ 条件: agent.add_datetime_to_context=True                │    │
│  │     │   └─ 内容: "The current time is {formatted_time}."           │    │
│  │     │   └─ 大小: ~30-50 tokens                                     │    │
│  │     │                                                              │    │
│  │     ├─ 4.3 当前位置 (add_location_to_context)                      │    │
│  │     │   └─ 条件: agent.add_location_to_context=True                │    │
│  │     │   └─ 内容: "Your approximate location is: {city, region...}" │    │
│  │     │   └─ 大小: ~20-40 tokens                                     │    │
│  │     │                                                              │    │
│  │     └─ 4.4 Agent名称 (add_name_to_context)                        │    │
│  │         └─ 条件: agent.name is not None AND add_name_to_context    │    │
│  │         └─ 内容: "Your name is: {agent.name}."                     │    │
│  │         └─ 大小: ~10-20 tokens                                     │    │
│  │                                                                     │    │
│  │  ⑤ tool_instructions     [工具指令]                                │    │
│  │     └─ 条件: agent._tool_instructions is not None                  │    │
│  │     └─ 来源: Toolkit.instructions (add_instructions=True时注入)    │    │
│  │     └─ 大小: 取决于工具数量，每个Toolkit约200-800 tokens           │    │
│  │                                                                     │    │
│  │  ── resolve_in_context 变量替换 ──                                 │    │
│  │     └─ 条件: agent.resolve_in_context=True                         │    │
│  │     └─ 作用: 将 {session_state} / {dependencies} 变量替换到文本中  │    │
│  │                                                                     │    │
│  │  ⑥ expected_output      [期望输出]                                │    │
│  │     └─ 条件: agent.expected_output is not None                     │    │
│  │     └─ 格式: <expected_output>\n{...}\n</expected_output>          │    │
│  │     └─ 大小: 用户自定义                                            │    │
│  │                                                                     │    │
│  │  ⑦ additional_context   [额外上下文]                               │    │
│  │     └─ 条件: agent.additional_context is not None                  │    │
│  │     └─ 大小: 用户自定义，无限制                                     │    │
│  │                                                                     │    │
│  │  ⑧ skills                [技能提示]                                │    │
│  │     └─ 条件: agent.skills is not None                              │    │
│  │     └─ 来源: agent.skills.get_system_prompt_snippet()              │    │
│  │                                                                     │    │
│  │  ⑨ memories              [用户记忆]                                │    │
│  │     ├─ 条件: agent.add_memories_to_context=True                    │    │
│  │     ├─ 有记忆时: <memories_from_previous_interactions>...</>       │    │
│  │     ├─ 无记忆时: "You have the capability to retain memories..."   │    │
│  │     └─ 大小: 取决于记忆数量，每条约20-50 tokens                    │    │
│  │     │                                                              │    │
│  │     └─ ⑨.1 agentic_memory指令 (enable_agentic_memory)             │    │
│  │         └─ 条件: agent.enable_agentic_memory=True                  │    │
│  │         └─ 内容: <updating_user_memories> 使用说明 </>             │    │
│  │         └─ 大小: ~150 tokens                                       │    │
│  │                                                                     │    │
│  │  ⑩ cultural_knowledge    [文化知识]                                │    │
│  │     ├─ 条件: agent.add_culture_to_context=True                     │    │
│  │     ├─ 有知识时: <cultural_knowledge>...</>                        │    │
│  │     ├─ 无知识时: "You have the capability to access shared..."      │    │
│  │     └─ 大小: 取决于知识条目数，每条约100-300 tokens                │    │
│  │     │                                                              │    │
│  │     └─ ⑩.1 agentic_culture指令 (enable_agentic_culture)           │    │
│  │         └─ 条件: agent.enable_agentic_culture=True                 │    │
│  │         └─ 内容: <contributing_to_culture> 贡献说明 </>            │    │
│  │         └─ 大小: ~150 tokens                                       │    │
│  │                                                                     │    │
│  │  ⑪ session_summary       [会话摘要]                                │    │
│  │     └─ 条件: add_session_summary_to_context AND session.summary    │    │
│  │     └─ 格式: <summary_of_previous_interactions>...</>              │    │
│  │     └─ 大小: 取决于摘要长度，约100-500 tokens                      │    │
│  │                                                                     │    │
│  │  ⑫ learnings             [学习内容]                                │    │
│  │     └─ 条件: agent._learning is not None AND add_learnings_to_ctx  │    │
│  │     └─ 来源: agent._learning.build_context()                      │    │
│  │                                                                     │    │
│  │  ⑬ knowledge_instructions [知识库搜索指令]                          │    │
│  │     └─ 条件: knowledge存在 AND search_knowledge AND add_search_... │    │
│  │     └─ 来源: knowledge.build_context()                             │    │
│  │                                                                     │    │
│  │  ⑭ model_system_message  [模型特定系统消息]                        │    │
│  │     └─ 来源: agent.model.get_system_message_for_model(tools)       │    │
│  │                                                                     │    │
│  │  ⑮ JSON输出提示         [结构化输出指令]                            │    │
│  │     └─ 条件: output_schema存在 AND 模型不支持原生结构化输出         │    │
│  │     └─ 来源: get_json_output_prompt(output_schema)                 │    │
│  │                                                                     │    │
│  │  ⑯ response_model_format [响应模型格式提示]                        │    │
│  │     └─ 条件: output_schema存在 AND parser_model存在                │    │
│  │     └─ 来源: get_response_model_format_prompt(output_schema)       │    │
│  │                                                                     │    │
│  │  ⑰ session_state         [会话状态]                                │    │
│  │     └─ 条件: add_session_state_to_context AND session_state存在    │    │
│  │     └─ 格式: <session_state>\n{session_state}\n</session_state>    │    │
│  │     └─ 大小: 取决于状态变量数量和值                                 │    │
│  │                                                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  最终: Message(role=agent.system_message_role, content=拼接结果.strip())    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 完整消息列表构建顺序 (get_run_messages)

```
┌──────────────────────────────────────────────────────────────┐
│              发送给LLM的完整消息列表顺序                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [1] System Message     ← 系统提示词(上述全部拼接)           │
│                                                              │
│  [2] Additional Input   ← agent.additional_input 额外消息    │
│       (用于few-shot学习或额外上下文)                          │
│                                                              │
│  [3] History Messages   ← add_history_to_context=True时      │
│       ├─ num_history_runs: 控制历史轮数                      │
│       ├─ num_history_messages: 控制历史消息数                 │
│       ├─ max_tool_calls_from_history: 限制历史工具调用数     │
│       └─ 跳过非标准角色的历史消息                             │
│                                                              │
│  [4] User Message       ← 用户消息(含knowledge/dependencies) │
│       ├─ 用户输入内容                                        │
│       ├─ knowledge references (add_knowledge_to_context)     │
│       │   └─ <references>...</references>                    │
│       └─ dependencies (add_dependencies_to_context)          │
│           └─ <additional context>...</additional context>    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 二、各上下文机制深度分析

### 2.1 add_datetime_to_context

**作用**: 为Agent提供时间感知能力，使其能理解"今天"、"明天"、"上周"等相对时间表达。

**实现方式**:
```python
# _messages.py L187-L207
if agent.add_datetime_to_context:
    from datetime import datetime
    tz = None
    if agent.timezone_identifier:
        from zoneinfo import ZoneInfo
        tz = ZoneInfo(agent.timezone_identifier)
    time = datetime.now(tz) if tz else datetime.now()
    if agent.datetime_format:
        formatted_time = time.strftime(agent.datetime_format)
    else:
        formatted_time = str(time)
    additional_information.append(f"The current time is {formatted_time}.")
```

**存在条件**: `add_datetime_to_context=True`
**大小**: ~30-50 tokens
**为什么这样实现**:
- LLM本身没有时间感知，必须显式注入当前时间
- 支持自定义时区(`timezone_identifier`)和格式(`datetime_format`)，满足全球化场景
- 放在`additional_information`块中，与核心instructions分离，语义清晰

---

### 2.2 add_location_to_context

**作用**: 为Agent提供位置感知能力，使其能理解"附近的餐厅"、"本地天气"等位置相关请求。

**实现方式**:
```python
# _messages.py L209-L226
if agent.add_location_to_context:
    from agno.utils.location import get_location
    location = get_location()
    if location:
        location_str = ", ".join(filter(None, [
            location.get("city"),
            location.get("region"),
            location.get("country"),
        ]))
        if location_str:
            additional_information.append(f"Your approximate location is: {location_str}.")
```

**存在条件**: `add_location_to_context=True` 且 `get_location()` 返回有效位置
**大小**: ~20-40 tokens
**为什么这样实现**:
- 位置信息通过IP推断，是"近似"的，措辞用"approximate"
- 只取city/region/country三级，避免信息过载
- 与datetime类似，放在additional_information块中保持一致性

---

### 2.3 add_name_to_context

**作用**: 让Agent知道自己的名字，实现自我认知和个性化。

**实现方式**:
```python
# _messages.py L228-L230
if agent.name is not None and agent.add_name_to_context:
    additional_information.append(f"Your name is: {agent.name}.")
```

**存在条件**: `agent.name` 不为空 且 `add_name_to_context=True`
**大小**: ~10-20 tokens
**为什么这样实现**:
- 简洁直接，名字是身份标识，不需要复杂格式
- 需要同时设置name和add_name_to_context，避免意外暴露

---

### 2.4 add_session_summary_to_context

**作用**: 在新会话中提供之前交互的摘要，实现跨会话的上下文连续性。

**实现方式**:
```python
# _messages.py L388-L397
if agent.add_session_summary_to_context and session.summary is not None:
    system_message_content += "Here is a brief summary of your previous interactions:\n\n"
    system_message_content += "<summary_of_previous_interactions>\n"
    system_message_content += session.summary.summary
    system_message_content += "\n</summary_of_previous_interactions>\n\n"
    system_message_content += (
        "Note: this information is from previous interactions and may be outdated. "
        "You should ALWAYS prefer information from this conversation over the past summary.\n\n"
    )
```

**存在条件**: `add_session_summary_to_context=True` 且 `session.summary` 不为空
**大小**: 取决于摘要长度，约100-500 tokens
**为什么这样实现**:
- 使用XML标签包裹，便于LLM识别边界
- 关键设计: 明确提示"优先使用当前对话信息"，防止摘要中的过时信息误导
- 摘要由`SessionSummaryManager`在run结束时自动生成
- 这是一种**压缩策略**: 将完整历史压缩为摘要，节省token

---

### 2.5 additional_context

**作用**: 允许开发者注入任意额外上下文到系统提示词末尾。

**实现方式**:
```python
# _messages.py L278-L280
if agent.additional_context is not None:
    system_message_content += f"{agent.additional_context}\n"
```

**存在条件**: `additional_context` 不为空
**大小**: 无限制，用户自定义
**为什么这样实现**:
- 提供最大的灵活性，作为"逃生舱口"
- 放在expected_output之后、memories之前，位置精心设计
- 不做任何格式化处理，原样插入

---

### 2.6 add_memories_to_context

**作用**: 将用户的持久化记忆注入系统提示词，实现跨会话的个性化。

**实现方式**:
```python
# _messages.py L287-L313
if agent.add_memories_to_context:
    if not user_id:
        user_id = "default"
    if agent.memory_manager is None:
        set_memory_manager(agent)  # 懒初始化

    user_memories = agent.memory_manager.get_user_memories(user_id=user_id)

    if user_memories and len(user_memories) > 0:
        system_message_content += "You have access to user info and preferences..."
        system_message_content += "<memories_from_previous_interactions>"
        for _memory in user_memories:
            system_message_content += f"\n- {_memory.memory}"
        system_message_content += "\n</memories_from_previous_interactions>\n\n"
        system_message_content += (
            "Note: this information is from previous interactions and may be updated. "
            "You should always prefer information from this conversation over the past memories.\n"
        )
    else:
        system_message_content += (
            "You have the capability to retain memories from previous interactions..."
        )
```

**存在条件**: `add_memories_to_context=True`
**大小**: 取决于记忆数量，每条约20-50 tokens
**为什么这样实现**:
- **懒初始化**: memory_manager在首次需要时才创建，避免不必要的资源消耗
- **双重提示**: 有记忆时展示内容+提醒优先级；无记忆时告知Agent具备记忆能力
- **优先级声明**: 与session_summary一样，明确告知"当前对话优先于历史记忆"
- 记忆存储在数据库中，通过`MemoryManager`管理，支持增删改查

---

### 2.7 enable_agentic_memory

**作用**: 赋予Agent主动管理用户记忆的能力（通过工具调用）。

**实现方式**:
```python
# _messages.py L315-L325
if agent.enable_agentic_memory:
    system_message_content += (
        "\n<updating_user_memories>\n"
        "- You have access to the `update_user_memory` tool...\n"
        "- If the user's message includes information that should be captured as a memory...\n"
        "- Memories should include details that could personalize ongoing interactions...\n"
        "- Use this tool to add new memories or update existing memories...\n"
        "- If you use the `update_user_memory` tool, remember to pass on the response to the user.\n"
        "</updating_user_memories>\n\n"
    )
```

**存在条件**: `enable_agentic_memory=True`（同时需要`add_memories_to_context=True`）
**大小**: ~150 tokens
**为什么这样实现**:
- **被动→主动的转变**: `add_memories_to_context`只是被动读取记忆，`enable_agentic_memory`让Agent能主动写入
- 通过注入工具使用指南，指导Agent何时、如何使用`update_user_memory`工具
- 工具本身由`MemoryTools` Toolkit提供，系统提示词中的指令指导Agent正确使用
- **设计哲学**: 记忆更新应该是Agent的自主行为，而非硬编码规则

**与add_memories_to_context的关系**:
```
add_memories_to_context = True  → 只读: 注入已有记忆到上下文
enable_agentic_memory = True    → 读写: 注入记忆管理工具使用指南
两者通常同时开启，形成完整的记忆闭环
```

---

### 2.8 add_session_state_to_context

**作用**: 将当前会话的状态变量注入系统提示词，让Agent了解和利用会话状态。

**实现方式**:
```python
# _messages.py L441-L443
if add_session_state_to_context and session_state is not None:
    system_message_content += f"\n<session_state>\n{session_state}\n</session_state>\n\n"
```

**存在条件**: `add_session_state_to_context=True` 且 `session_state` 不为空
**大小**: 取决于状态变量数量和值
**为什么这样实现**:
- 放在系统提示词最后，确保Agent能看到最新的状态
- session_state是持久化的字典，跨run保持
- 可通过`enable_agentic_state`让Agent通过工具动态更新状态
- 与`resolve_in_context`配合: resolve_in_context在构建时替换变量值，add_session_state_to_context将完整状态展示给Agent

---

### 2.9 add_knowledge_to_context

**作用**: 自动从知识库检索相关文档，附加到用户消息中（RAG机制）。

**实现方式**:
```python
# _messages.py L915-L964 (用户消息构建)
if agent.add_knowledge_to_context:
    docs_from_knowledge = get_relevant_docs_from_knowledge(
        agent, query=user_msg_content, filters=knowledge_filters, ...
    )
    if docs_from_knowledge is not None:
        # 添加到用户消息
        user_msg_content_str += "\n\nUse the following references from the knowledge base if it helps:\n"
        user_msg_content_str += "<references>\n"
        user_msg_content_str += convert_documents_to_string(agent, references.references) + "\n"
        user_msg_content_str += "</references>"
```

**存在条件**: `add_knowledge_to_context=True`
**大小**: 取决于检索到的文档数量和长度
**关键区别**: 知识引用附加到**用户消息**而非系统消息
**为什么这样实现**:
- **检索时机**: 每次用户请求时实时检索，确保信息最新
- **放在用户消息中**: 因为知识引用与具体查询相关，放在用户消息中语义更准确
- 与`search_knowledge`（Agentic RAG）互补:
  - `add_knowledge_to_context`: 自动检索，被动注入
  - `search_knowledge`: Agent主动调用工具检索

---

### 2.10 add_dependencies_to_context

**作用**: 将外部依赖数据注入用户消息，为Agent提供额外的上下文信息。

**实现方式**:
```python
# _messages.py L966-L969
if add_dependencies_to_context and dependencies is not None:
    user_msg_content_str += "\n\n<additional context>\n"
    user_msg_content_str += convert_dependencies_to_string(agent, dependencies) + "\n"
    user_msg_content_str += "</additional context>"
```

**存在条件**: `add_dependencies_to_context=True` 且 `dependencies` 不为空
**大小**: 取决于依赖数据量
**为什么这样实现**:
- 依赖数据放在用户消息中，因为它是请求相关的
- dependencies通过`run_context`传入，可以是任意字典
- 与`resolve_in_context`配合: dependencies中的值可以用于模板替换

---

### 2.11 add_history_to_context

**作用**: 将历史对话消息添加到消息列表中，实现多轮对话的上下文连续性。

**实现方式**:
```python
# _messages.py L1241-L1272
if add_history_to_context:
    skip_role = (
        agent.system_message_role if agent.system_message_role not in ["user", "assistant", "tool"] else None
    )
    history = session.get_messages(
        last_n_runs=agent.num_history_runs,
        limit=agent.num_history_messages,
        skip_roles=[skip_role] if skip_role else None,
        agent_id=agent.id if agent.team_id is not None else None,
    )
    if len(history) > 0:
        history_copy = [deepcopy(msg) for msg in history]
        for _msg in history_copy:
            _msg.from_history = True
        if agent.max_tool_calls_from_history is not None:
            filter_tool_calls(history_copy, agent.max_tool_calls_from_history)
        run_messages.messages += history_copy
```

**存在条件**: `add_history_to_context=True`
**大小**: 取决于历史消息数量，受`num_history_runs`/`num_history_messages`/`max_tool_calls_from_history`控制
**为什么这样实现**:
- **深拷贝**: 避免修改原始历史消息
- **from_history标记**: 区分历史消息和当前消息
- **工具调用过滤**: `max_tool_calls_from_history`控制历史中工具调用的数量，防止token爆炸
- **角色过滤**: 跳过历史中的system消息（避免重复），但保留user/assistant/tool消息
- 历史消息放在system message和user message之间，保持对话时序

---

### 2.12 reasoning=True

**作用**: 启用推理能力，让Agent在回答前先进行思考（Chain-of-Thought）。

**实现方式**:
推理机制有两种模式:

**模式1: 原生推理模型** (DeepSeek, Anthropic, OpenAI o1/o3, Gemini等)
```
Agent → ReasoningManager → 检测模型类型 → 调用模型原生推理API
                                                    ↓
                                            返回reasoning_message
```

**模式2: 默认CoT推理** (其他模型)
```
Agent → ReasoningManager → 创建reasoning_agent → 循环执行推理步骤
                                                    ↓
                                    Think → [Tool Calls] → Analyze → ... → Final Answer
```

**ReasoningManager核心逻辑** (`reasoning/manager.py`):
```python
class ReasoningManager:
    def reason(self, run_messages, stream=False):
        reasoning_model = self.config.reasoning_model
        if self.is_native_reasoning_model(reasoning_model):
            # 使用模型原生推理能力
            if stream:
                yield from self._stream_native_reasoning_events(...)
            else:
                yield from self._get_native_reasoning_events(...)
        else:
            # 使用默认CoT推理
            yield from self._run_default_reasoning_events(...)
```

**存在条件**: `reasoning=True` 且 `reasoning_model` 已设置
**大小**: 推理步骤本身不直接增加系统提示词，但增加推理消息
**为什么这样实现**:
- **统一接口**: 无论底层模型是否原生支持推理，上层API保持一致
- **模型适配**: 自动检测模型类型，选择最优推理路径
- **可配置**: `reasoning_min_steps`/`reasoning_max_steps`控制推理深度
- 推理结果作为消息插入到对话流中，供最终回答参考

---

### 2.13 Agent(instructions=get_instructions)

**作用**: 定义Agent的核心行为指令，是系统提示词中最重要的部分。

**实现方式**:
```python
# _messages.py L163-L179
instructions: List[str] = []
if agent.instructions is not None:
    _instructions = agent.instructions
    if callable(agent.instructions):
        _instructions = execute_instructions(
            agent=agent, instructions=agent.instructions,
            session_state=session_state, run_context=run_context
        )
    if isinstance(_instructions, str):
        instructions.append(_instructions)
    elif isinstance(_instructions, list):
        instructions.extend(_instructions)

# 追加模型特定的指令
_model_instructions = agent.model.get_instructions_for_model(tools)
if _model_instructions is not None:
    instructions.extend(_model_instructions)
```

**存在条件**: `instructions` 不为空
**大小**: 用户自定义
**为什么这样实现**:
- **支持三种形式**: 字符串、字符串列表、可调用函数
- **动态指令**: Callable形式允许根据session_state和run_context动态生成指令
- **模型指令合并**: 将模型特定的指令（如工具使用说明）追加到用户指令后
- **标签包裹**: `use_instruction_tags=True`时用`<instructions>`标签包裹，提高LLM识别度

---

## 三、Toolkit机制深度分析

### 3.1 Toolkit基类

**核心设计**: Toolkit是工具的容器，提供统一的注册、管理和指令注入机制。

```python
class Toolkit:
    def __init__(self, name, tools, instructions, add_instructions, ...):
        self.functions: Dict[str, Function] = OrderedDict()       # 同步工具
        self.async_functions: Dict[str, Function] = OrderedDict() # 异步工具
        self.instructions: Optional[str] = instructions           # 工具使用指令
        self.add_instructions: bool = add_instructions            # 是否注入指令
```

**指令注入机制**:
- 当`add_instructions=True`时，Toolkit的`instructions`会被添加到`agent._tool_instructions`
- 这些指令最终出现在系统提示词的位置⑤（tool_instructions）
- 每个Toolkit可以定义自己的DEFAULT_INSTRUCTIONS和FEW_SHOT_EXAMPLES

### 3.2 四大核心Toolkit对比

```
┌────────────────┬──────────────────────────────────────────────────────────────┐
│    Toolkit     │                      核心工具                                │
├────────────────┼──────────────────────────────────────────────────────────────┤
│ ReasoningTools │ think()     → 草稿板，分解问题、规划步骤                      │
│                │ analyze()   → 评估结果、决定下一步行动                         │
│                │                                                          │
│ 系统提示词指令  │ "You must ALWAYS `think` before making tool calls..."     │
│                │ 典型流程: Think → [Tool Calls] → Analyze → Final Answer    │
├────────────────┼──────────────────────────────────────────────────────────────┤
│ KnowledgeTools │ think()           → 规划搜索策略                             │
│                │ search_knowledge() → 执行知识库搜索                           │
│                │ analyze()         → 评估搜索结果质量                          │
│                │                                                          │
│ 系统提示词指令  │ "Use these tools as frequently as needed..."              │
│                │ 典型流程: Think → Search → Analyze → (循环) → Final Answer  │
├────────────────┼──────────────────────────────────────────────────────────────┤
│ MemoryTools    │ think()         → 规划记忆操作                               │
│                │ get_memories()  → 获取用户记忆                               │
│                │ add_memory()    → 添加新记忆                                 │
│                │ update_memory() → 更新已有记忆                               │
│                │ delete_memory() → 删除记忆                                   │
│                │ analyze()       → 评估操作结果                               │
│                │                                                          │
│ 系统提示词指令  │ "Use these tools as frequently as needed..."              │
│                │ 典型流程: Think → Memory Op → Analyze → Final Answer        │
├────────────────┼─────────────────────────────────────────────────────────────┤
│ WorkflowTools  │ think()        → 规划工作流执行                              │
│                │ run_workflow() → 执行工作流                                  │
│                │ analyze()      → 评估执行结果                                │
│                │                                                          │
│ 系统提示词指令  │ "Use these tools as frequently as needed..."              │
│                │ 典型流程: Think → Run Workflow → Analyze → Final Answer     │
└────────────────┴──────────────────────────────────────────────────────────────┘
```

### 3.3 共同的Think-Act-Analyze模式

所有四个Toolkit都遵循相同的**Think-Act-Analyze**三阶段模式:

```
┌─────────┐     ┌─────────┐     ┌──────────┐
│  Think  │ ──→ │   Act   │ ──→ │ Analyze  │
│ (规划)   │     │ (执行)   │     │ (评估)    │
└─────────┘     └─────────┘     └──────────┘
      ↑                               │
      └───────── 循环直到满意 ─────────┘
```

**为什么这样设计**:
1. **Think（思考）**: 作为"草稿板"(scratchpad)，让LLM在行动前先规划，减少盲目调用
2. **Act（行动）**: 执行具体的操作（搜索/记忆/工作流），是实际产生效果的阶段
3. **Analyze（分析）**: 评估结果质量，决定是否需要继续迭代
4. **内部性**: Think和Analyze的内容不暴露给用户，只作为内部推理过程
5. **迭代性**: 支持多轮循环，直到得到满意的最终答案

**实现细节**:
- 所有思考和分析结果存储在`run_context.session_state`中
- 每次调用think/analyze时，返回所有历史步骤的汇总，保持推理链的连贯性
- ReasoningTools额外支持`confidence`和`next_action`字段，更精细地控制推理流程

---

## 四、MemoryManager机制深度分析

### 4.1 双模式记忆管理

```
模式1: 自动记忆 (update_memory_on_run=True)
┌──────────┐    ┌──────────────┐    ┌──────────┐
│ 对话结束  │ ──→│ MemoryManager│ ──→│ 自动提取  │
│          │    │ .create_or_  │    │ 记忆并存储│
│          │    │ update_      │    │          │
│          │    │ memories()   │    │          │
└──────────┘    └──────────────┘    └──────────┘

模式2: 代理记忆 (enable_agentic_memory=True)
┌──────────┐    ┌──────────────┐    ┌──────────┐
│ 用户消息  │ ──→│ Agent自主    │ ──→│ 调用      │
│ 包含可记忆│    │ 判断是否需要  │    │ update_   │
│ 信息      │    │ 更新记忆      │    │ user_     │
│          │    │              │    │ memory()  │
└──────────┘    └──────────────┘    └──────────┘
```

### 4.2 MemoryManager内部工作流

```python
# memory/manager.py - create_or_update_memories()
1. 从数据库读取已有记忆
2. 构建系统提示词:
   - 角色定义: "You are a Memory Manager..."
   - 记忆捕获标准: <memories_to_capture>...</>
   - 已有记忆列表: <existing_memories>...</>
   - 可用工具: add_memory, update_memory, delete_memory, clear_memory
3. 将对话消息 + 系统提示词发送给LLM
4. LLM决定是否需要添加/更新/删除记忆，并调用相应工具
5. 工具直接操作数据库
```

**为什么这样实现**:
- **LLM作为决策者**: 由LLM判断哪些信息值得记忆，而非硬编码规则
- **增量更新**: 不是每次重建，而是基于已有记忆进行增删改
- **工具驱动**: 通过function calling实现，LLM自主决定操作
- **可定制**: `memory_capture_instructions`允许自定义记忆捕获标准

---

## 五、设计哲学总结

### 5.1 分层注入策略

```
┌─────────────────────────────────────────────────────────┐
│                    系统提示词分层                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  身份层:  description + role + name                     │
│  ↓        "你是谁"                                      │
│                                                         │
│  行为层:  instructions + tool_instructions              │
│  ↓        "你该怎么做"                                   │
│                                                         │
│  环境层:  datetime + location + additional_information  │
│  ↓        "你在什么环境下"                               │
│                                                         │
│  期望层:  expected_output + additional_context          │
│  ↓        "期望什么结果"                                 │
│                                                         │
│  记忆层:  memories + cultural_knowledge + session_summary│
│  ↓        "你记得什么"                                   │
│                                                         │
│  知识层:  knowledge_instructions + model_system_message │
│  ↓        "你知道什么"                                   │
│                                                         │
│  输出层:  JSON prompt + response_model_format           │
│  ↓        "如何输出"                                     │
│                                                         │
│  状态层:  session_state                                 │
│           "当前状态是什么"                                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 5.2 核心设计原则

1. **渐进式复杂度**: 每个功能都是可选的，默认关闭，按需开启
2. **关注点分离**: 不同类型的信息放在不同位置（系统消息vs用户消息）
3. **懒初始化**: MemoryManager、CultureManager等在首次需要时才创建
4. **优先级声明**: 历史信息（记忆/摘要）总是附带"当前对话优先"的提示
5. **XML标签包裹**: 使用`<tag>...</tag>`格式，帮助LLM识别信息边界
6. **Think-Act-Analyze**: 统一的推理模式，贯穿所有Toolkit
7. **双模式设计**: 被动注入（add_xxx_to_context）+ 主动管理（enable_agentic_xxx）

### 5.3 Token优化策略

| 策略 | 机制 | 效果 |
|------|------|------|
| 历史压缩 | session_summary替代完整历史 | 减少50-90%历史token |
| 工具调用过滤 | max_tool_calls_from_history | 避免工具结果膨胀 |
| 懒加载 | memory_manager按需创建 | 无记忆时不消耗token |
| 摘要优先 | 明确声明"当前对话优先" | 减少冲突导致的重复 |
| 知识按需检索 | add_knowledge_to_context只检索相关文档 | 避免全量知识注入 |

---

## 六、关键源码索引

| 机制 | 核心文件 | 关键行号 |
|------|---------|---------|
| 系统提示词构建 | `agent/_messages.py` | L106-L450 |
| 用户消息构建 | `agent/_messages.py` | L821-L983 |
| 消息列表组装 | `agent/_messages.py` | L1156-L1358 |
| Agent参数定义 | `agent/agent.py` | L69-L350 |
| ReasoningTools | `tools/reasoning.py` | L10-L290 |
| KnowledgeTools | `tools/knowledge.py` | L12-L223 |
| MemoryTools | `tools/memory.py` | L13-L420 |
| WorkflowTools | `tools/workflow.py` | L18-L286 |
| Toolkit基类 | `tools/toolkit.py` | L12-L367 |
| ReasoningManager | `reasoning/manager.py` | L106-L1258 |
| MemoryManager | `memory/manager.py` | L46-L1580 |
| 变量替换 | `agent/_messages.py` | L56-L98 |
