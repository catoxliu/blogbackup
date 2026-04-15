---
title: "Flickering issue with Indexed Indirect Draw"
seoTitle: "Resolving Indexed Indirect Draw Flickering"
seoDescription: "Resolve flickering in Indexed Indirect Draw by identifying and fixing buffer alignment issues between CPU and GPU using manual padding"
datePublished: 2026-01-20T02:33:33.538Z
cuid: cmklzcfvm000702l86iulbu8q
slug: flickering-issue-with-indexed-indirect-draw
tags: nvidia, unreal-engine, gpu

---

In [previous post](https://hashnode.com/post/cmk3ahsli000002jvdpxja7om) I have the procedual grass rendering working with the IndexedIndirectDraw, but while testing it, I find there’re flickering issue sometimes. And after playing with it for a while I can create an edge case with specific number of the size of my instance buffer, which making over 1/3 of the grasses flickering constantly.

Now I have a perfect test case, and I start debugging it closely. The first thing I noticed is that in the RenderDoc, and in the mesh viewer of my VS pass shows the output SV\_Position is shown as NaN (Not a Number) for the almost 1/3 of the instance buffer. The interesting part is that if I debug it step by step, everything just works fine as expected and we got a solid position there. Then I learnt that RenderDoc debug shader is a “simulation” on CPU side, and it cannot guarantee the result match what’s on GPU.

Then I thought it could be some sync issue of the instance buffer or the indirect arguments buffer, wasting a day working on it trying different buffer setup, explicit buffer transition and memory barrieers, no luck.

Ruling out the sync issue, I spent another day to verifying my compute pass to make sure all the data is generated correctly without any surprises.

As I almost gave up, I asked AI if my structure of the instance buffer is cursed : ( but surprisingly, it notice the culprit immediately : )

The real trouble began with a struct that looked like this:

```cpp
struct FData {
    float3 Position;
    float Width;
    // ... other floats ...
    int Payload;
    float4 ControlPos0; // <-- The Saboteur
};
```

The structure is 92 bytes in total.

And according to google, it seems that (for NVIDIA specifically), a `float4` isn't just four floats—it is a **16-byte vector entity**.

And this seems to be the root cause that some of the instance buffer will read "invalid data".

**The Fix: Manual Padding is Your Friend**

In fact, in latest [Unreal Engine](https://github.com/EpicGames/UnrealEngine/blob/0b917fe1ab67ca45e1233a866c92e791fc451ef8/Engine/Source/Runtime/D3D12RHI/Private/D3D12Buffer.cpp#L420) (5.7), they add a new variable "RHI\_RAW\_VIEW\_ALIGNMENT" to check if your structured buffer stride is `LeastCommonMultiplier` of it while creating the buffer.

So for my case, to fix it, we had to add a simple `float Padding;` after our `int Payload;`, we manually aligned to the 16-byte boundary. And then the flickering issue goes away.

**The ByteAddressBuffer Escape Hatch**

And some search result shows there's other work around for this, but I didn't test it.

If you’re tired of fighting with alignment, there is the **ByteAddressBuffer**. It treats memory as raw bytes, skipping the "smart" auto-padding of Structured Buffers. It's more manual, but it's the ultimate way to ensure what you send is exactly what you get.