---
title: "RDG-Shader参数和UniformBuffer"
date: 2022-10-31T21:34:21+08:00
draft: false
toc: true
categories: [UE5]
series:
  - RDG
tags:
  - 图形学
  - UE
  - RDG
  - ShaderParameters
  - UniformBuffer
---
## 概述
Render Dependency Graph 简称RDG，从UE4最后几个版本开始逐步取代原来的渲染代码。 到了UE5，逐步完善，在源码中应用范围也日趋广泛。后面关于RDG的描述均以 UE5 为背景。
根据官方文档，RDG会提供以下便利：

- 安排异步计算栅栏的执行
- 以最佳生命周期和内存别名分配临时资源
- 使用拆分屏障转换子资源，在GPU上隐藏延迟并改善重叠
- 命令列表并行记录
- 剔除图中未使用的资源和通道
- 验证API的使用和资源依赖关系
- 在RDG Insights中实现图结构和内存生命周期的可视化

可见使用RDG就不用设置计算栅栏，不用管理生命周期， 还有其他诸多便利。

## Shader Parameters 宏

在RDG出现之前，我们要给Shader添加参数需要三个步骤：[定义和使用一个属性的三个步骤]({{< ref "/posts/ue4-global-shaders-creation.md#defineparameter" >}})

在RDG出现之后，我们可以使用一组宏来简化这个工作：*BEGIN/END_SHADER_PARAMETERS* 以及*SHADER_PARAMETER[...]* 等

举例：
```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters, /** MODULE_API_TAG */)
    SHADER_PARAMETER(FVector2D, ViewportSize)
    SHADER_PARAMETER(FVector4, Hello)
    SHADER_PARAMETER(float, World)
    SHADER_PARAMETER_ARRAY(FVector, FooBarArray, [16])

    SHADER_PARAMETER_TEXTURE(Texture2D, BlueNoiseTexture)
    SHADER_PARAMETER_SAMPLER(SamplerState, BlueNoiseSampler)

    SHADER_PARAMETER_TEXTURE(Texture2D, SceneColorTexture)
    SHADER_PARAMETER_SAMPLER(SamplerState, SceneColorSampler)

    SHADER_PARAMETER_UAV(RWTexture2D, SceneColorOutput)
END_SHADER_PARAMETER_STRUCT()
```
### SHADER_PARAMETER_STRUCT的变体
我们还可以看到 *BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT* ，从名称上看，它是一个“全局的” Shader 参数结构体，它还有另外一个名称：
```cpp
 /** Legacy macro definitions. */
#define BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT \
	 BEGIN_UNIFORM_BUFFER_STRUCT
```
即 *UNIFORM_BUFFER_STRUCT* ， 两者完全一样。或者我们可以这样理解：UniformBuffer 就是“全局的着色器参数”。

关于Uniform Buffer，我们稍后再分析。

### SHADER_PARAMETER的种类
*SHADER_PARAMETER[...]* 系列宏还有这些种类：
```cpp
#define SHADER_PARAMETER(MemberType, MemberName)
#define SHADER_PARAMETER_EX(MemberType,MemberName,Precision)
#define SHADER_PARAMETER_ARRAY(MemberType,MemberName,ArrayDecl)
#define SHADER_PARAMETER_ARRAY_EX(MemberType,MemberName,ArrayDecl,Precision)

#define SHADER_PARAMETER_TEXTURE(ShaderType,MemberName)
#define SHADER_PARAMETER_TEXTURE_ARRAY(ShaderType,MemberName, ArrayDecl)
#define SHADER_PARAMETER_SRV(ShaderType,MemberName)
#define SHADER_PARAMETER_UAV(ShaderType,MemberName)
#define SHADER_PARAMETER_SAMPLER(ShaderType,MemberName)
#define SHADER_PARAMETER_SAMPLER_ARRAY(ShaderType,MemberName, ArrayDecl)

#define SHADER_PARAMETER_STRUCT(StructType,MemberName)
#define SHADER_PARAMETER_STRUCT_ARRAY(StructType,MemberName, ArrayDecl)
// 引用另外一个Shader Parameter Structure
#define SHADER_PARAMETER_STRUCT_INCLUDE(StructType,MemberName)
// 引用一个全局Shader Parameter Structure(即Uniform Buffer Structure).
#define SHADER_PARAMETER_STRUCT_REF(StructType,MemberName)

// RDG Shader Parameter.
#define SHADER_PARAMETER_RDG_TEXTURE(ShaderType,MemberName)
#define SHADER_PARAMETER_RDG_TEXTURE_SRV(ShaderType,MemberName)
#define SHADER_PARAMETER_RDG_TEXTURE_UAV(ShaderType,MemberName)
#define SHADER_PARAMETER_RDG_TEXTURE_UAV_ARRAY(ShaderType,MemberName, ArrayDecl)
#define SHADER_PARAMETER_RDG_BUFFER(ShaderType,MemberName)
#define SHADER_PARAMETER_RDG_BUFFER_SRV(ShaderType,MemberName)
#define SHADER_PARAMETER_RDG_BUFFER_SRV_ARRAY(ShaderType,MemberName, ArrayDecl)
#define SHADER_PARAMETER_RDG_BUFFER_UAV(ShaderType,MemberName)
#define SHADER_PARAMETER_RDG_BUFFER_UAV(ShaderType,MemberName)
#define SHADER_PARAMETER_RDG_BUFFER_UAV_ARRAY(ShaderType,MemberName, ArrayDecl)
#define SHADER_PARAMETER_RDG_UNIFORM_BUFFER(StructType, MemberName)
```
### 简化Shader定义
BEGIN_SHADER_PARAMETER_STRUCT 可以放在Shader类外部，也可以放在类内部，推荐放在类内部，并且SHADER_PARAMETER_STRUCT的名称设置为 FParameters。

配合 SHADER_USE_PARAMETER_STRUCT(ShaderName, ShaderType) 宏，可以极大的简化Shader的定义。例如：

```cpp
// Pixel Shader Sample
class FCrossSectionPS : public FGlobalShader
{
public:
	DECLARE_SHADER_TYPE(FCrossSectionPS, Global);
	SHADER_USE_PARAMETER_STRUCT(FCrossSectionPS,FGlobalShader)
	BEGIN_SHADER_PARAMETER_STRUCT(FParameters,)
		SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FRadiationUniformParameters, UniformBuffer)
		SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FViewMatricesUniformParameters, MatricesUniformBuffer)
		SHADER_PARAMETER_RDG_TEXTURE(Texture3D,VolumeTexture)
		SHADER_PARAMETER_SAMPLER(SamplerState, VolumeSampler)
		SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneTextureUniformParameters, SceneTextures)		
	END_SHADER_PARAMETER_STRUCT()
};
IMPLEMENT_GLOBAL_SHADER(FCrossSectionPS, "/Engine/Private/MyPassShader.usf", "CrossSectionPS", SF_Pixel)
```
## usf shader参数

### usf 数据类型
usf的基本数据类型只能是hlsl的数据类型（另外也有一部分结构体和普通的hlsl不一样），需要和虚幻C++中的数据类型进行对应。 下面举几个例子：

|  UE类型              |   usf/hlsl类型  | 
|:-----                |:-----           |
|  float				|	float		|
|  FVector2D/FVector2f	|	float2		|
|  FVector/FVector3f	|	float3		|
|  FVector4D/FVector4f	|	float4		|
|  FMatrix				|	float4x4	|
|  Texture2D			|	Texture2D	|
|  Texture3D			|	Texture3D   |
|  RWTexture3D          |   RWTexture3D |

注意其中的Texture类数据，我们在C++中会对其设定储存的数据格式，在usf端我们往往也需要在其数据类型后面添加一对尖括号，并在括号内说明数据类型。下面举一例：

1. 参数定义，使用的是Texture3D类型，是一种三维贴图格式，但是Shader参数中并未指定该三维贴图的详细信息，例如尺寸，例如数据格式(每一个像素存储的是单通道还是多通道)。
```cpp
SHADER_PARAMETER_RDG_TEXTURE(Texture3D,VolumeTexture)
```
2. 纹理描述，并创建纹理。这里通过FIntVector制定了纹理尺寸，通过EPixelFormat::PF_R32_FLOAT说明了纹理存储的格式是32位的浮点数。
```cpp
FRDGTextureDesc TextDesc = FRDGTextureDesc::Create3D(
			FIntVector(textureSize, textureSize, textureSize),
			EPixelFormat::PF_R32_FLOAT,
			FClearValueBinding::None,
			TexCreate_RenderTargetable | TexCreate_ShaderResource | TexCreate_UAV);

FRDGTexture* vt = GraphBuilder.CreateTexture(TextDesc, TEXT("VolumeTexture"));
```
3. 中间又经过一系列操作，填充了数据，这里略去，只展示最终传入Shader参数：
```cpp
PassParams->VolumeTexture = vt;
```
4. 在usf中该参数的格式也要对应：
```c
Texture3D<float> VolumeTexture;
```
如果在C++中指定贴图格式是 EPixelFormat::PF_FloatRGB 或 EPixelFormat::PF_FloatRGBA , 则usf 中要改为
```c
// 对应PF_FloatRGB
Texture3D<float3> VolumeTexture;
// 对应PF_FloatRGBA
Texture3D<float4> VolumeTexture;
```


### C++和usf参数对应
*UniformBuffer除外的大多数Shader参数，在usf文件的顶部都要进行声明。*

C++的Shader定义中，Shader参数列表一定要包含该shader的 usf 文件的函数定义中所使用到的所有参数，*包括通过函数间接引用到的*。

例如某个usf 的Shader中调用了ConvertFromDeviceZ 方法，这个方法中使用了View这个Uniform，因此我们需要在参数列表中加入
```cpp
SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
```
否则会出现类似这样的报错：
> Shader FMyPS, permutation 0 has unbound parameters not represented in the parameter struct:  View

另外由于 ConvertFromDeviceZ 函数是定义于 Common.ush中的，所以在usf文件开头要加入
```c
#include "Common.ush"
```

## Uniform Buffer
### 两种引用UniformBuffer的方法
如前所述，Uniform Buffer即一个全局的Shader参数。
我们在 [这篇文章]({{< ref "/posts/ue4-global-shaders-uniformbuffer.md" >}})
中已经讲过UniformBuffer的声明和定义，在RDG环境下，它的声明和定义方式仍然是一样的，有区别的是传入数据的方式。

声明使用的宏是 BEGIN_UNIFORM_BUFFER_STRUCT, 一般放在.h文件中， 定义使用的宏是 IMPLEMENT_UNIFORM_BUFFER_STRUCT. 必须放在.cpp文件中。

在SHADER_PARAMETER宏中引用Uniform Buffer，有两种方法可以可以实现这个目的：

- SHADER_PARAMETER_RDG_UNIFORM_BUFFER
- SHADER_PARAMETER_STRUCT_REF
  
他们实现的目的是一样的，都是引用一个 Uniform Buffer 作为本Shader的参数，但是前者(RDG版本) 适用于 由RDG 跟踪的Uniform Buffer，而后者适用于非 RDG 跟踪的Uniform Buffer。

### RDG跟踪的UniformBuffer
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
### 非RDG跟踪的UniformBuffer
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