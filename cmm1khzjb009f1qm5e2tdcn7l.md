---
title: "Missing Scene Depth for Custom Vertex Factories in Unreal Engine"
seoTitle: "Unreal Engine Vertex factory issue "
seoDescription: "why "
datePublished: 2026-02-25T05:01:59.208Z
cuid: cmm1khzjb009f1qm5e2tdcn7l
slug: missing-scene-depth-for-custom-vertex-factories-in-unreal-engine
tags: unreal-engine

---

As last time I started using the vertex factory to consume the instance buffer by drawing indexed primitive with indirect buffer, some issues have been found which is interesting (struggling).

Firstly, it's the missing scene depth for the generated mesh, even the material is set to opaque and as simple as possible. Without the depth there, the overlap meshes will have z-fighting all the time which is unacceptable.

I checked and play around with the vertex factory shader, the FPrimitiveViewRelevance and FMeshBatch configurations and nothing works.

Then I tried the material itself, and do find a workaround for it, setting a small number to the "Pixel Depth Offset" seems force it render to scene depth again. Which gives me a clue, so I follow it up in the render pipeline. So without the "Pixel Depth Offset", the vertex factory's indirect draw will disappear in `PrePass DDM_AllOpaque` 's depth pass. it's because the engine default shader `DepthOnlyVertexShader` is used in this case and for our vertex factory shader map, it's missing, and then the `DepthRendering::GetDepthPassShaders` will early return false.

Once find the root cause, the fix is easy, just make sure add the default shader in the vertex factory permutation using `MaterialParameters.bIsSpecialEngineMaterial`

```cpp
bool FVertexFactory::ShouldCompilePermutation(const FVertexFactoryShaderPermutationParameters& Parameters)
{
	return MyVertexFactoryPermutation() || Parameters.MaterialParameters.bIsSpecialEngineMaterial;
}
```

  
Then the next issue is while moving the camera around or turn on some vertex animation for the vertex factory generated mesh, part of the mesh will become black and some yellow text behind it saying:

<mark class="bg-yellow-200 dark:bg-yellow-500/30">"Your scene contains a skydome mesh with a sky material, but it does not cover that part of the screen..."</mark>

not my game, image from internet:

![](https://cdn.hashnode.com/uploads/covers/626dfd3d454011df6e86c25d/b8aab285-76a4-4b30-b5f9-307ca805917f.png align="center")

So I believe it's some corrupted buffer for my vertex factory, and I noticed that without the scene depth buffer fix, it's not showing up. After reading the compiled shader I realised that the default `DepthOnlyVertexShader` still needs call the vertex factory `GetVertexFactoryIntermediates`, which consume the instance buffer. But the instance buffer is written by a compute shader after the PrePass there. So a classical race condition : )

Then moving that compute shader pass before the PrePass get everyhing works as expected with the vertex factory now.