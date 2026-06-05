---
title: "Texture binding bug for Unreal Engine on Mac/IOS (Metal)"
seoTitle: "Fix for Shader TDepthOnlyVS never set texture at BaseIndex 0"
seoDescription: "When Metal's Separate Namespaces Meet Unreal's Flat Array: A Shader Binding Bug"
datePublished: 2026-06-05T04:13:56.485Z
cuid: cmq0etdvm000c1skcawte4p6a
slug: texture-binding-bug-for-unreal-engine-on-mac-ios-metal
tags: ios, macos, rendering, unreal-engine

---

As a recent change in our Vertex Factory, we add a texture binding there along with some other structured buffers. Everything works perfectly fine on Windows but failed startup on MacOS with the error "Shader TDepthOnlyVS<false> with vertex factory MyVertexFactory never set texture at BaseIndex 0."

At the very beginning, I thought it's a regression issue we had before that [a buffer is wrongly interpret as texture on Mac](https://blog.catoxliu.net/custom-render-passes-for-unreal-engine-on-ios#:~:text=Which%20is%20very%20wired,uint%3E%20in%20my%20shader.). But a quick check on the compiled shader shows nothing is wrong, the only texture(0) there is the new texture we added. And the debugger shows nothing wrong here, no nullptr, no silence failure and no skip logics. When I checked all the possibilities, I realised this could be a bug of the Engine itself, I have to dig deeper.

First start from where it should be supposed to bind the texture:

```cpp
ShaderBindings.AddTexture(
	MyTexture2D,
	MyTexture2DSampler,
	TStaticSamplerState<SF_Bilinear>::GetRHI(),
	UserData->MyTexture2D);
```

and follow that step by step it goes here:

```cpp
inline void WriteBindingTexture(FRHITexture* Value, uint32 BaseIndex)
{
	int32 FoundIndex = FindSortedArrayBaseIndex(MakeArrayView(ParameterMapInfo.SRVs), BaseIndex);
	if (FoundIndex >= 0)
	{
        GetSRVStart()[FoundIndex] = Value;
    }
}
```

And the BaseIndex here is 0, and the metal shader is "\[\[texture(0)\]\]", seems all good.

But then I notice that for the structured buffer

```cpp
ShaderBindings.Add(MyBuffer, UserData->MyBuffer);
```

it calls this function

```cpp
inline void WriteBindingSRV(FRHIShaderResourceView* Value, uint32 BaseIndex)
{
	int32 FoundIndex = FindSortedArrayBaseIndex(MakeArrayView(ParameterMapInfo.SRVs), BaseIndex);
	if (FoundIndex >= 0)
	{
        uint32 TypeByteIndex = FoundIndex / 8;
        uint32 TypeBitIndex = FoundIndex % 8;
        GetSRVTypeStart()[TypeByteIndex] |= 1 << TypeBitIndex;
        GetSRVStart()[FoundIndex] = Value;
    }
}
```

and for my very first structured buffer there, the BaseIndex is also 0! I double checked to make sure that they get exactly the same FoundIndex here, so the latter one is basically overwriting the former one. This is absolute wrong, but I still not sure one thing. Why even the AddTexture happened later than Add buffer, it still report no texture binding?

The answer is in the `FMeshDrawShaderBindings::SetShaderBindings`

```cpp
for (uint32 SRVIndex = 0; SRVIndex < NumSRVs; SRVIndex++)
{
	FShaderResourceParameterInfo Parameter = SRVParameters[SRVIndex];

	uint32 TypeByteIndex = SRVIndex / 8;
	uint32 TypeBitIndex = SRVIndex % 8;

	if (SRVType[TypeByteIndex] & (1 << TypeBitIndex))
	{
		FRHIShaderResourceView* SRV = (FRHIShaderResourceView*)SRVBindings[SRVIndex];
		SetSrvParameter(BatchedParameters, Parameter, SRV);
	}
	else
	{
		FRHITexture* Texture = (FRHITexture*)SRVBindings[SRVIndex];
		SetTextureParameter(BatchedParameters, Parameter, Texture);
	}
}
```

It's because the SRVType bit flag, even the AddTexture is called later, because it didn't change the bit flag, in the bindings it still think it's a buffer!

Ok, so now we know the cause is because of the **BaseIndex collision**, but why and how could it only affect my vertex factory, and I have all other shaders work fine. Then I realized that for the vertex factory, we're using the old way to binding the shader parameters, and for all other shaders, use `SHADER_USE_PARAMETER_STRUCT` which calls `FShaderParameterBindings`. Resources are recorded as `{BaseIndex, ByteOffset, BaseType}` in `ResourceParameters[]` \_ keyed by **name**, and then consumed in `RHISetShaderParametersShared`.

Before moving on, I realized that there's a work around for this situation, put the texture into a uniform buffer to avoid the BaseIndex collision. And in fact that's what the Virtual Heightfield Mesh plugin does for their vertex factory and works fine.

While since I'm already here I would like to figure out why there's a BaseIndex collision in the first place and try a fix if possible. And after some learning I know that on D3D, the structured buffer and texture shared the same descriptor table "t#" (shader resource views), but on Metal, \[\[buffer(N)\]\] and \[\[texture(N)\]\] are separate namespaces.

And then in MetalCompileShaderSPIRV, After SPIR-V reflection, resources are sorted into typed lists:

```plaintext
ReflectionBindings.SBufferSRVs     structured buffers  (uses BufferIndices)
ReflectionBindings.TextureSRVs     textures            (uses TextureIndices)
```

But **both** call the same output function:

```cpp
// SBufferSRVs loop (line 1108):
CCHeaderWriter.WriteSRV(UTF8_TO_TCHAR(Binding->name), Index);  // Index = 0, 1, 2...

// TextureSRVs loop (line 1214):
CCHeaderWriter.WriteSRV(UTF8_TO_TCHAR(Binding->name), Index);  // Index = 0 //collides!
```

`WriteSRV` is the **loss point**. It concatenates into a single string `Strings.SRVs` with no type tag:

```cpp
// HlslccHeaderWriter.cpp
void FHlslccHeaderWriter::WriteSRV(const TCHAR* ResourceName, uint32 BindingIndex, uint32 Count)
{
    MetaDataPrintf(Strings.SRVs, TEXT("%s(%u:%u)"), ResourceName, BindingIndex, Count);
}
```

And then in MetalShaderCompiler.cpp, Parsing Without Type Awareness

The annotation is printed as:

> @Samplers: MyBuffer(0:1),...,MyTexture2D(0:1)

The `CCHeader` parser reads this into `FSampler` structs <80><94> which have **no type field**:

```cpp
struct FSampler {
    FString Name;
    int32 Offset;     // slot number from annotation
    int32 Count;       // bind count
    TArray<FString> SamplerStates;  // only for combined image samplers
    // no "is this a buffer or texture?" field
};
```

Then the same handler processes everything identically:

```cpp
// MetalShaderCompiler.cpp:526
for (auto& Sampler : CCHeader.Samplers)
    HandleReflectedShaderResource(Sampler.Name, Sampler.Offset, Sampler.Count, Output);
//  AddParameterAllocation(Name, BindOffset=0, BaseIndex=Offset, ...)
```

By this point, `MyBuffer` (structured buffer) and `MyTexture2D` (texture) are indistinguishable both `{BufferIndex=0, BaseIndex=0}`.

So now we have the full picture, and then the question is how to fix it. In the first look, it seems that a simple fix made in SPIR-V cross-compiler by shifting the annotation index for textures past buffer slots, so every resource gets a unique `BaseIndex` in the ParameterMap. The it broke everything else because the slot in the shader won't changed.

Then I noticed that the `BufferIndex` and \`FSampler::SamplerStates\` are never used here, so we can change the annotation format + parsing. Add `BufferIndex` disambiguation through the annotation (`@SRV_Buffers:` vs `@SRV_Textures:`) and pass distinct `BindOffset` values. But that is not enough, even with different `BufferIndex`, `FindSortedArrayBaseIndex` still returns the same slot. Because:

```cpp
  // Shader.h _ only compares BaseIndex
  bool operator<(const FShaderResourceParameterInfo& Rhs) const {
      return BaseIndex < Rhs.BaseIndex;  // ignores BufferIndex!
}
```

To make BufferIndex work as the disambiguator, we'd also need to change `operator<` and `FindSortedArrayBaseIndex` to compare `BufferIndex`. That touches Shader.h, Shader.cpp, and MeshDrawShaderBindings.h \_ three files. Which seems too a bit too risky for the project.

Finally I realized there could be an easy low-risk fix, just in `WriteBindingTexture` and `WriteBindingSRV`, do a check to make sure the `FoundIndex` isn't bounding to other resource yet:

```cpp
	inline void WriteBindingTexture(FRHITexture* Value, uint32 BaseIndex)
	{
		int32 FoundIndex = FindSortedArrayBaseIndex(MakeArrayView(ParameterMapInfo.SRVs), BaseIndex);
		if (FoundIndex >= 0)
		{
#if PLATFORM_IOS || PLATFORM_MAC
			// Metal flattens buffer and texture SRVs into one sorted array, so multiple entries
			// can share the same BaseIndex. Skip already-written slots to avoid overwriting.
			const FShaderResourceParameterInfo * RESTRICT SRVParams = ParameterMapInfo.SRVs.GetData();
			const uint32 NumSRVs = ParameterMapInfo.SRVs.Num();
			while ((uint32)FoundIndex < NumSRVs && SRVParams[FoundIndex].BaseIndex == BaseIndex && GetSRVStart()[FoundIndex] != nullptr)
			{
				FoundIndex++;
			}
			if ((uint32)FoundIndex < NumSRVs && SRVParams[FoundIndex].BaseIndex == BaseIndex)
			{
				GetSRVStart()[FoundIndex] = Value;
			}
#else
			GetSRVStart()[FoundIndex] = Value;
#endif
		}
```

And because the structured buffer having a type bit flag while texture doesn't, it means the order (which one take the first 0 BaseIndex) doesn't matter!

Finally I get my vertex factory working again on Mac and IOS : )