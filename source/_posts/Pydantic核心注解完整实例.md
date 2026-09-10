# Pydantic核心注解实例

## 注解速查表

| 注解                                | 执行时机             | 能拿到什么              | 典型用途                                                      |
| --------------------------------- | ---------------- | ------------------ | --------------------------------------------------------- |
| `@field_validator`                | 字段解析时            | 单个字段的原始值 `v`       | 清洗输入（去空格、转小写）、单字段格式校验                                     |
| `@model_validator(mode='before')` | 所有字段解析**之前**     | 原始 `dict` 数据       | 兼容 LLM 不规范输出（如 JSON 字符串转 dict、百分比转小数）                     |
| `@model_validator(mode='after')`  | 所有字段解析**之后**     | 完整的实例 `self`       | 跨字段逻辑校验（如 status=success 时 confidence 必须够高）               |
| `@computed_field` + `@property`   | 访问/序列化时          | 实例 `self`          | 派生字段（如 `is_high_confidence`、`summary`），自动进 `model_dump()` |
| `@field_serializer`               | `model_dump()` 时 | 字段值                | 自定义单个字段的输出格式（如 `datetime` → 字符串、小数 → 百分比）                 |
| `@model_serializer`               | `model_dump()` 时 | 整个实例 + `handler()` | 完全控制输出结构（如分组、加版本号、过滤字段）                                   |
| `@validate_call`                  | 函数被调用时           | 函数参数               | 给普通工具函数加参数类型和约束校验，不用写一堆 `if`                              |

```python
"""
Pydantic v2 核心注解完整实例
场景：Agent 工具调用结果的结构化模型
"""

from pydantic import (
    BaseModel, Field, field_validator, model_validator,
    computed_field, field_serializer, model_serializer, validate_call
)
from datetime import datetime
from typing import Literal, Optional
import json


# ============================================================
# 场景：Agent 工具调用结果的结构化模型
# ============================================================

class ToolResult(BaseModel):
    """Agent 执行工具后的结构化结果"""

    name: str = Field(description="工具名称")
    status: Literal["success", "failed", "pending"] = Field(default="pending")
    confidence: float = Field(default=0.5, ge=0, le=1, description="置信度 0~1")
    metadata: dict = Field(default_factory=dict, description="附加信息")
    timestamp: Optional[datetime] = Field(default=None, description="执行时间")
    raw_output: str = Field(default="", description="原始输出文本")

    # --------------------------------------------------------
    # 1. @field_validator — 单字段校验 & 清洗
    # --------------------------------------------------------
    @field_validator("name")
    @classmethod
    def clean_name(cls, v: str) -> str:
        """自动去除首尾空格，并转成小写"""
        cleaned = v.strip().lower()
        if not cleaned:
            raise ValueError("工具名称不能为空")
        if " " in cleaned:
            raise ValueError("工具名称不能包含空格，请用下划线连接")
        return cleaned

    @field_validator("raw_output")
    @classmethod
    def truncate_raw_output(cls, v: str) -> str:
        """原始输出太长时自动截断，防止撑爆上下文"""
        max_len = 200
        if len(v) > max_len:
            return v[:max_len] + f"... [截断，原长度 {len(v)}]"
        return v

    # --------------------------------------------------------
    # 2. @model_validator(mode='before') — 原始数据进来时处理
    # --------------------------------------------------------
    @model_validator(mode='before')
    @classmethod
    def normalize_input(cls, data):
        """
        在字段解析之前执行，data 还是原始 dict。
        常用于：兼容 LLM 的不规范输出、字段别名映射、类型转换。
        """
        if isinstance(data, dict):
            # LLM 有时会把 metadata 写成 JSON 字符串，这里自动解析
            if "metadata" in data and isinstance(data["metadata"], str):
                try:
                    data["metadata"] = json.loads(data["metadata"])
                except json.JSONDecodeError:
                    data["metadata"] = {"error": "invalid json string"}

            # LLM 有时会把 confidence 写成百分比字符串 "95%"
            if "confidence" in data and isinstance(data["confidence"], str):
                val = data["confidence"].replace("%", "").strip()
                try:
                    data["confidence"] = float(val) / 100
                except ValueError:
                    data["confidence"] = 0.5
        return data

    # --------------------------------------------------------
    # 3. @model_validator(mode='after') — 实例创建后跨字段校验
    # --------------------------------------------------------
    @model_validator(mode='after')
    def check_consistency(self):
        """
        所有字段解析完成后执行，可以用 self.xxx 访问字段。
        常用于：跨字段逻辑校验。
        """
        # 规则：如果状态是 success，confidence 必须 >= 0.6
        if self.status == "success" and self.confidence < 0.6:
            raise ValueError(
                f"状态为 success 时，confidence 必须 >= 0.6，当前为 {self.confidence}"
            )

        # 规则：如果状态是 failed，metadata 里必须有 error 字段
        if self.status == "failed" and "error" not in self.metadata:
            raise ValueError("状态为 failed 时，metadata 中必须包含 'error' 字段")

        return self

    # --------------------------------------------------------
    # 4. @computed_field — 计算属性（会出现在序列化输出中）
    # --------------------------------------------------------
    @computed_field
    @property
    def is_high_confidence(self) -> bool:
        """是否为高置信度结果，model_dump() 时会自动包含"""
        return self.confidence >= 0.8

    @computed_field
    @property
    def summary(self) -> str:
        """生成一句话摘要"""
        return f"[{self.status.upper()}] {self.name}: 置信度 {self.confidence:.0%}"

    # --------------------------------------------------------
    # 5. @field_serializer — 自定义单个字段的序列化方式
    # --------------------------------------------------------
    @field_serializer("timestamp")
    def serialize_timestamp(self, timestamp: Optional[datetime]) -> Optional[str]:
        """把 datetime 对象序列化为 ISO 格式字符串"""
        if timestamp is None:
            return None
        return timestamp.strftime("%Y-%m-%d %H:%M:%S")

    @field_serializer("confidence")
    def serialize_confidence(self, confidence: float) -> str:
        """把 confidence 序列化为百分比字符串，更直观"""
        return f"{confidence:.0%}"

    # --------------------------------------------------------
    # 6. @model_serializer — 自定义整个模型的序列化逻辑
    # --------------------------------------------------------
    @model_serializer(mode='wrap')
    def serialize_model(self, handler):
        """
        完全控制 model_dump() 的输出结构。
        handler() 会先执行默认序列化，然后你可以在此基础上修改。
        """
        data = handler(self)
        # 添加一个版本号字段
        data["_schema_version"] = "1.0"
        # 把字段分组，让输出更结构化
        return {
            "tool_info": {
                "name": data["name"],
                "status": data["status"],
            },
            "quality": {
                "confidence": data["confidence"],
                "is_high_confidence": data["is_high_confidence"],
            },
            "details": {
                "metadata": data["metadata"],
                "raw_output": data["raw_output"],
                "timestamp": data["timestamp"],
            },
            "summary": data["summary"],
            "_schema_version": data["_schema_version"],
        }


# ============================================================
# 7. @validate_call — 给普通函数加参数校验
# ============================================================

@validate_call
def execute_tool(
    tool_name: str,
    timeout: int = Field(default=30, ge=1, le=300),
    retry_count: int = Field(default=3, ge=0, le=10),
    mode: Literal["sync", "async"] = "sync"
) -> ToolResult:
    """
    模拟执行一个工具函数。
    @validate_call 会自动校验传入的参数类型和约束，
    就像 Pydantic 模型一样，不用写一堆 if 判断。
    """
    return ToolResult(
        name=tool_name,
        status="success",
        confidence=0.92,
        metadata={"executor": "agent_v1", "duration_ms": 120},
        timestamp=datetime.now(),
        raw_output="搜索结果：Python 3.12 已发布..."
    )


# ==================== 运行示例 ====================

if __name__ == "__main__":
    print("【1】field_validator 自动清洗")
    r1 = ToolResult(name="  Web_SEARCH  ", confidence=0.85)
    print(f"  输入 '  Web_SEARCH  ' -> 输出 '{r1.name}'")

    print("\n【2】model_validator(before) 兼容不规范 LLM 输出")
    raw = {
        "name": "calculator",
        "status": "success",
        "confidence": "95%",
        "metadata": '{"expr": "1+1", "result": 2}',
        "raw_output": "计算结果为 2"
    }
    r2 = ToolResult.model_validate(raw)
    print(r2.model_dump())
    print(f"  confidence: {r2.confidence}, metadata: {r2.metadata}")

    print("\n【3】model_validator(after) 跨字段校验")
    try:
        ToolResult(name="search", status="success", confidence=0.3)
    except Exception as e:
        print(f"  拦截: {e}")

    print("\n【4】computed_field 计算属性")
    r4 = ToolResult(name="search", confidence=0.85)
    print(f"  is_high_confidence: {r4.is_high_confidence}")
    print(f"  summary: {r4.summary}")

    print("\n【5&6】序列化效果")
    r5 = ToolResult(
        name="search",
        status="success",
        confidence=0.85,
        metadata={"query": "pydantic tutorial", "hits": 42},
        timestamp=datetime(2026, 9, 7, 10, 30, 0),
        raw_output="Pydantic 是 Python 数据验证库..."
    )
    print(json.dumps(r5.model_dump(), indent=2, ensure_ascii=False))

    print("\n【7】validate_call 校验函数参数")
    try:
        execute_tool("search", timeout=500)  # 超限，会报错
    except Exception as e:
        print(f"  拦截: {e}")
```

输出如下：

```textile
【1】field_validator 自动清洗
  输入 '  Web_SEARCH  ' -> 输出 'web_search'

【2】model_validator(before) 兼容不规范 LLM 输出
{'tool_info': {'name': 'calculator', 'status': 'success'}, 'quality': {'confidence': '95%', 'is_high_confidence': True}, 'details': {'metadata': {'expr': '1+1', 'result': 2}, 'raw_output': '计算结果为 2', 'timestamp': None}, 'summary': '[SUCCESS] calculator: 置信度 95%', '_schema_version': '1.0'}
  confidence: 0.95, metadata: {'expr': '1+1', 'result': 2}

【3】model_validator(after) 跨字段校验
  拦截: 1 validation error for ToolResult
  Value error, 状态为 success 时，confidence 必须 >= 0.6，当前为 0.3 [type=value_error, input_value={'name': 'search', 'statu...ess', 'confidence': 0.3}, input_type=dict]
    For further information visit https://errors.pydantic.dev/2.13/v/value_error

【4】computed_field 计算属性
  is_high_confidence: True
  summary: [PENDING] search: 置信度 85%

【5&6】序列化效果
{
  "tool_info": {
    "name": "search",
    "status": "success"
  },
  "quality": {
    "confidence": "85%",
    "is_high_confidence": true
  },
  "details": {
    "metadata": {
      "query": "pydantic tutorial",
      "hits": 42
    },
    "raw_output": "Pydantic 是 Python 数据验证库...",
    "timestamp": "2026-09-07 10:30:00"
  },
  "summary": "[SUCCESS] search: 置信度 85%",
  "_schema_version": "1.0"
}

【7】validate_call 校验函数参数
  拦截: 1 validation error for execute_tool
timeout
  Input should be less than or equal to 300 [type=less_than_equal, input_value=500, input_type=int]
    For further information visit https://errors.pydantic.dev/2.13/v/less_than_equal
```
