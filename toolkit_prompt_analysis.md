# Agno Reasoning Toolkit 提示词与执行效果深度分析

> 分析日期: 2026-06-06
> 目标: 解析 ReasoningTools / KnowledgeTools / MemoryTools / WorkflowTools 的提示词内容、实际执行输出效果、工作作用

---

## 一、工具提示词（Instructions）完整解析

### 1.1 ReasoningTools 提示词

#### DEFAULT_INSTRUCTIONS（核心指令）

```
You have access to the `think` and `analyze` tools to work through problems step-by-step and structure your thought process. 
You must ALWAYS `think` before making tool calls or generating a response.

1. **Think** (scratchpad):
   - Purpose: Use the `think` tool as a scratchpad to break down complex problems, outline steps, 
     and decide on immediate actions within your reasoning flow. Use this to structure your internal monologue.
   - Usage: Call `think` before making tool calls or generating a response. 
     Explain your reasoning and specify the intended action (e.g., "make a tool call", "perform calculation", "ask clarifying question").

2. **Analyze** (evaluation):
   - Purpose: Evaluate the result of a think step or a set of tool calls. Assess if the result is expected, sufficient, or requires further investigation.
   - Usage: Call `analyze` after a set of tool calls. Determine the `next_action` based on your analysis: 
     `continue` (more reasoning needed), `validate` (seek external confirmation/validation if possible), 
     or `final_answer` (ready to conclude).
   - Explain your reasoning highlighting whether the result is correct/sufficient.

## IMPORTANT GUIDELINES
- **Always Think First:** You MUST use the `think` tool before making tool calls or generating a response.
- **Iterate to Solve:** Use the `think` and `analyze` tools iteratively to build a clear reasoning path. 
  The typical flow is `Think` -> [`Tool Calls` if needed] -> [`Analyze` if needed] -> ... -> `final_answer`. 
  Repeat this cycle until you reach a satisfactory conclusion.
- **Make multiple tool calls in parallel:** After a `think` step, you can make multiple tool calls in parallel.
- **Keep Thoughts Internal:** The reasoning steps (thoughts and analyses) are for your internal process only. 
  Do not share them directly with the user.
- **Conclude Clearly:** When your analysis determines the `next_action` is `final_answer`, 
  provide a concise and accurate final answer to the user.
```

**提示词核心要素分析**:

| 要素 | 内容 | 作用 |
|------|------|------|
| **强制先思考** | "MUST ALWAYS `think` before making tool calls" | 强制 LLM 进入推理模式，而非直接响应 |
| **工具并行** | "Make multiple tool calls in parallel" | 允许在 think 后并发调用多个工具，提高效率 |
| **内部隔离** | "Keep Thoughts Internal: Do not share them directly" | 推理过程对用户不可见，避免干扰 |
| **迭代循环** | "Think → [Tool Calls] → Analyze → ... → final_answer" | 定义明确的推理工作流 |
| **决策点** | next_action: continue / validate / final_answer | 赋予 LLM 自主决策能力 |

#### FEW_SHOT_EXAMPLES（示例）

**示例 1: 简单事实检索**
```
think(
  title="Understand Request",
  thought="The user wants to know the standard number of continents on Earth. This is a common piece of knowledge.",
  action="Recall or verify the number of continents.",
  confidence=0.95
)
```
```
analyze(
  title="Evaluate Fact",
  result="Standard geographical models list 7 continents...",
  analysis="The recalled information directly answers the user's question accurately.",
  next_action="final_answer",
  confidence=1.0
)
```
→ 简单问题直接 final_answer

**示例 2: 多步信息收集**
```
think(...) # 计划：搜索资本和人口
[并行工具调用] search(capital of France) + search(population of Paris)
analyze(...) # 分析：已获取资本
analyze(...) # 分析：已获取人口 → final_answer
```
→ 复杂问题需要多次迭代

---

### 1.2 KnowledgeTools 提示词

#### DEFAULT_INSTRUCTIONS

```
You have access to the Think, Search, and Analyze tools that will help you search your knowledge for relevant information. 
Use these tools as frequently as needed to find the most relevant information.

## How to use the Think, Search, and Analyze tools:
1. **Think**
   - Purpose: A scratchpad for planning, brainstorming keywords, and refining your approach. 
     You never reveal your "Think" content to the user.
   - Usage: Call `think` whenever you need to figure out what to do next, 
     analyze your approach, or decide new search terms before (or after) you look up documents.

2. **Search**
   - Purpose: Executes a query against the knowledge base.
   - Usage: Call `search` with a clear query string whenever you want to retrieve documents or data. 
     You can and should call this tool multiple times in one conversation.
       - For complex topics, use multiple focused searches rather than one broad search
       - Try different phrasing and keywords if initial searches don't yield useful results
       - Use quotes for exact phrases and OR for alternative terms (e.g., "protein synthesis" OR "protein formation")

3. **Analyze**
   - Purpose: Evaluate whether the returned documents are correct and sufficient. 
     If not, go back to "Think" or "Search" with refined queries.
   - Usage: Call `analyze` after getting search results to verify the quality and correctness of that information. Consider:
       - Relevance: Do the documents directly address the user's question?
       - Completeness: Is there enough information to provide a thorough answer?
       - Reliability: Are the sources credible and up-to-date?
       - Consistency: Do the documents agree or contradict each other?

**Important Guidelines**:
- Do not include your internal chain-of-thought in direct user responses.
- Use "Think" to reason internally. These notes are never exposed to the user.
- Iterate through the cycle (Think → Search → Analyze) as many times as needed until you have a final answer.
- When you do provide a final answer to the user, be clear, concise, and accurate.
- If search results are sparse or contradictory, acknowledge limitations in your response.
- Synthesize information from multiple sources rather than relying on a single document.
```

**与 ReasoningTools 的关键差异**:

| 维度 | ReasoningTools | KnowledgeTools |
|------|---------------|----------------|
| Act 阶段 | 任意外部工具 | 限定为 `search_knowledge` |
| Think 侧重点 | 解题思路 | 搜索关键词、查询策略 |
| Analyze 侧重点 | 结果正确性 | **相关性、完整性、可靠性、一致性** 四维评估 |
| 迭代触发 | 结果不够好 | **文档稀疏或矛盾时也触发** |

#### FEW_SHOT_EXAMPLES（示例）

**示例: 多次搜索与迭代优化**
```
Think: I'll start broad, then refine if needed.
Search: "dietary guidelines for mild hypertension"
Analyze: I got one document referencing the DASH diet, but it's quite brief.
Think: Let me refine my search to see if there are official guidelines.
Search: "WHO or American Heart Association guidelines for hypertension"
Analyze: Now I have daily sodium limits, recommended fruit/vegetable intake...
```

---

### 1.3 MemoryTools 提示词

#### DEFAULT_INSTRUCTIONS

```
You have access to the Think, Add Memory, Update Memory, Delete Memory, and Analyze tools 
that will help you manage user memories and analyze their operations. 
Use these tools as frequently as needed to successfully complete memory management tasks.

## How to use the Think, Memory Operations, and Analyze tools:

1. **Think**
   - Purpose: A scratchpad for planning memory operations, brainstorming memory content, and refining your approach.
   - Usage: Call `think` whenever you need to figure out what memory operations to perform, 
     analyze requirements, or decide on strategy.

2. **Get Memories**
   - Purpose: Retrieves a list of memories from the database for the current user.
   - Usage: Call `get_memories` when you need to retrieve memories for the current user.

3. **Add Memory**
   - Purpose: Creates new memories in the database with specified content and metadata.
   - Usage: Call `add_memory` with memory content and optional topics when you need to store new information.

4. **Update Memory**
   - Purpose: Modifies existing memories in the database by memory ID.
   - Usage: Call `update_memory` with a memory ID and the fields you want to change. Only specify the fields that need updating.

5. **Delete Memory**
   - Purpose: Removes memories from the database by memory ID.
   - Usage: Call `delete_memory` with a memory ID when a memory is no longer needed or requested to be removed.

6. **Analyze**
   - Purpose: Evaluate whether the memory operations results are correct and sufficient. 
     If not, go back to "Think" or use memory operations with refined parameters.
   - Usage: Call `analyze` after performing memory operations to verify:
       - Success: Did the operation complete successfully?
       - Accuracy: Is the memory content correct and well-formed?
       - Completeness: Are all required fields populated appropriately?
       - Errors: Were there any failures or unexpected behaviors?

**Important Guidelines**:
- Do not include your internal chain-of-thought in direct user responses.
- Use "Think" to reason internally.
- When you provide a final answer to the user, be clear, concise, and based on the memory operation results.
- If memory operations fail or produce unexpected results, acknowledge limitations and explain what went wrong.
- Always verify memory IDs exist before attempting updates or deletions.
- Use descriptive topics and clear memory content to make memories easily searchable and understandable.
```

**与前两个工具的关键差异**:

| 维度 | ReasoningTools | KnowledgeTools | MemoryTools |
|------|---------------|----------------|-------------|
| Think 侧重点 | 解题思路 | 搜索关键词 | 记忆操作策略 |
| Act 阶段 | 任意 | 知识库搜索 | **CRUD 四种操作** |
| Analyze 评估点 | 结果正确性 | 四维文档评估 | **操作成功性+内容准确性+字段完整性+错误检查** |
| 特殊约束 | — | — | **ID 存在性检查必须先执行** |

#### FEW_SHOT_EXAMPLES（示例）

**示例: 用户偏好更新**
```
User: Actually, update my dietary info - I'm now eating fish too, so I'm pescatarian.
Think: The user wants to update their previous dietary preference from vegetarian to pescatarian.
Update Memory: memory_id="previous_memory_id", 
                memory="User follows pescatarian diet (vegetarian + fish) and is allergic to nuts",
                topics=["dietary_preferences", "allergies", "food", "pescatarian"]
Analyze: Successfully updated the dietary preference memory. 
         The content now accurately reflects pescatarian diet...
```

---

### 1.4 WorkflowTools 提示词

#### DEFAULT_INSTRUCTIONS

```
You have access to the Think, Run Workflow, and Analyze tools 
that will help you execute workflows and analyze their results. 
Use these tools as frequently as needed to successfully complete workflow-based tasks.

## How to use the Think, Run Workflow, and Analyze tools:

1. **Think**
   - Purpose: A scratchpad for planning workflow execution, brainstorming inputs, and refining your approach.
   - Usage: Call `think` whenever you need to figure out what workflow inputs to use, 
     analyze requirements, or decide on execution strategy before (or after) you run the workflow.

2. **Run Workflow**
   - Purpose: Executes the workflow with specified inputs and parameters.
   - Usage: Call `run_workflow` with appropriate input data whenever you want to execute the workflow.
       - For all workflows, start with simple inputs and gradually increase complexity

3. **Analyze**
   - Purpose: Evaluate whether the workflow execution results are correct and sufficient. 
     If not, go back to "Think" or "Run Workflow" with refined inputs.
   - Usage: Call `analyze` after getting workflow results to verify the quality and correctness of the execution. Consider:
       - Completeness: Did the workflow complete all expected steps?
       - Quality: Are the results accurate and meet the requirements?
       - Errors: Were there any failures or unexpected behaviors?

**Important Guidelines**:
- Do not include your internal chain-of-thought in direct user responses.
- Use "Think" to reason internally.
- When you provide a final answer to the user, be clear, concise, and based on the workflow results.
- If workflow execution fails or produces unexpected results, acknowledge limitations and explain what went wrong.
- Synthesize information from multiple workflow runs if you execute the workflow several times with different inputs.
```

**与前三个工具的关键差异**:

| 维度 | ReasoningTools | KnowledgeTools | MemoryTools | WorkflowTools |
|------|---------------|----------------|-------------|---------------|
| Think 侧重点 | 解题思路 | 搜索关键词 | 操作策略 | **工作流输入规划** |
| Act 阶段 | 任意 | 知识库搜索 | CRUD | **执行工作流** |
| Analyze 评估点 | 结果正确性 | 四维文档评估 | 操作结果评估 | **步骤完整性+结果质量+错误检查** |
| 特殊 | — | 多次迭代优化 | ID 存在性检查 | **支持多轮不同输入执行** |

---

## 二、实际执行输出效果

### 2.1 think() 方法输出效果

#### ReasoningTools.think() 输出格式

```python
Step 1:
Title: Understand the Problem
Reasoning: The user wants to know how to get all items across the river safely...
Action: I'll break down the constraints: boat holds man + 1 item, fox eats chicken, chicken eats grain
Confidence: 0.9

Step 2:
Title: Identify the Solution Pattern
Reasoning: The fox and chicken can't be left alone together. The chicken and grain can't be left alone together...
Action: I need to move the chicken first, then the fox/grain, then bring the chicken back
Confidence: 0.95
```

**特点**: 每个 Step 包含 Title（简短标题）、Reasoning（详细推理）、Action（下一步动作）、Confidence（置信度）。

#### KnowledgeTools.think() 输出格式

```python
Thoughts:
- I need to find information about building agent teams in Agno
- Let me search for "agent team" related content
- I should also try "multi-agent" as an alternative search term
```

**特点**: 简单的思考列表，每行一个思考项。

#### MemoryTools.think() 输出格式

```python
Memory Thoughts:
- I need to store the user's dietary preferences
- Topics like "dietary_preferences" and "allergies" would be good for retrieval
- I should also store their travel preferences for personalized recommendations
```

**特点**: 带命名空间前缀的思考列表（Memory Thoughts 而非 Thoughts）。

#### WorkflowTools.think() 输出格式

```python
Workflow Thoughts:
- The user wants a blog post on AI trends
- I should run the content_creation_workflow with topic="AI, AI agents"
- Let me also add style guidance for the writer
```

**特点**: 带命名空间前缀的思考列表（Workflow Thoughts）。

---

### 2.2 analyze() 方法输出效果

#### ReasoningTools.analyze() 输出格式

```python
Step 3:
Title: Verify Solution
Reasoning: Trip 1: Man takes chicken across. Trip 2: Man returns alone, takes fox across, brings chicken back...
Analysis: The solution correctly handles all constraints. No item is ever left alone with a predator/prey.
Next Action: final_answer
Confidence: 1.0
```

**特点**: 包含 `Next Action` 决策点（continue / validate / final_answer），LLM 自主决定流程走向。

#### KnowledgeTools.analyze() 输出格式

```python
Analysis:
- The documents mention agent teams but are brief on the implementation details
- I need to search for more specific examples
- The documentation seems outdated - let me check for newer versions
```

**特点**: 四维评估导向：相关性、完整性、可靠性、一致性。

#### MemoryTools.analyze() 输出格式

```python
Memory Analysis:
- Successfully added the memory with the correct topics
- The memory content is well-structured and searchable
- All required fields (memory_id, memory, topics, user_id) are populated
```

**特点**: 操作结果导向：成功性、内容准确性、字段完整性、错误检查。

#### WorkflowTools.analyze() 输出格式

```python
Workflow Analysis:
- The workflow completed all 4 steps successfully
- The blog post content looks comprehensive and well-structured
- No errors were encountered during execution
```

**特点**: 执行结果导向：步骤完整性、结果质量、错误检查。

---

### 2.3 领域特定方法输出效果

#### search_knowledge() 输出

```json
[
  {
    "name": "Building Agent Teams",
    "content": "Agent teams allow you to distribute tasks across multiple specialized agents...",
    "source": "agno_docs",
    "location": "https://docs.agno.com/teams",
    "score": 0.92
  },
  {
    "name": "Team Communication",
    "content": "Teams can communicate through shared context and message passing...",
    "source": "agno_docs",
    "location": "https://docs.agno.com/teams/communication",
    "score": 0.85
  }
]
```

#### get_memories() 输出

```json
{
  "success": true,
  "operation": "get_memories",
  "memories": [
    {
      "memory_id": "550e8400-e29b-41d4-a716-446655440000",
      "memory": "User prefers vegetarian recipes and is allergic to nuts",
      "topics": ["dietary_preferences", "allergies", "food"],
      "user_id": "john_doe@example.com",
      "created_at": "2026-01-15T10:30:00Z",
      "updated_at": "2026-01-15T10:30:00Z"
    }
  ]
}
```

#### run_workflow() 输出

```json
{
  "workflow_id": "blog_post_workflow_v1",
  "workflow_name": "Blog Post Workflow",
  "run_id": "run_abc123",
  "status": "completed",
  "steps_completed": 4,
  "total_steps": 4,
  "output": {
    "content": "# AI Trends in 2024\n\n## Introduction\n...",
    "word_count": 1500,
    "style": "accessible"
  }
}
```

---

## 三、工具工作作用深度解析

### 3.1 核心工作作用

#### think() — 内部推理规划器

**工作作用**:
1. **强制推理**: 在 LLM 执行任何操作前强制调用，阻止"直觉反射"式响应
2. **工作记忆扩展**: 将 LLM 的隐式推理外化为可追溯的显式步骤
3. **策略规划**: 让 LLM 在行动前明确目标、方法和预期结果
4. **置信度自评**: 通过 confidence 参数让 LLM 量化自己对推理的确定性

**为什么会有效**:
- LLM 的"思维"在 token 生成过程中是隐式的，容易跳跃或遗漏
- `think()` 强制 LLM 停下来写出推理步骤，等同于给 LLM 增加了"草稿纸"
- 格式化的输出（Title + Reasoning + Action + Confidence）让推理结构化

#### analyze() — 结果评估与决策点

**工作作用**:
1. **结果验证**: 在执行后检查结果是否符合预期
2. **迭代决策**: 决定是继续迭代还是输出最终答案
3. **质量把控**: 通过 next_action 参数实现多级质量门控
4. **错误处理**: 发现问题后可以提前终止或调整策略

**为什么会有效**:
- `next_action` 参数将控制流决策权交给 LLM，实现"智能循环"
- 三种决策（continue / validate / final_answer）覆盖了大多数推理场景
- 评估标准（正确性/相关性/完整性等）引导 LLM 进行有目的性的检查

#### 领域方法（search_knowledge / CRUD / run_workflow）— 外部能力扩展

**工作作用**:
1. **能力边界扩展**: LLM 自身无法搜索知识库/操作数据库/执行工作流
2. **状态持久化**: 将操作结果存入 session_state，供后续推理使用
3. **并行能力**: 在 think 后可以并行调用多个领域方法

**为什么会有效**:
- LLM 的知识有截止日期，知识库搜索提供实时信息
- 记忆操作让 Agent 具备"个性化"能力
- 工作流执行让复杂任务自动化成为可能

---

### 3.2 对 LLM 行为的引导机制

```
┌─────────────────────────────────────────────────────────────────────┐
│                        提示词 → LLM 行为引导                         │
├─────────────────────────────────────────────────────────────────────┤
│  指令层           │  "MUST think before tool calls"                  │
│                   │  → 强制行为模式，改变默认响应方式                 │
├─────────────────────────────────────────────────────────────────────┤
│  方法说明层        │  "Purpose: scratchpad for planning..."           │
│                   │  → 定义工具语义，让 LLM 理解何时用什么工具        │
├─────────────────────────────────────────────────────────────────────┤
│  使用指南层        │  "Call think before making tool calls..."      │
│                   │  → 具体操作步骤，引导正确调用顺序                 │
├─────────────────────────────────────────────────────────────────────┤
│  示例层           │  think(title="...", thought="...", action="...") │
│                   │  → 格式规范，让 LLM 知道如何填写参数             │
├─────────────────────────────────────────────────────────────────────┤
│  约束层           │  "Do not share thoughts directly with user"     │
│                   │  → 边界约束，防止信息泄露                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 3.3 会话状态（Session State）隔离机制

每个 Toolkit 在 session_state 中使用独立的命名空间:

```python
session_state = {
    # ReasoningTools
    "reasoning_steps": {
        "<run_id>": [
            '{"title":"...","reasoning":"...","next_action":"..."}'
        ]
    },

    # KnowledgeTools
    "thoughts": ["思考1", "思考2"],
    "analysis": ["分析1", "分析2"],

    # MemoryTools
    "memory_thoughts": [...],
    "memory_operations": [...],
    "memory_analysis": [...],

    # WorkflowTools
    "workflow_thoughts": [...],
    "workflow_results": [...],
    "workflow_analysis": [...]
}
```

**为什么这样设计**:
1. **避免冲突**: 不同 Toolkit 可以共存于同一 Agent
2. **按 run_id 隔离**: 同一会话的多次运行互不干扰
3. **可追溯性**: 每种操作都有完整的历史记录

---

### 3.4 工具输出对后续推理的影响

think()/analyze() 返回完整历史的设计至关重要:

```
第1次迭代:
  think(...) → 返回 [Step1]
  
第2次迭代:
  think(...) → 返回 [Step1, Step2]
  
第3次迭代:
  analyze(...) → 返回 [Step1, Step2, Step3]
  
→ LLM 始终能看到完整的推理链，而非只看到最后一次调用
→ 这让 LLM 能够基于历史推理进行更高层次的综合
```

---

## 四、总结对比表

| 维度 | ReasoningTools | KnowledgeTools | MemoryTools | WorkflowTools |
|------|---------------|----------------|-------------|---------------|
| **核心问题** | "怎么解这道题" | "相关文档在哪" | "用户记忆如何管理" | "工作流如何执行" |
| **think() 输出** | 推理步骤列表 | 思考列表 | Memory Thoughts | Workflow Thoughts |
| **act() 方法** | 任意工具 | search_knowledge | CRUD 四操作 | run_workflow |
| **analyze() 评估** | 结果正确性 | 文档四维评估 | 操作结果评估 | 执行结果评估 |
| **next_action 选项** | continue / validate / final_answer | 隐式（稀疏则重搜） | 隐式（失败则重试） | 隐式（不完整则重跑） |
| **典型应用场景** | 逻辑推理、数学证明、决策分析 | RAG 问答、文档检索 | 个性化推荐、用户画像管理 | 自动化流程、多步骤任务 |
| **典型示例** | 渡河难题、财务分析 | 产品文档问答 | 旅行规划助手 | 博客自动生成 |

---

## 五、文件索引

| 文件 | 内容 |
|------|------|
| `agno/tools/reasoning.py` | ReasoningTools 源码 |
| `agno/tools/knowledge.py` | KnowledgeTools 源码 |
| `agno/tools/memory.py` | MemoryTools 源码 |
| `agno/tools/workflow.py` | WorkflowTools 源码 |
| `agno/reasoning/step.py` | ReasoningStep 模型定义 |
| `cookbook/10_reasoning/tools/` | 各工具使用示例 |
| `cookbook/10_reasoning/teams/` | 团队推理示例 |
