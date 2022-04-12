---
title: "Ue4 Structured Buffer"
date: 2022-04-12T16:33:27+08:00
draft: false
categories: [UE4]
tags: 
  - StructuredBuffer
  - UE
  - ComputeShader
  - C++
series:
  - UE4图形开发系列
---

*UE4版本:4.26.2*

>前置知识
> - DirectX中的Compute Shader, Structured Buffer
> - 推荐阅读《Introduction to 3D Game Programming with DirectX 11》的 Chapter12-The Compute Shader
> - [上一篇]({{< ref "posts/ue4-compute-shader" >}})

## 工程源码
[UE4GraphicsGuide](https://github.com/inlet511/UEGrahpicsGuide/releases/tag/StructuredBuffer)

测试场景: Levels/ComputeShader

## 其他参考源码
[UnrealComputeShaderExample](https://github.com/timdecode/UnrealComputeShaderExample)
这套源码中包含了一些很有价值的内容，包括：
- 对于新旧版本的图形API如何区分 (#if ENGINE_MINOR_VERSION < 25)
- 在BeginPlay中创建渲染需要的资源，避免在每次渲染的时候重复创建
- 如果把ComputeShader计算的结果拷贝回游戏线程
- UNIFORM BUFFER STRUCT 宏和LAYOUT_FIELD宏混合使用

## 知识点

**BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT** 是旧版宏，等价于**BEGIN_UNIFORM_BUFFER_STRUCT**
```cpp
 /** Legacy macro definitions. */
#define BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT \
	 BEGIN_UNIFORM_BUFFER_STRUCT
```

## 一、StructuredBuffer
StructuredBuffer是DirectX11新增的缓冲类型，它是一种特殊的缓冲，**仅可用于Compute Shader**，它可以存储一组相同类型的数据,类似于一个数组。
### 意义
1. StructuredBuffer存储的数据类型可以是原生类型，也可以是用户自定义的结构体。扩展了我们可以传入Shader的数据类型，不再仅仅局限于 float、向量、颜色等
2. StructuredBuffer允许我们传输的数据量非常大(具体限制硬件决定，DirectX11可以支持2GB)，方便我们传输大量数据到Compute Shader里。 

>我们在Shader中声明Structured Buffer的时候，需要指明其类型。

例子：
```c
struct Data
{
  float3 v1;
  float2 v2;
};
StructuredBuffer<Data> gInputA;
StructuredBuffer<Data> gInputB;
RWStructuredBuffer<Data> gOutput;
```
- 上述代码使用了自定义结构体作为Structured Buffer的数据类型。
- 我们创建了两种不同的属性，一种是StructuredBuffer，它是只读的，一种是 RWStructuredBuffer，它是可读写的。

StructuredBuffer可以通过SRV绑定到Compute Shader，RWStructuredBuffer可以通过UAV绑定到Compute Shader

## 二、StructuredBuffer in UE4
hlsl的Shader和UE4的ush、usf shader是完全相同的。
在UE4 C++中，它的创建是使用 RHICreateStructuredBuffer 函数执行的。
创建时，如果需要填入内容，可以通过 **FRHIResourceCreateInfo**, 将CreateInfo的ResourceArray属性指向一个TResourceArray。

## 三、修改usf shader
在前一篇usf文件中的Compute Shader基础上进行修改，增加StructuredBuffer参数。
```c
RWTexture2D<float4> RWOutputSurface;
StructuredBuffer<float3> MyStructuredBuffer;

[numthreads(32,32,1)]
void MainCS(
	uint3 GroupId: SV_GroupID,
	uint3 DispatchThreadId : SV_DispatchThreadID,
	uint3 GroupThreadId : SV_GroupThreadID)
{
    //RWOutputSurface[DispatchThreadId.xy] = float4(float(GroupThreadId.x) / float(32), float(GroupThreadId.y) / float(32), 0, 1);
    RWOutputSurface[DispatchThreadId.xy] = float4(MyStructuredBuffer[0], 1);
}
```
添加了一个StructuredBuffer，其数据类型为float3类型数组，usf shader里不能指定该buffer的长度，应该在C++端指定要传入的数据的长度。
作为演示，我们在ComputeShader中只是简单的把StructuredBuffer的第一个元素作为颜色进行输出。实际运用中我们可以结合DispatchThreadId等读取StructuredBuffer，参考本文开头提供的[其他参考源码]({{< ref "#其他参考源码" >}})

## 四、修改Compute Shader
在前一篇的Compute Shader基础上进行修改，这里仅列出有修改的部分
```cpp
	FMyComputeShader(const ShaderMetaType::CompiledShaderInitializerType& Initializer)
		: FGlobalShader(Initializer)
	{
		OutputSurface.Bind(Initializer.ParameterMap, TEXT("OutputSurface"));
		InData.Bind(Initializer.ParameterMap, TEXT("MyStructuredBuffer"));
	}

	void SetParameters(
		FRHICommandList& RHICmdList,
		FTexture2DRHIRef& InOutputSurfaceValue,
		FUnorderedAccessViewRHIRef& UAV,
		FRHIComputeShader* Shader,
		FShaderResourceViewRHIRef& SRV
	)
	{
		OutputSurface.SetTexture(RHICmdList, RHICmdList.GetBoundComputeShader(), InOutputSurfaceValue, UAV);		
		RHICmdList.SetShaderResourceViewParameter(Shader, InData.GetBaseIndex(), SRV);
	}

private:
	LAYOUT_FIELD(FRWShaderParameter, OutputSurface);
	LAYOUT_FIELD(FShaderResourceParameter, InData);
```
通过LAYOUT_FIELD添加了一个**FShaderResourceParameter**类型的属性，并在构造函数内部进绑定usf shader中的属性名称。
之所以选择FShaderResourceParameter, 是因为我们仅打算读取StructuredBuffer中的数据，也就是将其通过SRV方式绑定给Shader。
如果希望又读又修改，那么应该使用UAV，这里就应该像另一个参数OutputSurface一样，使用FRWShaderParameter类型，并且usf shader里也使用RWStructuredBuffer。

另外一个重要的变化是给SetParameters增加了两个参数，Shader和SRV。
传入Shader的作用是为了SetShaderResourceViewParameter函数使用，SRV则是要给Compute Shader绑定的资源。
SetParameters函数体内使用 RHICmdList.SetShaderResourceViewParameter实现了给Compute Shader绑定SRV资源(这个函数名称实在太长了，个人感觉可以简写为SetSRV)。

## 五、修改C++应用层
应用层也同样在前一章节的UseCompute_RenderThread基础上修改(UseCompute函数无需修改)。
增加了一段代码，给SetParameters函数的调用增加了两个参数。

增加的内容：
```cpp
// StructuredBuffer新增内容**********************************************************
//初始胡一个Resource Array，储存FVector类型数据
TResourceArray<FVector> MyResourcesArray;
// 初始化Resource Array为10个元素，并填充随机数据
MyResourcesArray.Init(FVector::ZeroVector, 10);
for (auto& v : MyResourcesArray)
{
  v = FVector(
    FMath::RandRange(.0f, 1.0f),
    FMath::RandRange(.0f, 1.0f),
    FMath::RandRange(.0f, 1.0f));
}
// 声明StructuredBuferRHI
FStructuredBufferRHIRef MyStructuredBuffer;
// 声明SRV
FShaderResourceViewRHIRef MyStructuredBufferSRV;
//创建StructuredBuffer
FRHIResourceCreateInfo CreateInfo2;
CreateInfo2.ResourceArray = &MyResourcesArray;
MyStructuredBuffer = RHICreateStructuredBuffer(sizeof(FVector), sizeof(FVector) * 10 , BUF_Static | BUF_ShaderResource, CreateInfo2);
// 创建SRV
MyStructuredBufferSRV = RHICreateShaderResourceView(MyStructuredBuffer);
// *****************************************************************
```
先声明了一个FVector类型的TResourceArray，初始化为10个元素，并对10个元素的x,y,z进行了0-1的随机填充，若作为颜色解释，就是随机颜色。

之后用TResourceArray作为内容，创建StructuredBuffer，并为其创建SRV。

设置参数：
```cpp
ComputeShader->SetParameters(RHICmdList, CreatedRHITexture, TextureUAV, ComputeShader.GetComputeShader(),MyStructuredBufferSRV);
```

> 注意，这里我们将创建资源(TResourceArray,StructuredBuffer,SRV)放在了渲染线程函数里，实际应用中，我们会在tick中调用渲染函数，那么资源创建会反复执行，这是没有必要的。更好的做法应该是在渲染之前将所有资源创建好，渲染线程中仅更新Buffer，以及调用DispatchComputeShader命令，可以参考本文开头提供的[其他参考源码]({{< ref "#其他参考源码" >}})