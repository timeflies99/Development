
# 🤝 LangGraph 人机协动：构建可中断、可干预、可协作的智能体工作流

> **核心思想**：\
> 真正可靠的 AI 系统不是“全自动”，而是“人在环路”（Human-in-the-Loop）。LangGraph 通过 **`interrupt()` 机制**，将人类专家无缝嵌入智能体执行流程，实现**可控、可信、可审计**的协作式 AI。

***

## 📌 一、背景与动机：为何需要人机协动？

当前 LLM 智能体面临三大信任瓶颈：

1. **不可控性**：一旦启动，无法中途干预或修正。
2. **不可解释性**：黑盒决策难以追溯，尤其在高风险场景（如医疗、金融）。
3. **知识盲区**：LLM 无法处理超出训练数据的专业问题。

LangGraph 的 **人机协动（Human-in-the-Loop Collaboration）** 机制，通过 **执行中断（Interruption） + 人工干预（Human Input） + 状态恢复（Resumption）** 三步闭环，解决了上述问题。

> 💡 **类比理解**：\
> `interrupt()` ≈ 编程中的 `breakpoint()` + 用户输入弹窗 + 执行上下文保存


***

## 🧱 二、核心机制：`interrupt()` 与 `Command`

### 2.1 中断：暂停执行并请求人工介入

```python
from langgraph.types import interrupt

def human_assistance(query: str) -&gt; str:
    """请求人类专家协助"""
    # 暂停图执行，并传递上下文给人工操作员
    human_response = interrupt({"query": query})
    return human_response["data"]
```

* `interrupt(payload)`：立即暂停当前图执行，保存完整状态。
* 返回值为后续恢复时传入的 `Command` 中的 `resume` 数据。

### 2.2 恢复：注入人工输入并继续执行

```python
from langgraph.types import Command

# 人工提供的答案
human_response = "We recommend using LangGraph for building reliable AI agents."

# 构造恢复命令
human_command = Command(resume={"data": human_response})

# 恢复执行（自动接续中断前的状态）
events = graph.stream(human_command, config, stream_mode="values")
```

> 🔍 **关键设计**：
>
> * `Command` 是恢复执行的**唯一合法入口**，确保干预可控。
> * `resume` 字段内容将作为 `interrupt()` 的返回值，无缝衔接业务逻辑。

***

## 🔄 三、构建支持人机协动的智能体

### 3.1 状态与工具定义

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]

@tool
def human_assistance(query: str) -&gt; str:
    """工具：请求人类协助"""
    human_response = interrupt({"query": query})
    return human_response["data"]

# 其他工具（如搜索）
search_tool = TavilySearch(max_results=2)
tools = [search_tool, human_assistance]
```

### 3.2 图结构：支持自动工具与人工干预

```python
from langgraph.graph import StateGraph, START
from langgraph.prebuilt import ToolNode, tools_condition

graph_builder = StateGraph(State)

# LLM 节点（绑定工具）
llm_with_tools = llm.bind_tools(tools)
def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    assert len(message.tool_calls) &lt;= 1  # 限制单次调用一个工具
    return {"messages": [message]}

graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", ToolNode(tools=tools))

# 控制流
graph_builder.add_edge(START, "chatbot")
graph_builder.add_conditional_edges("chatbot", tools_condition)
graph_builder.add_edge("tools", "chatbot")
```

> ⚠️ **注意**：`assert len(message.tool_calls) <= 1` 是为简化演示。生产环境可支持多工具并行或串行调用。

### 3.3 启用持久化记忆（支持跨中断会话）

```python
from langgraph.checkpoint.memory import InMemorySaver

memory = InMemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

* 中断期间状态自动保存，恢复时从断点继续。

***

## 🖼️ 四、执行流程：从请求到人工干预再到完成

### 4.1 用户发起请求（触发人工协助）

```python
user_input = "I need expert guidance for building an AI agent."
config = {"configurable": {"thread_id": "1"}}

events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    config,
    stream_mode="values"
)
# → LLM 调用 human_assistance 工具 → 执行中断
```

### 4.2 检查中断状态

```python
snapshot = graph.get_state(config)
print(snapshot.next)  # 输出: ('tools',) 
# → 表示下一步需执行 tools 节点（即等待人工输入）
```

### 4.3 注入人工响应并恢复

```python
human_response = "Use LangGraph—it's reliable and extensible."
human_command = Command(resume={"data": human_response})

events = graph.stream(human_command, config, stream_mode="values")
for event in events:
    event["messages"][-1].pretty_print()  # 输出最终回答
```

> ✅ **流程闭环**：
> 用户请求 → LLM 决策需人工 → 中断 → 人工输入 → 恢复 → 生成最终答案

***

## 📊 五、人机协动 vs 全自动智能体对比

| 能力 | 全自动智能体 | 人机协动智能体 |
|------|---------------|------------------|
| **可控性** | 低（黑盒执行） | 高（随时中断干预） |
| **可靠性** | 依赖 LLM 能力 | 人类兜底，错误率↓ |
| **适用场景** | 低风险、标准化任务 | 高风险、专业领域、合规场景 |
| **审计追踪** | 困难 | 完整记录人工介入点 |
| **开发调试** | 依赖日志回溯 | 实时交互式调试 |

***

## 🎯 六、典型应用场景

1. **金融风控**：大额转账前请求人工复核。
2. **医疗咨询**：遇到罕见病时转接医生。
3. **代码生成**：生成关键模块前请求开发者确认。
4. **内容审核**：敏感内容交由审核员判断。
5. **AI 教学**：学生请求“提示”时暂停并提供引导。

***

## 🛠️ 七、工程最佳实践

### ✅ 推荐做法

* **明确中断触发条件**：仅在高不确定性或高风险时调用 `human_assistance`。
* **结构化传递上下文**：`interrupt({"query": ..., "context": ..., "risk_level": ...})`。
* **设置超时恢复机制**：避免人工长时间未响应导致流程卡死（需自定义调度器）。
* **记录干预日志**：用于后续分析与模型迭代。

### ❌ 避免误区

* **误区1**：“人工干预 = 降低自动化”\
  → 实际是**提升系统整体可靠性**，实现“自动化为主，人工兜底”。

* **误区2**：“中断后状态丢失”\
  → LangGraph **自动持久化完整状态**，恢复后上下文无缝衔接。


***

## 📎 附录：完整代码结构（优化版）

```python
# 1. 初始化
llm = ChatDeepSeek(model="deepseek-chat", api_key=os.getenv("DEEPSEEK_API_KEY"))
search_tool = TavilySearch(max_results=2)

# 2. 定义人工协助工具
@tool
def human_assistance(query: str) -&gt; str:
    response = interrupt({"query": query})
    return response["data"]

tools = [search_tool, human_assistance]

# 3. 构建图
class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    msg = llm_with_tools.invoke(state["messages"])
    assert len(msg.tool_calls) &lt;= 1
    return {"messages": [msg]}

graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", ToolNode(tools=tools))
graph_builder.set_entry_point("chatbot")
graph_builder.add_conditional_edges("chatbot", tools_condition)
graph_builder.add_edge("tools", "chatbot")

# 4. 编译（启用记忆）
graph = graph_builder.compile(checkpointer=InMemorySaver())

# 5. 执行与干预
config = {"configurable": {"thread_id": "1"}}
graph.stream({"messages": [{"role": "user", "content": "Need expert help!"}]}, config)

# 6. 恢复（人工输入后）
human_cmd = Command(resume={"data": "Use LangGraph."})
graph.stream(human_cmd, config)
```

