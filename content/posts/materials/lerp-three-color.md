---
title: "Lerp3Color"
date: 2022-10-23T15:11:59+08:00
draft: false
toc: true
categories: [材质]
series:
  - 材质节点
tags: 
  - 材质
  - Postprocess
  - UE
  - lerp3color
---
## Lerp_3Color
![node](./node.png)
该节点的作用类似于lerp，但是lerp是对两个数值进行线性插值，而这个节点是使用一个数值对3个数值进行线性插值。
当Alpha值处于0-0.5之间会根据A和B进行线性插值，Alpha值处于0.5-1之间会根据B和C进行线性插值。
下图这样的设置很容易测试出该节点的作用。
![color](./color.png)
如果将Alpha换成一个渐变的数值，则会将渐变映射为3种不同的数值/颜色，例如：
![node2](./node2.png)
![color_gradient](./color_gradient.png)

