---
title: Langgraph基础
description: 主要学习langgraph中的最基础的state，nodes，edges
categories:
  - Agent开发
tags: 
  - Agent框架
  - Langgraph
---

# langgraph基础

### Reducer arguments

​	reducer 是理解节点中的更新如何应用于 `State` 的关键。 `State` 中的每个键都拥有自己独立的 reducer 函数。如果没有明确指定 reducer 函数，那么默认情况下，对该键的所有更新都会覆盖原有的 reducer 函数。

```python
from typing import Annotated
from typing_extensions import TypedDict

def append_strings(left: list[str], right: list[str]) -> list[str]:
    """Combine the existing state value (left) with a node update (right)."""
    return left + right
class State(TypedDict):
    # Annotated[参数一，参数二]参数一规定了tags里面的数据类型，参数二规定了有节点返回tags信息的时候tags的更新方式是怎么样的
    tags: Annotated[list[str], append_strings]
```

自定义 reducer 会合并左右两边的参数。默认的 reducer 会丢弃左边的参数，只保留右边的参数（也就是只保存最新节点返回的那个数据，之前的数据会被覆盖掉）。

### Overwrite

> 在某些情况下，之前我们定义的可能是希望能够新节点返回的时候追加，但是某一步骤我们想要全部覆盖，就可以使用overwrite 来处理这次特殊的情况，使用方法如下：

```python
class State(TypedDict):
    messages: Annotated[list, operator.add]

def node_a(state: TypedDict):
    # Normal update: uses the reducer (operator.add)
    return {"messages": ["a"]}

def node_b(state: State):
    # Overwrite: bypasses the reducer and replaces the entire value
    return {"messages": Overwrite(value=["b"])}
```

你也可以使用 JSON 格式，并附加一个特殊的键 `"__overwrite__"` ：

```python
def replace_messages(state: State):
    return {"messages": {"__overwrite__": ["replacement message"]}}
```

### state

​	在定义图结构时，首先要做的就是确定该图的 `State` 。 `State` 包含了图的模式信息，而 `reducer` 则定义了如何对状态进行更新。 `State` 的模式将作为所有 `Nodes` 和 `Edges` 的输入模式，它可以是 `TypedDict` 模型或 `Pydantic` 模型。所有的 `Nodes` 都会向 `State` 发送更新信息，而这些更新信息将通过指定的 `reducer` 函数进行应用。

### nodes

在langgraph中，nodes是一个python函数。接受这些参数：

 - state：graph的状态
 - config：runnableConfig对象（包含配置信息，跟踪数据）
 - runtime：一个Runnable对象，包含运行时数据以及一些其他信息

> 添加node的方式：

```python
class State(TypedDict):
    input: str
    results: str

@dataclass
class Context:
    user_id: str

builder = StateGraph(State)

def plain_node(state: State):
    return state

def node_with_runtime(state: State, runtime: Runtime[Context]):
    print("In node: ", runtime.context.user_id)
    return {"results": f"Hello, {state['input']}!"}

def node_with_execution_info(state: State, runtime: Runtime):
    print("In node with thread_id: ", runtime.execution_info.thread_id)
    return {"results": f"Hello, {state['input']}!"}

# 添加节点并且为这些节点取名，如果没有取名则赋予一个默认的名字
builder.add_node("plain_node", plain_node)
builder.add_node("node_with_runtime", node_with_runtime)
builder.add_node("node_with_execution_info", node_with_execution_info)
...
```

**START Node**是一个特殊的节点，他表示将用户输入传递给图的节点。引入这个节点的目的时确定哪些节点应该优先被调用。

```python
graph.add_edge(START, "node_a")
```

**END Node**也是一个特殊的节点，表示终端节点。当想表示某些边在完成之后不再有操作时，就会引用此节点。

```python
graph.add_edge(START, "node_a")
```

**Node caching**

langgraph支持根据节点的输入数据来缓存任务或者节点。要使用缓存功能，步骤如下：

	- 在编译图结构或指定入口点时，请指定一个缓存对象
 - 为节点指定一个缓存策略。每种缓存策略都支持以下功能
   - `key_func` 通常用于根据节点的输入生成缓存键。默认情况下，该键是通过使用 pickle 序列化方法从输入数据中提取出的 `hash` 数据生成的。
   - `ttl` ，指的是缓存数据的存活时间，单位为秒。如果未指定此值，那么缓存将永远有效，不会过期。

```python
class State(TypedDict):
    x: int
    result: int

builder = StateGraph(State)

def expensive_node(state: State) -> dict[str, int]:
    # expensive computation
    time.sleep(2)
    return {"result": state["x"] * 2}
# 1. 添加一个昂贵节点，并绑定缓存策略（TTL=3秒）
builder.add_node("expensive_node", expensive_node, cache_policy=CachePolicy(ttl=3))
# 2. 设置图的入口和出口都是该节点（单节点图）
# 这两种方法都是有效的，不过推荐使用 add_edge(START, ...) 和 add_edge(..., END) 这种现代语法。
# set_entry_point(node) 定义了图执行的首个节点。它相当于 builder.add_edge(START, node) 。
builder.set_entry_point("expensive_node")
# set_finish_point(node) 定义了图中的最后一个节点。它相当于 builder.add_edge(node, END) 
builder.set_finish_point("expensive_node")
# 3. 编译图时注入内存缓存实例
graph = builder.compile(cache=InMemoryCache())

print(graph.invoke({"x": 5}, stream_mode='updates'))
# [{'expensive_node': {'result': 10}}]
print(graph.invoke({"x": 5}, stream_mode='updates'))
# [{'expensive_node': {'result': 10}, '__metadata__': {'cached': True}}]
```

### Edges

​	边缘决定了逻辑如何被传递，以及图结构如何决定停止的时机。这是代理模块运作方式的重要部分，也涉及到不同节点之间如何相互通信。边缘主要有几种类型：

 - Normal Edges: 直接从一个节点走到下一个节点

   ```python
   # 直接从节点A到节点B，直接使用add_edge()
   graph.add_edge("node_a", "node_b")
   ```

 - Conditional Edges：调用一个函数来确定接下来应该访问哪个节点

   ```python
   # 如果你希望选择性地通往一个或多个边，或者选择性地终止路径，可以使用 add_conditional_edges 方法。该方法接受一个节点的名称，以及在该节点被执行后需要调用的“路由函数”
   def routing_function(state: State):
       if state["count"] > 0:
           return "node_b"
       else:
           return "node_c"
   
   graph.add_conditional_edges("node_a", routing_function)
   ```

   > 与节点类似， `routing_function` 也接受图的当前 `state` 值，并返回一个结果。
   >
   > 默认情况下， `routing_function` 的返回值被用作要发送状态到下一个阶段的节点（或节点列表）的名称。所有这些节点都将作为下一个超级步骤的一部分并行运行。
   >
   > 你可以选择提供一个字典，将 `routing_function` 的输出结果映射到下一个节点的名称上。

   ```python
   graph.add_conditional_edges("node_a", routing_function, {True: "node_b", False: "node_c"})
   ```

 - Entry Point：当用户输入到达时，优先调用哪个节点。

	- Conditional Entry Point：当用户输入时，调用一个函数来确定首先调用哪个节点。

​	一个节点可以拥有多个出边。如果一个节点有多个出边，那么这些目的节点都将作为下一个超级步骤的一部分被并行执行。

```python
graph.add_conditional_edges(START, routing_function)
```

### Send

`Send` 是一个特殊的对象，允许您动态地将状态发送到其他节点，特别适用于 map-reduce 设计模式：

```python
from langgraph.graph import StateGraph, Send
from typing_extensions import TypedDict

class OverallState(TypedDict):
    subjects: list[str]

class JokeState(TypedDict):
    subject: str

def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state['subjects']]

builder = StateGraph(OverallState)
builder.add_conditional_edges("node_a", continue_to_jokes)
```

`Send` 的执行流程就是：**一个节点完成 → 根据状态动态拆分出 N 个并行任务 → 各自独立执行 → 全部完成后自动汇合**。这是 LangGraph 实现 map-reduce 模式的核心机制。

### Command

Command对象允许我们在单个节点中同时执行状态更新和控制流 ：

```python
from langgraph.graph import StateGraph, Command
from typing_extensions import TypedDict, Literal

class State(TypedDict):
    foo: str

def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # 状态更新
        update={"foo": "bar"},
        # 控制流
        goto="my_other_node"
    )

builder = StateGraph(State)
builder.add_node("my_node", my_node)
builder.add_node("my_other_node", lambda state: state)
builder.add_edge("my_other_node", END)
```

### [运行时上下文](https://langchain-doc.cn/v1/python/langgraph/graph-api.html#运行时上下文)

创建图时，您可以为传递给节点的运行时上下文指定 `context_schema`。这对于传递不属于图状态的信息非常有用。例如，您可能想要传递依赖项，如模型名称或数据库连接：

```python
from dataclasses import dataclass
from langgraph.graph import StateGraph

@dataclass
class ContextSchema:
    llm_provider: str = "openai"

graph = StateGraph(State, context_schema=ContextSchema)

# 使用 context 参数将上下文传递到图中
graph.invoke(inputs, context={"llm_provider": "anthropic"})
```

### [递归限制](https://langchain-doc.cn/v1/python/langgraph/graph-api.html#递归限制)

递归限制设置了图在单次执行期间可以执行的最大 super-step 数。一旦达到限制，LangGraph 将抛出 `GraphRecursionError`。默认值设置为 25 步。递归限制可以在运行时设置在任何图上，并通过配置字典传递给 `invoke`/`stream`。重要的是，`recursion_limit` 是一个独立的 `config` 键，不应像所有其他用户定义的配置一样传递到 `configurable` 键内部:

```python
graph.invoke(inputs, config={"recursion_limit": 5}, context={"llm": "anthropic"})
```

限制 `recursion_limit` 的核心目的是**防止图执行陷入无限循环或失控膨胀，导致资源耗尽**。它本质上是一个安全熔断机制。

以下是会导致超限的典型场景：

#### **1. 条件边形成死循环（最常见）**

当路由逻辑存在缺陷，导致节点之间互相跳转且没有正确的终止条件时：

```
# ❌ 危险：缺少退出分支
def router(state):
    if state["score"] < 0.9:
        return "refine"      # 永远返回 refine
    # 忘记写 else: return "end"

# A → refine → A → refine → ... (每轮消耗2个super-step)
# 约12-13轮后触发 GraphRecursionError
```

#### **2. Agent 工具调用无收敛**

ReAct 类 Agent 中，LLM 反复调用工具但始终无法得出最终答案：

```
LLM → call_tool → LLM → call_tool → LLM → call_tool → ...
```

- LLM 产生幻觉，不断调用不相关的工具
- 工具返回错误，LLM 没有正确处理而是重试
- Prompt 中没有设置最大迭代次数的指令

#### **3. Send 扇出失控**

动态并行时，输入数据异常导致生成过多任务：

```
# ❌ 如果 subjects 被污染为超大列表
[Send("generate_joke", {"subject": s}) for s in state['subjects']]
# subjects = ["a"] * 10000 → 虽然只占1个super-step
# 但下一轮汇聚后的处理可能因状态过大而触发连锁问题
```

> ⚠️ 注意：Send 本身不会直接耗尽 super-step（N个并行=1步），但如果每个 Send 实例内部又触发了子循环，就会快速累积。

#### **4. 递归式图嵌套**

图中某个节点再次 invoke 同一个图或形成跨图循环：

```
outer_graph.node_A → inner_graph → outer_graph.node_A → ...
```

#### **🛡️ 为什么默认是 25？**

| 考量                 | 说明                                                  |
| :------------------- | :---------------------------------------------------- |
| **足够覆盖正常流程** | 大多数 Agent/工作流在 10-20 步内完成                  |
| **快速失败**         | 死循环通常在 5-10 步内就能暴露，25 留有余量           |
| **资源保护**         | 每步都可能包含 LLM 调用，25 步 ≈ 控制单次请求成本上限 |
| **调试友好**         | 报错时回溯路径短，容易定位问题                        |
