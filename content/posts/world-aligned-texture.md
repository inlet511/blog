---
title: "贴图对齐世界"
date: 2022-10-23T14:23:09+08:00
draft: false
toc: true
categories: [材质]
tags: 
  - 材质
  - UE
---
## WorldAlignedTexture
材质编辑器中这个节点功能很强大，可以将贴图对齐到世界坐标。
![](../main.png)
TextureObject和TextureSize即要贴的贴图和贴图尺寸。
WorldPosition应输入世界的坐标，默认值就是AbsoluteWorldPosition。
其中WorldSpaceNormal就是决定贴图朝向的，所以这里应该输入SceneTexture:WorldNormal，只有这样才能正确贴到世界坐标上。
输出三个节点，分别是XY, Z, 和XYZ. XY只保证在垂直的面上贴图正确分布，Z只保证在水平的面上贴图正确分布，而XYZ则保证在任何面上贴图都正确分布，但是缺点是可能会有一些接缝。
![XY](../xy.png)
![Z](../z.png)
![XYZ](../xyz.png)


