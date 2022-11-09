---
title: "RDG 03 AddPass"
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
	/** Pass没有任何输入输出的绑定，仅用于无参的AddPass函数 */
	None = 0,
	/** Pass在图形管线上使用了光栅化操作*/
	Raster = 1 << 0,
	/** Pass在图形管线上使用了compute操作*/
	Compute = 1 << 1,
	/** Pass在异步计算管线中使用了compute操作*/
	AsyncCompute = 1 << 2,
	/** Pass在图形管线上使用了复制命令 */
	Copy = 1 << 3,
	/** Pass和其生产者永不被剔除。在输出不能被Graph追踪的情况下是必须的 */
	NeverCull = 1 << 4,
	/** 忽略Render Pass的开始和结束，让用户去调用。只有与Raster结合时才有用。会在当前Pass上禁用Pass合并。	 */
	SkipRenderPass = 1 << 5,
	/**	Pass将永远不会与其他Pass合并 */
	NeverMerge = 1 << 6,
	/** Pass将永远不会离开渲染线程 */
	NeverParallel = 1 << 7,
	/** Pass uses copy commands but writes to a staging resource. Pass使用复制命令但是写入一个 Staging 资源 */
	Readback = Copy | NeverCull
};
ENUM_CLASS_FLAGS(ERDGPassFlags);
```
一个Pass可以有多个标记，用竖线 | 进行分割。

一个Pass的标记至少要包含 Copy,Compute,AsyncCompute, Raster之一。

如果一个Pass包含了Raster标记，则必须绑定RenderTarget，否则将出现报错：
> Pass 'XXX' is set to 'Raster' but is missing render target binding slots.

绑定RenderTarget的方法见下文。


### 3.4 Lambda函数

Lambda函数首先要将需要用到的参数添加到捕获列表中， 函数参数为 FRHICommandListImmediate& RHICmdList。

在函数体内，我们需要声明一个 FGraphicsPipelineStateInitializer 对象，通过该对象对Pass进行必要的设置。

之后还要设置各个Shader的参数，这个操作通常采用 SetShaderParameters 函数来完成。

最后还需要设置VertexBuffer(和IndexBuffer)。

最终调用绘制命令进行绘制。

## 4 辅助函数
RDG包含几个有用的辅助函数，用于添加常用的Pass。应尽可能使用这些函数。
- FComputeShaderUtils::AddPass 用于添加Compute Pass
- FPixelShaderUtils::AddFullScreenPass 用于添加全屏像素着色器 Pass
例：
```cpp
FComputeShaderUtils::AddPass(
				GraphBuilder,
				RDG_EVENT_NAME("ComputeVolumeTexture"),
				ComputeShader,
				CSParams,
				FIntVector(GroupCount, GroupCount, textureSize));

FPixelShaderUtils::AddFullscreenPass<RenderSkyAtmosphereEditorHudPS>(
				GraphBuilder, 
				View.ShaderMap, 
				RDG_EVENT_NAME("SkyAtmosphereEditor"), 
				PixelShader, 
				PassParameters, 
				View.ViewRect);

FPixelShaderUtils::AddFullscreenPass(
				GraphBuilder,
				View.ShaderMap,
				RDG_EVENT_NAME("DownsampleHZB(mip=%d) %dx%d", StartDestMip, DstSize.X, DstSize.Y),
				PixelShader,
				PassParameters,
				FIntRect(0, 0, DstSize.X, DstSize.Y));
```

## 5 不带着色器参数的Pass
```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FCopyTextureParameters, )

    // 声明CopySrc访问FRDGTexture*
    RDG_TEXTURE_ACCESS(Input,  ERHIAccess::CopySrc)

    // 声明CopyDest访问FRDGTexture*
    RDG_TEXTURE_ACCESS(Output, ERHIAccess::CopyDest)

END_SHADER_PARAMETER_STRUCT()

void AddCopyTexturePass(
    FRDGBuilder& GraphBuilder,
    FRDGTextureRef InputTexture,
    FRDGTextureRef OutputTexture,
    const FRHICopyTextureInfo& CopyInfo)
{
    FCopyTextureParameters* Parameters = GraphBuilder.AllocParameters<FCopyTextureParameters>();
    Parameters->Input = InputTexture;
    Parameters->Output = OutputTexture;

    GraphBuilder.AddPass(
        RDG_EVENT_NAME("CopyTexture(%s -> %s)", InputTexture->Name, OutputTexture->Name),
        Parameters,
        ERDGPassFlags::Copy,
        [InputTexture, OutputTexture, CopyInfo](FRHICommandList& RHICmdList)
    {
        RHICmdList.CopyTexture(InputTexture->GetRHI(), OutputTexture->GetRHI(), CopyInfo);
    });
}
```
这个Pass的两个参数都与着色器无关，只是分别指明了要复制的源以及目标。 实现了复制贴图的功能。

## 6 Raster Pass {#rendertargetbinding}
注意，每一个Raster Pass都需要一个RenderTarget，RDG通过 RENDER_TARGET_BINDING_SLOTS 参数为Raster Pass暴露了固定渲染管线的RenderTarget，我们只需要给参数增加一个 RENDER_TARGET_BINDING_SLOTS()

在给参数赋值的时候做这两件事：
- 为RenderTargets[0]赋值一个FRenderTargetBinding，设置颜色的绑定对象以及操作。
- 为RenderTargets.DepthStencil赋值一个FDepthStencilBinding，设置深度模板的绑定对象及操作。
  
```cpp
PSParams->RenderTargets[0] = FRenderTargetBinding(SceneColorTexture, ERenderTargetLoadAction::ELoad);
PSParams->RenderTargets.DepthStencil = FDepthStencilBinding(
	SceneDepthTexture, ERenderTargetLoadAction::ELoad, ERenderTargetLoadAction::ELoad,
	FExclusiveDepthStencil::DepthWrite_StencilWrite);
```
### FRenderTargetBinding 构造函数
```cpp
FRenderTargetBinding(FRDGTexture* InTexture, ERenderTargetLoadAction InLoadAction, uint8 InMipIndex = 0, int16 InArraySlice = -1)
```
第一个参数是 FRDGTextureRef类型对象，赋给SceneColorTexture即可。

第二个参数是 ERenderTargetLoadAction 枚举，有三个成员：
- ELoad: 保留Texture已存在的部分
- EClear: 清理Texture，采用其优化的清除值
- ENoAction: 可能不会保留内容，这个选项在某些硬件上速度更快（如果确保所有有效像素都被写入）

### FDepthStencilBinding 构造函数
```cpp
FDepthStencilBinding(
		FRDGTexture* InTexture,
		ERenderTargetLoadAction InDepthLoadAction,
		ERenderTargetLoadAction InStencilLoadAction,
		FExclusiveDepthStencil InDepthStencilAccess)
```
第一个参数赋给SceneDepthTexture即可。

第二、三个参数分别是设定深度和模板部分的操作。

第四个参数是 FExclusiveDepthStencil 类型，它控制着每个平面是否具有读取或写入访问权限
```cpp
enum Type
	{
		// 不要使用上面的单独选项，使用下面的组合
		// 4 bits are used for depth and 4 for stencil to make the hex value readable and non overlapping
		DepthNop = 0x00,
		DepthRead = 0x01,
		DepthWrite = 0x02,
		DepthMask = 0x0f,
		StencilNop = 0x00,
		StencilRead = 0x10,
		StencilWrite = 0x20,
		StencilMask = 0xf0,

		// 用这些：
		DepthNop_StencilNop = DepthNop + StencilNop,
		DepthRead_StencilNop = DepthRead + StencilNop,
		DepthWrite_StencilNop = DepthWrite + StencilNop,
		DepthNop_StencilRead = DepthNop + StencilRead,
		DepthRead_StencilRead = DepthRead + StencilRead,
		DepthWrite_StencilRead = DepthWrite + StencilRead,
		DepthNop_StencilWrite = DepthNop + StencilWrite,
		DepthRead_StencilWrite = DepthRead + StencilWrite,
		DepthWrite_StencilWrite = DepthWrite + StencilWrite,
	};
```
### 例子
颜色目标手动清除，而深度和模板目标使用硬件清除操作：
```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FRenderTargetParameters, RENDERCORE_API)

    // 这些绑定插槽包含颜色和深度模板目标。
    RENDER_TARGET_BINDING_SLOTS()

END_SHADER_PARAMETER_STRUCT()

void AddClearRenderTargetPass(FRDGBuilder& GraphBuilder, FRDGTexture* Texture, const FLinearColor& ClearColor, FIntRect Viewport)
{
    FRenderTargetParameters* Parameters = GraphBuilder.AllocParameters<FRenderTargetParameters>();

    Parameters->RenderTargets[0] = FRenderTargetBinding(
        Texture,
        ERenderTargetLoadAction::ENoAction // <- 不需要加载之前的render target 内容，因为我们要手动清理
    );

    GraphBuilder.AddPass(
        RDG_EVENT_NAME("ClearRenderTarget(%s) [(%d, %d), (%d, %d)] ClearQuad", Texture->Name, Viewport.Min.X, Viewport.Min.Y, Viewport.Max.X, Viewport.Max.Y),
        Parameters,
        ERDGPassFlags::Raster,
        [Parameters, ClearColor, Viewport](FRHICommandList& RHICmdList)
    {
        RHICmdList.SetViewport(Viewport.Min.X, Viewport.Min.Y, 0.0f, Viewport.Max.X, Viewport.Max.Y, 1.0f);
        DrawClearQuad(RHICmdList, ClearColor);
    });
}

void AddClearDepthStencilPass(FRDGBuilder& GraphBuilder, FRDGTextureRef Texture)
{
    auto* PassParameters = GraphBuilder.AllocParameters<FRenderTargetParameters>();

    PassParameters->RenderTargets.DepthStencil = FDepthStencilBinding(
        Texture,
        ERenderTargetLoadAction::EClear, // <- 清理深度到其优化的清理值.
        ERenderTargetLoadAction::EClear, // <- 清理模板到其优化的清理值.
        FExclusiveDepthStencil::DepthWrite_StencilWrite // <- 允许写入深度和模板
    );

    GraphBuilder.AddPass(
        RDG_EVENT_NAME("ClearDepthStencil (%s)", Texture->Name),
        PassParameters,
        ERDGPassFlags::Raster,
        [](FRHICommandList&)
    {
        // Lambda中无实际工作！RDG为我们处理渲染通道！清除通过Clear操作完成。
    });
}
```
## 7 UAV Raster Pass
RDG支持写入UAV而不是固定管线渲染目标的Pass。最简单的办法就是使用 FPixelShaderUtils::AddUAVPass 函数。它创建了一个没有绑定渲染目标的自定义的渲染Pass,并且将RHI Viewport 设置好。
```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FUAVRasterPassParameters, RENDERCORE_API)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D, Output)
END_SHADER_PARAMETER_STRUCT()

auto* PassParameters = GraphBuilder.AllocParameters<FUAVRasterPassParameters>();
PassParameters.Output = GraphBuilder.CreateUAV(OutputTexture);

// Specify the viewport rect.
FIntRect ViewportRect = ...;

FPixelShaderUtils::AddUAVPass(
    GraphBuilder,
    RDG_EVENT_NAME("Raster UAV Pass"),
    PassParameters,
    ViewportRect,
    [](FRHICommandList& RHICmdList)
{
    // Bind parameters and draw.
});
```