# Using Unreal’s RDG for Compute-shaders

The RDG is a really convenient way to get custom shaders running on the GPU. Unfortunately [documentation is still very young.](https://docs.unrealengine.com/4.26/en-US/ProgrammingAndScripting/Rendering/RenderDependencyGraph/) Because of this, I decided to document some of the things that took me the longest to figure out.

What I’ll cover:

-   Getting large amounts of data onto the GPU
-   How to get a texture result back
-   Executing the shader every frame

What I won’t cover:

-   Making a plugin
-   [Working with modules](https://www.youtube.com/watch?v=piPkLJerpTg&t=2s) (if you’re getting linking errors, this might be helpful)
-   Getting a custom struct/datatype onto the GPU

## [](https://github.com/nfgrep/CustomComputeExample/wiki#how-to-get-large-amounts-of-data-onto-the-gpu)How to get large amounts of data onto the GPU

### [](https://github.com/nfgrep/CustomComputeExample/wiki#shader-parameter-structs)Shader-Parameter Structs

The main vehicle for data headed from CPU to GPU is the shader-parameter struct. The idea is that you jam everything you want on the GPU into this struct, and Unreal does some of the DX12/Vulkan/GL black magic required to make it happen. Declaring and initializing this struct doesn’t look like you might expect. Presumably due to the aforementioned black-magic, the struct is declared with a swath of macros:

```
DECLARE_GLOBAL_SHADER(FCustomComputeShader)
    SHADER_USE_PARAMETER_STRUCT(FCustomComputeShader, FGlobalShader)
   	 BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )

   	 SHADER_PARAMETER(FVector, SinglePoint)

    END_SHADER_PARAMETER_STRUCT()
```

The main takeaway here is the `SHADER_PARAMETER()` macro in the middle. This macro is one of many from [ShaderParameterMacros.h](https://github.com/EpicGames/UnrealEngine/blob/release/Engine/Source/Runtime/RenderCore/Public/ShaderParameterMacros.h). (This file is included in [RenderGraph.h](https://github.com/EpicGames/UnrealEngine/blob/release/Engine/Source/Runtime/RenderCore/Public/RenderGraph.h)) These macros are what we’ll use to declare individual members in the struct. They each take a type and a name. The type is typically from the engine, like `TArray` or `FVector` (with a few exceptions, i’ll explain later). The name is just the name of the member in the struct, and it has to match an identically named variable in your shader code. This struct is of type `FParameters`, and can be initialized like so:

FParameters MyParams;
MyParams.SinglePoint = your_variable;

Now, this struct can only hold so much data on its own. If you were to try and store vertex data from a mesh for example, you would likely get an error complaining that you’ve gone over the max struct size.

### [](https://github.com/nfgrep/CustomComputeExample/wiki#allocating-buffers-with-the-rdg)Allocating Buffers with the RDG

Instead of storing the whole array of vertex positions in the shader-parameter struct, we can allocate a buffer to be filled on the GPU beforehand, and store some references to it in the shader-parameters struct. Our `SHADER_PARAMETER()` macro needs an upgrade:

```
SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<FVector>, Vertices)
```

> The `_SRV` identifies that this buffer will be a “Shader Resource View”. There is also a version with `_UAV` which identifies a “Unordered Access View”. I find the [msdn documentation on SRVs and UAVs](https://docs.microsoft.com/en-us/windows/uwp/graphics-concepts/shader-resource-view--srv-) a little ambiguous, but the understanding I have is this: SRVs are read-only while UAVs are read-write. SRVs are a little more performant; I imagine due to threads on the GPU being able to read from them willy-nilly whereas a UAV might need to be locked.

Notice that the type we’re passing into the above macro looks a little foreign now. StructuredBuffer is not a type that we can use in normal CPU-side C++, It’s a type from [HLSL](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl). So how do we set an HLSL type from CPU-side C++? Well our fancy new macro doesn’t declare a member of type StructuredBuffer, but of type `FRDGBufferSRVRef`, and we can get one of these buffer refs when we go to build our render graph.

### [](https://github.com/nfgrep/CustomComputeExample/wiki#enter-the-rdg)Enter the RDG

In order to build a render graph with the RDG, we need an `FRDGBuilder`, which is initialized with an `FRHICommandListImmediate`:

```
FRDGBuilder GraphBuilder(RHICmdList);
```

The RHICmdList can come from a few different places, I’ll touch on this in the section on [Executing the shader every frame](https://github.com/nfgrep/CustomComputeExample/wiki/).  
Then we can initialize our shader-parameter struct, though given we’re working with the RDG, initialization looks a little different than you might expect:

```
FParameters* PassParams = GraphBuilder.AllocParameters<FCustomComputeShader::FParameters>();
```

Now that we have a shader-parameter struct initialized, we can set some of its members. Remember we’re looking to store an `FRDGBufferSRVRef`. The process to get one of these is to first create a more generic structured buffer, which returns an `FRDGBufferRef`, and then turn it into an SRV. To make a structured buffer we can call [`CreateStructuredBuffer()`](https://github.com/EpicGames/UnrealEngine/blob/99b6e203a15d04fc7bbbf554c421a985c1ccb8f1/Engine/Source/Runtime/RenderCore/Private/RenderGraphUtils.cpp#L739-L768) (from [RenderGraphUtils.h](https://github.com/EpicGames/UnrealEngine/blob/release/Engine/Source/Runtime/RenderCore/Private/RenderGraphUtils.cpp#L739), included in `RenderGraph.h`):

```
FRDGBufferRef VerticesBuffer = CreateStructuredBuffer(
	GraphBuilder,    // Our FRDGBuilder
	TEXT(“Vertices_StructuredBuffer”),    // The name of this buffer (for debug purposes)
	sizeof(FVector),    // The size of a single element
	VerticesFromMesh    // Our array of vertices
);
```

This function takes some information on how big our buffer will be, the name it should assign to it, and the data to fill it with, and returns an `FRDGBufferRef` that we can use to make our SRV:

```
FRDGBufferSRVRef VerticesSRV = GraphBuilder.CreateSRV(VerticesBuffer, PF_R32_UINT);
```

We hand over the `FRDGBufferRef` we got to `GraphBuilder.CreateSRV()` along with an `EPixelFormat`: `PF_R32_UINT`.

> I’m unsure if the pixel format is important here, as we end up accessing this data as `float3` on the GPU.

Now that we have our `FRDGBufferSRVRef`, we can set the `Vertices` member of our shader-parameters struct with it:

```
PassParameters->Vertices = VerticesSRV;
```

## [](https://github.com/nfgrep/CustomComputeExample/wiki#adding-passes-to-the-rdg)Adding passes to the RDG

Along with the data we want to work with, we need to add passes to the render-graph. These passes encapsulate the HLSL shader-code that acts upon our data. We refer to our HLSL code via a class that inherits from `FGlobalShader`. In this class is also where we can declare our shader-parameter struct:

```
class FCustomComputeShader : public FGlobalShader
{
	DECLARE_GLOBAL_SHADER(FCustomComputeShader)

	SHADER_USE_PARAMETER_STRUCT(FCustomComputeShader, FGlobalShader)

	BEGIN_SHADER_PARAMETER_STRUCT(FParameters)
	…
	END_SHADER_PARAMETER_STRUCT()

	static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
	{
		return IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM5);
	}

	Void BuildAndExecuteGraph(FRHICommandListImmediate& RHICmdList, UTextureRenderTarget2D* RenderTarget, TArray<FVector> InVerts);
}
```

We associate this shader class with our HLSL via the [`IMPLEMENT_GLOBAL_SHADER`](https://github.com/EpicGames/UnrealEngine/blob/99b6e203a15d04fc7bbbf554c421a985c1ccb8f1/Engine/Source/Runtime/RenderCore/Public/GlobalShader.h#L383) macro. We can now use this class when adding a pass to our render-graph:

```
TShaderMapRef<FCustomComputeShader> ComputeShader(GetGlobalShaderMap(GMaxRHIFeatureLevel));

FComputeShaderUtils::AddPass(
    GraphBuilder,
    RDG_EVENT_NAME(“Custom_Compute_Pass),
    ComputeShader,
    PassParameters,
    FIntVector(32,32,1)
)
```

We first initialize a `TShaderMapRef` with our now implemented custom shader coming from the global shader map. We then call upon `FComputeShaderUtils::AddPass`, passing in our graph builder, a name for this event on the GPU, our compute-shader reference from the shader map, the now initialize shader-parameter struct and an `FIntVector` specifying the thread group count on the GPU.

## [](https://github.com/nfgrep/CustomComputeExample/wiki#getting-a-texture-result-back)Getting a texture result back

You can write any kind of data you want with a compute shader, it doesn’t have to be colour data. However, if you _do_ want colour data, this section is for you.

This process is a little more involved than getting bulk data into a buffer. First we have to pass a UAV onto the GPU for our compute-shader to write colour data to. Then, we have to copy the output from the compute-shader to the render-target we want it to show up on.

### [](https://github.com/nfgrep/CustomComputeExample/wiki#getting-a-uav-onto-the-gpu-for-the-compute-shader-to-write-to)Getting a UAV onto the GPU for the compute-shader to write to

In order to build a UAV for our compute-shader to write to, we need to know how big to make it. We can get that info from whatever render-target we want to output to. Like the rest of our data, we can pass this render-target in our shader-params struct:

SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D(FVector4), OutputTexture)

Again, notice that the type we’re passing in is a little strange. In this case its an HLSL type (`RWTexture2D`) that contains an Unreal type (`FVector4`). Like with our big buffer-o-vertices we passed earlier, this macro doesn’t declare a member with a type we can set without asking the RDG first. This macro declares a member of type `FRDGTextureUAVRef`. To get one of these the first thing we need is a 2D texture description, which we can get from [`FRDGTextureDesc::Create2D`](https://github.com/EpicGames/UnrealEngine/blob/99b6e203a15d04fc7bbbf554c421a985c1ccb8f1/Engine/Source/Runtime/RenderCore/Public/RenderGraphResources.h#L372-L381):

```
FRDGTextureDesc OutTextureDesc = FRDGTextureDesc::Create2D(
	FIntPoint(RenderTarget->SizeX, RenderTarget->SizeY),    // The size/extent of our texture
	PF_FloatRGBA,   // The format
	FClearValueBinding(),
	TexCreate_UAV,
	1,  // Number of mips
	1   // Some more mip info
);
```

This function takes in the size, format, mip info and some other data, and spits out our texture description in the form of an `FRDGTextureDesc`.

With a description of our texture, we can now ask the RDG to allocate some memory for it and give us a reference:

```
FRDGTextureRef OutTextureRef = GraphBuilder.CreateTexture(OutTextureDesc, TEXT(“Compute_Out”);
```

Here, [`CreateTexture`](https://github.com/EpicGames/UnrealEngine/blob/99b6e203a15d04fc7bbbf554c421a985c1ccb8f1/Engine/Source/Runtime/RenderCore/Private/RenderGraphBuilder.cpp#L434-L470) takes just our texture description and a name for our texture (for debug info purposes).  
Now our texture isn’t any good if we can’t write to it. Let’s initialize a UAV description with our texture reference:

```
FRDGTextureUAVDesc OutTextureUAVDesc(OutTextureRef);
```

Ok, almost there. Now that we have a description of our UAV, we can actually make one and initialize the member of our shader-parameter struct with it:

```
FRDGTextureUAVRef OutTextureUAV = GraphBuilder.CreateUAV(OutTextureUAVDesc);
PassParameters->OutputTexture = OutTextureUAV;
```

Now we can declare a `RWTexture2D<float4>`named `OutputTexture` in our shader code, and write to the UAV we’ve allocated.

### [](https://github.com/nfgrep/CustomComputeExample/wiki#copying-the-uav-contents-to-a-render-target)Copying the UAV contents to a render-target

Ok, so we have a UAV on the GPU and we can write to it with our compute-shader, now we need to get that result back somehow. One way of doing this is to somehow copy that buffer back from the GPU to the CPU and we can wrangle with our data from there. For the purposes of getting some colours on screen however, this is pretty sub-optimal: copying from GPU to CPU can be really slow! Luckily, Unreal has these lovely things called render-targets.

> My understanding of render-targets and the process of texture extraction is still a little murky. The method I used to copy a UAV into a render-target involves first ‘extracting’ the UAV contents to a pooled render-target, and then copying that into my render-target of choice.

We can copy the result of our compute-shader to one of these render-targets and use it however we please. We can even plug it in as a node in the material editor and avoid having to write our own pixel shader for it in HLSL!

The first order of business is to queue our UAV result for extraction to a pooled render-target:

```
TRefCountPtr<IPooledRenderTarget> PooledComputeTarget;

GraphBuilder.QueueTextureExtraction(OutTextureRef, &PooledComputeTarget);
```

[`QueueTextureExtraction()`](https://github.com/EpicGames/UnrealEngine/blob/99b6e203a15d04fc7bbbf554c421a985c1ccb8f1/Engine/Source/Runtime/RenderCore/Public/RenderGraphBuilder.inl#L267-L282) takes the `FRDGTextureRef` we created earlier, and a reference to a `TRefCountPtr<IPooledRenderTarget>`.

If my understanding is correct, this function queues the UAV result to be copied to the pooled render-target after the graph is executed.

At this point, as long as we have out passes setup to execute our compute-shader, we can call `GraphBuilder.Execute()` and just after that we can extract our result:

```
RHICmdList.CopyToResolveTarget(
    PooledComputeTarget.GetReference()->GetRenderTargetItem().TargetableTexture,
    RenderTarget->GetRenderTargetResource()->TextureRHI,
    FResolveParams()
);
```

The [`CopyToResolveTarget`](https://github.com/EpicGames/UnrealEngine/blob/99b6e203a15d04fc7bbbf554c421a985c1ccb8f1/Engine/Source/Runtime/RHI/Public/RHICommandList.h#L3377-L3386) function takes the targetable texture from our pooled render-target (where the result from our compute-shader should be) and the TextureRHI resource from our desired render-target. At this point the result should be in our chosen render-target.

## [](https://github.com/nfgrep/CustomComputeExample/wiki#executing-the-shader-every-frame)Executing the shader every frame

Now that we have our shader ready to produce some results, we’ll want to execute it. This entails running the code we made to build and execute the render-graph on the [render-thread](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ThreadedRendering/). In most cases, you’ll want to execute your compute-shader every frame; there are a few ways to do this. One of which involves hooking into the render-thread. Temarans [UE4ShaderPluginDemo](https://github.com/Temaran/UE4ShaderPluginDemo) does this; though it doesn’t use the RDG. The other approach is to call the [`ENQUEUE_RENDER_COMMAND`](https://github.com/EpicGames/UnrealEngine/blob/99b6e203a15d04fc7bbbf554c421a985c1ccb8f1/Engine/Source/Runtime/RenderCore/Public/RenderingThread.h#L262-L274) macro every frame:

```
ENQUEUE_RENDER_COMMAND(ComputeShader)(
    [
        ComputeShader,
        RenderTargetParam,
        VerticesParam
    ](FRHICommandListImmediate& RHICmdList)
    {
        ComputeShader->BuildAndExecuteGraph(
            RHICmdList,
            RenderTargetParam,
            VerticesParam);
    });
```

The `ENQUEUE_RENDER_COMMAND` macro takes two parameters; a string and a [lambda](https://docs.microsoft.com/en-us/cpp/cpp/lambda-expressions-in-cpp?view=msvc-160). The string is not particularily important (I _think_ its just the name to use when it initializes a struct on the render-thread.)

The lambda is what gets run on the render thread. The engine takes the lambda, throws an `FRDGCommandListImmediate` into its call-parameters, and executes the body on the render thread.

> The other way to get an instance of a command list would be to include `RHICommandList.h` and call `GRHICommandList.GetImmediateCommandList()`, like is done in the [UE4ShaderPluginDemo example](https://github.com/Temaran/UE4ShaderPluginDemo/blob/8e0f0e39f21460a29e909c5b9f6906b2f2f39623/Plugins/TemaranShaderTutorial/Source/ShaderDeclarationDemo/Private/ShaderDeclarationDemoModule.cpp#L108).

This macro can be called every frame, each time passing in updated scene data or whatever else you can manage to stuff in to some buffers.

## [](https://github.com/nfgrep/CustomComputeExample/wiki#additional-resources)Additional resources

Examples:

-   [UE4-OceanProject ComputeShaderDev](https://github.com/UE4-OceanProject/ComputeShaderDev)
-   [Temarans UE4ShaderPluginDemo](https://github.com/Temaran/UE4ShaderPluginDemo)
-   [Unreal RDG crash course slides](https://epicgames.ent.box.com/s/ul1h44ozs0t2850ug0hrohlzm53kxwrz)

Info:

-   [UE4 Shaders Intro by Rico Loggini](https://logins.github.io/graphics/2021/03/31/UE4ShadersIntroduction.html#shader-objects)
-   [Loading data to/from structured buffer](https://answers.unrealengine.com/questions/978809/loading-data-tofrom-structured-buffer-compute-shad.html)
-   [UE4 Rendering by Matt Hoffman](https://medium.com/@lordned/unreal-engine-4-rendering-overview-part-1-c47f2da65346)
-   [In-Depth Rendering Course from Unreal](https://www.unrealengine.com/en-US/onlinelearning-courses/an-in-depth-look-at-real-time-rendering)