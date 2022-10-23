---
title: "扰动贴图"
date: 2022-10-23T17:22:40+08:00
draft: false
toc: true
categories: [材质]
series:
  - 材质节点
tags: 
  - 材质
  - Postprocess
  - UE
  - motion4waychaos
---

## 效果
Motion_4WayChaos是引擎自带的一个Material Function，需要在ContentBrowser的选项种开启"Show Engine Content"来使用。
它本身的作用是对贴图进行4向的混乱动画，效果仅仅是简单的叠加，并不是非常优秀。
![原本效果](./origin.gif)
一个更好的用法是：
1. 将它应用于黑白噪波图进行运动动画
2. 然后将动画叠加到uv上
3. 用uv动画来驱动贴图动画
   
这样就可以实现更好的扰动效果，例如水面的波动。
先看效果：
![更好的效果](./result.gif)

## 使用
在材质编辑器中创建一个材质函数节点，然后在它的Material Function中选择Motion_4WayChaos。
![nodes](./nodes.png)
Speed是扰动速度。
Divisor是决定扰动强度的值，越小扰动越轻微。
Texture则是用来进行扰动的噪波贴图，这里我使用的是引擎自带的LowResBlurredNoise，它的效果不错。
只取输出的结果的一个通道，将他和一个UV进行叠加，并用这个叠加后的uv驱动一张贴图(这里使用的是棋盘格)，即可得到最终的效果。



