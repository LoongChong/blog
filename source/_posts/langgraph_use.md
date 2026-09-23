# Types

## Send

```python
def continue_to_jokes(state: OverallState):
    # 表示下一个节点是generate_joke，发送的消息是批量的state["subjects"]里的内容
	return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]
```

> Send(node,arg,timeout)
> 	node：要将消息发送到的目标节点的名称
> 	arg：发送到目标节点的状态或者消息
>         timeout：可以选择超市策略，若省略则默认使用目标节点的超时策略

Send在State Graph的条件边中使用，动态的调用节点

```python
builder.add_conditional_edges(START, continue_to_jokes)
```

## Command

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # 状态更新
        update={"foo": "bar"},
        # 控制流
        goto="my_other_node"
    )
```

> Command(graph,update,resume,goto)
>
>  - graph：要将命令发送的图表，支持的值为：
>    - None（当前图表）
>    - Command.PARENT(最接近的父图)
>  - update：更新state
>  - resume：恢复执行的值。与 [interrupt() ][langgraph.types.interrupt] 一起使用。支持的值为：
>    - 中断id到恢复值的映射
>    - 用于恢复下一个中断的单个值
>  - goto：支持的值为：
>    - 要导航到下一个节点的名称
>    - 导航到下一个的节点名称序列
>    - Send对象
>    - Send对象序列
>
> ​		

## Interrupt

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import MemorySaver

# 1. 定义状态
class State(TypedDict):
    number: int
    user_reply: str

# 2. 节点：生成数字后暂停，等人回复
def generate(state: State):
    print("生成数字：42")
    return {"number": 42}

# 3. 节点：这里是触发中断的地方
def ask_user(state: State):
    # ↓ 执行到这里，图会暂停，把问题返回给调用方
    #   user_reply 拿到的值 = 恢复时 Command(resume=xxx) 里的 xxx
    user_reply = interrupt(f"数字 {state['number']} 可以吗？")
    
    # ↓ 这行只有恢复后才会执行
    print(f"用户回复：{user_reply}")
    return {"user_reply": user_reply}

# 4. 节点：结束
def done(state: State):
    print(f"最终结果：数字={state['number']}, 用户说={state['user_reply']}")
    return {}

# 5. 组装图
builder = StateGraph(State)
builder.add_node("generate", generate)
builder.add_node("ask_user", ask_user)
builder.add_node("done", done)

builder.add_edge(START, "generate")
builder.add_edge("generate", "ask_user")
builder.add_edge("ask_user", "done")
builder.add_edge("done", END)

# 6. 必须加 Checkpointer，否则中断状态没地方存
graph = builder.compile(checkpointer=MemorySaver())

# ================== 第一次调用：会卡在 interrupt ==================
config = {"configurable": {"thread_id": "t1"}}

print("--- 第一次 invoke ---")
result = graph.invoke({"number": 0}, config)
print("返回值：", result)
print()

# ================== 第二次调用：恢复执行 ==================
print("--- 第二次 invoke（恢复）---")
result = graph.invoke(Command(resume="可以，很棒！"), config)
print("返回值：", result)
```

