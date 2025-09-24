
# 🧠 LangGraph 入门：从零构建最简智能体（Agent）

> **核心思想**：\
> LangGraph 的本质是一个**状态驱动的有向图执行引擎**。通过将 LLM 封装为图中的一个节点，我们可以构建出具备**可扩展性、可观测性与可组合性**的基础智能体，为后续集成工具、记忆、人机协作等高级能力奠定基石。

***

## 📌 一、为什么从“最简智能体”开始？

许多教程直接跳入“带工具的智能体”或“多智能体系统”，但忽略了最核心的抽象：

> **智能体 = 状态（State） + 决策函数（Node） + 控制流（Edge）**

LangGraph 的设计哲学正是如此：**一切皆图（Everything is a Graph）**。\
掌握最简形式，才能理解其扩展机制的底层逻辑。

***

## 🧱 二、核心组件解析

### 2.1 状态（State）：智能体的记忆载体

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]
```

* **`messages`**：对话历史列表，遵循 [LangChain Message 协议](https://python.langchain.com/docs/concepts/messages)。
* **`add_messages`**：一个 reducer 函数，确保新消息**追加到列表末尾**，而非覆盖。这是实现“有状态对话”的关键。

> 💡 **设计哲学**：\
> 状态是**唯一真相源（Single Source of Truth）**。所有节点通过读写状态进行协作，避免隐式全局变量。

***

### 2.2 节点（Node）：智能体的决策单元

```python
def chatbot_node(state: State) -&gt; dict:
    """LLM 决策节点：接收历史消息，生成 AI 回复"""
    response = llm.invoke(state["messages"])
    return {"messages": [response]}
```

* 节点是一个**纯函数**：输入状态 → 输出状态更新。
* 返回值为 `dict`，其键必须与 `State` 定义一致（此处为 `"messages"`）。
* `llm.invoke()` 自动处理消息格式转换，返回 `AIMessage` 对象。

> ⚙️ **工程提示**：\
> 节点函数应**无副作用**（不修改外部状态），便于测试、重放与调试。

***

### 2.3 边（Edge）：定义执行流程

```python
from langgraph.graph import StateGraph, START, END

graph_builder = StateGraph(State)
graph_builder.add_node("chatbot", chatbot_node)
graph_builder.add_edge(START, "chatbot")  # 启动后进入 chatbot 节点
graph_builder.add_edge("chatbot", END)    # 执行完毕后结束
```

* **`START` / `END`**：内置的起始与终止节点。
* **`add_edge(src, dst)`**：定义无条件跳转。

> 🔍 **对比传统流程**：\
> 传统脚本：`input → llm(input) → output`\
> LangGraph：`START → [State: input] → chatbot(node) → [State: input+output] → END`

***

## 🔧 三、完整构建与执行流程

### 3.1 初始化 LLM

```python
import os
from langchain_deepseek import ChatDeepSeek

llm = ChatDeepSeek(
    model="deepseek-chat",
    api_key=os.getenv("DEEPSEEK_API_KEY")
)
```

> ✅ **最佳实践**：使用环境变量管理密钥，避免硬编码。

### 3.2 编译图：生成可执行对象

```python
graph = graph_builder.compile()
```

* `compile()` 将图定义转换为可调用的执行器。
* 支持后续扩展：添加检查点（`checkpointer`）、中断（`interrupt`）等。

### 3.3 执行：流式获取结果

```python
def stream_graph_updates(user_input: str):
    events = graph.stream({"messages": [{"role": "user", "content": user_input}]})
    for event in events:
        for value in event.values():
            print("Assistant:", value["messages"][-1].content)

stream_graph_updates("What is LangGraph?")
```

* **`stream()`**：返回生成器，支持**流式输出**（适用于 Web 实时响应）。
* **`invoke()`**：返回最终状态（适用于批处理）。

***

## 🖼️ 四、可视化代码（Jupyter 中）

```python
from IPython.display import Image
Image(graph.get_graph().draw_mermaid_png())
```
***

## 📊 五、与传统 LLM 调用的对比

| 维度 | 传统 LLM 调用 | LangGraph 最简智能体 |
|------|----------------|------------------------|
| **状态管理** | 手动维护历史 | 自动通过 `State` 管理 |
| **扩展性** | 难以插入中间逻辑 | 轻松添加新节点/边 |
| **可观测性** | 黑盒 | 每一步状态可追踪 |
| **复用性** | 逻辑耦合 | 节点可独立测试/组合 |
| **未来演进** | 需重构 | 无缝升级为工具/记忆/人机协动智能体 |

***

## 🌟 六、为何这是“终极基础”？

这个最简智能体虽只有 **3 个组件（State + Node + Edge）**，却具备了所有高级智能体的基因：

1. **可扩展为工具调用智能体**：\
   → 添加 `tools` 节点 + 条件边（`tools_condition`）

2. **可扩展为记忆智能体**：\
   → 编译时传入 `checkpointer`

3. **可扩展为人机协动智能体**：\
   → 在节点中调用 `interrupt()`

> ✅ **一句话总结**：\
> **LangGraph 的强大，不在于它提供了多少功能，而在于它提供了一个清晰、一致、可组合的抽象，让复杂智能体的构建变得像搭积木一样简单。**

***

## ⚠️ 七、常见误区与最佳实践

### ✅ 推荐做法

* **始终使用 `TypedDict` 定义 State**：提升类型安全与可读性。
* **节点函数保持无状态**：仅通过参数接收状态，通过返回值更新状态。
* **使用 `stream()` 而非 `invoke()`**：为未来流式 UI 做准备。

### ❌ 避免误区

* **误区1**：“LangGraph 只适合复杂场景”\
  → 即使最简单对话，使用 LangGraph 也能获得状态管理与可观测性红利。

* **误区2**：“必须用 `add_messages`”\
  → `add_messages` 仅适用于消息列表。若状态是计数器、字典等，可自定义 reducer。

***

## 📎 附录：完整可运行代码（优化版）

```python
# 1. 初始化
import os
from langchain_deepseek import ChatDeepSeek
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

llm = ChatDeepSeek(model="deepseek-chat", api_key=os.getenv("DEEPSEEK_API_KEY"))

# 2. 定义状态
class State(TypedDict):
    messages: Annotated[list, add_messages]

# 3. 定义节点
def chatbot_node(state: State) -&gt; dict:
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

# 4. 构建图
graph_builder = StateGraph(State)
graph_builder.add_node("chatbot", chatbot_node)
graph_builder.add_edge(START, "chatbot")
graph_builder.add_edge("chatbot", END)

# 5. 编译与执行
graph = graph_builder.compile()

def run_agent(user_input: str):
    events = graph.stream({"messages": [{"role": "user", "content": user_input}]})
    for event in events:
        for value in event.values():
            print("Assistant:", value["messages"][-1].content)

run_agent("Explain LangGraph in one sentence.")
```
