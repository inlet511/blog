---
title: "Using View"
date: 2022-10-19T21:32:22+08:00
draft: false
toc: true
categories: [UE5]
tags: 
  - UE5
  - Rendering
---

*UE5版本:5.03*
---

经常可以在引擎的Shader中看到使用View这个UniformBuffer的情形，例如：
```c
void VSMain(
	in float2 InPosition : ATTRIBUTE0,
	out float2 OutTexCoord : TEXCOORD0,
	out float4 OutScreenVector : TEXCOORD1,
	out float4 OutPosition : SV_POSITION
	)
{	
	// screenspace position from vb
	OutPosition = float4(InPosition,0,1);
	// texture coord from vb
	OutTexCoord = InPosition * View.ScreenPositionScaleBias.xy + View.ScreenPositionScaleBias.wz;

	// deproject to world space
	OutScreenVector = mul(float4(InPosition,1,0), View.ScreenToTranslatedWorld);
}
```
另外还有些函数也简介使用了View，例如ConvertFromDeviceZ。

但是View的使用不是仅仅 #include “Common.ush” 就可以的。所有此类Shader，都需要在Shader参数中加入这样一段：
![View](./01.png)

然后在给Shader参数赋值的时候加入：
```cpp
PassParams->ViewUniformBuffer = View.ViewUniformBuffer;
```
这样才可以正确编译和使用View这个Buffer
