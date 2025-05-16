---
layout: post
title: "Swarms Analysis (一)"
date: 2024-07-18 10:00:00 +0800
categories: swarms ai
tags: [agent, swarms, analysis, architecture]
---

## 1. Agent架构
Swarms Agent的架构如下，大体架构其实跟其他Web3 AI Agent框架都差不多；

根据跟用户的交互数据生成对应的任务，然后把任务交给LLM(大语言模型)进行处理，大语言模型会根据提前设置好的Tool Resistry选取对应的Tool进行执行，同时执行结果以及过程中发生的一些信息提取都会被加入到Memory之中。

我们大概可以认为 Swarms Tool = Eliza Provider + Action = Arc Tool.
所以这一块大家并没有什么太大的差别，也不存在太强的技术壁垒。
![](/assets/images/swarm_agent_framework.png)

## 2. Swarms Framework
Swrms真正核心的其实是这套让各个Agent协作的架构
并且他有一个SwapRouter组件，通过这个组件它可以根据具体的任务内容启动不同的Agent协作框架，大体架构如下：
- SwarmRouter和SwarmMatcher会决定一个Task选择哪一个具体的Swarm框架进行执行
- 具体的Swarm框架会分配多个Agents来执行这个Task的不同部分，并且管理他们之间的输入输出，有些任务之间可能会有依赖关系，这个调度过程也是交给对应启动的Swarm框架来管理
- 最后就是被调度到的Agent执行自己的任务并且返回结果
![](/assets/images/swarm_framework.png)

### 2.1 目前支持的Swarm方式
- AgentRearrange (rearrange.py)
	- 用于优化 agent 排列顺序和工作流
	- 支持灵活的任务流程定义
	- 可以按照自定义的流程图执行任务

- MixtureOfAgents (mixture_of_agents.py)
	- 组合多个不同类型的专家 agents
	- 使用聚合器 agent 来汇总和分析其他 agents 的输出
	- 通过分层执行实现任务分解和综合
	
- SpreadSheetSwarm (spreadsheet_swarm.py)
	- 像电子表格一样处理并展示数据
	- 支持并发任务执行并保存元数据
	- 可以将处理结果导出为 CSV 文件

- SequentialWorkflow (sequential_workflow.py)
	- 按顺序执行任务的工作流
	- 支持任务依赖关系管理
	- 可以保存和恢复工作流状态

- ConcurrentWorkflow (concurrent_workflow.py)
	- 支持并发执行多个任务
	- 使用线程池进行任务调度
	- 可以监控资源使用情况
	-
我们下面详细深入下 SequentialWorkflow，ConcurrentWorkflow， MixtureOfAgents这几个协作方式，剩下的几个我有时间再来写，Swarms的协作代码太多了看不过来了。
#### 2.1 SequentialWorkflow 如何工作
- 按顺序执行任务的工作流
- 支持任务依赖关系管理
- 可以保存和恢复工作流状态
- 以下是一个使用的调用例子，假设我们有创建了两个Agent:
	- Agent1是一个负责研究的Ai Agent
	- Agent2是一个负责写作的Ai Agent
	- 这个时候我们通过SequentialWorkFlow创建一个task： 研究并写一篇关于AI的文章
	- 这个时候这个 task 会被先传递给 Agent1 ， 然后我们可以得到 output1 = Agent1.run(Task) 
	- 接着我们更新下Content = {当前的任务是task, 已经有的研究结构是 output1 },然后把它传递传递给 Agent2, 这样最终我们就可以实现一个顺序分工的Agent工作机制
![](/assets/images/sequential_workflow.png)
在以下代码里，我们也可以看到Sequential_flow就是一个顺序调用Agent的流程 
	![](/assets/images/sequential_core_code.png)	
#### 2.2 ConcurrentWorkflow 是如何工作
- 并发执行任务的工作流，支持多个Agent同时分析处理数据
- 以下是一个金融分析的使用示例:
	- 创建了三个专业的分析Agent:
		- Profit-Analysis-Agent: 负责分析利润相关指标
		- Cashflow-Analysis-Agent: 负责分析现金流相关指标
		- Risk-Analysis-Agent: 负责分析风险相关指标
	- 通过ConcurrentWorkflow将这三个Agent组织起来:
	- 工作流程:
		- 输入一份财务报告作为分析数据
		- 三个Agent会同时开始分析各自负责的指标
		- 最终汇总所有Agent的分析结果
	![](/assets/images/concurrent_workflow.png)
我们看ConcurrentWorkflow代码里的核心处理逻辑，可以总结成一句话，让所有Agent同时输入我们上面的financial_report作为Task，他们会根据我们提前给他的system_prompt来处理对应的数据并生成对应的结果，最终我们把所有结果聚合在一起。
![](/assets/images/concurrent_core_code.png)

#### 2.3 MixtureOfAgents 如何工作
下面用分析AI教育市场的例子来解释MixtureOfAgents
- 设置了三个专业分析Agent和一个聚合Agent
- 每个Agent专注于特定领域:
	- Market Agent专注于市场分析
	- Competitor Agent专注于竞品分析
	- User Agent 专注于用户行为分析
	- Aggregator Agent 根据以上三个Agent得到的市场，用户和竞品报告生成完整的分析报告以及未来的策略。
	- 分析流程![](/assets/images/mixture_process.png)

	- ![](/assets/images/mixture_swarm_example.png)


#### 2.4 SwarmRouter怎么自动选择最合适的Swarm框架
这里面自动选择最合适的Swarm框架啊的核心在于find_best_match函数，它首先会调用self.get_embedding函数把task转换成向量表示，然后跟提前注册好的self.swarm_types里面的swarm框架进行比较选择，最终分数最高的就是最适合的Swarm框架。
![](/assets/images/swarm_matcher.png)
下面这个代码是提前注册的Swarm框架的描述
![](/assets/images/swarms_type.png)
下面是一个实际的例子理解SwarmRouter的工作方式：
假设有这样一个任务：
`task = "分析这份财务报告，需要从利润、现金流和风险三个维度同时进行分析"`
匹配过程：
1. 将任务文本转换为向量表示
2. 计算与各个 Swarm 类型描述的相似度
3. 这个任务强调"同时"和"三个维度"，会与 ConcurrentWorkflow 的描述（parallel processing, simultaneous tasks）有较高相似度
4. 系统会选择 ConcurrentWorkflow 作为最佳匹配
## 结论
在以上整个流程里 Swarms 就像一个无所不能的"Human Manager"，它知道：
1. 有哪些 Agents（"员工"）
2. 每个 Agent 的专长是什么
3. 如何分配任务给 Agents
4. 如何让 Agents 以最搞笑的协同方式进行工作
5. 如何收集和整理 Agents 的工作成果
感觉如果让他来做老板人类得被压榨没了....

并且另外我们需要注意到的是Swarms的这个SwarmsRouter以及各个具体Swarms的实现是极其模块化的，感觉将这个框架搬到链上让现在这些已经存在的web3 ai agent互相协作是一个更有趣的事情，因为我们都知道现在这些web3 ai agent事实上已经会自发的互相调用，但是事实上还是只能很简单的处理一些简单的工作，如果可以采用Swarms这套协作框架来做，那么未来一个蓬勃发展的web3 ai agents society很值得期待～

同时我们也需要注意到Swarms框架的核心点在于Agent协作，它实际上开创了一条新的赛道，因为目前市面上的ElizaOS, Arc, GAME, Zerobro其实都没有实现任何跟Agent协作相关的部分。 