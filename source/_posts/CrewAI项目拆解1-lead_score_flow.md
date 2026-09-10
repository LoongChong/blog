---
title: Crew AI项目拆解-lead_score_flow
description: Flow处理非线性复杂的流程
categories: 
  - Agent开发
tags: 
  - Crew AI
  - Agent框架
cover: /img/crewai.png
date: 2026-09-09 19:35
---

**对于非线性复杂流程，现在更推荐使用 Crew AI 的 `Flow` 抽象（而非强行塞进一个 Crew）。`Flow` 支持条件路由、并行分支、状态持久化，是构建生产级 Agent 系统的标配。**

Crew AI的主流框架范式已经从早期的“纯代码定义”全面转向了 **“YAML 声明式配置 + Python 动态编排”** 的混合架构。这种模式在可维护性、团队协作和复杂流程控制之间取得了最佳平衡。将“是什么”（配置）与“怎么做”（逻辑）彻底解耦:

```text
my_crew_project/
├── config/
│   ├── agents.yaml      # ✅ Agent 的 role/goal/backstory
│   └── tasks.yaml       # ✅ Task 的 description/expected_output
├── src/
│   └── my_crew/
│       ├── __init__.py
│       ├── crew.py      # ✅ @CrewBase 类：组装、工具注入、LLM选择
│       ├── tools/       # ✅ 自定义工具模块
│       └── main.py      # ✅ 入口点 & CLI
├── pyproject.toml       # ✅ 依赖管理 (uv/poetry)
└── .env                 # ✅ API Keys (绝不硬编码)
```

## 新的项目骨架：`@CrewBase` 装饰器模式

对比 Trip Planner 的老式写法，`lead_score_crew.py` 用了 Crew AI **脚手架标准模式**：

Python

```python
@CrewBase
class LeadScoreCrew:
    agents_config = "config/agents.yaml"
    tasks_config = "config/tasks.yaml"

    @agent
    def hr_evaluation_agent(self) -> Agent:
        return Agent(config=self.agents_config["hr_evaluation_agent"], ...)

    @task
    def evaluate_candidate_task(self) -> Task:
        return Task(config=self.tasks_config["evaluate_candidate"], 
                    output_pydantic=CandidateScore)

    @crew
    def crew(self) -> Crew:
        return Crew(agents=self.agents, tasks=self.tasks, ...)
```

三个装饰器的分工：

| 装饰器             | 作用                                                         |
| :----------------- | :----------------------------------------------------------- |
| `@CrewBase`        | 类级标记，让这个类获得 `self.agents` / `self.tasks` 自动收集能力 |
| `@agent` / `@task` | 标记工厂方法，框架**自动把返回值收集**到 `self.agents` / `self.tasks` 列表——你不用手动写 `tasks=[...]` |
| `@crew`            | 标记最终的 Crew 组装方法                                     |

**设计意图**：Agent/Task 的定义**外置到 YAML**（配置即提示词），Python 代码只负责"组装"。这就是你之前用 `crewai create crew` 生成项目时看到的结构。以后自己写项目建议用这种。

## 配置文件：提示词与代码分离

### `config/agents.yaml`

```yaml
hr_evaluation_agent:
  role: >
    Senior HR Evaluation Expert
  goal: >
    Analyze candidates' qualifications and compare them against...
  backstory: >
    As a Senior HR Evaluation Expert, you have extensive experience...
```

就是 Trip Planner 里 `role/goal/backstory` 三元组的 YAML 版——**改提示词不用动 Python 代码**，产品和工程可以分开维护。

### `config/tasks.yaml` —— 有一个值得偷师的细节

```yaml
description: >
  ...
  Your final answer MUST include:
  - A score between 1 and 100. Don't use numbers like 100, 75, or 50. 
    Instead, use specific numbers like 87, 63, or 42.
```

这个反常规！为什么**禁止**整数分数？因为 LLM 偷懒时会扎堆给 100/75/50，导致候选人无法排序。强制"必须用 87、63 这种具体数字"是在**对抗模型的默认值倾向**——你写 prompt 时遇到模型输出雷同，可以借这个思路。

另外注意占位符 `{candidate_id}`、`{bio}`、`{job_description}`、`{additional_instructions}`——它们就是 `main.py` 里 `kickoff_async(inputs={...})` 注入的值。还记得吗，`additional_instructions` 来自用户反馈（human-in-the-loop 里选 2 输入的那段话），实现"带着新要求重新评分"。