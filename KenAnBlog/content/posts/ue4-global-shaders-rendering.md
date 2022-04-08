---
title: "Ue4 Global Shaders - 渲染"
date: 2022-04-08T17:24:36+08:00
draft: false
toc: true
categories: [UE4]
---
*UE4版本:4.26.2*

>前置知识
> - 了解图形学中的顶点着色器和像素着色器
> - UE4中创建和配置插件的方法
> - UE4C++
> - [上一章节]({{< ref "posts/../ue4-global-shaders-creation" >}})

## 使用Global Shaders
目前我们的GlobalShaders仅仅是输出一个纯色，为了让它起作用，我们将这样使用该Shader:
- 创建一个RenderTarget(渲染对象)