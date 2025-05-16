---
layout: post
title: "Buzz Swarm Architecture：如何优雅的实现"
date: 2023-10-01 10:00:00 +0800
categories: web3
tags: agent swarm ai buzz architecture
---

『Buzz 之 Swarm 如何优雅的实现』
昨日副标题 『 为何在buzz上亏钱 』
今日副标题 『 仓位不够大 』

buzz 如我上一条推特所说，已经切换成多个Agent互相协作的架构，目前是分为9个Agent来处理任务
![agents](/assets/images/agents.png)

比如我们举一个例子来理解这个调用过程： 
- Agent描述
Market Agent：负责获取代币信息，市场趋势
Trading Agent： 负责执行交易，比如swap交易，提供流动性交易，stake交易等等
Social Agent: 负责获取社交媒体上的信息
Boss Agent: 负责把用户的任务分配给对应的Agent去执行
- 任务描述
帮我获取推特上关于$ai16z的信息并且判断是否值得买入，如果可以买入帮我买入0.01sol. 
- 执行过程
那么这个任务，boss agent会判断任务要先分配给哪个Agent,当前这个任务会判断先分配给到Social Agent，然后social agent执行完成后会调用Trading Agent继续执行, Trading Agent会根据 Social Agent的输出来判断是否发起交易行为。

![buzz swarms pic](/assets/images/buzz-swarms-pic.png)

那么Buzz是怎么实现这个东西的：
1. 首先在原来Agent的Tools上增加了 Invoke Tool,一个Agent实例可以通过这个tool来调用其他Agent
![buzz app framework](/assets/images/buzz-app-framework.png)
2. 在最新的代码里，其实把整个Hive的逻辑分成了9个Agent,同时在这些Agent之间实现了互相调用

同时如果这些Agent在初始化的时候注册了Invoke-agent-tool, 那么它就拥有调用其他agent的能力。

比如下面这个例子，就是在trading_agent的system_prompt里同时注册了solana-trade-tool 和 invoke-agent-tool,
![trading agent prompt](/assets/images/trading-agent-prompt.png)
https://github.com/jasonhedman/the-hive/commit/cb3d79639382c2041a553bb06c89a4968f3bc1ed#diff-63b10dae85e24cd77d9b8e3dd074449ef69ea21d84b10fb1ab2e5ce40a3f7b78R7

3. 接下来我们看 invoke-agent-tool的具体实现
核心是system_prompt如下，核心在最后一行，把能调用的agent和这个agent的能力嵌入到这个prompt里
![Invoke agent prompt](/assets/images/Invoke-agent-prompt.png)
https://github.com/jasonhedman/the-hive/commit/cb3d79639382c2041a553bb06c89a4968f3bc1ed#diff-55697936e4c9080cbc4c5060b409aae2fc7ad75c25cd2d27cf5569581023b530R3

4. 同时还有一个chooseAgent来判断先调用哪个Agent
![hive route](/assets/images/hive-route.png)
主要关注下图的system_prompt,描述了当前的系统里拥有哪些agent可以供当前这个Boss Agent调用，所以Boss Agent在这里工作的关键在于
![choose agent](/assets/images/choose-agent.png)

5. 最后在前端按照调用agent的顺序，把每个Agent的结果拼接后显示给用户
![agg output](/assets/images/agg-output.png)

于是我们就可以得到这样的效果：
![hive message](/assets/images/hive-message.png)

总结 Buzz Swarm的流程如下： 
![buzz swarm framework](/assets/images/buzz-swarm-framework.png)
其中的Boss Agent负责判断哪个Agent第一个来执行任务，并且把任务分配给第一个Agent,第一个Agent执行完成之后再调用invoke-agent-tool来调用下一个Agent进行执行，并依次不断重复这个过程，直到任务执行完成。

不过目前整个Swarm架构还没有很完善，有些时候会有一些bug,同时对于越来越复杂的defai的场景来看，以及未来甚至需要支持并行执行一些任务的趋势来看，现在这个Swarm架构其实还不足以应对未来的所有场景，期待Buzz Swarm的进一步进化～ 