---
title: MCP 协议入门：为什么它是 Agent 的 USB-C
date: 2026-09-04 16:00:00
updated:
categories:
  - Agent 开发
tags:
  - MCP
  - Agent
  - 大模型
cover: /img/default-cover.jpg
description: 用 USB-C 的类比讲清楚 MCP（Model Context Protocol）解决什么问题、三层角色怎么分工、以及一个最小可用的 MCP Server 长什么样。
keywords: MCP, Model Context Protocol, Agent, Function Calling, 工具调用
top_img:
comments: true
toc: true
mathjax: false
copyright: true
---

<!-- more -->

## 背景

2024 年底 Anthropic 开源了 MCP（Model Context Protocol），当时很多人觉得"不就是另一个 Function Calling 规范吗"。

用了一段时间之后我的理解是：**它解决的不是"模型怎么调工具"，而是"工具怎么被发现"**。这两件事差别很大。

## 没有 MCP 的世界：M×N 问题

假设你有 3 个应用（Claude Desktop、Cursor、自己写的 Agent）和 4 个数据源（本地文件、数据库、GitHub、Slack）。

没有统一协议时，每个应用要为每个数据源单独写适配：

```
Claude Desktop  →  文件适配器、DB 适配器、GitHub 适配器、Slack 适配器
Cursor          →  文件适配器、DB 适配器、GitHub 适配器、Slack 适配器
自己的 Agent     →  文件适配器、DB 适配器、GitHub 适配器、Slack 适配器
```

**M 个应用 × N 个数据源 = M×N 套适配代码。**

每加一个数据源，所有应用都得改。这就是所谓的集成碎片化。

## MCP 的解法：M + N

MCP 在中间插了一层标准协议：

```
应用（MCP Client）  ←—— 统一协议 ——→  MCP Server  ←→  数据源
```

- 数据源提供者只写**一个** MCP Server
- 应用只实现**一次** MCP Client
- 复杂度从 M×N 降到 **M + N**

这和 USB-C 解决的问题是同一个形状：在 USB-C 之前，每种外设都要自己的接口和驱动；有了统一标准，设备厂商只管做 USB-C 设备，电脑厂商只管留 USB-C 口。

**所以 "Agent 的 USB-C" 这个说法不只是营销话术，它准确描述了标准化接口带来的组合爆炸抑制。**

## 三层角色

MCP 定义了三个角色，容易混淆，按"谁做什么"来记：

| 角色 | 是什么 | 举例 |
| --- | --- | --- |
| **Host** | 跑模型的宿主应用，负责编排 | Claude Desktop、Cursor、你自己写的 Agent |
| **Client** | Host 内部的连接器，1 个 Server 对应 1 个 Client | 由 Host 实现，你一般不用管 |
| **Server** | 暴露能力的独立进程 | 文件服务、数据库服务、GitHub 服务 |

一个 Host 里可以同时跑多个 Client，每个 Client 连一个 Server。

## Server 能暴露的三类能力

这是 MCP 最实用的部分，能力分成三种：

**1. Tools（工具）** —— 模型可以调用的函数，会产生副作用

```json
{
  "name": "query_database",
  "description": "执行只读 SQL 查询",
  "inputSchema": {
    "type": "object",
    "properties": {
      "sql": { "type": "string", "description": "要执行的 SQL" }
    },
    "required": ["sql"]
  }
}
```

**2. Resources（资源）** —— 可以被读取的数据，类似 GET 请求，只读

```
file:///home/user/report.pdf
postgres://db/customers/schema
```

**3. Prompts（提示词模板）** —— 预置的提示词，用户可以主动触发

区分 Tools 和 Resources 的关键：**Tools 由模型决定何时调用，Resources 由应用决定何时加载，Prompts 由用户决定何时使用。**

## 一个最小可用的 MCP Server

用 Python SDK 写一个暴露"查询本地文件列表"的 Server：

```python
from mcp.server.fastmcp import FastMCP
import os

mcp = FastMCP("local-files")

@mcp.tool()
def list_files(directory: str) -> list[str]:
    """列出指定目录下的所有文件名"""
    try:
        return os.listdir(directory)
    except FileNotFoundError:
        return [f"目录不存在: {directory}"]

@mcp.resource("file://readme")
def read_readme() -> str:
    """暴露项目 README 的内容"""
    with open("README.md", encoding="utf-8") as f:
        return f.read()

if __name__ == "__main__":
    mcp.run()
```

注册到 Claude Desktop（配置文件 `claude_desktop_config.json`）：

```json
{
  "mcpServers": {
    "local-files": {
      "command": "python",
      "args": ["/absolute/path/to/server.py"]
    }
  }
}
```

就这么简单。函数签名 + docstring 会被自动转换成工具描述，不需要手写 JSON Schema。

**注意 `args` 里必须用绝对路径**，相对路径在大多数 Host 里会因为工作目录不确定而启动失败。

## 和 Function Calling 的关系

这是最常被问的问题。一句话：

> **Function Calling 是模型的能力，MCP 是工具的接口标准。**

两者不是替代关系，而是不同层次：

- **Function Calling**：让模型输出结构化的函数调用意图（OpenAI 的能力）
- **MCP**：定义工具如何被发现、描述、调用和返回结果（跨模型的协议）

没有 Function Calling，模型不会调工具；没有 MCP，你每换一个工具都要重写一遍适配。

实际链路是：

```
用户提问 → 模型通过 Function Calling 决定调用某工具
        → Host 通过 MCP 协议把请求发给对应 Server
        → Server 执行并返回
        → 结果回灌给模型 → 模型组织回答
```

## 几个实践建议

1. **工具描述比函数名重要**。模型靠 description 判断何时调用，写得含糊会导致该调不调、不该调乱调。

2. **工具数量控制在 20 个以内**。太多会挤占上下文，也会显著降低选择准确率。

3. **优先做只读工具**。写操作（删文件、发消息）务必加二次确认，别让模型直接碰不可逆操作。

4. **错误处理要返回可读信息**。给模型返回 `Error: null pointer` 它没法自我纠正，返回 `目录不存在，请检查路径拼写` 它就知道该改参数重试。

## 小结

MCP 的价值不在技术创新，而在**标准化带来的网络效应**：

- 对工具作者：写一次，所有支持 MCP 的 Host 都能用
- 对应用开发者：接一次，所有 MCP Server 都能用
- 对用户：可以自由组合 Host 和工具，不再被单一厂商锁定

如果你的 Agent 需要接多个数据源，值得先看看有没有现成的 MCP Server，而不是自己写适配。

## 参考

- [MCP 官方文档](https://modelcontextprotocol.io/)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [Anthropic 发布公告](https://www.anthropic.com/news/model-context-protocol)
