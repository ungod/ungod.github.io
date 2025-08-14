---
layout: post
title: MotionMatching
tags: [游戏, 动画]
categories: [游戏, 动画]
published: false
math: true
typora-root-url: ..
---





## Motion Matching



现代动画技术中，Motion Matching是个绕不开的话题。为了解决动画过渡自然和真实性，动画系统从传统Locomotion状态机到混合节点，到Pose Matching、Motion Matching，从最初的特征向量基础到现在的ML/DL/RL的混合式变种。近年也很多GDC都有详细的介绍各种工程落地：

- [Reinforcement Learning Based Character Locomotion in Hitman: Absolution (GDC2013) - by Michael Buttner](https://gdcvault.com/play/1017727/Reinforcement-Learning-Based-Character-Locomotion)
- [Motion Matching and The Road to Next Gen Animation (GDC2016) -  by Simon Clavet](https://www.gdcvault.com/play/1023280/Motion-Matching-and-The-Road)
- [Animation Bootcamp: Motion Matching: The Future of Games Animation (GDC2016) by Kristjan Zadziuk](https://gdcvault.com/play/1023478/Animation-Bootcamp-Motion-Matching-The)
- [From Motion Matching to Motion Synthesis, and All the Hurdles In Between (GDC2019) by Fabio Zinno](https://gdcvault.com/play/1025890/ML-Tutorial-Day-From-Motion)
- [Ragdoll Motion Matching (GDC2020) by Simon Clavet](https://gdcvault.com/play/1026712/Machine-Learning-Summit-Ragdoll-Motion)
- [Motion Matching in 'The Last of Us Part II' (GDC2021) by Michal Mach, Maksym Zhuravlov](https://gdcvault.com/play/1027118/Motion-Matching-in-The-Last)
- [Take 'CONTROL' of Animation (GDC2021) - by Ilkka Kuusela, Ville Ruusutie](https://gdcvault.com/play/1027146/Animation-Summit-Take-CONTROL-of)
- [Environmental and Motion Matched Interactions; 'Madden', 'FIFA' and Beyond! (GDC2021) by Henry Allen](https://gdcvault.com/play/1027465/Animation-Summit-Environmental-and-Motion)
- [Machine Learning Summit: AI Animator : A Real Time Motion Completion System (GDC2022) by Yinglin Duan](https://gdcvault.com/play/1027981/Machine-Learning-Summit-AI-Animator)
- [Controlled Chaos: The Combat of 'Marvel's Guardians of the Galaxy' (GDC2022) by Rodrigo Barros Lima, Olivier Tremblay-Ross](https://gdcvault.com/play/1027917/AI-Summit-Controlled-Chaos-The)
- [Motion Matching at EA: Five Years Later (GDC2023) by JC Delannoy](https://gdcvault.com/play/1028945/Animation-Summit-Motion-Matching-at)
- Technical Animation Pipeline of 'Fort Solis' (GDC2024) by Matthew Lake 
- [MotorNerve: A Character Animation System Using Machine Learning (GDC2024) by Songnan Li, Yuchen Liao](https://gdcvault.com/play/1034685/MotorNerve-A-Character-Animation-System)





### Reinforcement Learning Based Character Locomotion in Hitman: Absolution

这个视频是2013年游戏开发者大会（GDC）的一个演讲，由Michael Buttner主讲。这是GDC最早讲述Motion Matching类似的技术原理，或者说是技术雏形，此视频极有观看价值。传统游戏角色移动系统通常依赖“手动混合树”和大量游戏代码来控制和微调动画（如从走路到奔跑的过渡）。这些方法效率低下，尤其在处理复杂场景时（如人群中的AI角色）。Buttner分享了他们在《杀手：赦免》中采用的一种新型方法，使用强化学习来自动化和优化Crowd的动画和移动。数据驱动是这个技术的核心，视频是把其称为Motion Graph，或者是Pose Matching，主要是应用到了其Crowd中（当年玩杀手系列的海量可交互Crowd确实震撼）。



![img](/assets/postasset/2024-08-07-MotionMatching/198a9690ca0ba.jpg)



上图是简单表述这个技术的关键思想（Key Idea）

- 给定一定数量的动画，我们需要通过识别匹配的姿势（poses）来找到可能的过渡（transitions）。

- 构建一个有向图（directed graph），其中边（edges）对应于运动的片段（fragments of motion）。

  

Motion Graph同时其描述了四个跟传统动画节点的特点：

1. **自动检测匹配姿势**：Motion Graph 通过自动识别相似姿势生成过渡，避免手动规则，确保动画自然流畅。

2. **基于物理属性的控制器逻辑**：系统依赖动画数据的物理特性（如速度、角度）构建逻辑，实现更真实的运动模拟。

3. **无需特定动画知识**：控制器不需预知动画细节，仅靠数据搜索和匹配，简化扩展和维护。

4. **数据驱动系统**：以动画数据为导向而非规则，提升灵活性和可扩展性，自动优化质量。

   

![img](/assets/postasset/2024-08-07-MotionMatching/198a98c2ff1bc.jpg)



上图展示动画帧的差异度量。这是一个度量函数，用于计算两个动画帧（如A_i和B_j）之间的“不相似度”D(A_i, B_j)。在Motion Matching中，不相似度越低，意味着两个帧越匹配，越适合作为过渡点。在构建Motion Graph时，系统需要扫描大量动画数据，找出相似部分来创建边（transitions）。这个度量就是“相似性判断器”，确保过渡自然，避免动画“跳跃”或不协调。D(A_i, B_j)是“不相似度分数”，用于比较动画序列A中的第i帧和序列B中的第j帧。分数越高，两个帧越不相似（e.g., 角色姿势差异大）；分数越低，越可能匹配。



不是单看一帧，而是取一个“窗口”（比如前后几帧），将它们转化为“点云”（point clouds）——一种3D点集合，代表角色关节的位置（如肩膀、膝盖等关键点在空间中的坐标）。动画是连续的，单帧匹配容易忽略上下文。通过窗口点云，系统能捕捉运动趋势（e.g., 角色在加速还是减速），让匹配更准确、更动态。同时度量中加入了“导数信息”（derivative information），即速度、加速度等变化率（数学上，导数表示）。这意味着不只比较静态位置，还考虑动态因素（如关节移动的速度）。



最终分数是“加权平方距离和”。对应点（e.g., A帧的左手点 vs B帧的左手点）计算欧几里得距离，然后加权求和（权重可能根据关节重要性调整，如腿部权重高于手指）。



![img](/assets/postasset/2024-08-07-MotionMatching/198a993165585.jpg)

上图是一个“误差函数”，通过采样方式计算两个动画序列中所有帧对的误差，形成一个2D表面（像热力图）。误差最低的点（局部最小值）被视为“良好过渡点”。图片中展示了示例动画序列（如“WalkFTurnR135”可能指“走路前转右135度”，“Idle-WalkRF”指“从idle到走路右前”），并用灰度图像表示误差函数：深色区域表示低误差（好匹配），浅色表示高误差（差匹配）。



系统对两个动画序列（如A和B）的**每一对帧**（e.g., A的第i帧 vs B的第j帧）计算误差，使用之前提到的Dissimilarity Metric（不相似度）。这些误差值组合成一个2D函数（像一个网格或表面图），x轴和y轴分别代表两个序列的帧索引。图片中的灰度图像就是这个2D函数的采样表示。左侧是“WalkFTurnR135 vs WalkFTurn180”的误差图，右侧是“Idle-WalkRF vs WalkFTurnR135”。深色斑点表示低误差区域（帧对高度匹配），形成“山谷”状的局部最小值。这不是简单的一对一比较，而是全面扫描所有可能配对，形成一个可优化的“误差景观”。在游戏中，这能快速定位最佳过渡，而不需要手动检查成千上万的帧。



