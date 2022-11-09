---
title: "RDG 05 StructuredBuffer的用法"
date: 2022-11-07T00:10:45+08:00
draft: false
toc: true
categories: [UE5]
series:
  - RDG
tags:
  - 图形学
  - UE
  - RDG
  - StructuredBuffer
  - 结构缓冲区
---
StructuredBuffer(结构缓冲区)适合于向Shader传入一组结构相同的数据，这组数据可以是基本数据类型，也可以是自定义的结构体(struct)。

在Shader参数以及usf shader中，使用StructuredBuffer<类型>来声明一个只读的结构缓冲区，使用RWStructuredBuffer<类型> 来声明一个可读可写的结构缓冲区。

StructuredBuffer的容量限制未查到准确资料，但其本质上也是一个Buffer，推测应该遵循其他Buffer的容量限制，也就是min(VRAM*25%,2GB)，但是至少可以保证128MB。

## 1 用于RDG_BUFFER_SRV
### 1.1 定义结构体
就是一个普通的C++结构体（不需要是USTRUCT)
```cpp
struct ShaderPrintItem
{
    FVector2D ScreenPos;
    int32 Value;
    int32 Type;
};
```
### 1.2 定义Shader参数
 使用 *SHADER_PARAMETER_RDG_BUFFER_SRV* 宏来定义一个由RDG跟踪的StructuredBuffer参数
```cpp
class FShaderDrawSymbols : public FGlobalShader
	{
	public:
		FShaderDrawSymbols()
		{}

		FShaderDrawSymbols(const ShaderMetaType::CompiledShaderInitializerType& Initializer)
			: FGlobalShader(Initializer)
		{}

		BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
			RENDER_TARGET_BINDING_SLOTS()
			SHADER_PARAMETER_STRUCT_REF(FUniformBufferParameters, UniformBufferParameters)
			SHADER_PARAMETER_TEXTURE(Texture2D, MiniFontTexture)
			SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<ShaderPrintItem>, SymbolsBuffer)
			RDG_BUFFER_ACCESS(IndirectDrawArgsBuffer, ERHIAccess::IndirectArgs)
		END_SHADER_PARAMETER_STRUCT()

		static bool ShouldCompilePermutation(FGlobalShaderPermutationParameters const& Parameters)
		{
			return IsSupported(Parameters.Platform);
		}
	};

class FShaderDrawSymbolsVS : public FShaderDrawSymbols
{
    DECLARE_GLOBAL_SHADER(FShaderDrawSymbolsVS);
    SHADER_USE_PARAMETER_STRUCT(FShaderDrawSymbolsVS, FShaderDrawSymbols);
};

IMPLEMENT_GLOBAL_SHADER(FShaderDrawSymbolsVS, "/Engine/Private/ShaderPrintDraw.usf", "DrawSymbolsVS", SF_Vertex);

class FShaderDrawSymbolsPS : public FShaderDrawSymbols
{
    DECLARE_GLOBAL_SHADER(FShaderDrawSymbolsPS);
    SHADER_USE_PARAMETER_STRUCT(FShaderDrawSymbolsPS, FShaderDrawSymbols);
};

IMPLEMENT_GLOBAL_SHADER(FShaderDrawSymbolsPS, "/Engine/Private/ShaderPrintDraw.usf", "DrawSymbolsPS", SF_Pixel);
```
### 1.3 创建StructuredDesc
```cpp
FRDGBufferDesc::CreateStructuredDesc(sizeof(ShaderPrintItem), GetMaxSymbolCount() + 1)
```
#### FRDGBufferDesc
FRDGBufferDesc是一个描述结构体的类型，创建*FRDGBufferDesc*一般使用静态函数 *CreateStructuredDesc*:
```cpp
static FRDGBufferDesc CreateStructuredDesc(uint32 BytesPerElement, uint32 NumElements)
```
两个参数分别描述了StructuredBuffer一个元素有多少个字节，以及一共有多少个元素。注意StructuredBuffer的元素数量在这里就需要指定，并且后面是无法修改的。也就是说不可以实现动态长度的StructuredBuffer。

### 1.4 创建Buffer
```cpp
FRDGBufferRef SymbolBuffer = GraphBuilder.CreateBuffer(FRDGBufferDesc::CreateStructuredDesc(sizeof(ShaderPrintItem), GetMaxSymbolCount() + 1), TEXT("ShaderPrintSymbolBuffer"));
```
对于RDG跟踪的Buffer，使用 *FRDGBuilder.CreateBuffer*函数来创建,它最常用的函数重载原型如下：
```cpp
inline FRDGBufferRef FRDGBuilder::CreateBuffer(
	const FRDGBufferDesc& Desc,
	const TCHAR* Name,
	ERDGBufferFlags Flags)
```
它接受一个FRDGBufferDesc，一个名称，和一个ERDGBufferFlags参数。其中FRDGBufferDesc已经在1.3中讲解

#### Name
名称是一个标识，用于图形调试。

#### ERDGBufferFlags
ERDGBufferFlags枚举定义如下：
```cpp
/** Flags to annotate a render graph buffer. */
enum class ERDGBufferFlags : uint8
{
	None = 0,

	/** Tag the buffer to survive through frame, that is important for multi GPU alternate frame rendering. */
	MultiFrame = 1 << 0,

	/** The buffer may only be used for read-only access within the graph. This flag is only allowed for registered buffers. */
	ReadOnly = 1 << 1, 

	/** Force the graph to track this resource even if it can be considered as readonly (no UAV, no RTV, etc.) This allows the graph copying from and to textures, and handling the corresponding transitions, for example.
	 This flag is only allowed for registered buffers. Mutually exclusive with ReadOnly. */
	ForceTracking = 1 << 2,
};
```
MultiFrame 表示让这个Buffer的生命周期是多帧的（不止一帧），ReadOnly表示这个Buffer在RDG中仅仅用于只读的用途。ForceTracking表示强制RDG追踪该资源，即使它可以被当作只读的（不是UAV, RTV等），这就允许rdg 向纹理拷贝这个buffer，或者从纹理拷贝到这个buffer，并处理好资源转换。

### 1.5 两步合一步
如果觉得前面两个步骤(1.3和1.4)比较麻烦，虚幻还为我们提供了工具函数*CreateStructuredBuffer*，将上面两个步骤合并为一步, 它定义于RenderGraphUtils.h， 使用时需要include该文件。

使用范例：
```cpp
FRDGBufferRef Buffer = CreateStructuredBuffer(GraphBuilder, TEXT("RadiationDataBuffer"), SceneInfo->GetRadiationData(),ERDGInitialDataFlags::NoCopy);
```
该函数有多个重载，以下是最常用的重载：

```cpp
/** 于从TArray获取初始数组
 * Helper to create a structured buffer with initial data from a TArray.
 */
template <typename ElementType, typename AllocatorType>
FORCEINLINE FRDGBufferRef CreateStructuredBuffer(
	FRDGBuilder& GraphBuilder,
	const TCHAR* Name,
	const TArray<ElementType, AllocatorType>& InitialData,
	ERDGInitialDataFlags InitialDataFlags = ERDGInitialDataFlags::None)
{
	static const ElementType DummyElement = ElementType();
	if (InitialData.Num() == 0)
	{
		// 如果没有数据，则调用以下重载
		return CreateStructuredBuffer(GraphBuilder, Name, InitialData.GetTypeSize(), 1, &DummyElement, InitialData.GetTypeSize(), ERDGInitialDataFlags::NoCopy);
	}
	return CreateStructuredBuffer(GraphBuilder, Name, InitialData.GetTypeSize(), InitialData.Num(), InitialData.GetData(), InitialData.Num() * InitialData.GetTypeSize(), InitialDataFlags);
}
```
第三个参数是要上传的数组，是一个TArray。

第四个参数只有两种取值：ERDGInitialDataFlags::None 和 ERDGInitialDataFlags::NoCopy。None是默认行为，当Graph执行的时候，会拷贝初始数组的内容，用户无需维护数据的指针的生命周期。如果用户可以确保初始数组的生命周期会存活到Graph执行结束，那么就可以使用NoCopy，这样upload pass就使用数据的引用去上传，这是一种优化选项，一定在确保数据生命周期的情况下再使用。

### 1.6 创建SRV视图
```cpp
typedef FShaderDrawSymbols SHADER;
TShaderMapRef< FShaderDrawSymbolsVS > VertexShader(GlobalShaderMap);
TShaderMapRef< FShaderDrawSymbolsPS > PixelShader(GlobalShaderMap);

SHADER::FParameters* PassParameters = GraphBuilder.AllocParameters<SHADER::FParameters>();
PassParameters->RenderTargets[0] = FRenderTargetBinding(OutputTexture.Texture, ERenderTargetLoadAction::ELoad);
PassParameters->UniformBufferParameters = UniformBuffer;
PassParameters->MiniFontTexture = FontTexture;
PassParameters->SymbolsBuffer = GraphBuilder.CreateSRV(SymbolBuffer);
```
使用*GraphBuilder.CreateSRV*函数为上一步创建的StructuredBuffer创建一个SRV视图，并传递给Shader参数。

CreateSRV有三种重载, 分别是对应使用TextureSRVDesc、BufferSRVDesc以及RDGBufferRef的情形：
```cpp
FRDGTextureSRVRef CreateSRV(const FRDGTextureSRVDesc& Desc);
FRDGBufferSRVRef CreateSRV(const FRDGBufferSRVDesc& Desc);
FORCEINLINE FRDGBufferSRVRef CreateSRV(FRDGBufferRef Buffer, EPixelFormat Format);
```
> 我们这里只给了一个FRDGBufferRef的参数，但是没有给第二个参数，且 FRDGBufferRef到FRDGBufferSRVDesc之间是可以直接进行隐式转换的，因此这里调用的是第二种重载。



### 1.7 随Shader参数传入Pass
```cpp
GraphBuilder.AddPass(
				RDG_EVENT_NAME("ShaderPrint::DrawSymbols"),
				PassParameters,
				ERDGPassFlags::Raster,
                //.....
```

### 1.8 Usf中读取StructuredBuffer
在usf文件的开头，声明一个StructuredBuffer，其泛型参数的结构体类型要和第一步C++中声明的结构体类型对应。
```c
struct FPackedShaderPrintItem
{
	float2 ScreenPos; // Position in normalized coordinates
	int Value;        // Cast to value or symbol
	uint TypeAndColor;//
};

StructuredBuffer<FPackedShaderPrintItem> SymbolsBuffer;
```

之后就可以当作数据去获取它的值，需要注意的是下标不可以超过1.3中定义的Buffer的长度。
```c
FShaderPrintItem Symbol = UnpackShaderPrintItem(SymbolsBuffer[InstanceId + SHADER_PRINT_VALUE_OFFSET]);
```
## 2 用于RDG_BUFFER_UAV
和用于RDG_BUFFER_SRV大致相同，这里只说两者的区别。
### 2.1 Shader参数类型不同
Shader的参数声明类型不同，SRV中是 StructuredBuffer<>，而这里是 RWStructuredBuffer<>，RW 意为ReadWrite，可读可写，这也是UAV的特性。
### 2.2 创建资源视图的函数不同
在SRV中我们使用的是CreateSRV函数，这里则使用CreateUAV函数，CreateUAV函数有四种重载：
```cpp
FRDGTextureUAVRef CreateUAV(FRDGTextureRef Texture, ERDGUnorderedAccessViewFlags Flags = ERDGUnorderedAccessViewFlags::None);
FRDGBufferUAVRef CreateUAV(FRDGBufferRef Buffer, EPixelFormat Format, ERDGUnorderedAccessViewFlags Flags = ERDGUnorderedAccessViewFlags::None);
FORCEINLINE FRDGTextureUAVRef CreateUAV(const FRDGTextureUAVDesc& Desc, ERDGUnorderedAccessViewFlags InFlags);
FORCEINLINE FRDGBufferUAVRef CreateUAV(const FRDGBufferUAVDesc& Desc, ERDGUnorderedAccessViewFlags InFlags);
```
依次对应的是使用RDGTextureRef、RDGBufferRef、RDGTextureUAVDesc、RDGBufferUAVDesc的情况，且参数各有不同。

一个使用范例:
```cpp
PassParameters->RWSymbolsBuffer = GraphBuilder.CreateUAV(SymbolBuffer, EPixelFormat::PF_R32_UINT);
```

## 3 完整范例
### 3.1 结构体定义
```cpp
struct RadiationData
{
public:
	FVector3f Position;
	float Value;
};
```
### 3.2 Shader中参数的声明
```cpp
	BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
	// 省略了其他参数
	SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<RadiationData>, InDataBuffer)
	END_SHADER_PARAMETER_STRUCT()
```
### 3.3 渲染函数
```cpp
	FRDGBufferRef Buffer = CreateStructuredBuffer(GraphBuilder, TEXT("RadiationDataBuffer"), SceneInfo->GetRadiationData(),ERDGInitialDataFlags::NoCopy);

	// Compute Shader Parameters and Textures
	FMyCS::FParameters* CSParams = GraphBuilder.AllocParameters<FMyCS::FParameters>();
	CSParams->InDataBuffer = GraphBuilder.CreateSRV(Buffer);

	FComputeShaderUtils::AddPass(
		GraphBuilder,
		RDG_EVENT_NAME("ComputeVolumeTexture"),
		ComputeShader,
		CSParams,
		FIntVector(16, 16, 32));
```
### 3.4 usf
```c
//声明和C++中对应的结构体
struct RadiationData
{
    float3 Position;
    float Value;
};
//声明StructuredBuffer
StructuredBuffer<RadiationData> InDataBuffer;

//访问数据范例：
float3 radisourcePos = float3(InDataBuffer[i].Position.x, InDataBuffer[i].Position.y, InDataBuffer[i].Position.z);
```


