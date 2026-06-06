# Agno Reasoning Toolkit 体系完整实现逻辑分析

> 源码位置: `/workspace/libs/agno/agno/tools/`
> 分析日期: 2026-06-06
> 目标: 完整解析 ReasoningTools / KnowledgeTools / MemoryTools / WorkflowTools 的实现逻辑，提取可复用设计模式

---

## 一、架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                        Agent / Team                          │
│  (持有 Toolkit 实例，通过 Function 注册机制调用工具方法)        │
└──────────────┬──────────────────────────────────────────────┘
               │ 继承
┌──────────────▼──────────────────────────────────────────────┐
│                    Toolkit (基类)                             │
│  - name, instructions, add_instructions                      │
│  - functions: Dict[str, Function]  (同步工具注册表)           │
│  - async_functions: Dict[str, Function]  (异步工具注册表)     │
│  - register() / _register_tools() / _register_async_tools() │
│  - auto_register: bool  (自动注册机制)                        │
│  - include_tools / exclude_tools  (工具过滤)                  │
│  - cache_results / cache_ttl / cache_dir  (结果缓存)         │
└──────────┬──────────────────────────────────────────────────┘
           │
     ┌─────┴──────────┬──────────────┬───────────────┐
     ▼                ▼              ▼               ▼
ReasoningTools  KnowledgeTools  MemoryTools   WorkflowTools
 think()         think()         think()        think()
 analyze()       search_knowledge() get_memories() run_workflow()
                 analyze()       add_memory()    analyze()
                                 update_memory()
                                 delete_memory()
                                 analyze()
```

---

## 二、核心依赖关系

### 2.1 RunContext — 运行时上下文（所有工具共享）

```python
@dataclass
class RunContext:
    run_id: str                    # 当前运行 ID
    session_id: str                # 会话 ID
    user_id: Optional[str]         # 用户 ID
    session_state: Optional[Dict]  # 会话状态（工具间共享数据的核心载体）
    messages: Optional[List]       # 当前运行的消息列表
    # ... 其他字段
```

**关键设计**: `session_state` 是一个 `Dict[str, Any]`，所有工具通过它共享推理步骤、思考记录、分析结果等中间状态。这是整个 Toolkit 体系的核心数据通道。

### 2.2 ReasoningStep — 推理步骤模型

```python
class NextAction(str, Enum):
    CONTINUE = "continue"
    VALIDATE = "validate"
    FINAL_ANSWER = "final_answer"
    RESET = "reset"

class ReasoningStep(BaseModel):
    title: Optional[str]           # 步骤标题
    reasoning: Optional[str]       # 推理内容
    action: Optional[str]          # 计划执行的动作
    result: Optional[str]          # 动作结果
    next_action: Optional[NextAction]  # 下一步行动枚举
    confidence: Optional[float]    # 置信度 (0.0-1.0)
```

### 2.3 Function — 工具函数注册模型

```python
class Function(BaseModel):
    name: str                      # 函数名
    description: Optional[str]     # 函数描述（LLM 可见）
    parameters: Dict[str, Any]     # JSON Schema 参数定义
    entrypoint: Optional[Callable] # 实际执行的函数
    instructions: Optional[str]    # 使用说明
    # ... hooks, cache, confirmation 等配置
```

---

## 三、四大 Toolkit 详细实现逻辑

### 3.1 ReasoningTools — 通用推理工具

**源码**: `agno/tools/reasoning.py`

**用途**: 提供分步推理能力，让 LLM 在执行前先"思考"，形成可追溯的推理链。

**核心工具方法**:

#### think(run_context, title, thought, action, confidence)

```
输入: title(步骤标题), thought(思考内容), action(计划动作), confidence(置信度)
处理:
  1. 创建 ReasoningStep 对象 (next_action=CONTINUE)
  2. 将 step 序列化为 JSON，追加到 session_state["reasoning_steps"][run_id]
  3. 遍历当前 run_id 下的所有步骤，格式化输出完整推理链
输出: 格式化的推理步骤列表字符串
```

**状态存储模式**:
```python
session_state = {
    "reasoning_steps": {
        "<run_id>": [
            '{"title":"...","reasoning":"...","action":"...","confidence":0.8}',  # Step 1
            '{"title":"...","reasoning":"...","action":"...","confidence":0.9}',  # Step 2
        ]
    }
}
```

#### analyze(run_context, title, result, analysis, next_action, confidence)

```
输入: title(分析标题), result(前一步结果), analysis(分析内容), next_action(continue/validate/final_answer), confidence
处理:
  1. 将 next_action 字符串映射为 NextAction 枚举
     - "continue" → CONTINUE
     - "validate" → VALIDATE
     - "final"/"final_answer"/"finalize" → FINAL_ANSWER
  2. 创建 ReasoningStep (含 result 和 next_action)
  3. 同样追加到 session_state["reasoning_steps"][run_id]
  4. 返回格式化的完整推理链
输出: 格式化的推理步骤列表字符串
```

**初始化参数**:
```python
ReasoningTools(
    enable_think=True,      # 启用 think 工具
    enable_analyze=True,    # 启用 analyze 工具
    all=False,              # 启用所有工具
    instructions=None,      # 自定义指令（覆盖默认）
    add_instructions=False, # 是否将指令添加到 Agent 系统消息
    add_few_shot=False,     # 是否添加 few-shot 示例
    few_shot_examples=None, # 自定义 few-shot 示例
)
```

**默认指令核心要点**:
- 必须在调用其他工具或生成响应前先调用 `think`
- 典型流程: `Think → [Tool Calls] → [Analyze] → ... → final_answer`
- 可以在 `think` 后并行发起多个工具调用
- 推理步骤对用户不可见（内部过程）

---

### 3.2 KnowledgeTools — 知识库推理工具

**源码**: `agno/tools/knowledge.py`

**用途**: 将推理与知识库搜索结合，实现 RAG 式的迭代检索-分析循环。

**核心工具方法**:

#### think(run_context, thought)

```
输入: thought(思考内容)
处理:
  1. 将 thought 追加到 session_state["thoughts"] 列表
  2. 格式化输出所有思考记录
输出: "Thoughts:\n- thought1\n- thought2\n..."
```

**状态存储模式**:
```python
session_state = {
    "thoughts": ["思考1", "思考2", ...],
    "analysis": ["分析1", "分析2", ...]
}
```

#### search_knowledge(run_context, query)

```
输入: query(搜索查询)
处理:
  1. 调用 self.knowledge.search(query=query) 获取相关文档
  2. 如果无结果，返回 "No documents found"
  3. 将文档列表序列化为 JSON
输出: JSON 字符串 [doc1.to_dict(), doc2.to_dict(), ...]
```

**关键依赖**: `self.knowledge: Knowledge` — 知识库实例，在构造时注入。

#### analyze(run_context, analysis)

```
输入: analysis(分析内容)
处理:
  1. 将 analysis 追加到 session_state["analysis"] 列表
  2. 格式化输出所有分析记录
输出: "Analysis:\n- analysis1\n- analysis2\n..."
```

**迭代循环模式**:
```
Think → Search → Analyze → (不够?) → Think → Search → Analyze → ... → Final Answer
```

**初始化参数**:
```python
KnowledgeTools(
    knowledge=Knowledge(...),   # 必须提供知识库实例
    enable_think=True,
    enable_search=True,
    enable_analyze=True,
    add_instructions=True,      # 默认添加指令到 Agent
    add_few_shot=False,
)
```

---

### 3.3 MemoryTools — 用户记忆推理工具

**源码**: `agno/tools/memory.py`

**用途**: 将推理与用户记忆 CRUD 操作结合，实现智能化的记忆管理。

**核心工具方法**:

#### think(run_context, thought)

```
输入: thought(关于记忆操作的思考)
处理:
  1. 追加到 session_state["memory_thoughts"]
  2. 格式化输出
输出: "Memory Thoughts:\n- thought1\n- thought2\n..."
```

**状态存储模式**:
```python
session_state = {
    "memory_thoughts": ["思考1", ...],
    "memory_operations": [
        {"operation": "get_memories", "success": True, "memories": [...], "error": None},
        {"operation": "add_memory", "success": True, "memory": {...}, "error": None},
    ],
    "memory_analysis": ["分析1", ...]
}
```

#### get_memories(run_context)

```
处理:
  1. 从 run_context.user_id 获取用户 ID
  2. 调用 self.db.get_user_memories(user_id)
  3. 将操作结果记录到 session_state["memory_operations"]
输出: JSON 格式的记忆列表
```

#### add_memory(run_context, memory, topics)

```
输入: memory(记忆内容), topics(可选主题标签列表)
处理:
  1. 生成 UUID 作为 memory_id
  2. 创建 UserMemory 对象
  3. 调用 self.db.upsert_user_memory(user_memory)
  4. 记录操作结果到 session_state["memory_operations"]
输出: JSON {"success": True, "operation": "add_memory", "memory": {...}}
```

#### update_memory(run_context, memory_id, memory, topics)

```
输入: memory_id(必填), memory(可选更新内容), topics(可选更新主题)
处理:
  1. 先查询现有记忆 self.db.get_user_memory(memory_id)
  2. 不存在则返回错误
  3. 合并更新字段，创建新 UserMemory
  4. 调用 self.db.upsert_user_memory(updated_memory)
  5. 记录操作结果
输出: JSON {"success": True, "operation": "update_memory", "memory": {...}}
```

#### delete_memory(run_context, memory_id)

```
输入: memory_id
处理:
  1. 先查询确认存在
  2. 调用 self.db.delete_user_memory(memory_id)
  3. 记录操作结果（含被删除的记忆内容）
输出: JSON {"success": True, "operation": "delete_memory", "memory_id": "...", "deleted_memory": {...}}
```

#### analyze(run_context, analysis)

```
处理: 追加到 session_state["memory_analysis"]，格式化输出
输出: "Memory Analysis:\n- analysis1\n- analysis2\n..."
```

**关键依赖**: `self.db: BaseDb` — 数据库实例，在构造时注入。

---

### 3.4 WorkflowTools — 工作流推理工具

**源码**: `agno/tools/workflow.py`

**用途**: 将推理与工作流执行结合，实现智能化的工作流编排。

**核心工具方法**:

#### think(run_context, thought)

```
处理: 追加到 session_state["workflow_thoughts"]
输出: "Workflow Thoughts:\n- thought1\n..."
```

**状态存储模式**:
```python
session_state = {
    "workflow_thoughts": ["思考1", ...],
    "workflow_results": [
        {workflow_run_output_dict_1},
        {workflow_run_output_dict_2},
    ],
    "workflow_analysis": ["分析1", ...]
}
```

#### run_workflow(run_context, input)

```
输入: input (RunWorkflowInput 类型)
  - input_data: str (工作流输入数据)
  - additional_data: Optional[Dict] (附加数据)
处理:
  1. 如果 input 是 dict，转换为 RunWorkflowInput
  2. 调用 self.workflow.run(
       input=input.input_data,
       user_id=run_context.user_id,
       session_id=run_context.session_id,
       session_state=run_context.session_state,
       additional_data=input.additional_data,
     )
  3. 将结果追加到 session_state["workflow_results"]
输出: JSON 格式的工作流执行结果
```

**RunWorkflowInput 模型**:
```python
class RunWorkflowInput(BaseModel):
    input_data: str = Field(..., description="The input data for the workflow.")
    additional_data: Optional[Dict[str, Any]] = Field(default=None, description="The additional data for the workflow.")
```

#### analyze(run_context, analysis)

```
处理: 追加到 session_state["workflow_analysis"]
输出: "Workflow Analysis:\n- analysis1\n..."
```

**异步支持**: WorkflowTools 是唯一支持异步的 Toolkit，每个方法都有 async 版本:
- `async_think()` / `async_run_workflow()` / `async_analyze()`
- 通过 `async_mode=True` 参数启用，注册时选择同步或异步版本

**关键依赖**: `self.workflow: Workflow` — 工作流实例，在构造时注入。

**特殊注册机制**:
```python
# WorkflowTools 使用 auto_register=False，手动注册工具
super().__init__(auto_register=False, ...)
# 然后根据 async_mode 选择注册同步或异步版本
if async_mode:
    self.register(self.async_think, name="think")
else:
    self.register(self.think, name="think")
```

---

## 四、通用设计模式提取

### 4.1 "Think-Act-Analyze" 三段式推理模式

所有四个 Toolkit 都遵循相同的核心推理循环:

```
┌─────────┐     ┌──────────────┐     ┌─────────┐
│  Think  │ ──→ │  Act/Execute │ ──→ │ Analyze │
│ (规划)  │     │ (执行操作)    │     │ (评估)  │
└─────────┘     └──────────────┘     └─────────┘
      ▲                                     │
      └───────── (不够? 继续迭代) ───────────┘
                    (足够? → Final Answer)
```

| Toolkit | Think | Act | Analyze |
|---------|-------|-----|---------|
| ReasoningTools | think() | (外部工具调用) | analyze() |
| KnowledgeTools | think() | search_knowledge() | analyze() |
| MemoryTools | think() | get/add/update/delete_memory() | analyze() |
| WorkflowTools | think() | run_workflow() | analyze() |

### 4.2 Session State 作为共享状态总线

所有工具通过 `run_context.session_state` 共享中间状态，使用命名空间隔离:

```python
session_state = {
    # ReasoningTools 命名空间
    "reasoning_steps": {run_id: [step_json, ...]},

    # KnowledgeTools 命名空间
    "thoughts": [str, ...],
    "analysis": [str, ...],

    # MemoryTools 命名空间
    "memory_thoughts": [str, ...],
    "memory_operations": [{operation_dict}, ...],
    "memory_analysis": [str, ...],

    # WorkflowTools 命名空间
    "workflow_thoughts": [str, ...],
    "workflow_results": [{result_dict}, ...],
    "workflow_analysis": [str, ...],
}
```

### 4.3 Toolkit 基类注册机制

```
Toolkit.__init__()
  ├── tools 列表 → _register_tools() → 逐个调用 register()
  ├── register(function, name)
  │     ├── 自动检测 iscoroutinefunction → 分配到 functions 或 async_functions
  │     ├── include_tools / exclude_tools 过滤
  │     └── 创建 Function 对象 (name, entrypoint, parameters, cache, hooks...)
  └── auto_register=True 时自动执行注册
```

### 4.4 Instructions + Few-Shot 引导模式

每个 Toolkit 都内置:
1. **DEFAULT_INSTRUCTIONS** — 告诉 LLM 如何使用这些工具
2. **FEW_SHOT_EXAMPLES** — 提供具体使用示例

这些指令通过 `add_instructions=True` 注入到 Agent 的系统消息中，引导 LLM 遵循 Think-Act-Analyze 循环。

### 4.5 构造时依赖注入模式

```python
# KnowledgeTools 注入知识库
KnowledgeTools(knowledge=knowledge_instance)

# MemoryTools 注入数据库
MemoryTools(db=db_instance)

# WorkflowTools 注入工作流
WorkflowTools(workflow=workflow_instance)
```

### 4.6 工具粒度控制模式

所有 Toolkit 都支持通过布尔开关控制启用哪些工具:

```python
# ReasoningTools
enable_think=True, enable_analyze=True

# KnowledgeTools
enable_think=True, enable_search=True, enable_analyze=True

# MemoryTools
enable_think=True, enable_get_memories=True, enable_add_memory=True,
enable_update_memory=True, enable_delete_memory=True, enable_analyze=True

# WorkflowTools
enable_think=False, enable_run_workflow=True, enable_analyze=False

# 全部启用的快捷方式
all=True
```

---

## 五、在其他智能体框架中的复用指南

### 5.1 最小复用: 仅实现核心接口

要在其他框架中复用此模式，最少需要实现:

1. **RunContext 等价物** — 一个运行时上下文对象，包含:
   - `session_state: Dict` — 工具间共享状态
   - `run_id: str` — 运行标识
   - `user_id: str` — 用户标识

2. **Toolkit 基类等价物** — 工具注册与管理:
   - 工具注册方法 (将方法转为 LLM 可调用的 function schema)
   - 指令注入机制
   - 工具过滤 (include/exclude)

3. **Think-Act-Analyze 三方法** — 推理循环:
   - `think()` — 记录思考到 session_state
   - `act()` — 执行领域特定操作
   - `analyze()` — 评估结果，决定是否继续

### 5.2 伪代码实现模板

```python
class ReasoningToolkit:
    """可移植到任意智能体框架的推理工具包"""

    def __init__(self, context_provider, tool_registry, instructions=True, few_shot=False):
        self.context_provider = context_provider  # 获取 RunContext 的方式
        self.tool_registry = tool_registry        # 注册工具到框架的方式
        self.instructions = self.DEFAULT_INSTRUCTIONS if instructions else None
        if few_shot:
            self.instructions += self.FEW_SHOT_EXAMPLES

        # 注册工具
        self.tool_registry.register(self.think)
        self.tool_registry.register(self.analyze)

    def think(self, context, title: str, thought: str, action: str = None, confidence: float = 0.8) -> str:
        """记录推理步骤"""
        step = {"title": title, "reasoning": thought, "action": action, "confidence": confidence}
        state = context.session_state
        state.setdefault("reasoning_steps", {}).setdefault(context.run_id, []).append(step)
        return self._format_reasoning_chain(state["reasoning_steps"][context.run_id])

    def analyze(self, context, title: str, result: str, analysis: str, next_action: str = "continue", confidence: float = 0.8) -> str:
        """分析结果并决定下一步"""
        step = {"title": title, "result": result, "reasoning": analysis, "next_action": next_action, "confidence": confidence}
        state = context.session_state
        state.setdefault("reasoning_steps", {}).setdefault(context.run_id, []).append(step)
        return self._format_reasoning_chain(state["reasoning_steps"][context.run_id])

    def _format_reasoning_chain(self, steps):
        return "\n".join([f"Step {i+1}: Title={s['title']}, Reasoning={s['reasoning']}, Confidence={s['confidence']}" for i, s in enumerate(steps)])
```

### 5.3 框架适配要点

| 目标框架 | RunContext 适配 | 工具注册适配 | 状态共享适配 |
|---------|---------------|------------|------------|
| LangChain | CallbackHandler + AgentState | @tool 装饰器 | Agent.memory/state |
| AutoGen | 上下文传递 | Function 注册 | 对象属性 |
| CrewAI | Context 参数 | @tool 装饰器 | Agent 的属性 |
| Semantic Kernel | KernelArguments | KernelFunction 注册 | Context 变量 |
| 自研框架 | 自定义 Context 对象 | 自定义注册器 | Dict 传递 |

### 5.4 关键实现细节

1. **think() 必须返回完整历史** — LLM 需要看到之前的所有推理步骤才能保持推理连贯性
2. **session_state 按 run_id 隔离** — 同一会话可能有多次运行，需要隔离推理链
3. **analyze() 的 next_action 是 LLM 自主决策** — LLM 通过参数决定继续/验证/结束，不是硬编码逻辑
4. **Instructions 是核心** — 工具本身只做记录和执行，真正的"推理"由 LLM 通过遵循指令完成
5. **工具结果不直接发给用户** — think/analyze 的内容是 LLM 内部过程，最终答案由 LLM 综合后输出

---

## 六、数据流完整图

```
用户输入
  │
  ▼
Agent 接收
  │
  ├── 读取 Toolkit.instructions → 注入系统消息
  │
  ▼
LLM 推理循环
  │
  ├──► think(title, thought, action, confidence)
  │      │
  │      ├── 创建 ReasoningStep → 序列化 → 存入 session_state
  │      └── 返回完整推理链 ← LLM 看到所有历史步骤
  │
  ├──► [领域特定操作]  (search_knowledge / add_memory / run_workflow / ...)
  │      │
  │      ├── 调用外部依赖 (Knowledge / DB / Workflow)
  │      ├── 记录操作结果到 session_state
  │      └── 返回操作结果
  │
  ├──► analyze(title, result, analysis, next_action, confidence)
  │      │
  │      ├── 创建 ReasoningStep → 序列化 → 存入 session_state
  │      ├── LLM 决定 next_action: continue / validate / final_answer
  │      └── 返回完整推理链
  │
  ├── (next_action != final_answer? → 回到 think 继续迭代)
  │
  └── (next_action == final_answer? → 生成最终响应)
         │
         ▼
用户收到最终答案 (推理过程不可见)
```

---

## 七、源码文件索引

| 文件 | 作用 |
|------|------|
| `agno/tools/toolkit.py` | Toolkit 基类 — 工具注册、过滤、缓存 |
| `agno/tools/function.py` | Function 模型 — 参数解析、执行、hook 链 |
| `agno/tools/reasoning.py` | ReasoningTools — think + analyze |
| `agno/tools/knowledge.py` | KnowledgeTools — think + search_knowledge + analyze |
| `agno/tools/memory.py` | MemoryTools — think + CRUD + analyze |
| `agno/tools/workflow.py` | WorkflowTools — think + run_workflow + analyze (含异步) |
| `agno/reasoning/step.py` | ReasoningStep / NextAction 模型 |
| `agno/run/base.py` | RunContext 数据类 |
