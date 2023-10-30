---
layout: post
title: 叠加动画原理细节
tags:
published: false
---

叠加动画常用于战斗受击、射击抖动、AO等。其作用是能同时两个动画的表现，如角色在跑步的时候受击，跑动动画是不能停的同时也要有被击中的抽搐表现；又如角色瞄准的时候射击，射击会因为武器的后坐力让角色躯体发生抖动，而角色也会下意识地维持着瞄准动作……叠加动画几乎占据角色动画逻辑的一大部分，不过似乎很少人讨论有关叠加动画的技术细节。本以UE实现方法为例，意在分析叠加动画的原理和一些细节。



# 基本原理

不管是Unity、UE还有大部分现代游戏引擎，动画系统的实现都少不了叠加动画，不过他们使用方法都不大一样。UE[^1]因为有动画蓝图的存在，叠加动画可以让动画TA以任意的前后逻辑叠加。Unity[^2]和CryEngine[^3]则通过动画层的方法，让叠加动画放置在一个单独的动画层上。

当然实际上叠加动画的实现都几乎相同，用比较粗浅的描述，就是把一个基础的动画为底，”加“上别的表演动画，而表现动画常常不是通过美术制作出来的，是通过表现动画本身，在”减“去基础姿势得出的。

以ALS[^4]的实现为例，这个动画框架本身实现了一个较为复杂的分层（将头、腰、腿、手都分了叠加层），但是它的最终混合实现都是：Overlay + (BasePose - BaseInput) = Output。框架中的分层结构是以Overlay作为最基础动作，BasePose即Locomotion动作，减去BaseInput即Idle单帧动作，求出叠加动作，然后Overlay加上刚刚的叠加动作，得到最终动画。如图所示：

![image-20231030152534727](C:\Github\assets\postasset\2023-09-10-Additive Animation\image-20231030152534727.png)
_ALS的分层叠加流程_

以动画的矩阵乘法来看，”+“是矩阵变换，即基础动画S左乘叠加动画A。”-“则是基础动画S左乘参考动画R的逆，即$SR^-1$。一般来说都会把矩阵拆分为SQT来进行运算，一个是为了性能，一个是为了插值，见游戏引擎架构[^5]。

$$
D_j = S_jR_j^-1 \tag{1}
$$

$$
A_j = D_jTj = (S_jR_j^-1)T_j \tag{2}
$$

其中$D_j$是计算出来的叠加动画，$S_j$是叠加源动画，$R_j$是参考动画。$A_j$是最终应用的叠加动画，$T_j$是被应用的基础动画。





# Mesh Additive和Local Additive

叠加动画中，UE区分了Mesh Addiive和Local Additive。Local Additive就是最常规的按父骨骼叠加，也是最普遍的一种叠加方式。有趣的是UE提供了Mesh Additive的叠加方式，这是非常有用的一项特性，在官方例子中广泛应用，不过很多时候动画TA都知道其然不知到其所以然。



## 官方解释

官方文档[^6]中使用了AimOffset作为举例，描述了Mesh Additive的应用。此例子突出了在使用AO的时候通过Mesh Additive方式，让角色的倾斜动画（Lean）不会影响到正常瞄准方向：

1. 带倾斜动画

2. 倾斜动画带AimOffset的Local Additive叠加，瞄准方向会随着倾斜角度偏移

3. 倾斜动画带AimOffset的Mesh Additive叠加，瞄准方向不会随着倾斜角度偏移

![image-20231030160646252](C:\Github\assets\postasset\2023-09-10-Additive Animation\image-20231030160646252.png)
_官方AimOffset例子_



官方例子是个应用，原理表述上实在模糊。不理解其原理，也没法对这项技术举一反三。



## 源码分析

本文公式（1）中是生成叠加动画的原理，在UE中其主要源码如下：

![image-20231030161455992](C:\Github\assets\postasset\2023-09-10-Additive Animation\image-20231030161455992.png)

上面这代码是Make Dynamic Additive节点的逻辑，AnimSequence实际上生成的叠加动画也是一样的逻辑。

可以看到bMeshSpaceAdditive为是否使用MeshAdditive的标志位，Mesh Space Additive的区别就是变换的Rotation部分会转换成Mesh空间，即模型空间。



![image-20231030162213952](C:\Github\assets\postasset\2023-09-10-Additive Animation\image-20231030162213952.png)

上图是Apply Mesh Additive动画节点的主要源码部分，可见Local Additive与Mesh Additive的区别在于：**把矩阵的旋转部分在叠加运算前，转换到Mesh空间**。更细节解释，Local Additive是把叠加动画的运算是直接在本地空间做的，而Mesh Additive是在Mesh空间做的。

源码的解释可以说是简单直接了，就是运算空间的不一样。但是即使Local Additive还是Mesh Additive，骨骼计算完叠加动画后，依然会受到父骨骼的影响，直觉来说Mesh Additive不会产生官方例子解释的这种情况——AO不受倾斜影响。



## 简化模型实例

用线性代数，通过运算的方式来解释上述情况不是难事，不过也不大直观。我这里通过一个简化的动画运算模型，来描述Mesh Additive期间运算流程，就知道为什么会产生官方文档的应用效果了。



### 参考


[^1]: Unreal Engine Apply Additive :https://docs.unrealengine.com/5.3/en-US/animation-node-technical-guide-in-unreal-engine/#poseinputs
[^2]: Unity Animation Layer  https://docs.unity3d.com/Manual/AnimationLayers.html
[^3]: CryEngine Additive Animation : https://docs.cryengine.com/display/CEMANUAL/Additive+Animations
[^4]: Advanced Locomotion System :https://github.com/dyanikoglu/ALS-Community#advanced-locomotion-system---community-version
[^5]: 游戏引擎架构【美】Jason Gregory(杰森.格雷戈瑞)，章节11.6.5，译本490页
[^6]: Mesh Space Additive: https://docs.unrealengine.com/5.3/en-US/aim-offset-in-unreal-engine/#meshspaceadditive