# PhysX-Omni：统一生成可仿真的 rigid、deformable 与 articulated 3D 资产

最近连续读下来，一个非常明显的趋势是：physical 3D generation 正在从“某一类对象能不能做好”，慢慢走向“能不能做成一个统一框架”。PhysX-Omni 给我的第一感觉就是，它不想再只解决 rigid object 或 articulated object 的局部问题，而是想把 **刚体、可变形体、关节物体** 全部拉进同一个 simulation-ready generation 范式里。

这件事本身就很难，因为三类对象背后的几何形式、物理属性和交互方式差别都很大。也正因为如此，读这篇文章时会很明显地感觉到：它的野心已经不只是“生成一个 3D 物体”，而是在试着往更一般的 **physical world modeling** 迈一步。

这篇文章主要整理我读 **PhysX-Omni: Unified Simulation-Ready Physical 3D Generation for Rigid, Deformable, and Articulated Objects** 时的一些想法。

## 方法主线：把 simulation-ready generation 做成统一范式

一句话概括，作者提出了一个统一框架，让模型能够从图像出发，生成适用于 **rigid、deformable、articulated** 三类对象的 simulation-ready 3D 资产。

我最看重的，不只是方法本身，而是它重新定义了问题：不是“某一类 3D 物体应该怎么生成”，而是“能不能用统一表示和统一流程，把 physical 3D generation 系统化”。

### 核心问题

过去很多 simulation-ready 3D 方法，其实只覆盖某一种对象类型。比如有的方法擅长刚体，有的方法关注 articulated object；可一旦场景转向开放世界，环境里显然不可能只有一种物体。

于是，现有系统在 unified physical 3D generation 上经常暴露出这些缺口：

- 不同物体类型使用不同表示，难以统一建模
- 物理属性与几何结构往往分开处理，整体链路并不一致
- 很多方法更强调视觉重建，而不是仿真可用性
- 缺少能同时覆盖多种对象类型的大规模数据和统一评测基准

### 核心方法

论文主线大致可以概括成三件事：

1. **统一生成框架**  
   基于 VLM 做从全局到局部的多轮推理，同时生成几何、尺度、材质、可供性和运动学等信息。  
2. **统一几何表示**  
   作者提出了一种基于模板的 **RLE 文本表示**，通过沿 z 轴切片编码体素，让高分辨率 3D 结构能被语言模型直接处理，而且不需要引入特殊 token。  
3. **统一数据与评测**  
   构建 **PhysXVerse** 数据集与 **PhysX-Bench** 基准，把 geometry、scale、material、affordance、kinematics、function description 等维度统一纳入评估。

## 从“某类物体能生成”到“开放世界对象都能生成”

对我来说，最有意思的就是这一层变化：问题不再停留在单一对象设定，而是被抬升到了更一般的 world modeling 视角。

### 为什么这点重要

如果目标是机器人仿真、交互式场景生成，或者 embodied world model，那么系统面对的对象一定是混杂的：

- 有的是 rigid object
- 有的是 deformable object
- 有的是 articulated object
- 它们不仅形态不同，物理行为也完全不同

因此，一个真正有用的 physical 3D generation 系统，最好不要只在某个单一子类上成立；更理想的情况是，它还能统一刻画“不同物体类型共享什么、差异又在哪里”。

**解决方案**

PhysX-Omni 的思路是：先在表示层把问题统一，再在数据和评测层把问题统一，最后才在生成层真正做到统一。

- 在表示层，它用 RLE 风格的文本化几何表示兼容 VLM 的处理范式。
- 在数据层，它提出 **PhysXVerse**，覆盖更一般的 simulation-ready 资产。
- 在评测层，它提出 **PhysX-Bench**，从六个维度判断“这个资产到底是不是真的可用”。

![PhysX-Omni teaser](./pics/physx-omni-teaser.png)

这张 teaser 最打动我的地方在于，它不只是展示“能生成哪些物体”，而是在强调：同一个框架下，刚体、软体、关节物体都被纳入了 simulation-ready generation 的统一叙事里。

**个人思考**

在我看来，这篇文章真正有价值的地方，是它把“统一性”放到了一个很高的位置。很多时候，做出一个局部最优解并不算太难；真正难的，是给出一个对开放世界更有延展性的统一方案。PhysX-Omni 至少在问题设定上，已经朝这个方向迈出了非常关键的一步。

## 论文的其他亮点

### PhysXVerse 数据集

论文提出了 **PhysXVerse**。据文中介绍，它包含 **8.7K+** 个 simulation-ready 3D 资产，覆盖 **2.9K+ categories**。再加上来自 PhysXNet、PhysX-Mobility 等数据，整体训练集规模达到 **42K+** 个仿真就绪物体。

这件事很重要，因为统一方法如果没有统一数据支撑，最后往往容易退化成“概念统一、训练不统一”。

### PhysX-Bench 评测基准

我很喜欢作者把 benchmark 也一起做出来。PhysX-Bench 不只看几何，还会同时检查：

- geometry
- scale
- material
- affordance
- kinematics
- function description

和单纯比较 Chamfer Distance 或 F-score 相比，这样的评测更接近“物体是否真的能进入仿真与交互”的目标。

### 一些关键结果

文中有几个数字很值得记住：

- PhysXVerse：**8.7K+ assets / 2.9K+ categories**
- 总训练数据：**42K+** simulation-ready objects
- PhysXVerse 上：**PSNR 21.52 / CD 2.95 / F-score 91.28**
- 绝对尺度误差：**2.79**
- 运动学分数：**0.9185**

在 PhysX-Bench 上，PhysX-Omni 还给出了：

- Absolute Scale：**64.26**
- Material：**59.89**
- Affordance：**70.57**
- Kinematics：**80.72**
- Description：**39.02**

这些结果说明，它不是只在单一几何指标上占优，而是试图在“统一物理理解 + 统一生成”这件事上建立系统优势。

## 这篇工作还没有完全解决的问题

当然，PhysX-Omni 的 ambition 很大；而 ambition 越大，边界条件往往越值得认真看。

### 统一框架不代表所有子问题都同样容易

刚体、可变形体、关节物体被纳入同一框架，当然很漂亮；但这三类对象本身的难度差异也非常大。统一表示和统一流程并不自动意味着每一类对象都被同样好地解决了。

也就是说，“统一”本身是一项贡献，但它也可能带来新的 trade-off：为了统一，某些对象类型的细节刻画难免会被简化。

### 复杂结构与细粒度几何仍有提升空间

论文也明确提到，对于 **highly complex structures and fine-grained details**，几何质量仍然有继续提升的空间。

这其实不难理解：当方法同时兼顾物理属性、尺度、材质、可供性和运动学时，模型容量与表示预算不可能无限，细节几何往往就是最容易被牺牲的部分之一。

### 统一评测仍然很难真正完备

PhysX-Bench 已经很有价值了，但像 material、affordance、function description 这类属性，本身就比纯几何更难定义，也更难评估。即便已经有 benchmark，也不代表“这个物体在真实交互里是否真的合理”这件事已经被完全衡量清楚。

所以我更愿意把它理解成：作者已经把 evaluation target 扩展得更像世界了，但距离完整刻画 world usability 仍然有一段路。

### 更偏统一物理建模，未必总是最强外观建模

论文也提到，这个方法更强调统一的物理理解与 simulation-ready generation，而不是纯粹面向外观的几何预训练。因此，在某些只强调视觉外观的几何指标上，它未必一定压过专做 appearance-oriented 3D generation 的方法。

这不算缺点，更像是一种主动取舍：它选择了更接近“可用性”的目标，因此也不再只是为了把某个视觉重建分数推到极致。

## 总结

如果说前一类工作是在问“怎么把某种对象做成 simulation-ready asset”，那 PhysX-Omni 更进一步，问的是：**能不能把 simulation-ready physical 3D generation 这件事本身做成一个统一框架？**

最让我印象深刻的，不是某个单独模块有多花哨，而是它把 **统一表示、统一数据、统一评测、统一生成** 这四层东西放在一起考虑了。

对关心 embodied AI、机器人仿真和物理世界建模的人来说，这篇文章很值得看，因为它在认真尝试把“开放世界里的多类对象”都纳入同一个 physical 3D generation 叙事里。

## 参考

- [Project Page](https://physx-omni.github.io/)
- [论文原文链接](https://arxiv.org/abs/2605.21572)
