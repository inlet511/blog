---
title: "RDG 01-Shader参数"
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
---
## 1 概述
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

## 2 Shader Parameters 宏

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
### 2.1 SHADER_PARAMETER_STRUCT的变体
我们还可以看到 *BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT* ，从名称上看，它是一个“全局的” Shader 参数结构体，它还有另外一个名称：
```cpp
 /** Legacy macro definitions. */
#define BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT \
	 BEGIN_UNIFORM_BUFFER_STRUCT
```
即 *UNIFORM_BUFFER_STRUCT* ， 两者完全一样。或者我们可以这样理解：UniformBuffer 就是“全局的着色器参数”。

关于Uniform Buffer，我们稍后再分析。

### 2.2 SHADER_PARAMETER的种类
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
### 2.3 简化Shader定义
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
		SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FDisplayControlParameters, CrossSectionUniformBuffer)
		SHADER_PARAMETER_RDG_TEXTURE(Texture3D, VolumeTexture)
		SHADER_PARAMETER_SAMPLER(SamplerState, VolumeSampler)
		SHADER_PARAMETER_TEXTURE(Texture2D, GradientTexture)
		SHADER_PARAMETER_SAMPLER(SamplerState, GradientSampler)
		SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneTextureUniformParameters, SceneTextures)
		RENDER_TARGET_BINDING_SLOTS()
	END_SHADER_PARAMETER_STRUCT()
};
IMPLEMENT_GLOBAL_SHADER(FCrossSectionPS, "/Engine/Private/MyPassShader.usf", "CrossSectionPS", SF_Pixel)
```
### 2.4 无参Shader
有些Shader不需要任何参数输入，就可以使用 
```cpp
using FParameters = FEmptyShaderParameters; 
```
例如：
```cpp
class FGenerateMipsVS : public FGlobalShader
{
public:
	DECLARE_GLOBAL_SHADER(FGenerateMipsVS);
	SHADER_USE_PARAMETER_STRUCT(FGenerateMipsVS, FGlobalShader);
	using FParameters = FEmptyShaderParameters;

	static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
	{
		return IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::ES3_1);
	}

	static void ModifyCompilationEnvironment(const FShaderPermutationParameters&, FShaderCompilerEnvironment& OutEnvironment)
	{
		OutEnvironment.SetDefine(TEXT("GENMIPS_COMPUTE"), 0);
	}
};
```

### 2.5 用法
在C++中定义完Shader以及其参数之后，首先在渲染函数中获取到该Shader，以GlobalShader为例：
```cpp
	const FGlobalShaderMap* GlobalShaderMap = GetGlobalShaderMap(GMaxRHIFeatureLevel);
	const TShaderMapRef<FMyVS> VertexShader(GlobalShaderMap);
	const TShaderMapRef<FMyPS> PixelShader(GlobalShaderMap);
	const TShaderMapRef<FMyCS> ComputeShader(GlobalShaderMap);
	const TShaderMapRef<FCrossSectionVS> CrossSectionVS(GlobalShaderMap);
	const TShaderMapRef<FCrossSectionPS> CrossSectionPS(GlobalShaderMap);
```
上述代码中，先通过GetGlobalShaderMap()函数获取全局Shader的Map，然后就可以通过TShaderMapRef<> 从GlobalShaderMap中初始化特定类型的Shader引用，这里创建了5个引用。

为参数分配内存并填充成员。
```cpp
// Create PS Parameters
FCrossSectionPS::FParameters* PSParams = GraphBuilder.AllocParameters<FCrossSectionPS::FParameters>();
PSParams->SceneTextures = SceneTexturesWithDepth;
PSParams->UniformBuffer = SceneInfo->CreateUniformBufferRDG(GraphBuilder);
PSParams->CrossSectionUniformBuffer = SceneInfo->CreateCrossSectionUBRDG(GraphBuilder);
PSParams->VolumeTexture = volumeTexture;
PSParams->VolumeSampler = TStaticSamplerState<SF_Bilinear, AM_Clamp, AM_Clamp, AM_Clamp>::GetRHI();
PSParams->GradientTexture = Scene->GetRadiationSceneInfo()->GradientTexture->TextureReference.TextureReferenceRHI;
PSParams->GradientSampler = TStaticSamplerState<SF_Bilinear, AM_Clamp, AM_Clamp, AM_Clamp>::GetRHI();
PSParams->RenderTargets[0] = FRenderTargetBinding(SceneColorTexture, ERenderTargetLoadAction::ELoad);
PSParams->RenderTargets.DepthStencil = FDepthStencilBinding(
	SceneDepthTexture, ERenderTargetLoadAction::ELoad, ERenderTargetLoadAction::ELoad,
	FExclusiveDepthStencil::DepthWrite_StencilWrite);
```
上述代码中，通过GraphBuilder.AllocParameters()泛型函数为Shader参数分配了空间，接着分别对Shader参数的各个成员赋值。

接下来通过 GraphBuilder.AddPass() 添加渲染Pass 或者通过 FComputeShaderUtils::AddPass() 增加Compute Pass.

AddPass添加Pass时要将上面的参数传入Pass。例如：

```cpp
GraphBuilder.AddPass(
				RDG_EVENT_NAME("RenderCrossSection"),
				PSParams, // 上面的参数
				ERDGPassFlags::Raster | ERDGPassFlags::NeverCull,
				[this, &View, CrossSectionVS, SceneInfo, CrossSectionPS, VSParams, PSParams](FRHICommandList& RHICmdList)
				{
					RenderCrossSection(RHICmdList, View, SceneInfo, CrossSectionVS, CrossSectionPS, VSParams, PSParams);
				});
```

特别注意的一点是，AddPass的第二个参数有以下情形：
- 可以是PixelShader的参数
- 可以是PixelShader和VertexShader共享的参数，两者都派生自一个共同的父类。可以查看 FShaderDrawSymbolsVS，FShaderDrawSymbolsPS的例子。





## 3 usf shader参数

### 3.1 usf 数据类型
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


### 3.2 C++和usf参数对应
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
