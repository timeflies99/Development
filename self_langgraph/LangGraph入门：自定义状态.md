
# 🧠 LangGraph 入门：通过自定义状态实现结构化智能体控制

> **核心思想**：\
> 消息列表（`messages`）只是状态的“通用容器”，而**自定义状态字段**才是实现**结构化决策、精准控制与高效协作**的关键。LangGraph 允许你将业务逻辑直接编码到状态中，使智能体从“会聊天”升级为“懂任务”。

***

## 📌 一、为什么需要自定义状态？

默认的 `State = { "messages": [...] }` 虽然通用，但在复杂场景中存在明显局限：

1. **信息隐式**：关键数据（如用户姓名、任务目标）埋藏在对话历史中，需反复解析。
2. **更新低效**：每次修改需重写整个消息列表。
3. **协作困难**：下游节点难以直接访问特定业务字段。

> 💡 **类比理解**：\
> 默认状态 ≈ 用聊天记录管理项目进度\
> 自定义状态 ≈ 用结构化数据库（含 `project_name`, `deadline`, `status` 字段）管理项目

***

## 🧱 二、自定义状态：定义结构化数据模型

### 2.1 扩展状态类型

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]  # 保留对话历史
    name: str                                # 新增：实体名称
    birthday: str                            # 新增：关键属性
```

* **`name` / `birthday`**：显式字段，直接反映业务语义。
* **类型安全**：`TypedDict` 提供 IDE 自动补全与静态检查。

> ✅ **设计原则**：\
> 状态应包含**所有节点需要共享的、可变的业务数据**，而非仅对话文本。

***

## 🔧 三、在工具中更新状态：使用 `Command`

### 3.1 问题：传统工具无法直接修改状态

普通工具返回字符串，只能追加到 `messages`，无法更新 `name` 或 `birthday`。

### 3.2 解决方案：返回 `Command` 对象

```python
from langgraph.types import Command, interrupt
from langchain_core.messages import ToolMessage
from langchain_core.tools import InjectedToolCallId, tool

@tool
def human_assistance(
    name: str,
    birthday: str,
    tool_call_id: Annotated[str, InjectedToolCallId]
) -&gt; Command:
    """请求人工审核，并返回结构化状态更新"""
    # 中断执行，等待人工输入
    human_response = interrupt({
        "question": "Is this correct?",
        "name": name,
        "birthday": birthday,
    })

    # 解析人工反馈
    if human_response.get("correct", "").lower().startswith("y"):
        verified_name, verified_birthday = name, birthday
        response_text = "Correct"
    else:
        verified_name = human_response.get("name", name)
        verified_birthday = human_response.get("birthday", birthday)
        response_text = f"Correction applied"

    # 构造状态更新指令
    state_update = {
        "name": verified_name,
        "birthday": verified_birthday,
        "messages": [ToolMessage(response_text, tool_call_id=tool_call_id)]
    }
    return Command(update=state_update)  # 👈 关键：返回 Command
```

> 🔍 **关键机制解析**：
>
> * **`Command(update=...)`**：显式声明要更新的状态字段。
> * **`InjectedToolCallId`**：自动注入工具调用 ID，用于构造合规的 `ToolMessage`。
> * **`interrupt()`**：暂停执行，将上下文传递给人工操作员。

***

## 🔄 四、完整工作流：从查询到人工审核再到状态固化

### 4.1 用户发起请求

```python
user_input = (
    "Can you look up when LangGraph was released? "
    "When you have the answer, use the human_assistance tool for review."
)
config = {"configurable": {"thread_id": "1"}}

# 启动图执行
events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    config,
    stream_mode="values"
)
```

→ LLM 调用搜索工具 → 获取结果 → 调用 `human_assistance(name="LangGraph", birthday="Jan 17, 2024")` → **执行中断**

### 4.2 人工审核并恢复

```python
# 人工确认信息正确
human_command = Command(resume={
    "name": "LangGraph",
    "birthday": "Jan 17, 2024",
})

# 恢复执行
events = graph.stream(human_command, config, stream_mode="values")
```

→ `human_assistance` 工具返回 `Command(update={...})` → 状态被更新 → LLM 生成最终回答

### 4.3 验证状态

```python
snapshot = graph.get_state(config)
print({k: v for k, v in snapshot.values.items() if k in ("name", "birthday")})
# 输出: {'name': 'LangGraph', 'birthday': 'Jan 17, 2024'}
```

***

## 🛠️ 五、手动更新状态：`graph.update_state()`

LangGraph 还支持**外部直接修改状态**，适用于调试或外部系统干预：

```python
# 手动修正名称
graph.update_state(config, {"name": "LangGraph (library)"})

# 验证更新
snapshot = graph.get_state(config)
print(snapshot.values["name"])  # 输出: "LangGraph (library)"
```

> ⚠️ **注意**：\
> `update_state()` 会**覆盖指定字段**，不影响其他字段（如 `messages` 保持不变）。

***

## 📊 六、状态管理策略对比

| 策略 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| **仅 `messages`** | 简单对话 | 实现简单 | 信息隐式，难维护 |
| **自定义状态 + `Command`** | 结构化任务（如表单填写、数据审核） | 字段显式，更新精准，协作高效 | 需定义状态模型 |
| **`update_state()`** | 调试、外部系统干预 | 灵活直接 | 绕过业务逻辑，慎用于生产 |

***

## 🌟 七、核心思想总结

> **LangGraph 的状态不仅是“记忆”，更是“控制平面”。通过自定义状态字段，你可以将智能体从“被动响应者”转变为“主动任务管理者”，实现：**
>
> * **数据与逻辑解耦**：状态承载数据，节点承载逻辑
> * **精准状态控制**：只更新必要字段，避免冗余
> * **人机无缝协作**：人工审核结果直接写入结构化状态
> * **下游高效消费**：后续节点可直接读取 `state["name"]`，无需解析文本

***

## 📎 附录：完整代码结构（优化版）

```python
# 1. 定义带自定义字段的状态
class State(TypedDict):
    messages: Annotated[list, add_messages]
    name: str
    birthday: str

# 2. 实现带状态更新的工具
@tool
def human_assistance(name: str, birthday: str, tool_call_id: Annotated[str, InjectedToolCallId]) -&gt; Command:
    human_response = interrupt({"name": name, "birthday": birthday})
    # ... 解析反馈 ...
    return Command(update={
        "name": verified_name,
        "birthday": verified_birthday,
        "messages": [ToolMessage("Done", tool_call_id=tool_call_id)]
    })

# 3. 构建图（略，同前文）
graph = graph_builder.compile(checkpointer=InMemorySaver())

# 4. 执行与干预
graph.stream({"messages": [user_msg]}, config)  # 触发中断
graph.stream(Command(resume={...}), config)     # 恢复并更新状态

# 5. 手动修正（可选）
graph.update_state(config, {"name": "Corrected Name"})
```
