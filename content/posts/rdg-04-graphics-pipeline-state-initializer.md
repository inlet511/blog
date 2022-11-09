---
title: "RDG 04 Graphics Pipeline State Initializer"
date: 2022-11-03T22:14:37+08:00
draft: false
toc: true
categories: [UE5]
series:
  - RDG
tags:
  - 图形学
  - UE
  - RDG
  - GraphicsPipeline
  - 混合状态
  - 混合运算
  - 混合因子
  - 栅格化状态
  - 深度模板状态
---
## 1 概述和范例
FGraphicsPipelineStateInitializer 是一个代表渲染管线状态的对象，一个使用范例：
```cpp
FGraphicsPipelineStateInitializer GraphicsPSOInit; // 声明
RHICmdList.ApplyCachedRenderTargets(GraphicsPSOInit);	// 应用RenderTargets

// 设置各种状态
GraphicsPSOInit.DepthStencilState = TStaticDepthStencilState<false, CF_Always>::GetRHI();
GraphicsPSOInit.BlendState = TStaticBlendState<CW_RGB, BO_Add, BF_One, BF_One>::GetRHI();
GraphicsPSOInit.RasterizerState = TStaticRasterizerState<FM_Solid, CM_None>::GetRHI();
GraphicsPSOInit.PrimitiveType = PT_TriangleList;
GraphicsPSOInit.BoundShaderState.VertexDeclarationRHI = MyVertexDeclaration.VertexDeclarationRHI;
GraphicsPSOInit.BoundShaderState.VertexShaderRHI = vs.GetVertexShader();
GraphicsPSOInit.BoundShaderState.PixelShaderRHI = ps.GetPixelShader();

// 应用
SetGraphicsPipelineState(RHICmdList, GraphicsPSOInit,0);
```

## 2 成员
FGraphicsPipelineStateInitializer 包括以下成员(并非全部都要设置)：
```cpp
FBoundShaderStateInput			BoundShaderState;       // 包含三个子成员，顶点Layout、顶点着色器、像素着色器
FRHIBlendState*					BlendState;		        // 混合状态
FRHIRasterizerState*			RasterizerState;        // 光栅化状态
FRHIDepthStencilState*			DepthStencilState;      // 深度、Stencil状态
FImmutableSamplerState			ImmutableSamplerState;  

EPrimitiveType					PrimitiveType;          // 图元类型
uint32							RenderTargetsEnabled;   // 是否启用渲染目标
TRenderTargetFormats			RenderTargetFormats;    // 渲染目标格式
TRenderTargetFlags				RenderTargetFlags;
EPixelFormat					DepthStencilTargetFormat;
ETextureCreateFlags				DepthStencilTargetFlag;
ERenderTargetLoadAction			DepthTargetLoadAction;
ERenderTargetStoreAction		DepthTargetStoreAction;
ERenderTargetLoadAction			StencilTargetLoadAction;
ERenderTargetStoreAction		StencilTargetStoreAction;
FExclusiveDepthStencil			DepthStencilAccess;
uint16							NumSamples;
ESubpassHint					SubpassHint;
uint8							SubpassIndex;
EConservativeRasterization		ConservativeRasterization;
bool							bDepthBounds;
uint8							MultiViewCount;
bool							bHasFragmentDensityAttachment;
EVRSShadingRate					ShadingRate;

// BoundShaderState 包含以下成员：
FRHIVertexDeclaration* VertexDeclarationRHI = nullptr;
FRHIVertexShader* VertexShaderRHI = nullptr;
FRHIPixelShader* PixelShaderRHI = nullptr;
```

### 2.1 BoundShaderState
主要是用来设置顶点布局、顶点着色器、像素着色器, 例：
```cpp
GraphicsPSOInit.BoundShaderState.VertexDeclarationRHI = MyVertexDeclaration.VertexDeclarationRHI;
GraphicsPSOInit.BoundShaderState.VertexShaderRHI = vs.GetVertexShader();
GraphicsPSOInit.BoundShaderState.PixelShaderRHI = ps.GetPixelShader();
```


### 2.2 BlendState
混合状态，看例子:
```cpp
GraphicsPSOInit.BlendState = TStaticBlendState<CW_RGB, BO_Add, BF_One, BF_One>::GetRHI();
```
看定义：
```cpp
class TStaticBlendState : public TStaticStateRHI<
	TStaticBlendState<
		RT0ColorWriteMask,RT0ColorBlendOp,RT0ColorSrcBlend,RT0ColorDestBlend,RT0AlphaBlendOp,RT0AlphaSrcBlend,RT0AlphaDestBlend,
		RT1ColorWriteMask,RT1ColorBlendOp,RT1ColorSrcBlend,RT1ColorDestBlend,RT1AlphaBlendOp,RT1AlphaSrcBlend,RT1AlphaDestBlend,
		RT2ColorWriteMask,RT2ColorBlendOp,RT2ColorSrcBlend,RT2ColorDestBlend,RT2AlphaBlendOp,RT2AlphaSrcBlend,RT2AlphaDestBlend,
		RT3ColorWriteMask,RT3ColorBlendOp,RT3ColorSrcBlend,RT3ColorDestBlend,RT3AlphaBlendOp,RT3AlphaSrcBlend,RT3AlphaDestBlend,
		RT4ColorWriteMask,RT4ColorBlendOp,RT4ColorSrcBlend,RT4ColorDestBlend,RT4AlphaBlendOp,RT4AlphaSrcBlend,RT4AlphaDestBlend,
		RT5ColorWriteMask,RT5ColorBlendOp,RT5ColorSrcBlend,RT5ColorDestBlend,RT5AlphaBlendOp,RT5AlphaSrcBlend,RT5AlphaDestBlend,
		RT6ColorWriteMask,RT6ColorBlendOp,RT6ColorSrcBlend,RT6ColorDestBlend,RT6AlphaBlendOp,RT6AlphaSrcBlend,RT6AlphaDestBlend,
		RT7ColorWriteMask,RT7ColorBlendOp,RT7ColorSrcBlend,RT7ColorDestBlend,RT7AlphaBlendOp,RT7AlphaSrcBlend,RT7AlphaDestBlend,
		bUseAlphaToCoverage
		>,
	FBlendStateRHIRef,
	FRHIBlendState*
	>
```
它定义了8个渲染对象，一般我们只用第一组，它的七个参数分别是：
- 颜色写入蒙版
- 颜色混合运算
- 颜色源混合因子
- 颜色目标混合因子
- Alpha混合运算
- Alpha源混合因子
- Alpha目标混合因子

颜色写入蒙版：
```cpp
enum EColorWriteMask
{
	CW_RED   = 0x01,
	CW_GREEN = 0x02,
	CW_BLUE  = 0x04,
	CW_ALPHA = 0x08,

	CW_NONE  = 0,
	CW_RGB   = CW_RED | CW_GREEN | CW_BLUE,
	CW_RGBA  = CW_RED | CW_GREEN | CW_BLUE | CW_ALPHA,
	CW_RG    = CW_RED | CW_GREEN,
	CW_BA    = CW_BLUE | CW_ALPHA,

	EColorWriteMask_NumBits = 4,
};
```
要继续讲解其他参数，需要先列出DirectX的混合方程作为参照：
$$
C = C_{src} \otimes F_{src} \oplus C_{dst} \otimes F_{dst}
$$
$$
A = A_{src} \otimes F_{src} \oplus A_{dst} \otimes F_{dst}
$$
第一个是颜色的混合方程，第二个是Alpha的混合方程。

#### 混合运算

混合运算符对于颜色混合方程和Alpha混合方程效果是一样的，这里就只用颜色混合方程来做讲解。
|BlendOperation|颜色混合方程|
|----|----|
|BO_Add|$$ C_{src} \otimes F_{src} + C_{dst} \otimes F_{dst}$$|
|BO_Subtract|$$C = C_{src} \otimes F_{src} - C_{dst} \otimes F_{dst}$$|
|BO_ReverseSubtract|$$C = C_{dst} \otimes F_{dst} - C_{src} \otimes F_{src}$$|
|BO_Min|$$C = Min(C_{src} , C_{dst} )$$|
|BO_Max|$$C = Max(C_{src} , C_{dst} )$$|
BO_Min和BO_Max忽略了混合因子

#### 混合因子
|BlendFactor|颜色混合因子|Alpha混合因子|
|----|----|----|
|BF_Zero|$$F = (0,0,0)$$|$$F=0$$|
|BF_One|$$F=(1,1,1)$$|$$F=1$$|
|BF_SourceColor|$$F=(r_{src},g_{src},b_{src})$$|--|
|BF_InverseSourceColor|$$F=(1-r_{src},1-g_{src},1-b_{src})$$|--|
|BF_SourceAlpha|$$F=(a_{src},a_{src},a_{src})$$|$$F=a_{src}$$|
|BF_InverseSourceAlpha|$$F=(1-a_{src},1-a_{src},1-a_{src})$$|$$F=1-a_{src}$$|
|BF_DestAlpha|$$F=(a_{dst},a_{dst},a_{dst})$$|$$F=a_{dst}$$|
|BF_InverseDestAlpha|$$F=(1-a_{dst},1-a_{dst},1-a_{dst})$$|$$F=1-a_{dst}$$|
|BF_DestColor|$$F=(r_{dst},g_{dst},b_{dst})$$|--|
|BF_InverseDestColor|$$F=(1-r_{dst},1-g_{dst},1-b_{dst})$$|--|
|BF_ConstantBlendFactor|$$F=(r,g,b)$$|$$F=a$$|
|BF_InverseConstantBlendFactor|$$F=(1-r,1-g,1-b)$$|$$F=1-a$$|
|BF_Source1Color|未知|未知|
|BF_InverseSource1Color|未知|未知|
|BF_Source1Alpha|未知|未知|
|BF_InverseSource1Alpha|未知|未知|

最后四个选项没有在DirectX中找到对应的选项，没有继续探究，前面的应该足够一般使用了。

### 2.3 RasterizerState
例子：
```cpp
GraphicsPSOInit.RasterizerState = TStaticRasterizerState<FM_Solid, CM_None>::GetRHI();
```
TStaticRasterizerState的泛型参数有三个
```cpp
TStaticRasterizerState<FillMode,CullMode,bEnableLineAA>
```
FillMode默认值是FM_Solid，另外可选择：	FM_Point, FM_Wireframe,	FM_Solid

CullMode默认值是CM_None， 另外可选择：	CM_CW(Cull Clockwise), CM_CCW(Cull Counter-Clockwise)

bEnableLineAA 默认是false

### 2.4 DepthStencilState
深度&模板状态
例子：
```cpp
GraphicsPSOInit.DepthStencilState = TStaticDepthStencilState<false, CF_Always>::GetRHI();
```
TStaticDepthStencilState的完整泛型参数列表：
```cpp
TStaticDepthStencilState<
		bEnableDepthWrite, // 是否启用深度写入
		DepthTest,          // 深度测试比较函数
		bEnableFrontFaceStencil,    // (正面)启用模板
		FrontFaceStencilTest,       // (正面)模板失败操作
		FrontFaceStencilFailStencilOp, //(正面)模板测试失败时如何更新模板缓冲区
		FrontFaceDepthFailStencilOp,  //(正面)深度测试失败时如何更新模板缓冲区
		FrontFacePassStencilOp,     //(正面)通过模板测试时如何更新模板红冲去
		bEnableBackFaceStencil,     // (背面)启用模板
		BackFaceStencilTest,        // (背面)模板失败操作
		BackFaceStencilFailStencilOp, //(背面)模板测试失败时如何更新模板缓冲区
		BackFaceDepthFailStencilOp, //(背面)深度测试失败时如何更新模板缓冲区
		BackFacePassStencilOp,      //(背面)通过模板测试时如何更新模板红冲去
		StencilReadMask,            // 模板读取Mask
		StencilWriteMask            // 模板写入Mask
		>
```

#### DepthTest
深度测试比较函数。

```cpp
enum ECompareFunction
{
	CF_Less,
	CF_LessEqual,
	CF_Greater,
	CF_GreaterEqual,
	CF_Equal,
	CF_NotEqual,
	CF_Never, // 总是返回false
	CF_Always, // 总是返回true

	ECompareFunction_Num,
	ECompareFunction_NumBits = 3,

	// Utility enumerations
	CF_DepthNearOrEqual		= (((int32)ERHIZBuffer::IsInverted != 0) ? CF_GreaterEqual : CF_LessEqual),
	CF_DepthNear			= (((int32)ERHIZBuffer::IsInverted != 0) ? CF_Greater : CF_Less),
	CF_DepthFartherOrEqual	= (((int32)ERHIZBuffer::IsInverted != 0) ? CF_LessEqual : CF_GreaterEqual),
	CF_DepthFarther			= (((int32)ERHIZBuffer::IsInverted != 0) ? CF_Less : CF_Greater),
};
```