![](https://alextardif.com/images/DXVK.jpg)  
  

### Overview

If you've spent time learning newer APIs like D3D12 and Vulkan, chances are you've come across the term "bindless" with regards to binding resources for shaders. Most commonly, this is describing a way to set up your resources such that you can largely avoid needing to recreate the classic slot-based binding model we've all become accustomed to from other APIs. The term itself is a bit misleading because you're still binding these resources, but leaving that aside, what makes it compelling is more about where and how you do it. Despite the fact that bindless has existed in some limited forms since the GL_NV_bindless_texture extension almost a decade ago, it has seen overall limited adoption since then, even in the cross-hardware compatible form it has today. This is largely due to games needing to support a wide variety of hardware, some of which does not support (or only partially supports) bindless, and trying to support both with and without is painful. However, we're now reaching the point where support for bindless is wide enough that it's becoming the new standard for how we set up our resource bindings. This is also one of those few modern API topics that helps improve accessibility to the modern graphics APIs, and should probably start becoming a part of "Intro to DX12/Vulkan" blogs, so here's one! Below I'll cover some advantages of bindless, implementation details, and a couple of simple but powerful examples. I will be assuming you're working with hardware that can use unbounded arrays of resources (or at least, large arrays of resources). [https://docs.microsoft.com/en-us/windows/win32/direct3d12/hardware-support](https://docs.microsoft.com/en-us/windows/win32/direct3d12/hardware-support)  
  
  

### Descriptor Management

Once you get past the initial hurdle of understanding the descriptor binding interfaces of modern APIs, perhaps the most familiar thing to do when you bind per-draw information is to copy non-shader visible descriptors into a shader-visible heap (D3D12) or update your descriptor sets all the time (Vulkan). With this approach, per-draw descriptor management during render passes is likely to become a hot spot for performance issues for you, once you start scaling up the number of draws. And of course, most of those descriptor bindings are probably for textures. There are a number of ways to avoid this overhead with clever management of your descriptor heaps/sets, but what if you didn't have to worry about binding per-draw textures? Without them, you're probably left with a constant buffer or two, which is much less of a problem to be doing simple binding with. This is where bindless comes in.  
  
  

### Bindless Table Setup

Let's start with your HLSL shader and set up bindless tables for 2D and cube textures.  
  

#define myTex2DSpace space1
#define myTexCubeSpace space2
 
Texture2D Texture2DTable[]  : register(t0, myTex2DSpace);
TextureCube TextureCubeTable[]  : register(t0, myTexCubeSpace);
                        

It's fairly common with bindless tables to name the HLSL spaces here to make it clear what the space is being used for. You can't bind anything in the "t" range after the Texture2DTable, for example, because... well, it's unbounded. It's kind of a natural boundary to use spaces for this, where 1 bindless "space" equates to 1 descriptor table (D3D12) or descriptor set binding (Vulkan). Side note, if you're reading this from the Vulkan side (which doesn't actually use spaces) you can make use of some neat features in DXC to map those bindings the way you want: [https://github.com/microsoft/DirectXShaderCompiler/blob/master/docs/SPIR-V.rst#descriptors](https://github.com/microsoft/DirectXShaderCompiler/blob/master/docs/SPIR-V.rst#descriptors).  
  
That's it for the shader setup. The other end of this is to set this up in the root signature (D3D12).  
  

D3D12_DESCRIPTOR_RANGE texture2DRange{};
texture2DRange.BaseShaderRegister = 0;
texture2DRange.NumDescriptors = MY_TEXTURE_2D_BINDLESS_TABLE_SIZE;
texture2DRange.OffsetInDescriptorsFromTableStart = 0;
texture2DRange.RangeType = D3D12_DESCRIPTOR_RANGE_TYPE_SRV;
texture2DRange.RegisterSpace = myTex2DSpace;
                        

MY_TEXTURE_2D_BINDLESS_TABLE_SIZE can be whatever you want it to be. Even unbounded, I like to give a known array maximum so that it's easy to do some debugging of bindless tables if things go wrong (like force copying an error texture descriptor to the full table range). Whatever that value is, that's how many shader resource views (SRVs) that you can put in that respective table. The cube texture range is the same. Next we add the descriptor table, like we would any other:  
  

D3D12_ROOT_PARAMETER texture2DTable{};
texture2DTable.ParameterType = D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE;
texture2DTable.ShaderVisibility = D3D12_SHADER_VISIBILITY_ALL;
texture2DTable.DescriptorTable.NumDescriptorRanges = 1;
texture2DTable.DescriptorTable.pDescriptorRanges = &texture2DRange;
                        

Then create your root signature(s) like you're already doing, and your shader creation side is good to go. If you're noticing a distinct lack of Vulkan information here (which is a bit more involved), that's because Will Pearce already has you covered: [http://roar11.com/2019/06/vulkan-textures-unbound/](http://roar11.com/2019/06/vulkan-textures-unbound/)  
  
From here, the way you manage your descriptor heap/pool allocations is up to your preferences. In D3D12, I like to reserve a starting range of the D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV shader-visible heap that I bind with SetDescriptorHeaps, specifically for these tables (again where setting eg MY_TEXTURE_2D_BINDLESS_TABLE_SIZE is handy). So it would look something like this:  
  

<----Reserved Heap Range---><---------------Everything Else---------------->
{2D textures}{cube textures}{forward pass descriptors}{post fx descriptors}
                        

But you can do it however makes sense to you. As long as your tables are contiguous, you're fine. In Vulkan you can just allocate chunks of descriptors out of a descriptor pool, and you're good to go.  
  
  

### Bindless Textures

Alright, we've set up our tables, so how do we use it? It's actually really simple from here on out.  
  
I recommend keeping a free-list of available texture table indices. When you create a new texture, you pull an index out of the free-list, create a shader resource view in the descriptor heap at that index (D3D12) or write the descriptor to the set (Vulkan), and free the index back to the list when you destroy the texture. Note here, synchronization considerations for accessing resources in-flight on the GPU are the same for indices in a bindless table: you don't want to destroy the resource or return the index for re-use until you know for sure the GPU isn't still using it. This is somewhat even more important with bindless than with older binding models, because GPU validation used to (generally) catch you when you didn't bind resources, but if you improperly access a resource from a bindless table, your GPU will likely just crash. Not a great feedback system, right? That's a trade-off here! It's something you can totally work around though. What I like to do is (for debugging), bind a known error texture to all texture slots in the bindless table before assigning indices out to anything else. That way, if you sample something unexpected, you'll see your error texture instead of a your monitor going black.  
  
Next you need to make sure you bind these descriptor tables/sets in your render passes like you normally do, wherever you're binding your per-pass resources after binding the pipeline object. Now the tables will be available for you to use in your shaders, and with textures properly bound to your bindless tables, the only thing left to do is to use them! A simple example is, in your per-draw constant buffer, add a field for your texture index like:  
  

cbuffer MyPerDrawConstants
{       
    float4x4 transform;
    float3 albedoMultiplier;
    uint albedoTextureIndex;
}
                        

And then sample it in your shaders like this:  
  

Texture2DTable[albedoTextureIndex].Sample(AlbedoSampler, uv0);
                        

Or, even better, you can easily make it an optional feature by having a known invalid index and check it:  
  

float3 myAlbedo = albedoMultiplier;
if(albedoTextureIndex != INVALID_INDEX)
{
    myAlbedo *= Texture2DTable[albedoTextureIndex].Sample(AlbedoSampler, uv0).rgb;
}
 

Now, suddenly, all your per-draw bindings have been reduced to (probably) a constant buffer or two, rather than that plus X number of textures per draw. This is just the tip of the iceberg of things you can do with bindless, but this alone offers a ton of value by making texture binding simpler and more efficient. You can also do other neat things like use the GPU to do some decision making about textures and write out indices that you then use later for sampling. Beware though, if that index is non-uniform, you must use NonUniformResourceIndex or you'll have problems, and it is not free performance-wise: [https://docs.microsoft.com/en-us/windows/win32/direct3d12/resource-binding-in-hlsl](https://docs.microsoft.com/en-us/windows/win32/direct3d12/resource-binding-in-hlsl).  
  
It also potentially opens up some easier draw data resource management, like the fact that you could replace all usages of a texture with another texture by simply writing over the descriptor at that table index (frame buffering considerations still matter here though). Anyway, the point is you've opened up a whole lot of power for yourself, and improved binding efficiency at the same time. Apart from hardware support limitations, I don't really know of any reason not to do this.  
  
  

### Goodbye, Vertex Input Layouts

Bindless textures are the obvious thing to reach for with bindless tables, but what else can we take advantage of with this new tech we just set up? Like descriptors, another pain you're probably now dealing with is managing shader pipeline permutations, and how any bit of different state can add headaches - like the vertex layout (eg D3D12_INPUT_LAYOUT_DESC) part of the pipeline state. As an alternative to binding vertex buffers and having to set up that information in your state descriptions, consider something like this instead:  
  

ByteAddressBuffer BufferTable[] : register(t0, bufferSpace);
 
struct MyVertexStructure
{
    float3 position;
    float3 normal;
    float2 uv0;
    uint color;
}
 
cbuffer MyPerDrawConstants
{
    uint vertexOffset;
    uint vertexBufferIndex;
}
 
....
 
VertexOutput VertexShader(uint vertexId : SV_VertexID)
{
    MyVertexStructure myVertex = BufferTable[vertexBufferIndex].Load<MyVertexStructure>((vertexOffset + vertexId) * sizeof(MyVertexStructure));
}
                        

There are some downsides to this. First is that you're no longer getting auto-magic translation of types you were probably using before (like the unorm8x4-ification of a uint color to a float4 in the shader). In these cases you just have to take care of that manually, like unpacking your vertex color yourself. The other issue is that some of the draw information might not be available to you, for example the "vertexOffset" part of a draw call will not be accounted for by SV_VertexID, you have to pass it in as a constant. That's generally not a big deal though, since you're already passing in the vertex buffer index anyway.  
  
All this accounted for, this gets you the same result as binding a vertex buffer, and is basically just as fast on modern graphics cards (at least, in my experience it is, on AMD/Nvidia). On top of that, you can get rid of calls to IASetVertexBuffers/vkCmdBindVertexBuffers, and remove all the vertex input layout code you needed for pipelines. You'll still want to bind index buffers to take advantage of the vertex cache though, so it's worth keeping those around. Also note that MyVertexStructure was just a simple example, in reality you probably want to split the positions from the material data for z-pre and compute skinning.  
  
The most obvious additional pain this can relieve is skinning-related permutations, to support dynamic bone counts. Gone is the need to define skinning data rigidly like {uint4 blendIndices, float4 blendWeights}. You can pass the counts, layout information, offsets, etc, in constant buffers, and then read the data in your buffers as you see fit from your bindless buffer table. This also sets you up for mesh shaders in the future, where you'll be reading your vertex data manually anyway, so the transition to that should become a lot easier.  
  
  

### The Future of Bindless

Bindless is definitely the way forward, and it may get even easier to use in the future. Discussions like this show that we may someday get HLSL functions that allow us to index into the bound descriptor heap directly! [https://github.com/microsoft/DirectXShaderCompiler/issues/1067](https://github.com/microsoft/DirectXShaderCompiler/issues/1067)  
  
Pseudo-snippet from the link:  
  

uint heapIndex;
float main(uint i:I): SV_Target 
{
    Buffer buf = GetResourceFromHeap(heapIndex);
    return buf[i];
}
                        

  
  

### Bindless, Bindless, Bindless!

If you're newer to DX12 and Vulkan, hopefully this helps the APIs feel a bit more accessible by making some of the more painful parts easier to work with. If you're already experienced with them but haven't looked at bindless yet, I'm hoping this post motivates you (as long as your hardware supports it) to move over to bindless. The examples I've shown above illustrate some straightforward usages and quality of life improvements that bindless offers, but it's only just the beginning of what you can do with it. For some more advanced and interesting practical examples, I recommend checking out Philip Hammer's "Bindless Deferred Decals in The Surge 2" talk ([slides](https://de.slideshare.net/philiphammer/bindless-deferred-decals-in-the-surge-2/)) ([video](https://youtu.be/e2wPMqWETj8)), and MJP's "Bindless Texturing for Deferred Rendering and Decals" [post](https://mynameismjp.wordpress.com/2016/03/25/bindless-texturing-for-deferred-rendering-and-decals/).