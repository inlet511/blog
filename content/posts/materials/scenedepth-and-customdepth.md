---
title: "关于深度"
date: 2022-10-21T23:59:09+08:00
draft: false
toc: true
categories: [材质]
series:
  - 材质节点
tags: 
  - SceneDepth
  - CustomDepth
  - 材质编辑器
---

## 什么是Custom Depth
UE中对物体勾选“Render CustomDepth Pass” 就会让对象渲染到一个特殊的深度Buffer中，即CustomDepth。在这个特殊的深度Buffer中，勾选了"Render CustomDepth Pass"物体的像素范围的深度值和普通的depth buffer是一样的，但是其他的部分则是<font color="#D2691E">用一个极大值填充</font>，类似于对这些物体进行了“抠图”，但是“透明区域”的深度值不是0，而是一个很大的数值。
![scene depth](./scenedepth.png)
![custom depth](./custom-depth.png)
如上图，只给角色和其脚下的一个方块勾选了custom depth，则custom depth中除了这两个物体之外的部分都是填充了一个极大值，但是在图中是以黑色显示。

## Custom Depth应用
如果希望给远处的天空球背景做个蒙版，可以这样操作：
![](./mask_01.png)
用custom depth减去scene depth，勾选了"Render CustomDepth Pass"的物体的区域，custom depth和scene depth深度相同，相减为0，而周边的黑色区域相减仍然是一个极大的值。接着对整个画面使用了saturate，等价于clamp(0,1), 将结果钳制在(0,1)之间，那么周边区域则为1，物体范围为0.
![](./mask_02.png)

## 场景深度的应用
如果希望给远处天空球制作一个蒙版，可以这样操作：
![](./mask_03.png)
场景深度除以一个较大的距离值（这里是参数Distance，默认1024，可以在材质实例中调整)，则这个距离范围内的值变为0-1。而这个范围之外的值均大于1。
使用floor，可以把这个距离范围内的区域深度值变为0（因为他们原本为0-1），而这个范围之外的值则为大于等于1的整数，最后再使用clamp(0,1)，则这个距离范围内为0，范围外为1.
因此只要这个距离值选的比较合适（即覆盖住了场景中的所有物体，又没有大到包含住天空球的距离），就可以得到一个远处天空球的蒙版

![](./mask_04.png)




