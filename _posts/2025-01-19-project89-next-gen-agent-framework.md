---
layout: post
title: "我在 Project89 内看到下一代Agent框架"
date: 2024-07-18 10:04:00 +0800
categories: project89 ai
tags: [agent, project89, analysis, architecture, ecs]
---

先说结论，project89 采用一种全新的方式来设计Agent Framework, 这是一个针对游戏开发的高性能的Agent Framework,与目前使用的Agent Framework相比有更好的性能。


## 一、为什么要用ECS来设计Agent Framework

从游戏领域的应用来看
目前采用ECS架构的游戏有：
区块链游戏：Mud、Dojo
传统游戏： 守望先锋、星际公民等等
而且目前主流的游戏引擎也在往ECS的方向进化比如Unity。 
#### 什么是 ECS

ECS（Entity-Component-System）是一种在游戏开发与模拟系统中常用的架构模式。它将数据与逻辑彻底分离，以便在大规模可扩展场景下高效管理各种实体及其行为：

1. **Entity（实体）**  
   • 仅仅是一个 ID（数字或字符串），不包含任何数据或逻辑。  
   • 可以根据需要，挂载不同的组件来赋予它各种属性或能力。
2. **Component（组件）**  
   • 用来存储实体的具体数据或状态。  
3. **System（系统）**  
   • 负责执行与某些组件相关的逻辑。  

以一个具体的Agent行动的例子来理解这套体系：
在 ArgOS 中将每一个Agent看成一个 Entity,它可以注册不同的组件，比如在下面这幅图里，我们这个Agent拥有以下四个组件：
Agent Component: 主要存储类似 Agent 名称，模型名字等等基础信息
Perception Component: 主要用来存储感知到的外界数据
Memory Component: 主要用来存储 Agent Entity的Memory数据，类似做过的事情等等
Action Component: 主要存储要执行的 Action 数据

System 的工作流程：
1. 在这个游戏里比如感知到了自己面前有一个武器，那么这个就会调用 Perception System的执行函数来执行更新这个Agent Entity 的Perception Component 里面的数据
2. 然后再触发 Memory System， 同时 调用 Perception Component 和 Memory Component, 把感知到的数据通过Memory持久化到数据库之后
3. 然后 Action System 再调用Memory Component 和 Action Component ,从记忆中获取周边环境的信息，然后最终执行相应的动作。
4. 接着我们就得到一个每个Component数据 都被更新的 Update Agent Entity
所以我们可以看到这里面System 主要负责定义要对哪些Component 执行对应的处理逻辑。

![](/assets/images/89-1.png)
而且很明显在 project89 里面它是一个世界里面充斥着各种类型的 Agent，比如有些Agent不仅仅拥有以上的基础能力还有做计划的能力。
那么就会如下面这个图所示:
![](/assets/images/89-2.png)

#### 怎么利用LLM
但是这里面我们还要继续思考一个过程，就是什么时候会调用到system，答案是在每个system得处理流程里，比如从 Memory System就会调用 LLM的能力从 Perception Component 提取Memory Component 所需要的数据加载到 Memory Component之中。
由比如 Plan System 处理过程需要使用到LLM的做规划的能力
同时最后Action的触发则是利用我们提前封装在 LLM中的tool来执行。

#### System的运行流程
但是实际的 system 执行流程不是我们想象的 Perception System执行完之后调用 Memory System 这种传统做法，不同System之间是不存在调用关系的，每个System都会在一个规定的周期内执行一次，比如：
- Perception System可能2s执行一次来更新接收到的外界感知，并把它们更新到 Perception Component之中
- Memory System 可能会是每1s执行一次，从Perception Component 之中提取数据加载到Memory Component之中
- Plan System可能是每 1000s执行一次，根据接收到的信息判断自己需不需要根据对应的目标优化制定合理的计划，然后把这部分更新记录到 Plan Component之中
- Action System 可能也是每2s执行一次，这样可以根据外界的信息及时做出反应，同时如果Plan Component有更新则需要根据这部分数据更新自己的Action Component，进而影响最初做出的Action. 

文章到这里都是基于我对ArgOS 的理解大大简化了它的架构方便大家更好理解，接下来我们来看真实的ArgOS. 
#### 二、ArgOS System 架构
ArgOS之中为了让 Agent 可以进行更加深度的思考执行更复杂的任务，设计了很多Component，以及很多System.

并且ArgOS之中将System分为"三种层次" (ConsciousnessLevel)：
##### 1）有意识(CONSCIOUS) 系统
- 包含 RoomSystem, PerceptionSystem, ExperienceSystem, ThinkingSystem, ActionSystem, CleanupSystem)
- 更新频率通常较高 (如每 10 秒)。
- 更贴近"实时"或"显意识"层面的处理，如环境感知、实时思考、执行动作等。
##### 2）潜意识(SUBCONSCIOUS) 系统
-  GoalPlanningSystem, PlanningSystem
- 更新频率相对较低 (如每 25 秒)。
- 处理"思考"的逻辑，如周期性检查/生成目标和计划。
##### 3）无意识(UNCONSCIOUS) 系统
- 目前暂时还没有启用
- 更新频率更慢(如 50 秒以上)，

所以在 ArgOS之中是将不同的System根据ConsciousnessLevel划分，来规定这个System多久之后执行一次。

为什么要这么设计呢？
因为ArgOS之中各个system之间的关系极其复杂，如下图：

--------------------------------------------------------------------------------

![](/assets/images/89-6.png)



1. PerceptionSystem  
   - 负责从外界或其他实体那里收集"刺激"(stimuli)，并将其更新到代理(Agent)的 Perception 组件中。  
   - 判断刺激是否显著变化，根据稳定度、处理模式 (ACTIVE/REFLECTIVE/WAITING) 等做相应更新。  
   - 最终为后续的 ExperienceSystem、ThinkingSystem 等提供"当前感知"的信息。

2. ExperienceSystem  
   - 将 PerceptionSystem 收集到的 Stimuli 转换为更加抽象的"体验"(Experience)。  
   - 会调用 LLM 或规则逻辑 (extractExperiences) 来识别新的体验，并存储到 Memory 组件。  
   - 去重、筛选、校验体验，同时通过 eventBus 触发"experience"事件给其他系统或外部监听者。

3. ThinkingSystem  
   - 智能体自身的"思考"系统。  
   - 从 Memory、Perception 等组件里提取当前状态，通过 generateThought(...) 与 LLM / 规则逻辑生成"思考结果"(ThoughtResult)。  
   - 根据思考结果，可能：  
     • 更新 Memory 中的 thoughts (思考历史)。  
     • 触发新的 Action（放到 Action.pendingAction[eid]）。  
     • 改变 agent 的外在 Appearance（表情、姿势等），并生成相关 Stimulus 让其他实体"看到"变化。

4. ActionSystem  
   - 若某个 Agent 的 Action.pendingAction 非空，则通过 runtime.getActionManager().executeAction(...) 来真正执行动作。  
   - 执行完后将结果写回 Action.lastActionResult，并通知房间或其他实体。  
   - 也会借此产生 CognitiveStimulus（认知刺激），以便后续系统"知道"该动作已完成，或可以纳入记忆。

5. GoalPlanningSystem  
   - 周期性地评估 Goal.current[eid] 列表中目标的进度，或检查外部/自身记忆是否出现重大变化 (detectSignificantChanges)。  
   - 当需要新目标或进行目标调整时，通过 generateGoals(...) 生成并写入 Goal.current[eid]。  
   - 同时更新在进度中 (in_progress) 的目标，若符合完成或失败条件就改变状态，并向相应 Plan 发出完成/失败信号。

6. PlanningSystem  
   - 对"已有目标"(Goal.current[eid])生成或更新 Plan（执行计划）。  
   - 如果检测到某些目标没有对应的活动中 (active) 计划，则通过 generatePlan(...) 产生一个包含若干 steps 的执行路线图，并将其写入 Plan.plans[eid]。  
   - 也会在目标完成或失败时更新与其关联的 Plan 状态，并产生相应认知刺激。

7. RoomSystem  
   - 处理与房间(Room)相关的更新：  
     • 获取房间内的占用者列表 (occupants)，为每位 agent 生成"外观"(appearance)刺激，让其他实体"看到"他的外貌或动作。  
     • 创建房间环境 Stimulus (如贴切的"房间氛围"信息) 并进行关联。  
   - 确保当 Agent 处于某个空间环境时，其他正在感知该空间的实体可以感知到他的外貌变化。

8. CleanupSystem  
   - 定期查找并移除标记了 Cleanup 组件的实体。  
   - 用于回收不再需要的 Stimulus 或其他对象，防止 ECS 中遗留大量无效的实体。

--------------------------------------------------------------------------------
#### 示例：从"看到物体"到"执行动作"的一次循环

下面的场景示例展示了各个 System 如何在一回合(或几帧)内依次配合完成一次完整流程。

1) 场景准备：  
   - 世界(World)中存在一个 Agent (EID=1)，正在"Active"状态，且处在某个 Room (EID=100)。  
   - 该 Room 里新出现了一个道具"MagicSword"，生成了相应的 Stimulus。

2) PerceptionSystem  
   - 检测到"MagicSword"的出现，对 Agent(1) 生成 Stimulus (type="item_appearance") 并添加到 Perception.currentStimuli[1]。  
   - 对比上一次 Stimuli Hash，认定"有显著变化"，"重新激活"代理的 ProcessingState (ACTIVE 模式)。

3) ExperienceSystem  
   - 看到 Agent(1) 的 Perception.currentStimuli 非空，于是把"Sword appears"之类的信息抽取为1条或多条新 Experience (type: "observation")。  
   - 存储到 Memory.experiences[1] 里，并发射「experience」事件。

4) ThinkingSystem  
   - 读取 Memory、Perception 等状态信息，调用 generateThought：  
     "我看到了 MagicSword，也许能捡起来看看它能干什么..."  
   - 该思考结果包含一个待执行的 Action：{ tool: "pickUpItem", parameters: { itemName: "MagicSword" } }  
   - ThinkingSystem 将此 Action 写入 Action.pendingAction[1]。  
   - 若有 appearance 改动（例如"面带好奇的表情"），则更新 Appearance 并产生视觉刺激。

5) ActionSystem  
   - 看到 Action.pendingAction[1] = { tool: "pickUpItem", parameters: ... }。  
   - 通过 runtime.getActionManager().executeAction("pickUpItem", 1, { itemName: "MagicSword" }, runtime) 执行"拾取"动作逻辑。  
   - 得到结果：{ success: true, message: "你捡起了魔法剑" }，更新到 Action.lastActionResult[1]，并触发「action」事件广播给房间(100)。  
   - 同时产生认知刺激 (type="action_result")，写入 Memory 或让 ThinkingSystem 下个回合捕获。

6) GoalPlanningSystem (若该 agent 有目标)  
   - 周期性地评估 agent 的目标，如果此时 agent 的某个目标是"获得强力武器"，并检测到 MagicSword 已到手，可能将该目标标记为完成。  
   - 如果产生/keySeatTr new changes (比如"新物体出现在房间"影响了 agent 追求的目标？)，则根据 detectSignificantChanges 生成新的目标或放弃旧目标。

7) PlanningSystem (若有相关目标)  
   - 对"获得强力武器"这样已完成或刚生成的目标，检查是否需要新 Plan 或更新既有 Plan。  
   - 如果已完成，则把关联的 Plan [status] 置为 "completed"；或若目标要扩展后续流程 ("研究魔法剑")，则生成更多步骤。

8) RoomSystem (每帧或每回合)  
   - 更新房间(100)中的 Occupants 列表和可见实体。  
   - 若 agent(1) 外观改变（比如 Appearance.currentAction = "holding sword"），就创建新的「appearance」视觉刺激，让其他在同房间的 Agent2, Agent3 知道"agent1 拿起了剑"。

9) CleanupSystem  
   - 移除已标记(Cleanup)的实体或刺激。若拾取后"MagicSword"这个 Stimulus 不再需要保留，可以在 CleanupSystem 中删除对应 Stimulus 实体。

通过这些系统的衔接，AI Agent 就实现了：  
• **感知环境变化(Perception) → 记录或转化为内在经验(Experience) → 自我思考并决策(Thinking) → 付诸行动(Action) → 动态调整目标与计划(GoalPlanning + Planning) → 同步环境(Room) → 及时回收无用实体(Cleanup)**  


## 三、ArgOS整体架构解析

### 1. 核心架构分层

| 层级        | 说明                          | 实例                          |
|-----------|-----------------------------|-----------------------------|
| Component | 数据容器                       | Memory, Emotion, Goal       |
| System    | 逻辑处理器                      | Perception, Decision, Social|
| Manager   | 资源调度与生命周期管理               | MemoryPool, EntityFactory   |
| Runtime   | 主循环与事件调度                   | FrameScheduler, EventQueue  |
| Entity    | 实体标识（eid）                 | #1001, #1002                |
![](/assets/images/89-7.png)

--------------------------------------------------------------------------------
### 2、组件(Component)分类示例

在 ECS 中，每个实体 (Entity) 可拥有若干组件 (Component)。根据在系统中的性质和生命周期，大致可以将组件分为以下几类：

1. 核心身份类 (Identity-Level Components)  
   • Agent / PlayerProfile / NPCProfile 等  
   • 用来唯一标识实体、存放核心角色或单位信息，一般需要持久化到数据库。  

2. 行为与状态类 (Behavior & State Components)  
   • Action, Goal, Plan, ProcessingState 等  
   • 代表了实体当前要做的事情或目标，以及对外部命令、内部思维的响应状态。  
   • 包含 pendingAction, goalsInProgress, plans, 以及在队列中的思考或任务等。  
   • 通常为中/短期状态，很多会随着游戏回合或业务周期动态变化。  
   • 是否需要落库视情况而定。如果希望断点续跑，可能定期写入数据库。

3. 感知与记忆类 (Perception & Memory Components)  
   • Perception, Memory, Stimulus, Experience 等  
   • 记录了实体感知到的外部信息 (Stimuli)，以及感知后提炼成的体验 (Experiences)。  
   • Memory 往往可积累大量数据，例如对话记录、事件历史等；常常需要做持久化。  
   • Perception 可能是实时或临时信息，多为短期内有效，可根据需求决定是否写入数据库（比如只存重要的感知事件）。

4. 环境与空间类 (Room, OccupiesRoom, Spatial, Environment, Inventory 等)  
   • 代表房间、环境、位置、物品容器等信息。  
   • Room.id、OccupiesRoom、Environment 等字段常常需要持久化，如房间首页描述、地图结构等。  
   • 不断变化的组件 (比如 Entity 在不同房间之间移动) 可以做事件式或定期写入。

5. 外观与交互类 (Appearance, UIState, Relationship 等)  
   • 记录实体对外的"可见"或"交互"部分，如 Avatar, pose, facialExpression, 与其他实体的社交关系网络等。  
   • 一部分可只在内存中处理(实时表现)，另一部分（比如关键社交关系）可能要持久化。

6. 辅助或运维类 (Cleanup, DebugInfo, ProfilingData 等)  
   • 用于标记哪些实体需要回收 (Cleanup)、或者记录调试信息 (DebugInfo) 以便在监控和分析时使用。  
   • 一般只在内存中存在，很少会同步到数据库 unless 日志或审计需要。

--------------------------------------------------------------------------------

### 3. System架构

上面已经介绍

#### 4.Manager 架构
有了Component和 System 之外我们其实还缺少一个资源管理者，比如如何访问数据库，当状态更新有冲突怎么处理等等
![](/assets/images/89-5.png)


###### 左侧 Systems (PerceptionSystem、ExperienceSystem、ThinkingSystem 等)：

• 每个系统在 ECS 循环中被 SimulationRuntime 调度执行，查询并处理自己关心的实体（通过组件条件）。

• 在执行逻辑时，需要和 Managers 进行交互，例如：

- 调用 RoomManager (RM) 查询/更新房间信息。

- 使用 StateManager (SM) 获取或保存世界/代理状态，如 Memory、Goal、Plan 等。

- 借助 EventBus (EB) 对外广播或监听事件。

- 在某些需要自然语言处理或提示语时，调用 PromptManager (PM)。

###### 右侧 Managers (EventBus、RoomManager、StateManager、EventManager、ActionManager、PromptManager 等)：

• 提供系统级功能，基本不主动"驱动"逻辑，而是被 Systems 或 Runtime 调用。

• 典型示例：

- ActionManager 专门管理动作(Action)的注册与执行。

- EventManager / EventBus 用于事件发布与订阅机制。

- RoomManager 管理 rooms、布局与 occupant。

- StateManager 负责ECS与数据库或存储的同步。

- PromptManager 提供LLM Prompt模板、上下文管理等扩展。

- 中间的 SimulationRuntime (R)：

• 是所有 Systems 的"调度者"，启动或停止不同层级(Conscious/Subconscious等)的系统循环；

• 也在构造阶段创建 Managers 并传给各 System 使用。

- CleanupSystem：

• 特别注意它还和 ComponentSync (CS) 交互，用于在回收实体时同步移除组件或事件订阅。

结论： 
每个 System 会在需要时通过对应的 Manager 读写数据或调用服务，而 Runtime 则在更高层面统一调度所有 System 与 Manager 的生命周期及行为。

#### 5、 如何向量与数据库进行交互

在 ECS 中，Systems 是真正执行逻辑的地方，而数据库读写可以通过一个"持久化管理器 (PersistenceManager / DatabaseManager)"或"状态管理器 (StateManager)"来完成。大致流程可如下：

1. 启动或加载时 (Initial Load)  
   • StateManager / PersistenceManager 从数据库加载 Agents、Rooms、Goals 等核心持久化组件的数据，创建对应实体 (Entities) 并初始化相关组件字段。  
   • 例如，读取一批 agent 记录插入 ECS 世界，给他们初始化 Agent、Memory、Goal 等组件。

2. ECS 运行时 (Systems Update Loop)  
   • 系统在每个帧 (或回合) 内做事情：  
     1) PerceptionSystem 收集"感知"并写到 Perception 组件 (多为短期不落库)。  
     2) ExperienceSystem 把新的"认知体验"写入 Memory.experiences，若是关键体验可能同时调用 StateManager 做立即存储，或打上"needsPersistence"标记以便后续批量写入。  
     3) ThinkingSystem / ActionSystem / GoalPlanningSystem 等根据组件内容做决策并更新 ECS 中的字段。  
     4) 如果某些组件 (比如 Goal.current) 发生重大变更且需要持久化(例如新的长远目标)，通过组件监听或系统事件，通知 StateManager 将该字段写入数据库。

3. 定期或事件驱动的持久化 (Periodic or Event-Driven)  
   • 可以在系统中的某些关键点（如目标计划发生更新或Agent发生重要事件）时，调用 PersistenceManager.storeComponentData(eid, "Goal") 这类接口进行落库。  
   • 也可以在 CleanupSystem 或定时器里，让 StateManager 扫描含有"needsPersistence"标记的组件或实体，一次性写回数据库。  
   • 此外，日志或审计数据（如动作历史、思考日志）也可在这里进行归档存储。

4. 退出或断点保存 (Manual or Shutdown Save)  
   • 当服务器或进程要关闭时，通过 StateManager.saveAll() 将还没写入的数据统一写入数据库，以确保下次加载能恢复 ECS 状态。  
   • 对于一些单机/离线场景，也可以手动触发存档。  

--------------------------------------------------------------------------------
##### 完整示例流程

以下给一个简单的场景，演示组件和数据库交互的可能方式：

1) 启动时：  
   - StateManager.queryDB("SELECT * FROM agents") → 得到一批 agent 记录，依次为每条记录创建 Entity (EID=x)，并给其初始化 Agent、Memory、Goal 等组件字段。  
   - 同时从 "rooms" 表里加载房间信息，创建 Room 实体。

2) 运行阶段：  
   - PerceptionSystem 在某个房间检测到事件"MagicSword 出现"，写入 Perception.currentStimuli[eid]。  
   - ExperienceSystem 将 Stimuli 转化为 Experience，赋值到 Memory.experiences[eid]。  
   - ThinkingSystem 通过 Memory、Goal 等信息决定执行下一步行动，生成 Action.pendingAction[eid]。  
   - ActionSystem 执行该动作后，将结果写到 Memory 或 Action.lastActionResult。若这是一次重大剧情事件，会把 Memory.experiences[eid] 的最新部分标记为 needsPersistence。  
   - StateManager 在一段时间后发现 Memory.experiences[eid] 带有 "needsPersistence"，则将其写入数据库（INSERT INTO memory_experiences ...）。

3) 停止或断点：  
   - 基于 ECS 或系统调度，在"服务器关闭"时调用 StateManager.saveAll()，把仍在内存中的关键组件字段（Agent, Memory, Goal 等）它们的最新状态写进数据库。  
   - 下次重启时，就能从数据库中加载并恢复 ECS 的世界状态。

--------------------------------------------------------------------------------

• 将组件分类，既方便在项目中清晰地管理实体数据，也能帮助我们控制"需要持久化"与"仅在内存中存在"的数据边界。  
• 与数据库的交互通常由一个专门的 Manager（如 StateManager）来处理，Systems 在需要读写数据库时通过它进行操作，避免在 System 里直接写 SQL 或类似底层语句。  
• 这样能同时享受到 ECS 在逻辑上的高效与灵活，以及数据库带来的持久化、断点续跑与数据统计分析优势。


## 五、架构创新点

![](/assets/images/89-3.png)
整个架构的亮点在于：
每个System都是独立运行的，不会跟其他的System之间有调用关系，因此即便我们希望实现Agent的"**感知环境变化(Perception) → 记录或转化为内在经验(Experience) → 自我思考并决策(Thinking) → 付诸行动(Action) → 动态调整目标与计划(GoalPlanning + Planning) → 同步环境(Room) → 及时回收无用实体(Cleanup)** " 能力的时候，各个System在功能上会有很多互相依赖的点，但是我们依然可以通过ECS架构把整体结构成各个互不相关的System，每个System依然可以独立运行，不会跟其他的System有人和耦合关系。
我想这也是为什么Unity这几年越来越往ECS架构迁移最主要的原因。

而且比如我现在只是希望一个Agent拥有一些基本能力，我只需要在定义Entity的时候减少注册一些Component以及减少注册System就可以轻易的实现，都不用改几行代码。
如下图：

![](/assets/images/89-4.png)
同时如果你在开发过程中希望增加新的功能，也不会对其他的System有任何影响，可以很轻易的把自己希望的功能加载进去。
另外当前这种架构的性能其实会比传统面向对象架构的性能强很多，这在游戏圈是一个公认的事实，因为ECS的设计更加适合进行并发，那么当我们在复杂的Defai场景下，采用ECS架构也有可能会更有优势，特别是希望采用Agent做量化交易的场景下，ECS也会有用武之地（不仅仅是在Agent游戏场景下）。

从我个人的观点来看，这是一个极其模块化，性能极其优秀的框架，同时代码质量也很高并且包含了很好的设计文档，不过可惜的是 $project89 项目一直缺少对这个框架的宣传，因此我才花了很长的时间(4天)写了这篇文章，我觉得好的东西还是值得被发现，明天应该还会发一个英文版本，希望能有更多的游戏团队或者Defai 团队开到这个框架，为大家提供一种新的潜在的架构选择！ 