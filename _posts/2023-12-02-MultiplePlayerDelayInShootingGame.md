---
layout: post
title: 射击游戏的延迟对抗
tags:[游戏,网络]
categories: [游戏, 动画]
published: false 
typora-root-url: ..
---

实现网络实时对战游戏的时候，延迟是一个躲不开的难题，总是需要各种经验和技巧。只要是发布在互联网上的游戏，延迟就客观存在，以目前基础技术来说延迟不可能消除。假设光纤以光速一秒绕地球7圈半来说，1000ms/7.5=133ms，就算用光纤直连也有133ms的延迟（严谨一点的话应该算半球，也有66ms），而且互联网庞大且复杂，还要把同一根光纤的网络拥堵和路由器协调运算时间算进去。要做世界服务器网络延迟不可避免，除了搭建区间服务器外，就是需要做延迟对抗的优化。

在快节奏的游戏中，延迟的存在必然影响手感，就算几毫秒也能感受出来。如果让玩家感受到延迟的存在，就会觉得输入反应很慢。网络必然会有波动，不稳定的网络会让玩家感觉一卡一卡的，不流畅感会严重影响玩家感受。每个玩家的延迟都不一样，高延迟玩家有可能影响低延迟玩家的体验，这是非常不公平的，一些ACT游戏比如黑暗之魂中的“延迟战士”，高延迟的玩家因为延迟而低延迟玩家看不到对方真实位置，却能轻易击杀对方，甚至有人会恶意把自己延迟调高。现在很多非重度PVP游戏这方面都没做得很好。不过也没法做到完全公平，后面会提及到。所见即所得，比如玩家的准星明明打中，特效也出来了，对面却没扣血。逻辑的触发都是基于玩家所见，不能因为延迟的存在而忽略所见合理性。





## 基本Client-Server模型

常说的CS同步方式，即Client-Server模型，其实就是来自CS（不是🤣）。Valve的Source引擎，也就是半条命/CS的引擎，网络模型被几乎所有的现代射击游戏都被使用参考和迭代。里面包含平滑插值，输入预测和延迟补偿的那几个概念。Valve的游戏一般每秒以20-60个数据包，也即是20Hz到60Hz的频率进行网络同步。同时缓存100毫秒的快照，用作网络回滚。详情可以查看Source引擎相关文档：[https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)

![image-20231202163010983](/assets/postasset/2023-12-02-MultiplePlayerDelayInShootingGame/image-20231202163010983.png)
_SourceEngine的网络模型_



当然上面的TickRate都是Source的默认值，或者说是老版CS的值。不同的游戏设定的TickRate都不一样，或者是为了竞技性，或者是削弱竞技性，或者是为了成本，高Tick必然能改善游戏体验，包括能降低错误预测带来的影响，同步位置更准确，游戏更公平，后面会提及，当然肯定会提高服务器成本。

| Counter-Strike: Source -- 66                                 |
| ------------------------------------------------------------ |
| Team Fortress 2 -- 66                                        |
| Left 4 Dead -- 30                                            |
| Division 2 -- 30                                             |
| Overwatch -- 60                                              |
| Halo Reach -- 60                                             |
| Counter-Strike: Global Offensive -- 可调整，一般60，现在很多私人服务器128 |
| Valorant -- 128                                              |

拳头的Valorant的TickRate达到惊人的128，这也是他们对外宣传重点。



不管同步频率是多少，同步始终是离散的假设20Hz的同步频率，那就是每50ms同步一次，必然会让游戏感觉断断续续和抖动，可以想象一下20FPS的游戏的流畅感有多差。网络丢包的情况影响更大，快照之间必须有平滑插值



![image-20231202165727807](/assets/postasset/2023-12-02-MultiplePlayerDelayInShootingGame/image-20231202165727807.png)
_快照的平滑插值_



插值平常又分为内插值和外插值（预测）：

- 内插值(Interpolation)指的是在已知快照之间进行插值，插值方法如线性插值、指数插值等。
- 外插值(Extrapolation)指的是根据已知快照，推断丢失快照的值（如丢包），或者是推测未来快照（如输入预测），插值方法如Dead Reckoning算法，或者是现在已被多次落地的神经网络算法。可见论文：[https://uwspace.uwaterloo.ca/bitstream/handle/10012/16960/Walker_Tristan.pdf?sequence=3](https://uwspace.uwaterloo.ca/bitstream/handle/10012/16960/Walker_Tristan.pdf?sequence=3)

![image-20231202170714303](/assets/postasset/2023-12-02-MultiplePlayerDelayInShootingGame/image-20231202170714303.png)
_Dead Reckoning预测算法Projective Velocity Blending的实现_



然后当然，实时游戏都习惯使用UDP来作为收发网络包。TCP虽然是可靠的但是因为重发机制导致传输效率低下，UDP更适合实时游戏。现在有很多可靠的UDP实现，如KCP，都是为了解决冗余的TCP包及其繁琐的连接校验流程，并且有足够的参数让对应的游戏有优化空间，但是可靠UDP依然有丢包的延迟损耗。如果对延迟做到极致，不可靠UDP最合适，不可靠UDP不对数据包进行重发，而是忽略该包，服务器做输入外插值，尽可能减少来回的延迟。守望先锋使用的是不可靠UDP，很多FPS游戏都是不可靠UDP。





## 预测与回滚
