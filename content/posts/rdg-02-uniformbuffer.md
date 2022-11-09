---
title: "RDG 02 Uniformbuffer"
date: 2022-11-01T14:01:29+08:00
draft: false
toc: true
categories: [UE5]
series:
  - RDG
tags:
  - 图形学
  - UE
  - RDG
  - Uniform Buffer
---
## 1 两种引用UniformBuffer的方法
如上文所述，Uniform Buffer即一个全局的Shader参数。
我们在 [这篇文章]({{< ref "/posts/ue4-global-shaders-uniformbuffer.md" >}})
中已经讲过UniformBuffer的声明和定义，在RDG环境下，它的声明和定义方式仍然是一样的，有区别的是传入数据的方式。

声明使用的宏是 BEGIN_UNIFORM_BUFFER_STRUCT, 一般放在.h文件中， 定义使用的宏是 IMPLEMENT_UNIFORM_BUFFER_STRUCT. 必须放在.cpp文件中。

在SHADER_PARAMETER宏中引用Uniform Buffer，有两种方法可以可以实现这个目的：

- SHADER_PARAMETER_RDG_UNIFORM_BUFFER
- SHADER_PARAMETER_STRUCT_REF
  
他们实现的目的是一样的，都是引用一个 Uniform Buffer 作为本Shader的参数，但是前者(RDG版本) 适用于 由RDG 跟踪的Uniform Buffer，而后者适用于非 RDG 跟踪的Uniform Buffer。

## 2 RDG跟踪的UniformBuffer
  所谓“由RDG跟踪”的Uniform Buffer的使用流程如下：
  
  1. 声明和定义的部分省略
  2. 在Shader的c++定义中加入对UniformBuffer的引用
```cpp
class FMyPS : public FGlobalShader
{
public:
	DECLARE_GLOBAL_SHADER(FMyPS);
	SHADER_USE_PARAMETER_STRUCT(FMyPS,FGlobalShader)

	BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
		SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FRadiationUniformParameters, UniformBuffer)
        // ...
	END_SHADER_PARAMETER_STRUCT()
```
  
  1. 使用RDG方法为Shader参数分配内存：
  ```cpp
  // 先分配整个Shader参数的内存
  FMyPS::FParameters* PassParams = GraphBuilder.AllocParameters<FMyPS::FParameters>();
  ```
  4. 使用RDG方法分配UniformBuffer的内存并且创建UniformBuffer：
```cpp
  // 使用RDG方法分配UniformBuffer的内存
  RadiationParameters = GraphBuilder.AllocParameters<FRadiationUniformParameters>();
  // 填充结构体数值，此处省略
  // 使用RDG方法创建UniformBuffer
  TRDGUniformBufferRef<FRadiationUniformParameters> ub = GraphBuilder.CreateUniformBuffer(RadiationParameters);
```
  注意GraphBuilder.CreateUniformBuffer生成的对象是 *TRDGUniformBufferRef* 类型。

  5. 把创建好的UniformBuffer传入Shader参数：
```cpp
    PassParams->UniformBuffer = ub;
```
  6. Shader中使用
在usf shader中使用uniform buffer的方法实际上和旧的方法是完全一样的，同样是*无需在shader中定义*，在shader中直接通过IMPLEMENT_GLOBAL_SHADER_PARAMETER_STRUCT宏定义的名称进行引用即可。
```cpp
IMPLEMENT_GLOBAL_SHADER_PARAMETER_STRUCT(FRadiationUniformParameters, "RadiationUniform");
```
```c
// shader文件中无需声明，直接通过 "RadiationUniform"引用即可。
uv = (p - RadiationUniform.BoundsMin) / RadiationUniform.BoundsSize;
```

注意，对于非自定义的、系统已提供的UniformBuffer，在Render函数中可以直接获取到，这样就可以省略上面的3、4两步，而直接进入第5步。

例如常见的 TRDGUniformBufferRef<FSceneTextureUniformParameters> SceneTextures，它包含场景颜色、场景深度、GBuffer等等内容。
```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FSceneTextureUniformParameters, RENDERER_API)
	// Scene Color / Depth
	SHADER_PARAMETER_RDG_TEXTURE(Texture2D, SceneColorTexture)
	SHADER_PARAMETER_RDG_TEXTURE(Texture2D, SceneDepthTexture)

	// GBuffer
	SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferATexture)
	SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferBTexture)
	SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferCTexture)
	SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferDTexture)
	SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferETexture)
	SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferFTexture)
	SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferVelocityTexture)

	// SSAO
	SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenSpaceAOTexture)

	// Custom Depth / Stencil
	SHADER_PARAMETER_RDG_TEXTURE(Texture2D, CustomDepthTexture)
	SHADER_PARAMETER_SRV(Texture2D<uint2>, CustomStencilTexture)

	// Misc
	SHADER_PARAMETER_SAMPLER(SamplerState, PointClampSampler)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```
在DeferredShadingRenderer的Render函数中，它先被创建出来:
```cpp
TRDGUniformBufferRef<FSceneTextureUniformParameters> SceneTextures = CreateSceneTextureUniformBuffer(GraphBuilder, FeatureLevel, SceneTexturesSetupMode);

// CreateSceneTextureUniformBuffer的定义：
TRDGUniformBufferRef<FSceneTextureUniformParameters> CreateSceneTextureUniformBuffer(
	FRDGBuilder& GraphBuilder,
	ERHIFeatureLevel::Type FeatureLevel,
	ESceneTextureSetupMode SetupMode)
{
	FSceneTextureUniformParameters* SceneTextures = GraphBuilder.AllocParameters<FSceneTextureUniformParameters>();
	FSceneRenderTargets& SceneContext = FSceneRenderTargets::Get(GraphBuilder.RHICmdList);
	SetupSceneTextureUniformParameters(&GraphBuilder, FeatureLevel, SceneContext, SetupMode, *SceneTextures);
	return GraphBuilder.CreateUniformBuffer(SceneTextures);
}
```

随后内容被逐步填充，所以在合适的时机，我们可以直接使用SceneTextures引用该Uniform Buffer。
```cpp
//定义
	BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
    //...
		SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneTextureUniformParameters, SceneTextures)
	END_SHADER_PARAMETER_STRUCT()
//使用
PassParams->SceneTextures = SceneTextures;
```
## 3 非RDG跟踪的UniformBuffer
非RDG跟踪的UniformBuffer指的是从创建到传值都使用的是非RDG方法的UniformBuffer，可以参考 [这篇文章]({{< ref "/posts/ue4-global-shaders-uniformbuffer.md" >}})。
使用 SHADER_PARAMETER_STRUCT_REF 宏的典型情况是 FViewUniformShaderParameters。

这个例子我们在"usf shader参数"部分也引用了，因此我们在需要使用View的情况下，参考"usf shader参数"部分的讲解进行设置，最后通过以下代码获取ViewUniformBuffer并传入shader的参数：
```cpp
PassParams->View = View.ViewUniformBuffer; 
```
其中View结构一般是从Render函数或者渲染委托中得到的，例如：
```cpp
for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
{
	const FViewInfo& View = Views[ViewIndex];
	//...
}
```