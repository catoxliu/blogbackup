---
title: "An issue with indexed indirect draw in Unreal"
seoTitle: "Unreal Indexed Indirect Draw Issue"
seoDescription: "Troubleshoot an Unreal indexed indirect draw shader issue due to VS-To-PS struct mismatch and compiler bugs, and AI's role in debugging"
datePublished: Wed Jan 07 2026 00:38:01 GMT+0000 (Coordinated Universal Time)
cuid: cmk3ahsli000002jvdpxja7om
slug: an-issue-with-indexed-indirect-draw-in-unreal
tags: unreal-engine, shader

---

Long story short, I’m working on a shader for an indexed indirect draw in Unreal. And I had a VS-To-PS struct here as simple as:

```cpp
struct VertexOutput
{
  float4 Position : SV_POSITION;
}
```

And this works just perfect fine.

But things get buggy once I add another variable into the struct, which is still very simple

```cpp
struct VertexOutput
{
  float4 Position : SV_POSITION;
  float4 Test : TEXCOORD0;
}
```

The render result shows that the position data is still right, but “test” data is always default.

I have no idea what’s wrong so goes RenderDoc to check step by step, and everything seems just working until the PS stage as the input from VS is all default.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767743714477/c7c20212-637d-4ed3-af14-eb99d5b30145.png align="center")

But I double check the output of the VS which is definitely fine, and also, the render result position is right.

Drop all the shaders into AI and ask why it happened, and it can hardly understand the real issue, other than giving possible solutions. So I simplified the question as “what possible causes for a PS only get default output from VS?”, and then I can narrow it down to the “VS-To-PS struct mismatch“. While my struct is so simple I cannot tell how it could mismatch, I suddenly realize it could be the unreal shader complier.

Get into the RenderDoc again and check the VS there, and my guess is correct, in the compiled VS shader, the SV\_POSITION is reordered to last position in the struct, but no changes for PS shader. That’s why I get a mismatch here.

A quick search in the Unreal Engine source code shows it’s a bug introduced a while ago which only affect indexed indirect draw and fixed later for the shader compiler, and I’m the unlucky guy who is not using the latest version…

Again this is another disappointing experience using AI while coding with shader in Unreal, another case is months ago while I turned on shader debugging the editor cannot start due to shader validation failed:

```cpp
Get a shader compilation error when I disable shader optimize for current project (Unreal 5.3)
5 Shader compiler errors compiling global shaders for platform PCD3D_SM6:
Shader debug info dumped to: "/Saved/ShaderDebugInfo/PCD3D_SM6/Global/FShadingFurnaceTestPassPS/0"
Engine/Shaders/Private/ShadingFurnaceTest.usf(): Shader FShadingFurnaceTestPassPS, Permutation 0, VF None:
    error: validation errors
Engine/Binaries/Win64/(6395,6): Shader FShadingFurnaceTestPassPS, Permutation 0, VF None:
    zzz.dxil(6395,6): error: Instructions should not read uninitialized value.
/Engine/Shaders/Private/ShadingFurnaceTest.usf(): Shader FShadingFurnaceTestPassPS, Permutation 0, VF None:
    note: at 'br i1 undef, label %1664, label %1699' in block '#89' of function 'MainPS'.
/Engine/Shaders/Private/ShadingFurnaceTest.usf(): Shader FShadingFurnaceTestPassPS, Permutation 0, VF None:
    Validation failed.
/Engine/Shaders/Private/ShadingFurnaceTest.usf(): Shader FShadingFurnaceTestPassPS, Permutation 0, VF None:
    D3DCompileToDxil failed. Error code: Unspecified error (0x80004005).
```

I think this is an easy one, so I give the shader file and error message to AI and it tried several times and cannot fix it… Then I have to let it explain the shader line by line and ask if the variable is properly initialized, still no luck, I highlight that any custom struct has to be initialized explicitly, and then it finally find the fix:

```cpp
-                FDeferredLightData LightData;
+                FDeferredLightData LightData = (FDeferredLightData)0;
```

But this time I’m not that frustrated, I have found a way to “work” with AI now and it does help me to get more efficient : )