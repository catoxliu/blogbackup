---
title: "Custom Render Passes for Unreal Engine on iOS"
seoTitle: "Unreal Engine iOS: Custom Render Pass Guide"
seoDescription: "Custom render passes in UE for iOS: learn how to transition from Windows to mobile, address debugging, and optimize performance"
datePublished: Mon Feb 16 2026 04:14:10 GMT+0000 (Coordinated Universal Time)
cuid: cmlonttz3000s02l56ldj5q3m
slug: custom-render-passes-for-unreal-engine-on-ios
tags: ios, rendering, unreal-engine

---

Once I get the custom render pass for Unreal working fine on Windows, I started the painful journey to make it working on mobile (iOS to be specific).

To simplify, the custom render pass has two parts, the first one is computing shader to generate the instance data in the pre render base pass and then draw these instances in the base pass. For destop platform, it’s very straight forward to add the two custom passes into `FDeferredShadingSceneRenderer::Render`.

For the mobile platform, the compute pass added into `FMobileSceneRenderer::Render` and draw pass added into `FMobileSceneRenderer::RenderDeferredMultiPass`, which works as expected in Unreal editor preview on both Windows and Mac.

But while testing on iOS, it’s not working. Going into the GPU debug in XCode shows that the compute pass is working fine but the draw pass is never called. And then I learnt that for iOS, it’s using `FMobileSceneRenderer::RenderDeferredSinglePass`.

But adding the draw pass there after the `SceneColorRendering` pass is still not working on device and GPU debug shows that the draw pass is correctly run and the SceneColor/GBuffer result is as expected. The problem is that it’s too late, because SceneColor/GBuffer is already consumed for back buffer in the `SceneColorRendering` pass previously (yeah, it’s called for `RenderDeferredSinglePass` for a reason : ). So the correct way is to make the draw pass a subpass inside the `SceneColorRendering` pass. Or a better solution is to use unreal vertex factory for drawing our instances, more powerful and flexible, and also it will be handled by `FMobileBasePassMeshPassProcessor`, which then will be part of the subpass there.

A quick test with a simple vertex factory shows promising result, so I start to fully implement all our features into the vertex factory and then I start getting a crash error while starting the game.

> \[UE\] Ensure condition failed: Texture \[File:./Runtime/Renderer/Private/MeshPassProcessor.cpp\] \[Line: 828\] Shader TBasePassVSFNoLightMapPolicy with vertex factory FMyCustomVertexFactory never set texture at BaseIndex 0. This can cause GPU hangs, depending on how the shader uses it.

Which is very wired because I don’t have any texture parameters in my shader. But surprisingly, after checking the compiled shader file for METAL, it turns out somehow my `Buffer<uint> ReserveBuffer` is translated to `texture_buffer ReservedBuffer [[texture(0)]]`! Some searching saying this is expected behavior, so for now my fix is to use StructuredBuffer&lt;uint&gt; in my shader.

Now, no crash or errors, but one feature is not working correctly on iOS. And I notice the data structure of this feature is not aligned with 16-bytes.

```cpp
 struct FMyDataStructure 
{
	FVector3f Pos;
	FVector3f UpVector;
	float Height;
	float Radius;
}
```

So change it into

```cpp
struct FMyDataStructure 
{
    FVector4f PosAndHeight;    // .xyz = Pos, .w = Height
    FVector4f UpVectorAndRadius; // .xyz = UpVector, .w = Radius
};
```

And now everything works on iOS : )