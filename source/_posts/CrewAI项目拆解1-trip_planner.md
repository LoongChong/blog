---
title: CrewAi项目拆解-trip_planner
description: 学习Crew AI项目通过项目学习框架
categories:
  - Agent开发
tags:
  - CrewAi
  - Agent框架
date: 2026-09-09 16:25
cover: /img/crewai.png
---

# Trip Planner

## 源代码

```python
# main.py
class TripCrew:

  def __init__(self, origin, cities, date_range, interests):
    self.cities = cities
    self.origin = origin
    self.interests = interests
    self.date_range = date_range

  def run(self):
    agents = TripAgents()
    tasks = TripTasks()

    city_selector_agent = agents.city_selection_agent()
    local_expert_agent = agents.local_expert()
    travel_concierge_agent = agents.travel_concierge()

    identify_task = tasks.identify_task(
      city_selector_agent,
      self.origin,
      self.cities,
      self.interests,
      self.date_range
    )
    gather_task = tasks.gather_task(
      local_expert_agent,
      self.origin,
      self.interests,
      self.date_range
    )
    plan_task = tasks.plan_task(
      travel_concierge_agent, 
      self.origin,
      self.interests,
      self.date_range
    )

    # 核心执行代码
    crew = Crew(
      agents=[
        city_selector_agent, local_expert_agent, travel_concierge_agent
      ],
      tasks=[identify_task, gather_task, plan_task],
      verbose=True
    )
    # 启动引擎
    result = crew.kickoff()
    return result

```

```python
# trip_agents.py

class TripAgents():

  def city_selection_agent(self):
    return Agent(
        role='City Selection Expert',
        goal='Select the best city based on weather, season, and prices',
        backstory=
        'An expert in analyzing travel data to pick ideal destinations',
        # Agent 不会自动有手有脚，它能做什么完全取决于你给了哪些 tools。
        # CrewAI 会把工具名+描述注入到 Agent 的 prompt 里，模型自己决定何时调用。
        tools=[
            SearchTools.search_internet,
            BrowserTools.scrape_and_summarize_website,
        ],
        # verbose=True：运行时在终端打印这个 Agent 的完整"内心戏"
        # 你第一次跑的时候一定要开着，看它是怎么决定先搜索、再点开网页、再综合结论的，这是最好的学习材料。
        verbose=True)

  def local_expert(self):
    return Agent(
        role='Local Expert at this city',
        goal='Provide the BEST insights about the selected city',
        backstory="""A knowledgeable local guide with extensive information
        about the city, it's attractions and customs""",
        tools=[
            SearchTools.search_internet,
            BrowserTools.scrape_and_summarize_website,
        ],
        verbose=True)

  def travel_concierge(self):
    return Agent(
        role='Amazing Travel Concierge',
        goal="""Create the most amazing travel itineraries with budget and 
        packing suggestions for the city""",
        backstory="""Specialist in travel planning and logistics with 
        decades of experience""",
        tools=[
            SearchTools.search_internet,
            BrowserTools.scrape_and_summarize_website,
            CalculatorTools.calculate,
        ],
        verbose=True)
```



