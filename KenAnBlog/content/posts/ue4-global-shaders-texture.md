---
title: "Ue4 Global Shaders - 贴图"
date: 2022-04-09T18:00:28+08:00
toc: true
categories: [UE4]
---
*UE4版本:4.26.2*

>前置知识
> - 图形学中的texturing和sampler知识
> - 图形学中的input layout
> - [上一篇]({{< ref "posts/ue4-global-shaders-rendering" >}})

上一篇我们实现了向自定义的Global Shaders传递颜色参数，并绘制到一张RenderTarget上，这一节我们来实现传递一张贴图。

## 一、修改usf shader
```c
#include "/Engine/Public/Platform.ush"

float4 MainColor;
Texture2D MainTexture;
SamplerState MainTextureSampler;

void MainVS(
    in float4 InPosition : ATTRIBUTE0,
    in float2 InUV : ATTRIBUTE1,
    out float2 OutUV : TEXCOORD0,
    out float4 OutPosition : SV_POSITION
)
{
    OutPosition = InPosition;
    OutUV = InUV;
}

void MainPS(
    in float2 UV : TEXCOORD0,
    out float4 OutColor : SV_Target0
)
{
    OutColor = MainColor * MainTexture.Sample(MainTextureSampler, UV.xy);
}
```
我们添加了两个属性，Texture2D和SamplerState，并且在Vertex Shader的参数中加入了输入和输出的UV，直接将UV传递给Pixel Shader。
在Pixel Shader中，对贴图进行采样，并叠加上MainColor作为最后的输出。
注意Vertex Shader输入的顶点现在除了位置以外，还需要UV，因此我们也需要在C++中做相应的修改。

## 二、修改C++Shader
### 1 增加属性
对应usf shader,我们也需要在C++ shader中添加一个Texture属性，一个SamplerState属性，这两个属性的类型是**FShaderResourceParameter**，先使用LAYOUT_FIELD在**FMyShaderBase**中添加这两个属性：
```cpp
private:
	LAYOUT_FIELD(FShaderParameter, MainColorVal);
	LAYOUT_FIELD(FShaderResourceParameter, MainTextureVal);
	LAYOUT_FIELD(FShaderResourceParameter, MainTextureSamplerVal);
```
### 2 绑定属性
之后要绑定到usf shader中对应的属性，在**FMyShaderBase**的构造参数中进行：
```cpp
    MainColorVal.Bind(Initializer.ParameterMap, TEXT("MainColor"));
    MainTextureVal.Bind(Initializer.ParameterMap, TEXT("MainTexture"));
    MainTextureSamplerVal.Bind(Initializer.ParameterMap, TEXT("MainTextureSampler"));
```
### 3 SetParameters {#setparameters}
当然，设置属性的**SetParameters**函数也要将一个贴图传递给shader：
```cpp
void SetParameters(FRHICommandListImmediate& RHICmdList, const FLinearColor& MyColor, FRHITexture2D* InTexture)
{
    SetShaderValue(RHICmdList, RHICmdList.GetBoundPixelShader(), MainColorVal, MyColor);
    SetTextureParameter(
        RHICmdList, 
        RHICmdList.GetBoundPixelShader(), 
        MainTextureVal, 
        MainTextureSamplerVal, 
        TStaticSamplerState<SF_Trilinear, AM_Clamp, AM_Clamp, AM_Clamp>::GetRHI(), 
        InTexture);
}
```
注意这里传入了一个静态的SamplerState

## 三、修改应用层面的C++
### 1. 修改顶点数据
因为要增加一个UV来进行采样，所以顶点数据中除了位置外，还有增加一个额外的UV数据，因此顶点数据的结构要进行修改(前一篇文章中我们只使用了FVector4作为顶点数据类型)，因此先定义一个顶点数据类型的结构体。(这里放在UtilityFunctions.cpp的开头)
```cpp
struct FVertex_Pos_UV
{
	FVector4 Position;
	FVector2D UV;
}
```

顶点增加好以后，有图形学知识的人应该能够意识到，我们需要修改顶点的Input Layout了。
在上一篇中我们是如何做的呢？
```cpp
GraphicsPSOInit.BoundShaderState.VertexDeclarationRHI = GetVertexDeclarationFVector4();
```
我们此前顶点的Input Layout使用的是一个**GetVertexDeclarationFVector4**函数获取的，这是UE给我们提供的一个工具函数，它位于**RenderUtils**类中，查看这个函数我们可以发现它就是一个简单的FVector4的顶点位置layout，不包含任何其他内容，这正好满足我们上一篇的需求。
```cpp
Elements.Add(FVertexElement(0, 0, VET_Float4, 0, sizeof(FVector4)));
```
>RenderUtils还提供了一些类似的函数，方便我们设置其他常见的简单Input Layout。

但是在这里，我们这里需要自己定义匹配现在的顶点数据的的Input Layout：
```cpp
class FVertex_Pos_UV_Declaration : public FRenderResource
{
public:
	FVertexDeclarationRHIRef VertexDeclarationRHI;

	virtual void InitRHI()
	{
		FVertexDeclarationElementList Elements;
		uint32 Stride = sizeof(FVertex_Pos_UV);
		Elements.Add(FVertexElement(0, STRUCT_OFFSET(FVertex_Pos_UV, Position), VET_Float4, 0, Stride));
		Elements.Add(FVertexElement(0, STRUCT_OFFSET(FVertex_Pos_UV, UV), VET_Float2, 1, Stride));
		VertexDeclarationRHI = RHICreateVertexDeclaration(Elements);
	}

	virtual void ReleaseRHI() override
	{
		VertexDeclarationRHI->Release();
	}
};
```
STRUCT_OFFSET是UE提供的一个方便我们获取Offset的宏

### 2. 传递贴图参数

#### 渲染线程参数
接着修改渲染线程函数 DrawToQuad_RenderThread。
首先需要添加一个贴图的参数, FRHITexture2D* MyRHITexture2D
```cpp
static void DrawToQuad_RenderThread(
	FRHICommandListImmediate& RHICmdList,
	FTextureRenderTargetResource* OutputRenderTargetResource,
	FLinearColor MyColor,
	FRHITexture2D* MyRHITexture
	)
```
#### VertexDeclaration
在Graphic pipeline state 中，我们需要把VertexDeclaration设置为前面自己定义的：
```cpp
// 新增的两行
FVertex_Pos_UV_Declaration VertexDesc;
VertexDesc.InitRHI();

FGraphicsPipelineStateInitializer GraphicsPSOInit;
// ...
//忽略一些没变动的部分
// ...
// 这里是重点，使用自定义的VertexDeclaration
GraphicsPSOInit.BoundShaderState.VertexDeclarationRHI = VertexDesc.VertexDeclarationRHI;
// ...
```
#### 给Shader传递贴图
对像素着色器使用修改的**SetParameter**方法，传入贴图
```cpp
PixelShader->SetParameters(RHICmdList, MyColor, MyRHITexture);
```
#### 修改Vertex Buffer
VertexBuffer也要相应的修改，增加UV的数据：
```cpp
// Vertex Buffer Begins --------------------------
FRHIResourceCreateInfo createInfo;
FVertexBufferRHIRef MyVertexBufferRHI = RHICreateVertexBuffer(sizeof(FVertex_Pos_UV) * 4, BUF_Static, createInfo);
void* VoidPtr = RHILockVertexBuffer(MyVertexBufferRHI, 0, sizeof(FVertex_Pos_UV) * 4, RLM_WriteOnly);

FVertex_Pos_UV v[4];
// LT
v[0].Position = FVector4(-1.0f, 1.0f, 0.0f, 1.0f);
v[0].UV = FVector2D(0, 1.0f);
// RT
v[1].Position = FVector4(1.0f, 1.0f, 0.0f, 1.0f);
v[1].UV = FVector2D(1.0f, 1.0f);
// LB
v[2].Position = FVector4(-1.0f, -1.0f, 0.0f, 1.0f);
v[2].UV = FVector2D(0.0f, 0.0f);
// RB
v[3].Position = FVector4(1.0f, -1.0f, 0.0f, 1.0f);
v[3].UV = FVector2D(1.0f, 0.0f);

FMemory::Memcpy(VoidPtr, &v, sizeof(FVertex_Pos_UV) * 4);
RHIUnlockVertexBuffer(MyVertexBufferRHI);
// Vertex Buffer Ends --------------------------
```

#### 游戏线程
我们要能够在蓝图中指定一个贴图，从而传递给shader，因此修改DrawToQuad函数，为其增加一个可以在蓝图中接受贴图的参数(UTexture2D*类型)
```cpp
UFUNCTION(BlueprintCallable, Category = "KenUtility", meta = (WorldContext = "WorldContexObject"))
static void DrawToQuad(class UTextureRenderTarget2D* OutputRenderTarget,FLinearColor MyColor, UTexture2D* MyTexture);
```

渲染线程接受的贴图参数类型为FRHITexture2D*, 但是游戏线程中的贴图类型是UTexture2D*，因此我们需要进行一下转换，这个转换较为复杂，但方法比较固定：
```cpp
FRHITexture2D* MyRHITexture2D = MyTexture->TextureReference.TextureReferenceRHI->GetReferencedTexture()->GetTexture2D();
```
之后，再将这个FRHITexture2D类型的参数通过Lambda传递给渲染线程即可。
