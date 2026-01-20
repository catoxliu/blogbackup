---
title: "Flickering issue with Indexed Indirect Draw"
seoTitle: "Resolving Indexed Indirect Draw Flickering"
seoDescription: "Resolve flickering in Indexed Indirect Draw by identifying and fixing buffer alignment issues between CPU and GPU using manual padding"
datePublished: Tue Jan 20 2026 02:33:33 GMT+0000 (Coordinated Universal Time)
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

On the CPU (C++), this might look like a continuous block of 80 bytes. But on the GPU (NVIDIA specifically), a `float4` isn't just four floats—it is a **16-byte vector entity**.

If the members preceding it don't add up to a perfect multiple of 16, the HLSL compiler "pushes" that `float4` to the next 16-byte boundary to keep the hardware happy. This creates **hidden padding** that your C++ code doesn't know about.

**The result?** The CPU uploads 80 bytes, but the GPU expects 96. Every element in your buffer after the first one is "shifted" by 16 bytes. To the GPU, your data looks like static or garbage, causing the infamous **flicker**.

**The Fix: Manual Padding is Your Friend**

We discovered that Unreal Engine doesn't "assert" or warn you about this. Why? Because the C++ compiler and the Shader compiler are two different worlds. They don't talk to each other.

To fix it, we had to take control of the layout. By adding a simple `float Padding;` after our `int Payload;`, we manually aligned the `float4` to the 16-byte boundary. Suddenly, the CPU and GPU were speaking the same language.

**The ByteAddressBuffer Escape Hatch**

If you’re tired of fighting with alignment, there is the **ByteAddressBuffer**. It treats memory as raw bytes, skipping the "smart" auto-padding of Structured Buffers. It's more manual, but it's the ultimate way to ensure what you send is exactly what you get.

In long run, I may optimize the buffer structure without using either float4 and padding.