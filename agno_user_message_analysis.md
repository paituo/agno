# Agno 框架用户消息层机制深度分析

> 基于 Agno 框架源码（`/workspace/libs/agno/agno/`）的完整逆向分析
> 核心文件: `agent/_messages.py`, `agent/_tools.py`, `agent/_storage.py`, `agent/_utils.py`, `context/provider.py`, `compression/manager.py`

---

## 一、完整消息列表全景图

在系统提示词之外，Agno 框架通过**用户消息层**和**消息列表层**注入大量上下文信息。以下是发送给LLM的完整消息列表结构：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              发送给LLM的完整消息列表 (get_run_messages)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [1] System Message        ← 系统提示词 (前文已分析)                        │
│                                                                             │
│  ──────────── 以下为用户消息层注入 ────────────                             │
│                                                                             │
│  [2] Additional Input      ← 额外消息 (few-shot/上下文)                     │
│       ├─ 条件: agent.additional_input is not None                           │
│       ├─ 位置: 紧跟系统消息，在历史消息之前                                   │
│       ├─ 类型: List[Union[str, Dict, BaseModel, Message]]                   │
│       ├─ 大小: 用户自定义，无限制                                            │
│       └─ 特点: 不保留在会话记忆中，每次run重新注入                           │
│                                                                             │
│  [3] History Messages      ← 历史对话消息                                   │
│       ├─ 条件: add_history_to_context=True                                  │
│       ├─ 位置: 在additional_input之后，用户消息之前                          │
│       ├─ 控制: num_history_runs / num_history_messages                      │
│       │       max_tool_calls_from_history                                   │
│       ├─ 过滤: 跳过非标准角色(system)的历史消息                              │
│       ├─ 标记: from_history=True (深拷贝，不影响原始消息)                    │
│       └─ 大小: 取决于历史长度，受上述参数控制                                │
│                                                                             │
│  [4] User Message          ← 用户消息 (核心构建对象)                        │
│       ├─ 位置: 消息列表最后                                                  │
│       └─ 内容构成见下方详细分析                                              │
│                                                                             │
│  [5] Input Messages        ← 多消息输入 (List[Message])                     │
│       ├─ 条件: input为List[Message]或List[Dict with role]                   │
│       ├─ 位置: 与user_message并列追加                                       │
│       └─ 特点: 直接追加到消息列表，不经过构建流程                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、用户消息(User Message)内部构成示意图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    用户消息 (User Message) 内部构成                          │
│                   (get_user_message / aget_user_message)                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─ 前置判断 ──────────────────────────────────────────────────────────┐    │
│  │ 1. build_user_context=False? → 直接返回原始input，不做任何处理      │    │
│  │ 2. input=None? → 返回空消息(有媒体)或None(无媒体)                   │    │
│  │ 3. input=Message? → 直接返回，不做任何处理                          │    │
│  │ 4. input=Dict? → 验证为Message后返回                               │    │
│  │ 5. input=BaseModel? → 序列化为JSON后返回                           │    │
│  │ 6. input=List[str]? → join后返回                                   │    │
│  │ 7. input=str? → 进入默认构建流程 ↓                                 │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─ 默认构建流程 (仅当input为str时) ──────────────────────────────────┐    │
│  │                                                                     │    │
│  │  ① 用户原始输入           [核心内容]                                │    │
│  │     └─ 条件: 始终存在                                              │    │
│  │     └─ 大小: 用户输入决定                                          │    │
│  │                                                                     │    │
│  │  ② resolve_in_context     [变量替换]                               │    │
│  │     └─ 条件: agent.resolve_in_context=True (默认True)              │    │
│  │     └─ 作用: 将{session_state}/{dependencies}/{metadata}替换为值   │    │
│  │     └─ 大小: 不增加，只替换                                        │    │
│  │                                                                     │    │
│  │  ③ knowledge_references   [知识库检索结果]                          │    │
│  │     └─ 条件: add_knowledge_to_context=True AND 检索到文档           │    │
│  │     └─ 格式: "\n\nUse the following references..."                  │    │
│  │     │        "<references>\n{docs_json}\n</references>"            │    │
│  │     └─ 大小: 取决于检索文档数量和长度，可能很大                     │    │
│  │     └─ 格式: JSON(默认) 或 YAML(references_format="yaml")          │    │
│  │                                                                     │    │
│  │  ④ dependencies          [依赖上下文]                              │    │
│  │     └─ 条件: add_dependencies_to_context=True AND dependencies存在  │    │
│  │     └─ 格式: "\n\n<additional context>\n{json}\n</additional context>"│    │
│  │     └─ 大小: 取决于依赖数据量                                      │    │
│  │                                                                     │
│  │  ⑤ 媒体附件              [多媒体内容]                              │    │
│  │     ├─ images: send_media_to_model=True时附加                      │    │
│  │     ├─ audio:  send_media_to_model=True时附加                      │    │
│  │     ├─ videos: send_media_to_model=True时附加                      │    │
│  │     └─ files:  send_media_to_model=True时附加                      │    │
│  │     └─ 大小: 取决于媒体文件大小                                    │    │
│  │                                                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  最终: Message(role="user", content=拼接结果, images=..., audio=...)        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 三、各机制深度分析

### 3.1 additional_input (额外消息/Few-Shot学习)

**作用**: 在系统消息之后、历史消息之前注入额外消息，主要用于few-shot学习和提供额外上下文。

**实现方式**:
```python
# _messages.py L1213-L1238 (get_run_messages)
if agent.additional_input is not None:
    messages_to_add_to_run_response: List[Message] = []
    if run_messages.extra_messages is None:
        run_messages.extra_messages = []

    for _m in agent.additional_input:
        if isinstance(_m, Message):
            messages_to_add_to_run_response.append(_m)
            run_messages.messages.append(_m)
            run_messages.extra_messages.append(_m)
        elif isinstance(_m, dict):
            _m_parsed = Message.model_validate(_m)
            messages_to_add_to_run_response.append(_m_parsed)
            run_messages.messages.append(_m_parsed)
            run_messages.extra_messages.append(_m_parsed)
```

**存在条件**: `agent.additional_input` 不为空
**大小**: 用户自定义，无限制
**位置**: 消息列表中系统消息之后、历史消息之前

**为什么这样实现**:
- **不保留在记忆中**: additional_input每次run都重新注入，不存储到session历史。这确保了few-shot示例不会"污染"对话历史
- **位置精心设计**: 放在系统消息之后，确保LLM先理解身份和指令，再看到few-shot示例
- **支持多种类型**: Message对象、Dict、BaseModel，灵活适配不同使用场景
- **典型用途**:
  ```
  # Few-shot学习示例
  additional_input=[
      Message(role="user", content="What is 2+2?"),
      Message(role="assistant", content="4"),
      Message(role="user", content="What is 3+5?"),
      Message(role="assistant", content="8"),
  ]
  ```

---

### 3.2 introduction (介绍消息)

**作用**: 在新会话创建时，自动添加一条Agent的"自我介绍"消息作为第一条assistant回复。

**实现方式**:
```python
# _storage.py L328-L341 (read_or_create_session)
if agent.introduction is not None:
    agent_session.upsert_run(
        RunOutput(
            run_id=str(uuid4()),
            session_id=session_id,
            agent_id=agent.id,
            agent_name=agent.name,
            user_id=user_id,
            content=agent.introduction,
            messages=[
                Message(role=agent.model.assistant_message_role, content=agent.introduction)
            ],
        )
    )
```

**存在条件**: `agent.introduction` 不为空 且 会话是新创建的（非已有会话）
**大小**: 用户自定义
**位置**: 作为会话的第一条assistant消息存入session历史

**为什么这样实现**:
- **只在新建会话时触发**: 已有会话不会重复添加introduction
- **作为历史消息存在**: introduction不是注入到系统提示词，而是作为assistant的历史回复。当`add_history_to_context=True`时，它会出现在历史消息中
- **用户体验**: 让Agent在首次交互时就能"自我介绍"，建立角色认知
- **与system_message的区别**: system_message是系统级指令（LLM不应暴露），introduction是Agent对用户说的话

**关键设计洞察**:
```
introduction → 存入session历史 → 作为assistant消息出现在对话中
                  ↓
          后续run时通过add_history_to_context加载
                  ↓
          LLM看到"自己之前说过的话"，保持角色一致性
```

---

### 3.3 add_history_to_context (历史消息注入)

**作用**: 将之前的对话历史注入到消息列表中，实现多轮对话的上下文连续性。

**实现方式**:
```python
# _messages.py L1241-L1272 (get_run_messages)
if add_history_to_context:
    skip_role = (
        agent.system_message_role 
        if agent.system_message_role not in ["user", "assistant", "tool"] 
        else None
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
**大小控制参数**:
| 参数 | 作用 | 默认值 |
|------|------|--------|
| `num_history_runs` | 加载最近N次run的消息 | None(全部) |
| `num_history_messages` | 加载最近N条消息 | None(全部) |
| `max_tool_calls_from_history` | 历史中最多保留N次工具调用 | None(全部) |

**为什么这样实现**:
- **深拷贝**: 避免修改原始历史消息，确保session数据的不可变性
- **from_history标记**: 区分历史消息和当前消息，便于后续处理（如压缩时只压缩历史的工具结果）
- **角色过滤**: 跳过历史中的system消息（因为当前run已有新的system message），但保留user/assistant/tool消息
- **工具调用过滤**: `max_tool_calls_from_history`是关键的token优化手段——工具调用结果通常很长，限制历史中的工具调用数可以显著减少token消耗
- **Team场景**: `agent_id`过滤确保在Team中只加载属于当前Agent的历史

---

### 3.4 resolve_in_context (上下文变量替换)

**作用**: 在用户消息和系统消息中，将`{变量名}`占位符替换为session_state、dependencies、metadata中的实际值。

**实现方式**:
```python
# _messages.py L56-L98 (format_message_with_state_variables)
def format_message_with_state_variables(agent, message, run_context=None):
    session_state = run_context.session_state if run_context else None
    dependencies = run_context.dependencies if run_context else None
    metadata = run_context.metadata if run_context else None
    user_id = run_context.user_id if run_context else None

    format_variables = ChainMap(
        session_state if session_state is not None else {},
        dependencies or {},
        metadata or {},
        {"user_id": user_id} if user_id is not None else {},
    )

    # 将 {var_name} 转换为 ${var_name} 以使用Template安全替换
    converted_msg = deepcopy(message)
    for var_name in format_variables.keys():
        pattern = r"\{" + re.escape(var_name) + r"\}"
        replacement = "${" + var_name + "}"
        converted_msg = re.sub(pattern, replacement, converted_msg)

    template = string.Template(converted_msg)
    return template.safe_substitute(format_variables)
```

**存在条件**: `resolve_in_context=True`（默认True）
**大小**: 不增加内容，只做替换
**替换优先级**: `session_state` > `dependencies` > `metadata` > `user_id`

**为什么这样实现**:
- **ChainMap优先级**: session_state优先级最高，允许运行时状态覆盖静态依赖
- **安全替换**: 使用`safe_substitute`而非`substitute`，未匹配的变量保持原样而不报错
- **两步转换**: 先将`{var}`转为`${var}`，再使用`string.Template`，避免与JSON/代码中的花括号冲突
- **应用范围**: 同时应用于系统消息和用户消息，确保所有模板变量都能被解析
- **典型用途**:
  ```python
  # instructions中引用session_state
  instructions="You are helping {user_name} with their project {project_name}"
  
  # 运行时session_state = {"user_name": "Alice", "project_name": "Agno"}
  # 替换后: "You are helping Alice with their project Agno"
  ```

---

### 3.5 add_knowledge_to_context (知识库检索注入到用户消息)

**作用**: 自动从知识库检索与用户查询相关的文档，将检索结果附加到用户消息中（RAG机制）。

**实现方式**:
```python
# _messages.py L915-L964 (get_user_message)
if agent.add_knowledge_to_context:
    docs_from_knowledge = get_relevant_docs_from_knowledge(
        agent, query=user_msg_content, filters=knowledge_filters, run_context=run_context
    )
    if docs_from_knowledge is not None:
        references = MessageReferences(
            query=user_msg_content,
            references=docs_from_knowledge,
            time=round(retrieval_timer.elapsed, 4),
        )
        # 追加到用户消息
        user_msg_content_str += "\n\nUse the following references from the knowledge base if it helps:\n"
        user_msg_content_str += "<references>\n"
        user_msg_content_str += convert_documents_to_string(agent, references.references) + "\n"
        user_msg_content_str += "</references>"
```

**存在条件**: `add_knowledge_to_context=True` 且检索到相关文档
**大小**: 取决于检索到的文档数量和长度
**位置**: 用户消息中，原始输入之后

**与search_knowledge(Agentic RAG)的对比**:
```
┌──────────────────────┬─────────────────────────────────────────────────────┐
│  机制                │  add_knowledge_to_context  │  search_knowledge      │
├──────────────────────┼───────────────────────────┼────────────────────────┤
│  触发方式            │  自动（每次run）           │  Agent主动调用工具      │
│  注入位置            │  用户消息                  │  工具调用结果           │
│  检索时机            │  构建消息时                │  Agent决定何时检索      │
│  灵活性              │  低（自动检索）            │  高（Agent自主判断）    │
│  Token消耗           │  每次都消耗                │  按需消耗               │
│  适用场景            │  查询明确、知识库稳定      │  查询复杂、需多轮检索   │
└──────────────────────┴───────────────────────────┴────────────────────────┘
```

**为什么放在用户消息而非系统消息**:
- **查询相关性**: 知识检索结果与具体查询强相关，放在用户消息中语义更准确
- **系统消息稳定性**: 系统消息应保持相对稳定（身份、指令、记忆等），而知识引用每次查询都不同
- **LLM注意力机制**: 用户消息中的引用与查询紧邻，LLM更容易建立关联

**文档格式化**:
```python
# _utils.py L60-L69
def convert_documents_to_string(agent, docs):
    if agent.references_format == "yaml":
        import yaml
        return yaml.dump(docs)
    return json.dumps(docs, indent=2, ensure_ascii=False)  # 默认JSON
```

---

### 3.6 add_dependencies_to_context (依赖上下文注入)

**作用**: 将外部依赖数据（通过run_context.dependencies传入）注入到用户消息中。

**实现方式**:
```python
# _messages.py L966-L969 (get_user_message)
if add_dependencies_to_context and dependencies is not None:
    user_msg_content_str += "\n\n<additional context>\n"
    user_msg_content_str += convert_dependencies_to_string(agent, dependencies) + "\n"
    user_msg_content_str += "</additional context>"
```

**存在条件**: `add_dependencies_to_context=True` 且 `dependencies` 不为空
**大小**: 取决于依赖数据量
**位置**: 用户消息中，知识引用之后

**依赖数据序列化**:
```python
# _utils.py L72-L104
def convert_dependencies_to_string(agent, context):
    try:
        return json.dumps(context, indent=2, default=str)
    except (TypeError, ValueError, OverflowError):
        # 降级处理：逐个序列化，失败的转为字符串
        sanitized_context = {}
        for key, value in context.items():
            try:
                json.dumps({key: value}, default=str)
                sanitized_context[key] = value
            except Exception:
                sanitized_context[key] = str(value)
        return json.dumps(sanitized_context, indent=2)
```

**为什么这样实现**:
- **健壮的序列化**: 使用`default=str`处理不可序列化的对象，再逐个降级，确保不会因单个字段失败导致整个序列化崩溃
- **放在用户消息中**: dependencies是请求相关的运行时数据，与具体查询关联
- **与resolve_in_context互补**:
  - `resolve_in_context`: 将dependencies中的值替换到模板占位符中
  - `add_dependencies_to_context`: 将完整的dependencies展示给LLM

---

### 3.7 send_media_to_model (媒体附件)

**作用**: 控制是否将图片、音频、视频、文件等媒体内容发送给LLM模型。

**实现方式**:
```python
# _messages.py L847-L856 (get_user_message)
if not agent.build_user_context:
    return Message(
        role=agent.user_message_role or "user",
        content=input,
        images=None if not agent.send_media_to_model else images,
        audio=None if not agent.send_media_to_model else audio,
        videos=None if not agent.send_media_to_model else videos,
        files=None if not agent.send_media_to_model else files,
    )
```

**存在条件**: `send_media_to_model=True`（默认True）
**大小**: 取决于媒体文件大小
**位置**: Message对象的images/audio/videos/files属性

**为什么这样实现**:
- **默认开启**: 大多数多模态模型需要媒体数据
- **可关闭**: 某些场景下媒体只供工具使用（如图片分析工具），不需要发送给LLM
- **统一控制**: 一个开关控制所有媒体类型，简化配置

---

### 3.8 build_user_context (用户消息构建开关)

**作用**: 控制是否对用户消息进行默认构建（添加知识引用、依赖上下文等）。

**实现方式**:
```python
# _messages.py L847-L856
if not agent.build_user_context:
    # 直接返回原始input，不做任何处理
    return Message(
        role=agent.user_message_role or "user",
        content=input,
        images=None if not agent.send_media_to_model else images,
        ...
    )
```

**存在条件**: `build_user_context=False`时跳过所有用户消息构建
**为什么这样实现**:
- **完全控制**: 当开发者需要完全自定义用户消息格式时，可以关闭默认构建
- **性能优化**: 某些场景不需要知识检索和依赖注入，关闭可以节省处理时间

---

### 3.9 input_schema (输入验证)

**作用**: 为用户输入定义JSON Schema，用于验证和格式化输入。

**实现方式**:
```python
# _messages.py L1309-L1313
if agent.input_schema and is_typed_dict(agent.input_schema):
    import json
    content = json.dumps(input, indent=2, ensure_ascii=False)
    user_message = Message(role=agent.user_message_role, content=content)
```

**存在条件**: `input_schema` 不为空 且 input为Dict
**大小**: 取决于输入数据
**为什么这样实现**:
- **类型安全**: 确保用户输入符合预期格式
- **TypedDict处理**: 当input_schema是TypedDict时，将dict序列化为JSON字符串

---

## 四、ContextProvider机制深度分析

### 4.1 ContextProvider架构

ContextProvider是Agno框架中一种特殊的上下文注入机制，它通过**工具**和**指令**两种方式将外部信息源暴露给Agent。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ContextProvider 架构                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ContextProvider (ABC)                                                      │
│  ├── query(question) → Answer      # 自然语言查询                           │
│  ├── update(instruction) → Answer  # 自然语言写入                           │
│  ├── status() → Status             # 健康检查                               │
│  ├── instructions() → str          # 使用指南                               │
│  └── get_tools() → list            # 暴露的工具                             │
│                                                                             │
│  三种暴露模式 (ContextMode):                                                │
│  ┌────────────────┬──────────────────────────────────────────────────┐      │
│  │  mode=default  │ 推荐暴露方式（通常为query+update两个工具）        │      │
│  │  mode=agent    │ 只读query工具（通过子Agent）                      │      │
│  │  mode=tools    │ 暴露底层工具（直接操作）                          │      │
│  └────────────────┴──────────────────────────────────────────────────┘      │
│                                                                             │
│  内置ContextProvider:                                                       │
│  ├── SlackContextProvider     # Slack工作区读写                             │
│  ├── WebContextProvider       # Web搜索和阅读                               │
│  ├── WorkspaceContextProvider # 本地项目文件                                │
│  ├── MCPContextProvider       # MCP协议服务器                               │
│  ├── DatabaseContextProvider  # 数据库查询                                  │
│  ├── FilesystemContextProvider# 文件系统访问                                │
│  ├── GoogleDriveContextProvider# Google Drive                               │
│  ├── GmailContextProvider     # Gmail邮件                                   │
│  ├── GoogleCalendarContextProvider# Google日历                              │
│  └── WikiContextProvider      # Wiki知识库                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 ContextProvider如何注入到Agent

ContextProvider通过两种渠道影响Agent:

**渠道1: 工具注入** (通过`agent.tools`列表)
```
ContextProvider.get_tools() → 返回工具函数 → 添加到agent_tools
    ↓
agent_tools在run时注册到模型 → LLM可以调用这些工具
```

**渠道2: 指令注入** (通过`provider.instructions()`)
```
ContextProvider.instructions() → 返回使用指南字符串
    ↓
作为Toolkit的instructions → 添加到agent._tool_instructions
    ↓
注入到系统提示词的位置⑤ (tool_instructions)
```

### 4.3 SlackContextProvider详细分析

**作用**: 让Agent能够读写Slack工作区，通过自然语言查询和更新。

**架构设计**:
```
SlackContextProvider
├── 读取模式
│   ├── Bot Read Agent      # 使用Bot Token读取
│   │   └── SlackTools(只读工具集)
│   └── Assisted Read Agent # 使用Action Token读取(含搜索)
│       └── SlackTools(只读+搜索工具集)
└── 写入模式
    └── Write Agent         # 发送消息
        └── SlackTools(写入+查询工具集)
```

**三种模式的指令差异**:
| 模式 | 指令内容 | 工具 |
|------|---------|------|
| `default` | "call query_slack(question) to read. Use update_slack(instruction) to post." | query + update |
| `agent` | "call query_slack(question) to read Slack." | query only |
| `tools` | "get_channel_history(channel) for latest messages..." | 原始SlackTools |

**为什么用子Agent**:
- **职责分离**: 读取和写入是不同的职责，分离后每个子Agent的指令更聚焦
- **Token优化**: 7个Slack工具直接暴露会膨胀主Agent的提示词，子Agent只暴露2个高层工具
- **安全性**: 写入操作通过独立Agent控制，可以单独配置指令和限制
- **懒初始化**: 子Agent在首次使用时才创建，避免不必要的资源消耗

---

## 五、CompressionManager机制深度分析

### 5.1 作用

压缩工具调用结果以节省上下文空间，同时保留关键信息。

### 5.2 实现方式

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CompressionManager 工作流程                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 判断是否需要压缩                                                        │
│     ├── Token阈值: compress_token_limit (总token超过阈值时压缩)              │
│     └── 数量阈值: compress_tool_results_limit (未压缩工具结果数超过阈值)     │
│                                                                             │
│  2. 对每个未压缩的tool消息执行压缩                                          │
│     ├── 构建压缩请求:                                                       │
│     │   System: DEFAULT_COMPRESSION_PROMPT (压缩指令)                       │
│     │   User: "Tool Results to Compress: {tool_content}"                   │
│     │                                                                      │
│     └── 调用LLM压缩 → 返回压缩后的内容                                     │
│                                                                             │
│  3. 将压缩结果存入 msg.compressed_content                                   │
│     └── 后续发送消息时，使用compressed_content替代原始content               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**压缩提示词核心原则**:
```
ALWAYS PRESERVE:   事实、时间、实体、标识符、关键引用
COMPRESS:          描述→关键属性，解释→核心洞察，列表→最相关项
REMOVE ENTIRELY:   开头/结尾/过渡语，模糊语言，元评论，格式残留
```

**压缩示例**:
```
输入: "According to recent market analysis and industry reports, OpenAI has made 
several significant announcements... The company revealed ChatGPT Atlas on October 
21, 2025... Additionally, on October 6, 2025, OpenAI launched Apps in ChatGPT..."

输出: "OpenAI - Oct 21 2025: ChatGPT Atlas (AI browser, macOS, search competitor); 
Oct 6 2025: Apps in ChatGPT + SDK; Partners: Spotify, Zillow, Canva"
```

### 5.3 为什么这样实现

- **LLM驱动的压缩**: 不是简单的截断，而是由LLM理解语义后提取关键信息
- **双阈值触发**: Token阈值防止上下文溢出，数量阈值防止工具结果堆积
- **增量压缩**: 只压缩未压缩的工具结果（`compressed_content is None`），避免重复压缩
- **异步并行**: `acompres`使用`asyncio.gather`并行压缩多个工具结果
- **统计追踪**: 记录压缩前后的size，便于监控压缩效果

---

## 六、用户消息层 vs 系统消息层 对比总结

```
┌──────────────────────┬───────────────────────────┬───────────────────────────┐
│  维度                │  系统消息层                │  用户消息层                │
├──────────────────────┼───────────────────────────┼───────────────────────────┤
│  核心职责            │  定义Agent身份和行为       │  提供请求相关的上下文      │
│  稳定性              │  相对稳定(跨run不变)       │  每次run都变化             │
│  LLM视角            │  "我是谁，我该怎么做"      │  "这次用户要什么"          │
│  典型内容            │  instructions, memories,   │  knowledge references,    │
│                      │  session_state, culture    │  dependencies, media      │
│  Token消耗           │  相对固定                  │  随请求变化                │
│  优化手段            │  session_summary压缩历史   │  compression压缩工具结果  │
│                      │  懒初始化                  │  max_tool_calls过滤       │
│  变量替换            │  resolve_in_context        │  resolve_in_context       │
│  少样本学习          │  不适用                    │  additional_input         │
└──────────────────────┴───────────────────────────┴───────────────────────────┘
```

---

## 七、消息列表层的特殊机制

### 7.1 消息列表完整构建顺序

```
get_run_messages() 构建顺序:

[1] system_message          ← get_system_message()
[2] additional_input        ← agent.additional_input (few-shot)
[3] history_messages        ← session.get_messages() (历史对话)
[4] user_message            ← get_user_message() (含knowledge+dependencies)
[5] input_messages          ← List[Message] 直接追加
```

### 7.2 消息过滤与优化策略

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    消息优化策略全景                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  系统消息层优化:                                                            │
│  ├── session_summary替代完整历史     (减少50-90%历史token)                  │
│  ├── 懒初始化MemoryManager          (无记忆时不消耗token)                   │
│  └── 优先级声明                     (减少冲突导致的重复)                    │
│                                                                             │
│  用户消息层优化:                                                            │
│  ├── knowledge按需检索              (只检索相关文档)                        │
│  ├── dependencies序列化降级         (确保不因序列化失败而中断)               │
│  └── build_user_context开关         (完全跳过构建)                          │
│                                                                             │
│  消息列表层优化:                                                            │
│  ├── max_tool_calls_from_history    (限制历史工具调用数)                    │
│  ├── num_history_runs/messages      (限制历史范围)                          │
│  ├── CompressionManager            (压缩工具结果)                           │
│  ├── from_history标记              (区分历史/当前消息)                      │
│  └── 角色过滤                       (跳过历史system消息)                    │
│                                                                             │
│  ContextProvider优化:                                                       │
│  ├── 子Agent封装                    (减少主Agent工具数)                     │
│  ├── 懒初始化                       (首次使用时创建)                        │
│  └── 模式选择                       (default/agent/tools按需暴露)           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 八、关键源码索引

| 机制 | 核心文件 | 关键行号 |
|------|---------|---------|
| 用户消息构建 | `agent/_messages.py` | L821-L983 |
| 消息列表组装 | `agent/_messages.py` | L1156-L1358 |
| 变量替换 | `agent/_messages.py` | L56-L98 |
| additional_input | `agent/_messages.py` | L1213-L1238 |
| introduction | `agent/_storage.py` | L328-L341, L393-L406 |
| 历史消息 | `agent/_messages.py` | L1241-L1272 |
| 知识检索 | `agent/_messages.py` | L915-L964 |
| 依赖注入 | `agent/_messages.py` | L966-L969 |
| 文档序列化 | `agent/_utils.py` | L60-L69 |
| 依赖序列化 | `agent/_utils.py` | L72-L104 |
| CompressionManager | `compression/manager.py` | L52-L280 |
| ContextProvider基类 | `context/provider.py` | L77-L297 |
| SlackContextProvider | `context/slack/provider.py` | L41-L270 |
| WebContextProvider | `context/web/provider.py` | L25-L111 |
| WorkspaceContextProvider | `context/workspace/provider.py` | L28-L123 |
| 工具注册 | `agent/_tools.py` | L105-L205 |
| Agent初始化 | `agent/_init.py` | L240-L265 |
| Agent参数定义 | `agent/agent.py` | L205-L290 |
