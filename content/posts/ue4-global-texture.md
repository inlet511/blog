---
title: "Ue4 Global Texture"
date: 2022-08-28T17:06:11+08:00
draft: false
toc: true
categories: [UE4]
tags:
    - UE
    - GlobalTextures
    - C++
series:
    - UE4图形开发系列
---

*UE4版本:4.26.2*

>前置知识
> - 从源码编译UE4引擎

## 全局纹理
UE代码中经常看到GEngine->DefaultTexture之类的全局纹理，可以在引擎代码的任何地方方便的引用，列举如下：
    - DefaultTexture
    - DefaultDiffuseTexture
    - HighFrequencyNoiseTexture
    - DefaultBokehTexture
    - DefaultBloomKernelTexture
    - PreIntegratedSkinBRDFTexture
    - MiniFontTexture
    - WeightMapPlaceholderTexture
    - LightMapDensityTexture
    - BlueNoiseTexture
  
本篇分析自己如何添加一个这样的纹理。

## 一、制作资源
经过分析发现其他纹理资源都放在引擎的Engine\Content\EngineResources目录下，因此我们在任意工程中先显示引擎资源(Show Engine Content)，向该路径(Engine/EngineResources)下导入一张纹理，假设纹理名称为GradientTexture。

## 二、声明
在Engine.h文件中，UEngine的public区域添加贴图的指针以及名称，这里命名为GradientTexture
```cpp
	UPROPERTY()
	class UTexture2D* GradientTexture;

	UPROPERTY(globalconfig)
	FSoftObjectPath GradientTextureName;
```

## 三、加载
在UnrealEngine.cpp中，搜索到LoadEngineTexture，发现一组加载纹理的语句，仿照着添加加载新纹理的语句
```cpp
LoadEngineTexture(GradientTexture,*GradientTextureName.ToString());
```

## 四、配置
GadientTextureName是从哪里来的？答案是配置文件，因为声明GradientTextureName的时候在UPROPERTY中指定了globalconfig。打开Engine\Config\BaseEngine.ini文件，在[/Script/Engine.Engine]区域添加一行配置:
```ini
GradientTextureName=/Engine/EngineResources/GradientTexture.GradientTexture
```

重新编译引擎，我们就可以在任意c++代码中使用GEngine->GradientTexture 获取到该纹理了。

