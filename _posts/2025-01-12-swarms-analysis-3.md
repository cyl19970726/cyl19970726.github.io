---
layout: post
title: "Swarms Analysis (三)"
date: 2024-07-18 10:02:00 +0800
categories: swarms ai
tags: [agent, swarms, analysis, architecture]
---

这篇文章讲介绍 #SWARMS 框架里最好用的两个东西，一个用于调度多个Agent的Swarm框架

## AgentRearrange 
AgentRearrange 的核心机制：
1. Agent 任务分配机制：
- AgentRearrange 主要通过 `_run` 方法处理任务分配，核心逻辑如下：

```python
# 1. 并行处理多个 Agent
if len(agent_names) > 1:
    # 多个 Agent 并行执行
    results = []
    for agent_name in agent_names:
        agent = self.agents[agent_name]
        result = agent.run(
            task=task_with_context,
            img=img, 
            is_last=is_last
        )
        results.append(result)
    current_task = "; ".join(results) # 合并结果

# 2. 顺序处理单个 Agent
else:
    agent_name = agent_names[0]
    agent = self.agents[agent_name]
    current_task = agent.run(
        task=task_with_context,
        img=img,
        is_last=is_last
    )
```

2. 核心代码主要集中在:

- Flow 解析和验证:
```python
def validate_flow(self):
    if "->" not in self.flow:
        raise ValueError("Flow must include '->' to denote direction")
    
    tasks = self.flow.split("->")
    agents_in_flow = []
    
    for task in tasks:
        agent_names = [name.strip() for name in task.split(",")]
        for agent_name in agent_names:
            if agent_name not in self.agents and agent_name != "H":
                raise ValueError(f"Agent '{agent_name}' not registered")
            agents_in_flow.append(agent_name)
```

- 任务执行流程:
```python
def _run(self, task: str = None, img: str = None):
    tasks = self.flow.split("->")
    current_task = task
    
    for task_idx, task in enumerate(tasks):
        agent_names = [name.strip() for name in task.split(",")]
        
        # 处理并行或串行任务
        if len(agent_names) > 1:
            # 并行处理逻辑
            pass 
        else:
            # 串行处理逻辑
            pass
```

3. Flow 处理机制:

- Flow 定义采用 `->` 箭头表示任务流向,`,` 逗号表示并行执行
- 例如 `"A -> B,C -> D"` 表示:
  1. A 先执行
  2. B 和 C 并行执行
  3. D 最后执行

- Flow 解析过程:
```python
# 1. 先用 -> 分割获取执行阶段
tasks = self.flow.split("->")  # ["A", "B,C", "D"]

# 2. 对每个阶段,用逗号分割得到并行 agents
for task in tasks:
    agent_names = [name.strip() for name in task.split(",")]
    # "B,C" 会得到 ["B", "C"] - 表示并行
```

核心流程总结:
1. 初始化时注册所有 Agent 并定义 flow
2. 执行时解析 flow 字符串确定执行顺序
3. 处理每个阶段:
   - 单个 Agent 串行执行
   - 多个 Agent 并行执行
4. 结果在 Agents 间传递,作为下一个阶段的输入

**这种设计让多个 Agent 可以灵活组合,既支持串行又支持并行执行。通过简单的 flow 定义就能实现复杂的任务调度**
### Human Input 

```python

from swarms import Agent, AgentRearrange
from typing import List

# 定义一个自定义的人工审查函数
def custom_human_review(prompt: str) -> str:
    """
    自定义的人工审查函数，可以添加特定的验证逻辑
    """
    print("\n=== Custom Human Review Required ===")
    print(f"Task to review: {prompt}")
    
    # 添加特定的验证逻辑
    while True:
        response = input("Please enter your review (or 'reject' to decline): ")
        if response.lower() == 'reject':
            return "Task rejected by human reviewer"
        
        # 验证响应长度
        if len(response) < 10:
            print("Response too short! Please provide more details.")
            continue
            
        # 检查是否包含必要关键词
        if 'approved' not in response.lower():
            print("Please explicitly include 'approved' in your response!")
            continue
            
        break
    
    return f"[HUMAN APPROVED] {response}"

# 初始化代理
reviewer = Agent(
    agent_name="Senior Reviewer",
    system_prompt="Reviews financial decisions",
    llm="gpt-4"
)

analyst = Agent(
    agent_name="Financial Analyst",
    system_prompt="Analyzes financial data",
    llm="gpt-4"
)

# 创建带有人工审查的代理系统
agent_system = AgentRearrange(
    agents=[reviewer, analyst],
    flow="Financial Analyst -> H -> Senior Reviewer",  # H 表示人工介入
    human_in_the_loop=True,
    custom_human_in_the_loop=custom_human_review  # 使用自定义的人工审查函数
)

# 运行系统
result = agent_system.run(
    "Review the proposed investment in Project X with a budget of $1M"
)
print("\nFinal Result:", result)
```
custom_human_in_the_loop 是一个自定义的人工介入函数，允许你定义特定的人工交互逻辑。我来举个实际的例子：

在这个例子中：
1. custom_human_review 函数实现了特定的人工审查逻辑：
   - 要求响应不能太短（至少10个字符）
   - 要求响应中必须包含 "approved" 关键词
   - 提供了 "reject" 选项来拒绝任务
   - 为审批的内容添加了特定的格式（[HUMAN APPROVED] 前缀）

2. 工作流程：
   - Financial Analyst 首先分析数据
   - 结果传给人工审查（H），使用自定义的 custom_human_review 函数
   - 人工审查后，结果传给 Senior Reviewer 做最终审核

3. 使用场景：
   - 金融审批流程
   - 合规检查
   - 质量控制
   - 敏感操作验证

如果不设置 custom_human_in_the_loop，系统就会使用默认的 input() 函数来获取人工输入，没有任何验证或特殊处理。而使用自定义函数，你可以：

- 添加输入验证
- 实现特定的业务逻辑
- 集成外部系统
- 添加日志记录
- 实现更复杂的交互流程

这对于需要严格控制人工介入流程的场景特别有用，如金融审批、医疗诊断等需要特定验证流程的应用。



##  SwarmArrange  
让我来深入分析这个框架:

1. 框架核心机制和设计原因:
graph TD
    A[SwarmRearrange] -->|Orchestrates| B[AgentRearrange 1]
    A -->|Orchestrates| C[AgentRearrange 2]
    A -->|Orchestrates| D[AgentRearrange 3]
    
    B -->|Controls| E[Agent 1.1]
    B -->|Controls| F[Agent 1.2]
    
    C -->|Controls| G[Agent 2.1]
    C -->|Controls| H[Agent 2.2]
    
    D -->|Controls| I[Agent 3.1]
    D -->|Controls| J[Agent 3.2]

    style A fill:#f9f,stroke:#333,stroke-width:4px
    style B fill:#bbf,stroke:#333,stroke-width:2px
    style C fill:#bbf,stroke:#333,stroke-width:2px
    style D fill:#bbf,stroke:#333,stroke-width:2px

![](/assets/images/截屏2025-01-06 上午2.02.20.png)

核心设计机制：
1. 分层结构
- SwarmRearrange (顶层): 管理和协调多个 AgentRearrange
- AgentRearrange (中层): 管理单个任务流程中的多个 Agent
- Agent (底层): 执行具体的任务

2. 流程控制
```python
# SwarmRearrange 顶层流程
flow = "Blackstone-Acquisition-Analysis -> Blackstone-Investment-Strategy -> Blackstone-Market-Analysis"

# AgentRearrange 中层流程
flows = [
    "Industry-Analyst -> Tech-Expert -> Market-Researcher -> Regulatory-Specialist -> Managing-Director -> VP-Finance",
    "Managing-Director -> VP-Finance -> Industry-Analyst -> Tech-Expert -> Market-Researcher -> Regulatory-Specialist"
]
```

3. 设计原因：
- 可扩展性：支持复杂的多层级任务编排
- 灵活性：每层都可以独立配置和修改
- 可重用性：相同的 Agent 可以在不同的 AgentRearrange 中复用
- 并行处理：支持并行和串行任务执行

2. SwarmRearrange 和 AgentRearrange 的关系：

1. 层级关系：
- SwarmRearrange 是更高层的抽象，用于管理多个 AgentRearrange
- AgentRearrange 专注于单个任务流程的 Agent 编排

2. 功能分工：
```python
# SwarmRearrange 管理多个业务流程
swarm_arrange = SwarmRearrange(
    swarms=[
        blackstone_acquisition_analysis,  # AgentRearrange 实例
        blackstone_investment_strategy,   # AgentRearrange 实例
        blackstone_market_analysis        # AgentRearrange 实例
    ]
)

# AgentRearrange 管理具体任务执行
blackstone_acquisition_analysis = AgentRearrange(
    agents=[managing_director, vp_finance, industry_analyst],
    flow="Managing-Director -> VP-Finance -> Industry-Analyst"
)
```

3. 适合处理的任务类型：

1. 复杂业务流程：
- 投资分析流程（如示例中的 Blackstone 案例）
- 多阶段审批流程
- 复杂的数据处理管道

2. 具体应用场景：
```python
# 示例：投资分析流程
# 第一层：整体投资流程
swarm_flow = "收购分析 -> 投资策略 -> 市场分析"

# 第二层：具体分析环节
agent_flow = "行业分析 -> 技术评估 -> 市场研究 -> 合规审查 -> 总监审批 -> 财务审核"
```

3. 特别适合：
- 需要多个专家协作的任务
- 有严格执行顺序的流程
- 需要人机协作的复杂决策
- 需要并行处理的大规模任务
- 有多个子流程的复杂业务系统

这个框架的设计非常适合处理企业级的复杂业务流程，特别是那些需要多个专家或系统协作的场景，比如投资分析、风险评估、产品研发等领域 