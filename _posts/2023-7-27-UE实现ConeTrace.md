---
layout: post
title: UE实现ConeTrace
published: false
typora-root-url: ..
---

有时候实现Gameplay逻辑的时候，会要求判断屏幕的圆是否与场景的碰撞体相交（如瞄准辅助）。屏幕的圆与场景碰撞相交本质是视锥体与场景物体是否相交。此时ConeTrace能解决此问题，然而UE并不支持，Physx貌似也没有此类基本几何体。本文意在实现快速的ConeTrace。



## 实现的几种方案

