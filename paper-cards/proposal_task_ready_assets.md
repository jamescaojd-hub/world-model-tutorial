# Proposal：从 Simulation-Ready 到 Task-Ready 的 Embodied AI 物理 3D 资产生成

## 1. 问题定义

近年来，physical 3D generation 已经有了明显进展。模型不再只能生成外观像样的 3D 物体，而是开始能够输出带有几何、关节结构、尺度乃至部分物理属性的资产，并导出为仿真器可以接受的格式。即便如此，**simulation-ready** 和真正的 **task-ready** 之间，仍然存在一条很大的鸿沟。

在实际使用中，一个生成资产可能：
- 可以被正确导入仿真器，
- 外观看起来也比较合理，
- mesh 或 URDF 格式是有效的，
- 但一进入抓取、开合、推拉、形变操作或长程交互任务，就会暴露问题。

本 proposal 关注的正是这条鸿沟。

核心问题可以概括为：

> 我们怎样才能把“仅仅能够进入仿真流程的资产”，推进成“能够稳定支撑 embodied interaction 与下游任务的资产”？

---

## 2. 为什么要做这个问题？

### 2.1 Embodied AI 真正缺的不是更多 3D 资产，而是更可用的 3D 资产

机器人、交互式仿真和 embodied world model 都高度依赖交互数据。但真实世界的数据采集成本高、速度慢，部署难度也大，因此 simulation 已经成为训练和评估 embodied agent 的核心工具之一。

这也意味着，高质量 3D 资产变得非常关键。

然而，对下游系统来说，一个资产真正需要具备的，不只是视觉上“像不像”，而是它能否支持：
- 稳定接触，
- 正确的 articulation，
- 合理的物理响应，
- 有意义的 affordance，
- 以及交互过程中的一致状态变化。

只要这些属性出错，物体即便看起来再真实，也可能直接拖垮整个任务链路。

### 2.2 当前工作已经推动了 simulation-ready generation，但还没有真正解决 task-readiness

近期代表性工作，例如 **PhysX-Anything**、**PhysForge** 和 **PhysX-Omni**，都显著扩展了 physical 3D generation 的能力边界：
- PhysX-Anything 把单图生成推进到了 simulation-ready asset。
- PhysForge 用 hierarchical blueprint 显式引入 physical planning。
- PhysX-Omni 尝试用统一框架覆盖 rigid、deformable 和 articulated 对象。

这些都很重要，但它们目前大多仍停留在 **asset-level** 的成功判据上，例如：
- 几何质量，
- 尺度精度，
- 材质预测，
- articulation 结构，
- 运动学合理性，
- 仿真格式导出能力。

真正还没有被充分回答的问题是：

> 这些生成资产是否真的能够在 embodied tasks 里正常工作？

### 2.3 Asset-ready 并不等于 task-ready

这是本 proposal 最核心的动机。

一个对象可能已经具备：
- 有效 mesh，
- 合法 articulation tree，
- 预测出的 material label，
- 与 simulator 兼容的格式，

但它仍然会在真实交互中失败，因为：
- joint axis 稍微偏了，
- 质量分布不合理，
- friction 估计有误，
- 接触面不稳定，
- affordance 标注和交互方式对不上，
- 或者 deformable 行为在数值上不稳定。

所以真正缺的并不只是生成能力，而是：

> **面向任务可用性的评估、诊断与修复。**

---

## 3. 具体研究问题

本项目围绕三个明确的 research questions 展开。

### RQ1：simulation-ready 的 physical 3D 资产，究竟会在下游交互任务的哪里失败？

这个问题首先要求我们做系统性的 failure analysis。

我们希望弄清楚：
- 哪些生成资产会在交互过程中失败，
- 失败具体表现成什么形式，
- 不同对象类别是否呈现不同的 failure pattern。

更具体地说，我们会重点分析与以下因素相关的失败：
- geometry，
- articulation，
- physical parameters，
- 以及 semantic / affordance representation。

### RQ2：现有的 asset-level 指标，是否高估了资产真实的下游可用性？

当前很多评测默认认为，只要 geometry 或 articulation 分数高，资产就应该是“好用的”。但这件事未必成立。

我们希望验证：
- geometry 分数高，是否真的意味着抓取得更稳；
- articulation 分数高，是否真的意味着开关、拉合等交互会成功；
- material 标注正确，是否真的意味着接触行为合理；
- 更一般地，asset-level quality 与 task success 之间到底有多强的相关性。

如果这些指标对下游表现的预测能力很弱，那么说明当前 benchmark 其实漏掉了问题中最关键的一层。

### RQ3：能否利用 simulator-in-the-loop 的反馈，对生成资产进行自动修复，从而提升 task-readiness？

单图生成和基于 VLM 的推断不可避免会犯错。因此，我们不打算把第一次生成的资产当作最终结果，而是想引入一个 repair loop。

核心问题是：是否可以用交互反馈去自动修正资产，例如：
- 修 joint axis 和 joint range，
- 调 mass 和 friction 参数，
- 改 part hierarchy，
- 修 affordance label，
- 调 collision geometry，
- 或者修 interaction-critical 区域的局部物理配置。

长期目标是把资产生成变成一个闭环：

> image → asset generation → simulator test → failure diagnosis → asset repair

---

## 4. 核心假设

本 proposal 建立在如下假设之上：

> 对 embodied AI 来说，一个生成物理 3D 资产的真实价值，主要不取决于它看起来有多像，而取决于它能否稳定支撑交互、控制和任务执行。

更具体地说：

1. task-level evaluation 会暴露出很多 asset-level evaluation 看不到的失败模式；
2. articulation、physical parameters 和 affordance 表示中的错误，是下游失败的重要来源；
3. simulator-in-the-loop repair 能够显著提升资产的真实可用性；
4. task-aware repair 很可能比单纯继续优化 geometry 更有效。

---

## 5. 方法设计

整个项目被组织成三个 practical work packages。

## WP1：构建 Task-Ready Evaluation Benchmark

第一步不是继续堆生成模型，而是先建立一种评估方式：不只把资产当作“仿真对象”来评，而是把它当作“任务对象”来评。

### 任务范围

我们计划覆盖三类对象上的交互任务：

#### Rigid objects
- grasp and place（抓取与放置）
- push and slide（推动与滑动）
- stack and reposition（堆叠与重定位）

#### Articulated objects
- open drawer（拉开抽屉）
- open cabinet door（打开柜门）
- turn faucet（转动水龙头）
- press switch（按下开关）

#### Deformable objects
- lift cloth（提起布料）
- fold towel（折叠毛巾）
- compress soft object（压缩软体物体）
- deform container（使容器发生形变）

### 评估层次

我们会把每个资产分成三个层次来评估。

#### Level 1：Asset Validity
- 资产能否被正常导入仿真器？
- mesh / articulation 格式是否合法？
- 仿真是否数值稳定？
- 是否出现明显穿模、抖动、爆炸等异常？

#### Level 2：Interaction Validity
- scripted controller 能否真实地与对象发生交互？
- articulation 是否沿正确轴运动？
- deformable object 是否表现出合理响应？
- 接触行为是否稳定且可复现？

#### Level 3：Task Utility
- agent 或控制器能否完成目标任务？
- 资产是否能稳定支撑任务成功？
- world model 是否能可靠预测它的状态变化？
- 这些资产是否真正提升下游训练效用？

这个 benchmark 的目标，就是把 simulation-readiness 和 task-readiness 之间的具体差距量化出来。

---

## WP2：构建 Failure Taxonomy 与 Diagnosis Pipeline

当失败暴露出来之后，下一步就不是简单地说“它坏了”，而是要把失败类型系统地拆开。

### 拟议的 Failure Taxonomy

#### 1. Geometry failures
- 缺失几何结构
- 接触表面不稳定
- 拓扑不合理
- collision shape 错误

#### 2. Kinematic failures
- joint axis 错误
- joint range 错误
- parent-child hierarchy 错误
- articulation layout 不合理

#### 3. Physical parameter failures
- 质量估计错误
- friction 或 compliance 错误
- 尺度不正确
- 材料行为不真实

#### 4. Semantic / affordance failures
- 抓取区域预测错误
- 可动部件识别错误
- 功能状态描述错误
- 可见结构与 affordance 表示不一致

### Diagnosis 的目标

我们的目标不是只报告“这个资产失败了”，而是进一步回答：

> 到底哪里失败了？哪个属性最直接导致了下游任务崩溃？

这个诊断步骤也会让 rigid、deformable、articulated 三类资产，第一次能够放在同一个 error framework 下进行比较。

---

## WP3：构建 Task-Aware Asset Repair Loop

第三部分是整个 proposal 最 practical 的地方：用 simulation feedback 驱动资产修复。

### 基本思路

生成好的资产先被放入 simulator，再通过 scripted interaction policies 进行测试。如果在交互中出现失败，系统就对失败进行诊断，并据此修改资产。

### 可用的反馈信号

潜在的 repair signals 包括：
- scripted trajectory 失败记录，
- 接触不稳定事件，
- articulation 运动异常，
- world model prediction error，
- 重复任务失败，
- 以及基于 VLM 的部件或 affordance 重新检查。

### 可能修复的对象

repair system 可以尝试修改：
- joint axis 和 joint limits，
- mass 或 friction 参数，
- part hierarchy，
- collision geometry，
- blueprint-level affordance labels，
- interaction-critical 区域的局部 geometry。

### 目标结果

我们希望最终得到的，不是一个 one-shot generator，而是一条能够不断提升可用性的反馈链：

> generate → test → diagnose → repair → re-test

这也是本 proposal 之所以 practical 的原因：它不只描述问题，而是明确要形成一个可操作的闭环。

---

## 6. 最小可行范围（MVP）

为了让项目可控，初期版本不应该一口气覆盖所有对象类型，而应该先聚焦一个具体设定。

### 推荐 MVP：articulated household objects

例如：
- drawers
- cabinet doors
- microwaves
- fridge doors
- faucets

### 输入
- 单张图片，或单张图片加简短文本描述

### 输出
- simulation-ready articulated asset

### 目标任务
- open / close
- pull / push
- rotate handle

### 为什么这个 MVP practical

1. 相比 rigid + deformable + articulated 全覆盖，这个设定更容易 benchmark；
2. 它天然能暴露 asset-ready 和 task-ready 之间的差距；
3. joint-axis 和 hierarchy 错误更容易分析；
4. repair loop 的实现难度也明显低于完整 deformable physics。

---

## 7. 如何对生成候选执行 task-ready probes？

这里最关键的一点是：生成候选必须能被表示成“可操作资产”，而不只是一个单纯的 mesh。也就是说，一个 articulated candidate 至少需要包含以下信息：
- 可视 mesh（visual mesh）
- 碰撞几何（collision geometry）
- part hierarchy
- joint type / joint axis / joint limits
- handle 或可抓取区域等 affordance 表示
- 基本物理参数（如质量、摩擦系数）

只有在这种结构化表示下，候选资产才真正具备进入 probe evaluation 的条件。

### 7.1 两阶段 probe 评估流程

为了既保证评估有效，又避免把机器人控制误差和资产本身错误混在一起，本项目采用两阶段 probe evaluation。

#### Stage 1：Object-Centric Probe

第一阶段不直接使用完整机器人，而是先使用 object-centric 的虚拟操作者来测试对象本身的可操作性。对于 articulated household objects，可以定义标准化的 probe，如：

> reach handle → attach → pull / rotate → check open state

但这里的 “reach handle” 不一定真的需要机械臂到达，而可以简化为：
1. 在 handle center 附近创建一个虚拟控制点；
2. 通过约束把该控制点 attach 到 handle；
3. 沿预定义方向施加位移或力；
4. 观察对象是否沿正确 joint 运动；
5. 检查是否达到目标 open state，例如 drawer 打开 10 cm，或 door 转到 60°。

这样做的优点是：
- 可以快速筛掉 articulation 本身错误的候选；
- 不需要先解决机械臂 IK 和 grasp planning；
- 能更干净地测出 joint、collision、mass 等资产级错误。

#### Stage 2：Robot-Centric Probe

第二阶段只针对 object-centric probe 得分较高的少量候选，运行更真实的机器人交互测试，例如：

> reach handle → grasp → pull → open to 60°

这一阶段会引入标准化 gripper 或机械臂，并执行：
1. 根据 handle region 构造 pre-grasp pose；
2. 通过 IK 或 motion planning 让 gripper 接近 handle；
3. 执行 grasp；
4. 沿关节允许的方向执行 pull / push / rotate；
5. 检查任务是否成功完成。

这种分层测试方式的核心价值在于：第一阶段负责验证“这个资产物理上能不能被操作”，第二阶段再验证“它是否真的适合被机器人用于任务执行”。

### 7.2 具体如何运行 “reach handle → pull → open to 60°”？

对于 articulated object，这个 probe 不应被理解为一句口头描述，而应该被转成可执行的 simulator script。

#### 对 slider object（如 drawer）

如果候选提供了 slider joint axis，那么系统可以：
- 读取 handle center；
- 读取 slider axis；
- 沿 joint axis 的正负方向分别试探；
- 选择能让 joint opening 增大的方向作为 pull direction；
- 控制虚拟点或 gripper 沿该方向移动固定距离（例如 8–10 cm）；
- 记录 joint displacement、contact stability、是否卡死、是否成功达到 opening threshold。

#### 对 hinge object（如 cabinet door）

如果候选提供了 hinge axis 与 joint pivot，那么系统可以：
- 读取 handle center 与 pivot；
- 用 handle 相对 pivot 的位置估计切向开门方向；
- 沿该方向施加位移，或驱动 gripper 执行局部圆弧轨迹；
- 观察 hinge angle 是否增长到目标值（如 60°）；
- 同时记录是否出现穿透、抖动或非预期运动。

这意味着，所谓的 “open to 60°” 本质上是一个由候选资产结构自动生成的标准化脚本，而不是依赖人工逐个调试。

### 7.3 如果候选没有显式 handle 标注怎么办？

如果生成候选没有直接给出 handle region，仍然可以通过三种方式继续运行 probe：

1. **affordance predictor**：由模型额外输出 handle heatmap 或 grasp region；
2. **几何启发式**：从 movable part 的局部几何中推断可抓取区域，例如 door 的远铰链边缘、drawer 的前面板中央区域；
3. **attach-point probe**：在前期 ranking 阶段，先不要求真实 grasp，只给 movable part 指定一个候选交互点，测试它是否物理上可打开。

也就是说，候选不一定需要一开始就具备完美的语义标注，但必须能够被映射到一个可执行的 probe 接口上。

### 7.4 为什么 probe 能指导生成？

probe 的作用并不是直接反向传播到整个生成模型，而是提供结构化 failure signature，用来定位应该修哪一部分资产表示。

例如：
- 如果已经接触到 handle，但 joint 几乎不动，那么优先怀疑 joint axis / joint range / collision geometry；
- 如果 motion direction 正确，但交互中持续打滑，那么优先修 friction、mass 或 grasp region；
- 如果 object-centric probe 成功，但 robot-centric probe 经常失败，那么问题更可能出在 affordance 表示或 graspability，而不一定是 articulation 本身。

因此，simulator feedback 会先被转成 failure diagnosis，再映射到局部 repair，而不是直接去更新整个生成网络。

### 7.5 最 practical 的执行策略

为了让系统真正可跑，本项目建议采用如下筛选流程：

1. **静态合法性检查**：先筛掉导入失败、joint 非法、无 movable part 等明显错误的候选；
2. **object-centric probe ranking**：对所有剩余候选运行快速虚拟交互测试；
3. **robot-centric probe validation**：只对 top-K 候选执行完整机器人交互；
4. **failure-guided repair**：根据 probe 结果，只修改最可疑的 articulation、collision、physical parameter 或 affordance 变量；
5. **re-test**：重新运行 probe，比较修复前后 task-readiness 的提升。

这样的设计既能支持大规模 candidate ranking，又能保证 simulator feedback 最终能够真正指导生成与修复过程。

---

## 8. 预期贡献

### Contribution 1
提出一个 **task-ready evaluation framework**，把 physical 3D asset generation 的评估从“能不能进仿真器”推进到“能不能支撑任务”。

### Contribution 2
给出一套系统性的 **failure taxonomy**，用于分析生成物理资产在 embodied interaction 中的失败模式。

### Contribution 3
研究 **asset-level metrics** 与 **task-level performance** 之间的关系，明确当前 benchmark 在哪里高估了下游可用性。

### Contribution 4
提出一个 **simulator-in-the-loop repair pipeline**，能够基于交互反馈自动修复生成资产。

### Contribution 5
为 robotics simulation、embodied agents 和 world model training 提供一条更可靠的 task-ready asset pipeline。

---

## 8. Practical Impact

本 proposal 的价值主要体现在三个层面。

### 学术价值
它指向了当前文献中的一个明确空白：

> 现有工作已经能够生成 simulation-ready 资产，但仍缺乏一套系统框架，去评估并提升这些资产在 embodied tasks 中的真实可用性。

### 工程价值
如果这套方法成功，它可以直接服务于：
- robotics simulation，
- synthetic data generation，
- interactive virtual world construction，
- embodied world model training。

### 方法论价值
它试图把问题的核心从：
- “我们能否生成一个看起来合理的物理对象？”

推进到：
- “我们能否生成一个能够稳定支持交互与任务执行的对象？”

这一步既 practical，也非常必要。

---

## 9. 总结

总的来说，这个项目的目标，是把 physical 3D asset generation 从 **simulation-ready** 推进到 **task-ready**。

它不再只停留在 geometry、articulation 和 export format，而是进一步追问：一个生成资产能否真的支撑抓取、开合、推拉、形变以及更一般的 task-level interaction。为了解决这个问题，本项目将构建 task-level benchmark，分析失败模式，并设计 simulator-in-the-loop repair loop。

最终目标不是简单生成更多 3D 资产，而是生成真正对 embodied AI 有用的资产。
