---
title: "UE4 Global Shaders - 01 创建"
date: 2022-04-08T11:09:14+08:00
toc: true
categories: [UE4]
tags: 
  - 图形学
  - UE
  - GlobalShader
  - C++
series:
  - UE4图形开发系列
---
*UE4版本:4.26.2*

>前置知识
> - 了解图形学中的顶点着色器和像素着色器
> - UE4中创建和配置插件的方法，了解插件的文件结构
> - UE4C++

Global Shaders 是只应用于特定模型(例如全屏Quad),不需要和材质交互的Shader，每个Global Shader在一个项目中只有一个实例。

## 创建插件
本案例中将使用一个插件来创建和使用Global Shaders。
在空的UE4C++工程中创建一个空插件，插件命名为GlobalShaderPlug。

## 修改uplugin文件
打开GlobalShaderPlug.uplugin文件，将其中"GlobalShaderPlug"对象的**LoadingPhase**字段的值修改为**PostConfigInit**，因为Shader要在引擎完全初始化之前加载。
关于**LoadingPhase**，可以在C++工程中搜索**ELoadingPhase**，源码中对每个phase都进行了注释。

```json
"Modules": [
    {
        "Name": "GlobalShaderPlug",
        "Type": "Runtime",
        "LoadingPhase": "PostConfigInit"
    }
]
```

## 修改Build.cs文件
打开GlobalShaderPlug.Build.cs文件，
- 在**PublicDependency**中加入**RenderCore**和**RHI**
- 在**PrivateDependency**中加入**Projects**和**AssetRegistry**

```csharp
PublicDependencyModuleNames.AddRange(
    new string[]
    {
        "Core",
        "RenderCore"，
        "RHI"
        // ... add other public dependencies that you statically link with here ...
    }
    );   

PrivateDependencyModuleNames.AddRange(
    new string[]
    {
        "CoreUObject",
        "Engine",
        "Slate",
        "SlateCore",
        "Projects",
        "AssetRegistry"
        // ... add private dependencies that you statically link with here ...	
    }
    );
```

## 创建Shader文件
先在插件中(GlobalShaderPlug文件夹中)创建一个Shaders文件夹，用于存放本插件中的Shaders文件。
在刚创建的文件夹中放入一个MyGlobalShader.usf文件，其内容如下
```c
#include "/Engine/Public/Platform.ush"

float4 MainColor;

void MainVS(
in float4 InPosition : ATTRIBUTE0,
out float4 OutPosition : SV_POSITION)
{
    OutPosition = InPosition;
}

void MainPS(out float4 OutColor : SV_Target0)
{
    OutColor = MainColor;
}

```
这样就创建了两个基本的Shader，MainVS是顶点着色器，MainPS是像素着色器。其中MainColor是一个float4类型的参数，后面我们将使用C++向Shader中传入这个参数，像素着色器将把它作为一个颜色进行输出。

## 设置Shader加载路径
在模块类的StartupModule函数中添加一个ShaderSourceDirectory的映射，这样后面使用C++读取shader的时候才能正确识别路径
```cpp
#include "GlobalShaderPlug.h"
#include "Modules/ModuleManager.h"
#include "Interfaces/IPluginManager.h"
#include <ShaderCore.h>

#define LOCTEXT_NAMESPACE "FGlobalShaderPlugModule"

void FGlobalShaderPlugModule::StartupModule()
{
	FString ShaderPath = FPaths::Combine(IPluginManager::Get().FindPlugin(TEXT("GlobalShaderPlug"))->GetBaseDir(), TEXT("Shaders"));
	AddShaderSourceDirectoryMapping(TEXT("/GlobalShaderPlug"), ShaderPath);
}
```

## 创建C++ Shader文件
接下来我们需要在C++文件中声明**Global顶点着色器**和**Global像素着色器**，他们都派生自**FGlobalShader**类，这是UE抽象出来的一个类型。我们需要这两个C++着色器类对Shader进行设置、传参等工作。
在Plugins/GlobalShaderPlug/Source/GlobalShaderPlug/Public文件夹中手动创建一个**MyShaders.h**文件，刷新工程。

>刷新工程方法:
>在uproject文件图标上右键 > Generate Visual Studio project files

打开生成/刷新的vs工程，在MyShaders.h文件中输入以下内容

```cpp {.line-numbers}
#pragma once

#include "GlobalShader.h"
#include "Shader.h"

class FMyShaderBase : public FGlobalShader
{
	DECLARE_INLINE_TYPE_LAYOUT(FMyShaderBase, NonVirtual);
public:
	FMyShaderBase(){}
	FMyShaderBase(const ShaderMetaType::CompiledShaderInitializerType& Initializer)
		: FGlobalShader(Initializer)
	{
		MainColorVal.Bind(Initializer.ParameterMap, TEXT("MainColor"));
	}

	static bool ShouldCache(EShaderPlatform Platform)
	{
		return true;
	}

	static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
	{
		return true;
	}

	static void ModifyCompilationEnvironment(const FGlobalShaderPermutationParameters& Parameters, FShaderCompilerEnvironment& OutEnvironment)
	{
		FGlobalShader::ModifyCompilationEnvironment(Parameters, OutEnvironment);
	}

	void SetParameters(FRHICommandListImmediate& RHICmdList, const FLinearColor& MyColor)
	{
		SetShaderValue(RHICmdList, RHICmdList.GetBoundPixelShader(), MainColorVal, MyColor);
	}

private:
	LAYOUT_FIELD(FShaderParameter, MainColorVal);
};

class FShader_VS: public FMyShaderBase
{
	DECLARE_GLOBAL_SHADER(FShader_VS);
public:
	FShader_VS() {}
	FShader_VS(const ShaderMetaType::CompiledShaderInitializerType& Initializer)
		:FMyShaderBase(Initializer)
	{ }	
};

class FShader_PS : public FMyShaderBase
{
	DECLARE_GLOBAL_SHADER(FShader_PS);
public:
	FShader_PS() {}
	FShader_PS(const ShaderMetaType::CompiledShaderInitializerType& Initializer)
		:FMyShaderBase(Initializer)
	{
	}
};

IMPLEMENT_SHADER_TYPE(, FShader_VS, TEXT("/GlobalShaderPlug/MyGlobalShader.usf"),TEXT("MainVS"),SF_Vertex)
IMPLEMENT_SHADER_TYPE(, FShader_PS, TEXT("/GlobalShaderPlug/MyGlobalShader.usf"), TEXT("MainPS"), SF_Pixel)
```
这里定义了三个类，其中**FShaderBase**是派生自FGlobalShader自定义的Shader基类，**FShader_VS**和**FShader_PS**派生自FShaderBase，分别是顶点着色器和像素着色器。

注意自定义的Shader基类开头的宏， DECLARE_INLINE_TYPE_LAYOUT，这是自定义Shader基类必须的。

定义完Shader，在使用Shader之前，必须要使用 *IMPLEMENT_SHADER_TYPE* 宏。

### 定义和使用一个属性的三个步骤 {#defineparameter}
1. 声明
在FMyShaderBase的private区域使用LAYOUT_FIELD宏声明了一个参数，名称为MainColorVal
```cpp
LAYOUT_FIELD(FShaderParameter, MainColorVal);
```
2. 绑定shader文件中相应的属性
在FMyShaderBase的构造函数中使用Bind函数将上一步声明的属性(MainColorVal)和shader文件中的MainColor属性进行绑定
```cpp
MainColorVal.Bind(Initializer.ParameterMap, TEXT("MainColor"));
```
3. 传值
在SetParameters函数中，我们使用了SetShaderValue函数，将属性值传入shader。
```cpp
void SetParameters(FRHICommandListImmediate& RHICmdList, const FLinearColor& MyColor)
{
	SetShaderValue(RHICmdList, RHICmdList.GetBoundPixelShader(), MainColorVal, MyColor);
}
```
SetParameter的函数原型:
```cpp
template<typename ShaderRHIParamRef, class ParameterType, typename TRHICmdList>
void SetShaderValue(
	TRHICmdList& RHICmdList,
	const ShaderRHIParamRef& Shader,
	const FShaderParameter& Parameter,
	const ParameterType& Value,
	uint32 ElementIndex = 0
	)
```
  - 第一个参数是RHICmdList
  - 第二个参数是Shader,这里获取PixelShader的方式是RHICmdList.GetBoundPixelShader()，也可以通过类似的一组方法获取VertexShader等其他Shader
  - 第三个参数是属性，就是第一步中声明的属性
  - 第四个参数是值，我们自己给SetParameters函数增加了一个FLinearColor类型的参数，并传递给这个形参。由于FLinearColor是一个包含了四个float类型元素的结构体，因此和usf Shader中的float4类型是匹配的。

因为我们在基类中已经绑定好了参数，所以在后面的FShader_VS和FShader_PS中只做了很少的事情，基本就是使用DECLARE_GLOBAL_SHADER宏声明了Shader。

```cpp
DECLARE_GLOBAL_SHADER(FShader_VS)

DECLARE_GLOBAL_SHADER(FShader_PS)
```
最后，使用IMPLEMENT_SHADER_TYPE宏(61-62行)，定义了像素shader的内容和顶点shader的内容，注意第一个参数之前有个逗号，第二个参数是前面的C++类名，第三个参数是usf文件的路径，第四个参数是从usf文件中找出shader函数的名字，最后一个参数是要定义的shader类型。

至此，Global Shader的创建已经完成，剩下的就是如何使用他们了。

编译，确保没有报错即可，目前还看不到任何内容。