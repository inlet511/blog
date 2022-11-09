---
title: "Rdg 06 Resources"
date: 2022-11-09T15:12:32+08:00
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
  - 资源
---

## 创建资源
### 创建资源描述
要创建资源，首先要创建描述，例：
```cpp
FRDGTextureDesc TextDesc = FRDGTextureDesc::Create3D(
		FIntVector(textureSize, textureSize, textureSize),
		EPixelFormat::PF_R32_FLOAT,
		FClearValueBinding::None,
		TexCreate_ShaderResource | TexCreate_UAV);
```
上例中创建了一个3D纹理资源，尺寸大小为textureSize的立方体纹理，其每个“像素”存储的时一个32位的float类型数据，没有设置Clear值，将来可以用于创建SRV以及UAV视图。

其他创建资源描述的函数：
```cpp
static FRDGTextureDesc Create2D(FIntPoint InExtent,EPixelFormat InFormat,FClearValueBinding InClearValue,ETextureCreateFlags InFlags,uint8 InNumMips = 1,uint8 InNumSamples = 1);
static FRDGTextureDesc Create2DArray(FIntPoint InExtent,EPixelFormat InFormat,FClearValueBinding InClearValue,ETextureCreateFlags InFlags,uint32 InArraySize,uint8 InNumMips = 1,uint8 InNumSamples = 1);
static FRDGTextureDesc Create3D(FIntVector InSize,EPixelFormat InFormat,FClearValueBinding InClearValue,ETextureCreateFlags InFlags,uint8 InNumMips = 1,uint8 InNumSamples = 1);
static FRDGTextureDesc CreateCube(uint32 InSizeInPixels,EPixelFormat InFormat,FClearValueBinding InClearValue,ETextureCreateFlags InFlags,uint8 InNumMips = 1,uint8 InNumSamples = 1);
static FRDGTextureDesc CreateCubeArray(uint32 InSizeInPixels,EPixelFormat InFormat,FClearValueBinding InClearValue,ETextureCreateFlags InFlags,uint32 InArraySize,uint8 InNumMips = 1,uint8 InNumSamples = 1);
```
### 创建资源
例：
```cpp
FRDGTextureRef VolumeTexture = GraphBuilder.CreateTexture(TextDesc, TEXT("RadiationVolumeTexture"));
```
创建纹理资源使用:
```cpp
FRDGTextureRef CreateTexture(const FRDGTextureDesc& Desc, const TCHAR* Name, ERDGTextureFlags Flags = ERDGTextureFlags::None);
```
创建Buffer资源使用：
```cpp
FRDGBufferRef CreateBuffer(const FRDGBufferDesc& Desc, const TCHAR* Name, ERDGBufferFlags Flags = ERDGBufferFlags::None);
```
### RHI资源
一个RDG资源包含RHI Resource的描述符, 关联的RHI资源仅在(将此资源当作Pass参数传入的)Pass的Lambda函数体内有效。

所有RDG资源类型都提供一个FRDGResource::GetRHI()函数override，用于获取相应类型的RHI资源。 *此函数严格限定在Pass的Lambda函数中执行。*

下面是特定于Buffer和Texture的性质：
- 资源可以是Transient(临时的), 因此它的生命周期被限制在图上，并且可以与其他生命周期不相交的临时资源以别名方式共存。
- 资源可以是External(外部的), 其生命周期延伸到图外。如果用户将现有RHI资源注册到图中，或者在执行完成后从图中提取资源，就会发生这种情况。

RDG Buffer和 Texture可以通过RDG UAV 或 SRV 进行处理。和其他资源一样，底层RHI资源在执行期间将按需分配，*仅允许声明为Pass参数的Pass访问*。

资源创建演示:
```cpp
// 创建一个新的临时纹理实例。此时未分配GPU内存，仅分配了描述符。
FRDGTexture* Texture = GraphBuilder.CreateTexture(FRDGTextureDesc::Create2D(...), TEXT("MyTexture"));

// 无效！将触发断言。如果在通道上声明，则仅允许在通道Lambda中使用！
FRHITexture* TextureRHI = Texture->GetRHI();

// 创建一个新的UAV，引用特定mip级别的纹理。
FRDGTextureUAV* TextureUAV = GraphBuilder.CreateUAV(FRDGTextureUAVDesc(Texture, MipLevel));

// 无效！
FRHIUnorderedAccessView* UAVRHI = TextureUAV->GetRHI();

// 创建一个新的临时结构化缓冲区实例。
FRDGBuffer* Buffer = GraphBuilder.CreateBuffer(FRDGBufferDesc::CreateBufferDesc(...), TEXT("MyBuffer"));

// 无效！
FRHIBuffer* BufferRHI= Buffer->GetRHI();

// 创建一个新的SRV，引用具有R32浮点格式的缓冲区。
FRDGBufferSRV* BufferSRV = GraphBuilder.CreateSRV(Buffer, PF_R32_FLOAT);

// 无效！
FRHIShaderResourceView* SRVRHI = TextureSRV->GetRHI();
```

## 外部资源
如果资源的生命周期延伸到图外，则资源被视为External，这可能在两种情况下发生：
- 外部资源被注册到图中
- 从图中提取资源出来

注册外部资源会将资源的生命周期延长到图的前面。资源分配是发生在图的Setup阶段的。

提取资源则是相反的作用，它将资源的生命周期延长到图的末尾，因为用户现在持有了一个引用。

### 注册到图中
如果我们已经有非RDG创建的资源，可以通过“注册外部资源”将资源注册到RDG系统中。

注册外部纹理要用到RegisterExternalTexture()函数， 注册外部Buffer要用到 RegisterExternalBuffer()函数。

```cpp
FRDGTextureRef FRDGBuilder::RegisterExternalTexture(
		const TRefCountPtr<IPooledRenderTarget>& ExternalPooledTexture,
		ERenderTargetTexture Texture = ERenderTargetTexture::ShaderResource,
		ERDGTextureFlags Flags = ERDGTextureFlags::None);

FRDGTextureRef FRDGBuilder::RegisterExternalTexture(
		const TRefCountPtr<IPooledRenderTarget>& ExternalPooledTexture,
		const TCHAR* NameIfNotRegistered,
		ERenderTargetTexture RenderTargetTexture = ERenderTargetTexture::ShaderResource,
		ERDGTextureFlags Flags = ERDGTextureFlags::None);

FRDGBufferRef FRDGBuilder::RegisterExternalBuffer(const TRefCountPtr<FRDGPooledBuffer>& ExternalPooledBuffer, ERDGBufferFlags Flags = ERDGBufferFlags::None);
FRDGBufferRef FRDGBuilder::RegisterExternalBuffer(const TRefCountPtr<FRDGPooledBuffer>& ExternalPooledBuffer, ERDGBufferFlags Flags, ERHIAccess AccessFinal);

	/** Register an external buffer with a custom name. The name is only used if the buffer has not already been registered. */
FRDGBufferRef FRDGBuilder::RegisterExternalBuffer(
		const TRefCountPtr<FRDGPooledBuffer>& ExternalPooledBuffer,
		const TCHAR* NameIfNotRegistered,
		ERDGBufferFlags Flags = ERDGBufferFlags::None);
```
注册得到的对象是FRDGTextureRef或FRDGBuffer。需要注意的是，外部注册的资源，RDG无法控制和管理其生命周期，需要保证RDG使用期间外部资源的生命周期处于正常状态，否则将引发异常甚至程序崩溃。

使用上述这些方法将创建一个新的RDG资源, 用一个预分配的、引用计数的、池化的RHI资源指针来引用：
- *TRefCountPtr\<IPooledRenderTarget\>* 用于纹理，
- *TRefCountPtr\<FRDGPooledBuffer\>* 用于缓冲区。

范例:
```cpp
PassParameters->SkylightPdf = GraphBuilder.RegisterExternalTexture(GSystemTextures.BlackDummy);
```
上例中BlackDummy是一个TRefCountPtr\<IPooledRenderTarget\>类型的指针。 函数返回值是一个FRDGBufferRef类型的指针，我们可以把它传入Shader参数中。

### 提取资源
提取资源使用两个方法：
```cpp
//有几个重载，但至少都包含这两个形参:
void FRDGBuilder::QueueTextureExtraction(FRDGTextureRef Texture, TRefCountPtr<IPooledRenderTarget>* OutTexturePtr)

//有几种重载，但至少都包含这两个形参:
void FRDGBuilder::QueueBufferExtraction(FRDGBufferRef Buffer, TRefCountPtr<FRDGPooledBuffer>* OutBufferPtr)
```

从图中提取资源成功后，同样也会填充一个池化的资源指针。

如果用户并未持有图之外的引用，注册或提取的资源仍可在技术上稍后或更早地与帧中的其他RDG资源共享池化内存。

下面的代码演示了注册或提取纹理的方法。请注意，注册和提取如何使用相同的池化纹理类型，从而允许资源从图到图的传递。

```cpp
// 提取池化渲染目标。调用Execute()后，指针被填充。
TRefCountPtr<IPooledRenderTarget> ExtractedTexture;

// 第一个图生成纹理并提取该纹理。
{
    FRDGBuilder GraphBuilder(RHICmdList);

    FRDGTexture* Texture = GraphBuilder.CreateTexture(...);

    // ...

    GraphBuilder.QueueTextureExtraction(Texture, &ExtractedTexture);
    GraphBuilder.Execute();

    check(ExtractedTexture); // Valid
}

// 第二个图注册池化纹理。
{
    FRDGBuilder GraphBuilder(RHICmdList);

    // 注册池化渲染目标以获取RDG纹理。
    FRDGTexture* Texture = GraphBuilder.RegisterExternalTexture(ExtractedTexture);

    // ...

    GraphBuilder.Execute();
}
```

### 提取资源的替代方法
```cpp
void ConvertToExternalTexture(FRDGBuilder& GraphBuilder, FRDGTextureRef Texture, TRefCountPtr<IPooledRenderTarget>& OutPooledRenderTarget)
void ConvertToExternalBuffer(FRDGBuilder& GraphBuilder, FRDGBufferRef Buffer, TRefCountPtr<FRDGPooledBuffer>& OutPooledBuffer)
```

他们执行底层池化资源的立即分配并返回该资源。

在无法等到在图末尾提取资源的情况下，此方法很有用。转换和提取之间的最大区别是生命周期范围。转换将生命周期延伸到图的开头，而提取将其延伸到图的末尾，这意味着资源将无法与框架中的任何其他资源共享底层分配。