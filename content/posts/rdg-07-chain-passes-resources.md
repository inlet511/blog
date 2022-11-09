---
title: "Rdg 07 串接Pass和资源"
date: 2022-11-09T21:40:41+08:00
draft: false
toc: true
categories: [UE5]
series:
  - RDG
tags:
  - 图形学
  - UE
  - RDG
  - Resources
  - Pass
---

通常为了实现一个功能/效果，需要多个Pass配合，一般都是采用串接的方式，即先执行一个Pass，再执行其他Pass。多个Pass之间一定会有资源的传递才有意义。

## Mip生成案例
一个典型的例子就是生成Mipmap的功能。它采用了两个Pass：Compute Pass 和 Raster Pass。

Compute Pass是由Compute Shader计算的，Raster Pass是由Vertex Shader和Pixel Shader一起渲染的。

### 入口函数
先看一个已经Deprecated的执行的入口, (虽然已经过时，但可以学到一些知识)：
```cpp
void FGenerateMips::Execute(FRHICommandListImmediate& RHICmdList, FRHITexture* Texture, TSharedPtr<FGenerateMipsStruct>& ExternalMipsStructCache, FGenerateMipsParams Params, bool bAllowRenderBasedGeneration)
{
	TRefCountPtr<IPooledRenderTarget> PooledRenderTarget = CreateRenderTarget(Texture, TEXT("MipGeneration"));

	FMemMark MemMark(FMemStack::Get());
	FRDGBuilder GraphBuilder(RHICmdList);
	Execute(GraphBuilder, GraphBuilder.RegisterExternalTexture(PooledRenderTarget), Params, bAllowRenderBasedGeneration ? EGenerateMipsPass::Raster : EGenerateMipsPass::Compute);
	GraphBuilder.Execute();
}
```
执行函数传入了一个FRHITexture*类型的纹理，然后调用 *CreateRenderTarget* 创建了一个未被追踪的池化的Render Target对象，它是TRefCountPtr\<IPooledRenderTarget\>类型。

之后创建了一个新的RDG，调用了RDG版本的Execute，最后调用GraphBuilder的Execute()函数执行。这里唯一资源就是这个纹理。

再观察一下RDG版本的执行入口(其中一个)：
```cpp
void FGenerateMips::Execute(FRDGBuilder& GraphBuilder, FRDGTextureRef Texture, FRHISamplerState* Sampler, EGenerateMipsPass Pass)
{
	if (Pass == EGenerateMipsPass::AutoDetect)
	{
		Pass = WillFormatSupportCompute(Texture->Desc.Format) ? EGenerateMipsPass::Compute : EGenerateMipsPass::Raster;
	}

	if (Pass == EGenerateMipsPass::Compute)
	{
		ExecuteCompute(GraphBuilder, Texture, Sampler);
	}
	else
	{
		ExecuteRaster(GraphBuilder, Texture, Sampler);
	}
}
```
基本上就是调用了ExecuteCompute和ExecuteRaster两个函数。

### Compute Pass
简化以后的Compute Pass:
```cpp
class FGenerateMipsCS : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FGenerateMipsCS)
    SHADER_USE_PARAMETER_STRUCT(FGenerateMipsCS, FGlobalShader)

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER(FVector2D, TexelSize)
        SHADER_PARAMETER_RDG_TEXTURE_SRV(Texture2D, MipInSRV)
        SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D, MipOutUAV)
        SHADER_PARAMETER_SAMPLER(SamplerState, MipSampler)
    END_SHADER_PARAMETER_STRUCT()
};

IMPLEMENT_GLOBAL_SHADER(FGenerateMipsCS, "/Engine/Private/ComputeGenerateMips.usf", "MainCS", SF_Compute);

void FGenerateMips::ExecuteCompute(FRDGBuilder& GraphBuilder, FRDGTexture* Texture, FRHISamplerState* Sampler)
{
    TShaderMapRef<FGenerateMipsCS> ComputeShader(GetGlobalShaderMap(GMaxRHIFeatureLevel));

    // Loop through each level of the mips that require creation and add a dispatch pass per level.
    for (uint32 MipLevel = 1, MipCount = TextureDesc.NumMips; MipLevel < MipCount; ++MipLevel)
    {
        const FIntPoint DestTextureSize(
            FMath::Max(TextureDesc.Extent.X >> MipLevel, 1),
            FMath::Max(TextureDesc.Extent.Y >> MipLevel, 1));

        FGenerateMipsCS::FParameters* PassParameters = GraphBuilder.AllocParameters<FGenerateMipsCS::FParameters>();
        PassParameters->TexelSize  = FVector2D(1.0f / DestTextureSize.X, 1.0f / DestTextureSize.Y);
        PassParameters->MipInSRV   = GraphBuilder.CreateSRV(FRDGTextureSRVDesc::CreateForMipLevel(Texture, MipLevel - 1));
        PassParameters->MipOutUAV  = GraphBuilder.CreateUAV(FRDGTextureUAVDesc(Texture, MipLevel));
        PassParameters->MipSampler = Sampler;

        FComputeShaderUtils::AddPass(
            GraphBuilder,
            RDG_EVENT_NAME("GenerateMips DestMipLevel=%d", MipLevel),
            ComputeShader,
            PassParameters,
            FComputeShaderUtils::GetGroupCount(DestTextureSize, FComputeShaderUtils::kGolden2DGroupSize));
    }
}
```
可以看到Compute Shader有一个SRV参数，一个UAV参数。其中SRV参数就是将原始贴图或者上一次生成的Mip等级贴图作为输入，而UAV则是本次要生成的贴图等级(可以注意MipInSRV和MipOutUAV后面的MipLevel数值关系)。

所以总结Compute Shader的作用，就是提取上一级别的Mip作为源，将其压缩作为本级别的Mip。

### Raster Pass
再阅读简化后的Raster Pass:
```cpp
class FGenerateMipsVS : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FGenerateMipsVS);
};

IMPLEMENT_GLOBAL_SHADER(FGenerateMipsVS, "/Engine/Private/ComputeGenerateMips.usf", "MainVS", SF_Vertex);

class FGenerateMipsPS : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FGenerateMipsPS);
    SHADER_USE_PARAMETER_STRUCT(FGenerateMipsPS, FGlobalShader);

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER(FVector2D, HalfTexelSize)
        SHADER_PARAMETER(float, Level)
        SHADER_PARAMETER_RDG_TEXTURE_SRV(Texture2D, MipInSRV)
        SHADER_PARAMETER_SAMPLER(SamplerState, MipSampler)
        RENDER_TARGET_BINDING_SLOTS()
    END_SHADER_PARAMETER_STRUCT()
};

IMPLEMENT_GLOBAL_SHADER(FGenerateMipsPS, "/Engine/Private/ComputeGenerateMips.usf", "MainPS", SF_Pixel);

void FGenerateMips::ExecuteRaster(FRDGBuilder& GraphBuilder, FRDGTexture* Texture, FRHISamplerState* Sampler)
{
    auto ShaderMap = GetGlobalShaderMap(GMaxRHIFeatureLevel);
    TShaderMapRef<FGenerateMipsVS> VertexShader(ShaderMap);
    TShaderMapRef<FGenerateMipsPS> PixelShader(ShaderMap);

    for (uint32 MipLevel = 1, MipCount = Texture->Desc.NumMips; MipLevel < MipCount; ++MipLevel)
    {
        const uint32 InputMipLevel = MipLevel - 1;

        const FIntPoint DestTextureSize(
            FMath::Max(Texture->Desc.Extent.X >> MipLevel, 1),
            FMath::Max(Texture->Desc.Extent.Y >> MipLevel, 1));

        FGenerateMipsPS::FParameters* PassParameters = GraphBuilder.AllocParameters<FGenerateMipsPS::FParameters>();
        PassParameters->HalfTexelSize = FVector2D(0.5f / DestTextureSize.X, 0.5f / DestTextureSize.Y);
        PassParameters->Level = InputMipLevel;
        PassParameters->MipInSRV = GraphBuilder.CreateSRV(FRDGTextureSRVDesc::CreateForMipLevel(Texture, InputMipLevel));
        PassParameters->MipSampler = Sampler;
        PassParameters->RenderTargets[0] = FRenderTargetBinding(Texture, ERenderTargetLoadAction::ELoad, MipLevel);

        FPixelShaderUtils::AddFullscreenPass(
            GraphBuilder,
            ShaderMap,
            RDG_EVENT_NAME("GenerateMips DestMipLevel=%d", MipLevel),
            PixelShader,
            PassParameters,
            FIntRect(FIntPoint::ZeroValue, DestTextureSize));
    }
}
```
Pixel Shader有一个SRV，它的作用就是读取Texture的某个Mip级别。

Raster Pass 通过
```cpp
PassParameters->RenderTargets[0] = FRenderTargetBinding(Texture, ERenderTargetLoadAction::ELoad, MipLevel);
```
绑定了Texture的某个Mip级别作为该Pass的渲染目标。

所以两个Pass先手串联起来，对一个Texture资源的不同Mip等级进行读取和储存，就完成了计算一组MipMap的工作。

## 更复杂的一个案例

这个案例中，同样要使用一个ComputePass和一个Raster Pass，但是区别是Compute Pass并不是每帧都要执行，而Raster Pass是每帧都要执行。

Compute Pass的作用是读取外部数据，计算一个立体贴图，计算量比较大，因此我们应该尽量避免频繁执行这个Pass，只是再外部数据发生变化的时候执行一次即可。

Raster Pass的作用是读取Compute Pass计算出来的立体贴图作为SRV，在屏幕上进行绘制。
### Compute Shader
```cpp
class FMyCS : public FGlobalShader
{
public:
	DECLARE_GLOBAL_SHADER(FMyCS)
	SHADER_USE_PARAMETER_STRUCT(FMyCS, FGlobalShader)

	BEGIN_SHADER_PARAMETER_STRUCT(FParameters,)
		SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture3D, OutUAV)
		SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FRadiationUniformParameters, UniformBuffer)
		SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<RadiationData>, InDataBuffer)
		END_SHADER_PARAMETER_STRUCT()

	static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
	{
		return RHISupportsComputeShaders(Parameters.Platform);
	}
};

IMPLEMENT_GLOBAL_SHADER(FMyCS, "/Engine/Private/MyPassShader.usf", "MainCS", SF_Compute);
```
这里要关注的重点是 SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture3D, OutUAV) 这个参数，我们最后计算出来的立体贴图就是它。

### Compute Pass
```cpp
// Raster Pass
if (SceneInfo != nullptr &&
    (SceneInfo->VolumeRT == nullptr || SceneInfo->IsVolumeTextureDirty()))
{
    ComputeVolumeTexture(GraphBuilder, ComputeShader, Scene, textureSize);
    SceneInfo->SetVolumeTextureDirty(false);
}
// Compute VolumeTexture，省略部分内容
void ComputeVolumeTexture(FRDGBuilder& GraphBuilder, const TShaderMapRef<FMyCS>& ComputeShader, FScene* Scene,
                          const uint32 textureSize)
{
	FRDGTextureDesc TextDesc = FRDGTextureDesc::Create3D(
		FIntVector(textureSize, textureSize, textureSize),
		EPixelFormat::PF_R32_FLOAT,
		FClearValueBinding::None,
		TexCreate_ShaderResource | TexCreate_UAV);

	// Create VolumeTexture, use this as a bridge between CS and PS
	Scene->GetRadiationSceneInfo()->VolumeTexture = GraphBuilder.
		CreateTexture(TextDesc, TEXT("RadiationVolumeTexture"));
    // ...

	FComputeShaderUtils::AddPass(
		GraphBuilder,
		RDG_EVENT_NAME("ComputeVolumeTexture"),
		ComputeShader,
		CSParams,
		FIntVector(GroupCount, GroupCount, textureSize));

	// Extract Volume Texture Resource
	GraphBuilder.QueueTextureExtraction(Scene->GetRadiationSceneInfo()->VolumeTexture,
	                                    &Scene->GetRadiationSceneInfo()->VolumeRT);
}
```
首先看到ComputeVolumeTexture的前面是有一个判断的，满足这个条件才进行计算。其中的条件主要是判断贴图是否已经"dirty"，即是否有新的数据来了，需要重新计算了。这样我们就可以减少执行这个Pass的次数。

另外注意到最后一行，使用了QueueTextureExtraction将贴图提取到外部进行保存。保存的指针格式是 TRefCountPtr\<IPooledRenderTarget\>

###  Pixel Shader
Vertex Shader这里略去不表。看一下简化后的Pixel Shader：
```cpp
class FMyPS : public FGlobalShader
{
public:
	DECLARE_GLOBAL_SHADER(FMyPS);
	SHADER_USE_PARAMETER_STRUCT(FMyPS, FGlobalShader)

	BEGIN_SHADER_PARAMETER_STRUCT(FParameters,)
		SHADER_PARAMETER_RDG_TEXTURE(Texture3D, VolumeTexture)
		SHADER_PARAMETER_SAMPLER(SamplerState, VolumeSampler)
		RENDER_TARGET_BINDING_SLOTS()
	END_SHADER_PARAMETER_STRUCT()
};

IMPLEMENT_GLOBAL_SHADER(FMyPS, "/Engine/Private/MyPassShader.usf", "MainPS", SF_Pixel);
```
Pixel Shader 使用一个类型为Texture3D的SHADER_PARAMETER_RDG_TEXTURE作为参数，目的就是载入Compute Pass计算出来的VolumeTexture。

### Raster Pass
简化后的Raster Pass代码：
```cpp
if (SceneInfo != nullptr && SceneInfo->VolumeRT != nullptr)
{
	// Register pooled Texture
	volumeTexture = GraphBuilder.RegisterExternalTexture(SceneInfo->VolumeRT);

    if(SceneInfo->ShouldRenderVolume())
    {
        // Render To Screen 
        if (View.IsPerspectiveProjection())
        {
            check(SceneDepthTexture);

            FMyPS::FParameters* PassParams = GraphBuilder.AllocParameters<FMyPS::FParameters>();
            PassParams->RenderTargets[0] = FRenderTargetBinding(SceneColorTexture, ERenderTargetLoadAction::ELoad);
            PassParams->RenderTargets.DepthStencil = FDepthStencilBinding(
                SceneDepthTexture, ERenderTargetLoadAction::ELoad, ERenderTargetLoadAction::ELoad,
                FExclusiveDepthStencil::DepthRead_StencilWrite);
            PassParams->VolumeSampler = TStaticSamplerState<SF_Bilinear, AM_Clamp, AM_Clamp, AM_Clamp>::GetRHI();        
            PassParams->VolumeTexture = volumeTexture;

            GraphBuilder.AddPass(
                RDG_EVENT_NAME("RenderToScreen"),
                PassParams,
                ERDGPassFlags::Raster,
                [this, &View, LocalParameter, VertexShader, PixelShader, PassParams, UVScale](
                    FRHICommandList& RHICmdList)
                {
                    RenderToScreen(RHICmdList, View, LocalParameter, VertexShader, PixelShader, PassParams,
                        UVScale);
                });
        }
    }
}
```
开头先判断VolumeRT(ComputePass中提取出来的纹理资源)是否为空，不为空的情况下，调用 RegisterExternalTexture 函数将它再注册到本Graph中。

> 提取的动作不是即时完成的，在Raster Pass时是不能保证VolumeRT一定不为空，因此我们需要加以判断。

本例中两个Pass很大可能不是在同一帧执行，也就不是在同一个图中（图是每一帧都要重新创建的），因此要通过两个步骤：*提取、注册* 来共享资源。

