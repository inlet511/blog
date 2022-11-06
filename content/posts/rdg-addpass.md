---
title: "RDG03 AddPass"
date: 2022-11-02T23:19:27+08:00
draft: false
toc: true
categories: [UE5]
series:
  - RDG
tags:
  - 图形学
  - UE
  - RDG
  - AddPass
---

## 1 原型
RDG流程中，添加一个Pass的方法是 *FRDGBuilder::AddPass()*,其应用的最广泛的一个重载的原型如下：
```cpp
template <typename ParameterStructType, typename ExecuteLambdaType>
FRDGPassRef FRDGBuilder::AddPass(
	FRDGEventName&& Name,
	const ParameterStructType* ParameterStruct,
	ERDGPassFlags Flags,
	ExecuteLambdaType&& ExecuteLambda)
{
	return AddPassInternal(Forward<FRDGEventName>(Name), ParameterStructType::FTypeInfo::GetStructMetadata(), ParameterStruct, Flags, Forward<ExecuteLambdaType>(ExecuteLambda));
}
```
## 2 调用入口
### 2.1 系统内的Render函数
如果直接修改引擎代码，调用该函数的位置一般是在 FDeferredShadingSceneRenderer 的 Render函数中，这个函数中可以直接访问到 GraphBuilder ，它是一个FRDGBuilder对象，我们可以直接通过该对象调用 AddPass方法：
```cpp
 GraphBuilder.AddPass(...)
```
### 2.2 渲染委托
避免修改引擎源码，我们可以调用渲染委托，通过 RegisterPostOpaqueRenderDelegate 和 RegisterOverlayRenderDelegate 两个方法添加渲染的函数。例如：
```cpp
RenderHandle = GetRendererModule().RegisterOverlayRenderDelegate(FPostOpaqueRenderDelegate::CreateRaw(this, &FRadiationRenderer::Render));
```
该函数必须包含一个参数： FPostOpaqueRenderParameters& InParameters, 通过这个参数我们就可以获取到GraphBuilder:
```cpp
FRDGBuilder& GraphBuilder = *InParameters.GraphBuilder;
```

## 3 参数分析
它接受4个参数，下面逐个说明。

### 3.1 Pass名称
第一个参数是该Pass的名称，主要用于进行图形调试，例如使用RenderDoc这样的工具，可以看到这个Pass的名称。一般使用 RDG_EVENT_NAME("EventName") 来定义，双引号中间填写该Pass的名称。

### 3.2 Pass参数
第二个参数是该Pass的参数，注意这个Pass的参数可以是PixelShader的参数， 也可以是VertexShader和PixelShader共享的参数，如果是一个纯Compute Shader Pass，则可以是ComputeShader的参数。
我们可以看到如下不同的应用情形：
1. 直接使用PixelShader的参数作为Pass的参数
2. VertexShader和PixelShader共享一组Shader参数（从一个父类派生），将这组Shader参数传入。如 FShaderDrawSymbolsVS和 FShaderDrawSymbolsPS都是从 FShaderDrawSymbols派生。


### 3.3 Pass标记
第三个参数是一个标记，它的定义：
```cpp
enum class ERDGPassFlags : uint8
{
	/** Pass doesn't have any inputs or outputs tracked by the graph. This may only be used by the parameterless AddPass function. */
	None = 0,
	/** Pass uses rasterization on the graphics pipe. */
	Raster = 1 << 0,
	/** Pass uses compute on the graphics pipe. */
	Compute = 1 << 1,
	/** Pass uses compute on the async compute pipe. */
	AsyncCompute = 1 << 2,
	/** Pass uses copy commands on the graphics pipe. */
	Copy = 1 << 3,
	/** Pass (and its producers) will never be culled. Necessary if outputs cannot be tracked by the graph. */
	NeverCull = 1 << 4,
	/** Render pass begin / end is skipped and left to the user. Only valid when combined with 'Raster'. Disables render pass merging for the pass. */
	SkipRenderPass = 1 << 5,
	/** Pass will never have its render pass merged with other passes. */
	NeverMerge = 1 << 6,
	/** Pass will never run off the render thread. */
	NeverParallel = 1 << 7,
	/** Pass uses copy commands but writes to a staging resource. */
	Readback = Copy | NeverCull
};
ENUM_CLASS_FLAGS(ERDGPassFlags);
```
一个Pass可以有多个标记，用竖线 | 进行分割。

一个Pass的标记至少要包含 Copy,Compute,AsyncCompute, Raster之一。

如果一个Pass包含了Raster标记，则必须绑定RenderTarget，否则将出现报错：
> Pass 'XXX' is set to 'Raster' but is missing render target binding slots.

绑定RenderTarget的方法: [这篇文章]({{< ref "/posts/rdg-shader-params.md#rendertargetbinding" >}})


### 3.4 Lambda函数

Lambda函数首先要将需要用到的参数添加到捕获列表中， 函数参数为 FRHICommandListImmediate& RHICmdList。

在函数体内，我们需要声明一个 FGraphicsPipelineStateInitializer 对象，通过该对象对Pass进行必要的设置。

之后还要设置各个Shader的参数，这个操作通常采用 SetShaderParameters 函数来完成。

最后还需要设置VertexBuffer(和IndexBuffer)。

最终调用绘制命令进行绘制。
