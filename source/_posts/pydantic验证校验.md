---
title: pydantic验证校验
categories:
  - Agent开发
description: 一句话说清这篇文章解决什么问题
comments: true
toc: true
mathjax: false
copyright: true
date: 2026-09-04 20:51:15
updated:
tags:
keywords:
top_img:
cover: /img/lachang.png
---

<!-- more -->

## 在 **Agent 应用开发** 中，Pydantic 远不止"校验 JSON"这么简单

---

### 1. `Field()` 的完整用法（核心中的核心）

Agent 开发中，字段的 `description` 就是给大模型看的"说明书"。

```python
from pydantic import BaseModel, Field
from typing import Optional, Literal

class SearchTool(BaseModel):
    query: str = Field(
        description="搜索关键词，需要包含核心实体名",
        min_length=1,
        max_length=100
    )
    source: Literal["news", "academic", "web"] = Field(
        default="web",
        description="搜索来源类型"
    )
    top_k: int = Field(default=5, ge=1, le=20)
```

**重点参数：**

- `description`：大模型靠这个理解字段含义
- `default` / `default_factory`：让字段可选，降低 LLM 输出压力
- `ge` / `le` / `min_length` / `max_length`：数值/长度约束
- `examples`：给模型提供输出示例（OpenAI 等支持）

---

### 2. `model_json_schema()` — 把模型变成 LLM 的"指令"

这是 Agent 框架（LangChain、OpenAI SDK 等）的底层原理：

```python
schema = SearchTool.model_json_schema()
print(schema)
# 输出标准 JSON Schema，直接传给 LLM 的 function calling / structured output
```

**关键理解：** 你定义的 Pydantic 模型，最终会被转换成 JSON Schema 塞进 LLM 的 system prompt 里，告诉它"必须按这个格式输出"。

---

### 3. 嵌套模型 — 处理复杂 Agent 输出

Agent 的输出往往是多层结构，比如"思考过程 + 最终回答 + 工具调用"：

```python
from typing import List

class ToolCall(BaseModel):
    tool_name: str
    parameters: dict

class AgentOutput(BaseModel):
    thought: str = Field(description="逐步思考过程")
    tool_calls: List[ToolCall] = Field(default_factory=list)
    final_answer: Optional[str] = Field(default=None)
```

---

### 4. 自定义校验器 — `field_validator` 和 `model_validator`

LLM 输出经常"看起来对但逻辑错"，需要业务层校验：

```python
from pydantic import field_validator, model_validator

class DateRange(BaseModel):
    start: str
    end: str

    @field_validator('end')
    @classmethod
    def end_after_start(cls, v: str, info):
        if v < info.data['start']:
            raise ValueError('结束日期必须晚于开始日期')
        return v

    @model_validator(mode='after')
    def check_range(self):
        if self.start == self.end:
            raise ValueError('起止日期不能相同')
        return self
```

- **`field_validator`**：单个字段清洗（如去除空格、统一大小写）
- **`model_validator`**：跨字段逻辑校验

---

### 5. `Optional` 和 `Union` — 应对 LLM 的不稳定输出

LLM 有时会漏字段、会输出 `null`，必须显式声明可空性：

```python
from typing import Optional, Union

class LLMResponse(BaseModel):
    # 可能返回字符串，也可能返回 null
    summary: Optional[str] = Field(default=None)

    # 可能是整数也可能是字符串（LLM 经常搞混数字类型）
    confidence: Union[int, float] = Field(default=0.0)
```

---

### 6. 序列化与反序列化 — 不只是 `model_validate_json`

```python
# JSON 字符串 → 模型
obj = SearchTool.model_validate_json(json_str)

# 模型 → Python 字典（传给下游工具/函数）
data = obj.model_dump()

# 模型 → JSON 字符串（存日志、发请求）
json_str = obj.model_dump_json(indent=2)

# 包含默认值和 None 的字段控制
data = obj.model_dump(exclude_none=True, exclude_defaults=True)
```

**`exclude_none=True`** 特别常用——LLM 输出里大量 `null` 字段，清理后更干净。

---

### 7. `Literal` 和 `Enum` — 限制 LLM 的选择范围

这是控制 LLM 输出的利器，比写 "请从以下选项中选择" 有效得多：

```python
from typing import Literal
from enum import Enum

class ActionType(str, Enum):
    SEARCH = "search"
    CALCULATE = "calculate"
    ANSWER = "answer"

class AgentStep(BaseModel):
    action: ActionType  # 或 Literal["search", "calculate", "answer"]
    reasoning: str
```

---

### 8. 错误处理 — 优雅接住 `ValidationError`

LLM 输出格式错误是常态，必须处理：

```python
from pydantic import ValidationError

try:
    result = AgentOutput.model_validate_json(raw_json)
except ValidationError as e:
    # e.errors() 返回结构化错误信息，可以反馈给 LLM 让它重试
    errors = e.errors()
    print(errors)
    # 例如：反馈给 LLM "你输出的 JSON 中缺少必填字段 'thought'，请修正后重试"
```

---

### 9. `ConfigDict` / `model_config` — 控制模型行为

```python
from pydantic import ConfigDict

class AgentOutput(BaseModel):
    model_config = ConfigDict(
        strict=False,        # 允许类型自动转换（如 "5" → 5）
        str_strip_whitespace=True,  # 自动去除字符串首尾空格
        validate_assignment=True    # 赋值时也校验
    )
    content: str
```

---

### 10. 与主流 Agent 框架的集成模式

| 框架             | Pydantic 用法                                           |
| -------------- | ----------------------------------------------------- |
| **OpenAI SDK** | `response_format=YourModel`（Beta 版 structured output） |
| **LangChain**  | `@tool` 装饰器自动从 Pydantic 模型生成函数定义                      |
| **CrewAI**     | 任务输出直接绑定 Pydantic 模型做结构化                              |
| **AutoGen**    | 工具函数的参数模型用 Pydantic 定义                                |

---

### 建议的学习路径

1. **先熟练** `BaseModel` + `Field()` + `model_json_schema()`
2. **再掌握** 嵌套模型 + `Optional`/`Union` + `model_dump()`
3. **然后学** `field_validator` / `model_validator` 处理业务逻辑
4. **最后看** 与具体框架（LangChain / OpenAI SDK）的集成源码

如果你正在用某个具体的 Agent 框架（比如 LangChain、CrewAI、AutoGen 或原生 OpenAI API），我可以结合那个框架给你更具体的 Pydantic 实战示例。
