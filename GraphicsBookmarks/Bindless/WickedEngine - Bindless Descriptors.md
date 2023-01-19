The Vulkan and DX12 graphics devices now support bindless descriptors in Wicked Engine. Earlier and in DX11, it was only possible to access textures, buffers (resource descriptors) and samplers in the shaders by binding them to specific slots. First, the binding model limitations will be described briefly, then the bindless model will be discussed.

Binding model

Slots are available in limited amounts, so a shader could only access up to 128 shader resource views (read only resources), 8 unordered access views (read/write resources), 16 constant buffers and 16 samplers. These limits were configurable in the DX12 and Vulkan devices before now, but it still meant that shaders had to declare resources on hard coded resource slots, so it was not flexible enough. Just to give an example, this is how you access a texture on a slot by a shader:

1

2

3

`Texture2D<float4> texture : register(t27);`

`// ...`

`float4 color = texture.Sample(sampler_linear_clamp, uv);`

To bind the texture on the CPU side, do this:

1

2

3

`Texture texture;`

`// ...`

`device->BindResource(PS, &texture, 27, cmd);`

The other problem is the CPU cost of binding descriptors, which is the main cause of CPU performance problems in rendering code. In some cases, the CPU code will bind a lot of descriptors, especially when rendering the scene with complex shaders that require many inputs like textures, buffers, etc. Binding descriptors is CPU intensive because the implementation will copy all descriptors used by a shader into a contiguous memory area resembling the layout that the shader expects (which will be mostly different for every shader). In the Vulkan and DX12 implementations it was possible to make some simplified assumptions regarding descriptor binding, so they would perform more optimally compared to DX11. These simplifications included:

-   using shader reflection, determine for each shader exactly how many descriptors will be needed, instead of creating a layout that fits all
-   only use one set of slots for all shader stages combined, so no separate slots for vertex and pixel shader
-   put some constant buffers to “root descriptor” instead of descriptor table

But the number one highest CPU costing function that still always showed up was the CopyDescriptorsSimple (DX12) or vkUpdateDescriptorSets (Vulkan).

Bindless model

The bindless model solves both the flexibility and CPU performance problems. Instead of declaring slots in shaders, just say that we access the entire descriptor heap as an unbounded array and access a specific one by a simple index:

1

2

3

`Texture2D<float4> bindless_textures[] : register(t0, space1);`

`// ...`

`float4 color = bindless_textures[27].Sample(sampler_linear_clamp, uv);`

The great thing is that the index value “27” in this example can come from anywhere, for example a material structure:

1

2

3

4

5

6

7

`struct Material`

`{`

  `int texture_basecolormap_index;`

  `int texture_normalmap_index;`

  `int texture_emissivemap_index;`

  `// ...`

`};`

Which means, instead of binding all textures required by the material on the CPU, we just bind the material constant buffer for example (previously you’d need to bind the material constant buffer and all textures as well). The shader will be able to access all textures referenced by the material trivially. From the CPU side there is only one thing we need to do, to get access to the descriptor index, so the value of “texture_basecolormap_index” for example. For that, you can now use this API on the application side:

1

2

3

`Texture texture;`

`// ...`

`int index = device->GetDescriptorIndex(&texture, SRV);`

The GetDescriptorIndex() API can be used after the resource/sampler was created (for example by the CreateTexture function). If the resource wasn’t created yet, or the device doesn’t support bindless, this API will return the value -1. This is important, because shaders must make sure to not use any descriptor which was not created. For example, it is common that a material doesn’t contain a normal map, so the texture_normalmap_index would be -1. In this case the shader must use branching to avoid reading this texture:

1

2

3

4

`if(material.texture_normalmap_index >=0)`

`{`

  `normal = bindless_textures[material.texture_normalmap_index].Sample(sampler_linear_clamp, uv);`

`}`

To avoid branching in shaders, you can choose to provide a dummy texture (like a 1×1 white texture or something similar) if you detect that one of the texture indices is -1.

If the shader tries to access an invalid resource, undefined behaviour will occur, most likely a GPU hang. To help detecting this, you can start the application with the _debugdevice_ and _gpuvalidation_ command line arguments and the Visual Studio debugger attached. If a problem is detected, the debugging will break and the error messages will be posted to the Visual Studio output window.

Using the binding model alongside the bindless model is supported and makes a lot of sense. You wouldn’t want to access your per frame constant buffer in a bindless way, that is less convenient (but still possible). This is also useful to transition to the bindless model in small steps, replacing your shaders with bindless one step at a time. The only thing to keep in mind, that space0 (or if no space binding is provided) is reserved for descriptor bindings in Wicked Engine, so use explicit space1 and greater for bindless declarations.

You can go even further than always binding a material constant buffer, for example you can have all your materials inside a StructuredBuffer, or just bindless buffers. If you also reference all your vertex buffers in the bindless way, it means you will not have to bind anything separately for any of your scene objects. At some point you will need some way to provide descriptor indices to the shaders, and the simplest facility for that is the push constants. This is the most efficient way to set a small number of constant values for shaders. To use it, declare a push constant block in the shader using the PUSHCONSTANT macro (which provides consistent interface for both Vulkan and DX12):

1

2

3

4

5

6

7

8

`struct MyPushConstants`

`{`

  `int mesh;`

  `int material;`

  `int instances;`

  `uint instance_offset;`

`};`

`PUSHCONSTANT(push, MyPushConstants);`

From the CPU side, set the push constants for the next draw/dispatch call like this:

1

2

3

4

`MyPushConstants push;`

`push.mesh = device->GetDescriptorIndex(&meshBuffer, SRV);`

`// fill the rest of push structure...`

`device->PushConstants(&push, sizeof(push), cmd);`

This API is designed for simple, per draw call small data uploads (up to 128 bytes). Each shader can use one push constant block for simplicity. This doesn’t exhibit the same CPU cost as binding a small constant buffer, and has potentially better GPU performance (less memory indirection cost), so prefer this way of specifying per draw data. However, this is only available in DX12 and Vulkan devices and will have no effect in DX11. Check whether push constants can be used by checking bindless support:

1

2

3

4

`if(device->CheckCapability(GRAPHICSDEVICE_CAPABILITY_BINDLESS_DESCRIPTORS))`

`{`

  `// OK to use push constants and bindless`

`}`

By having this flexibility of specifying Mesh and Material structures to be used on GPU, it becomes possible to bind no per draw call vertex/instance/material buffers and let CPU performance will be significantly improved.

Subresources

Subresources are fully supported in bindless just like in non-bindless. The CreateSubresource() API is used to create a subresource on a resource and returns an integer ID, and the GetDescriptorIndex() API will accept this as an optional parameter if you want to query a specific subresource’s own descriptor index. By default, the whole resource is always viewed. Because the subresources have the same lifetime as their main resource, this doesn’t require any special consideration. Small example of retrieving the descriptor index of MipLevel=5 SRV (shader resource view) for a texture:

1

2

`int mip5 = device->CreateSubresource(&texture, SRV, 0, 1, 5, 1);`

`int descriptor = device->GetDescriptorIndex(&texture, SRV, mip5);`

Constant Buffer problem in Vulkan

A corner case to look out for is the bindless constant buffers. There is no problem with this declaration in DX12:

1

`ConstantBuffer<MyType> bindless_cbuffers : register(b0, space1);`

Which makes it easy to use bindless constant buffers, however in Vulkan you can run into unexpected limits regarding constant buffer descriptor count limit. This limit can be queried from here: _VkPhysicalDeviceVulkan12Properties::maxDescriptorSetUpdateAfterBindUniformBuffers_

For example on a RTX 2060 GPU I got an “unlimited” amount for this, but on GTX 1070 I got a limit of 15, which is not sufficient at all for bindless usage. At the same time I had no problem using bindless constant buffers in DX12 on the same GPU. In this case I had to rewrite the bindless constant buffers as bindless ByteAddressBuffers instead. With the ByteAddressBuffer, you can make use of the `Load<T>(address)` templated function in HLSL6.0+ to get the same logical result as if reading from a constant buffer. An example how bindless constant buffer can be rewritten:

1

2

3

4

5

6

7

`// Bindless constant buffer:`

`ConstantBuffer<ShaderMesh> bindless_meshes : register(b0,space2);`

`ShaderMesh mesh = bindless_meshes[42];`

`// Can be rewritten as:`

`ByteAddressBuffer bindless_buffers[] : register(t0, space2);`

`ShaderMesh mesh = bindless_buffers[42].Load<ShaderMesh>(0);`

The ByteAddressBuffer is also nice because less amount of descriptor tables must be bound by the API implementation layer, while supporting more buffer types. I haven’t noticed a performance difference compared to constant buffers on the GPUs/scenes I tested on.

Raytracing

Bindless makes it easier to support raytracing too, as using local root signatures is no longer required. Instead we can provide the mesh descriptor index via the top level acceleration structure InstanceID (which is basically userdata) and InstanceID() intrinsic function that is available in raytracing hit-shaders. The raytracing tier 1.1 (and Vulkan) also provides the GeometryIndex() intrinsic in the shader, which is useful to query a subset of a mesh. For example a flexible bindless scene description in a raytracing shader can look like this:

1

2

3

4

5

6

7

8

9

`Texture2D<float4> bindless_textures[] : register(t0, space1);`

`ByteAddressBuffer bindless_buffers[] : register(t0, space2);`

`StructuredBuffer<ShaderMeshSubset> bindless_subsets[] : register(t0, space3);`

`// Then inside the hit shader:`

`ShaderMesh mesh = bindless_buffers[InstanceID()].Load<ShaderMesh>(0);`

`ShaderMeshSubset subset = bindless_subsets[mesh.subsetbuffer][GeometryIndex()];`

`ShaderMaterial material = bindless_buffers[subset.material].Load<ShaderMaterial>(0);`

`// continue shading...`

This example works well, but watch out as there are a lot of dependent resource dereferencing like this, so there should be further experimentation with flattening some hierarchy, which could improve shader performance. For example: now to read a MeshSubset, you need to load the Mesh first, but in most cases there is only 1 subset per mesh, so it could be better to duplicate mesh data for each subset and load them as one. Remember that Mesh and Subset data here are just descriptor indices to reference vertex buffers and such, so the data duplication would not be very excessive.

The above example shows that a variety of bindless resources are declared, which is a bit questionable. Consider the StructuredBuffer case, for every type of StructuredBuffer, now we have to declare a bindless resource with the appropriate type. Internally the DX12 and Vulkan API implementations will bind a descriptor table (descriptor set in Vulkan) for each declaration to satisfy this that views the entire descriptor heap (Vulkan: descriptor pool). ~~This means that the same heap will be bound multiple times, for each bindless declaration which just doesn’t make a lot of sense~~ (Update: As Simon Coenen pointed out in the comments, it is valid to create one descriptor table with multiple unbounded ranges, so there won’t be a need for more than 2 SetDescriptorTable calls per pipeline. Thanks!). DX12 addresses this with the shader model 6.6 feature, which lets you just index a global resource descriptor heap or sampler descriptor heap and cast the result to the appropriate descriptor type: [https://devblogs.microsoft.com/directx/in-the-works-hlsl-shader-model-6-6/](https://devblogs.microsoft.com/directx/in-the-works-hlsl-shader-model-6-6/)

This makes sense in DX12, but Vulkan needs to create separate descriptor pools for every distinct resource type (storage buffer, uniform buffer, sampled image, etc…), so I’m not yet sure how it would be possible to make the two APIs compatible like this.

NonUniformResourceIndex

If the descriptor index is not a uniform value, you will need to make it one before you use it to index into a bindless heap. You can use the NonUniformResourceIndex() function in HLSL for this. If the value comes from a constant buffer, push constant, or WaveReadFirstLane() then the value is scalar. Otherwise, the NonUniformResourceIndex() will ensure that the scalarization happens correctly for each divergent lane. It is unfortunate that it can be missed easily, because the HLSL compiler doesn’t complain about it. Apparently it is valid to use a divergent index on some hardware, which seems to be the case for me on the Nvidia. To read more about this, visit this blog: [Direct3D 12 – Watch out for non-uniform resource index! (asawicki.info)](https://asawicki.info/news_1608_direct3d_12_-_watch_out_for_non-uniform_resource_index.html)

Under the hood

Perhaps it’s worth to describe briefly how all this is implemented with Vulkan and DX12 graphics APIs. In DX12, there are two descriptor heaps at any time that are visible by shaders: one for samplers only, the other is for constant buffers, shader resource views and unordered access views (I call the second one the “resource descriptor heap”). It is not recommended to switch shader visible heaps at any time, so at application start they are created to fit the max amount of descriptors: 2048 samplers and 1 million resource descriptors (tier1 limit of DX12 model). These heaps will hold both bindless descriptors and binding slot layouts alike. The way it works is to just split the heap into two parts: the lower half will contain the bindless descriptors, the upper half of the heap is used as a lock-free ring buffer allocator and if a draw call requires new binding layout, it will be dynamically allocated. I really like the DX12 way of just giving you access to the heap and a way to address and copy descriptors within it, leaves a lot of freedom for implementation. For the bindless allocation, a simple free list is used. When shaders use bindless descriptors (as seen from shader reflection), then the sampler or resource heap will be automatically set to the appropriate root parameter just before the draw/dispatch is called – the user will not need to care about this.

In Vulkan, support for bindless relies on the _VkPhysicalDeviceVulkan12Features::descriptorIndexing_ feature. I decided to create one descriptor pool per descriptor type for bindless, so that’s 8 pools currently that can hold a large number of descriptors. These are created once (using _VK_DESCRIPTOR_POOL_CREATE_UPDATE_AFTER_BIND_BIT_ flag) and the full descriptor sets are allocated immediately. A custom free list allocator is responsible to keep track of which descriptors are free (in a similar same way as in DX12). The binding layouts are allocated from a separate descriptor pool that can support multiple different types of descriptors, and doesn’t support freeing descriptors – the whole pool will always be reset when the GPU finished with it. Based on shader reflection, the device layer will detect which descriptor sets need to be bound where. The downside with Vulkan is that register space declaration maps into descriptor set index. So if you declare a bindless table in space12 for example and no other, the device layer will need to bind dummy descriptor sets starting from set=0, ending with set=11. DX12 doesn’t have this problem, you can declare something in space687 and not care about it, since the root signature handles the mapping. But if you plan to use Vulkan, the advice is to find the minimum space number you can use.

In both of these implementations, allocating bindless descriptors will happen the first time you create the resource, or when creating a subresource, and these are immediately accessible for shaders. Their lifetime is bound to the resource itself, so the user mostly doesn’t need to care about it.

Difficulty of moving to bindless

Because it is possible to use bindless and the old school binding model together, it becomes really easy to move to bindless in baby steps – or even larger steps. The fun problem is coming up with new flexible data structures that make sense. The less fun part is supporting DX11 at the same time when bindless is used. This will be slightly intrusive depending on how far you are going. I decided to go full bindless in the most frequently used shaders – the scene rendering “object shaders” – when using DX12 or Vulkan. This means that the CPU part that renders all meshes will have a large if(bindless) branching. If bindless is supported, only a push constant block is being set, otherwise a material constant buffer and all textures are bound, and it seems manageable so far. The “object shaders” are using a compile time #ifdef path for bindless and all the texture accesses and vertex buffer reads are redefined to read from bindless buffers, a simplified example:

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

`#ifdef BINDLESS`

`Texture2D<float4> bindless_textures[] : register(t0, space1);`

`SamplerState bindless_samplers[] : register(t0, space2);`

`ByteAddressBuffer bindless_buffers[] : register(t0, space3);`

`PUSHCONSTANT(push, ObjectPushConstants);`

`inline ShaderMaterial GetMaterial()`

`{`

  `return bindless_buffers[push.material].Load<ShaderMaterial>(0);`

`}`

`#define texture_basecolormap  bindless_textures[GetMaterial().texture_basecolormap_index]`

`#else // Non-bindless path below:`

`cbuffer materialCB : register(b0)`

`{`

  `ShaderMaterial g_xMaterial;`

`};`

`inline ShaderMaterial GetMaterial()`

`{`

  `return g_xMaterial;`

`}`

`Texture2D<float4> texture_basecolormap : register(t0);`

`#endif // BINDLESS`

Thankfully the shaders were already using branching to avoid reading material textures that are not loaded, so guarding against uninitialized descriptors was not a problem that needed to be tackled in bindless. It’s pretty important to have this branching because bindless will not set “null descriptors” (descriptors are declared volatile), and referencing a descriptor that’s uninitialized can result in GPU hangs.

The vertex input struct is using functions to get vertex position and other properties and those functions will be redefined based on bindless. In DX11, input layouts are still used to provide vertex data, but in bindless I fetch everything by SV_VertexID and SV_InstanceID system provided input semantics:

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

`struct VertexInput`

`{`

`#ifdef BINDLESS`

  `// Data coming from bindless fetching:`

  `uint vertexID : SV_VertexID;`

  `uint instanceID : SV_InstanceID;`

  `float4 GetPosition()`

  `{`

    `return float4(bindless_buffers[push.vertexbuffer_pos_wind].Load<float3>(vertexID * 16), 1);`

  `}`

`#else`

  `// Data coming from input layout:`

  `float4 pos : POSITION_NORMAL_WIND;`

  `float4 GetPosition()`

  `{`

    `return float4(pos.xyz, 1);`

  `}`

`#endif // BINDLESS`

`}`

Wicked Engine has ubershader that includes mostly all “object shader” code in one file and relies on compile time defines to enable specific paths. With that it was not a huge effort to move to bindless.

Performance

Expect big savings in CPU performance with bindless. In large scenes this can be easily be worth lots of milliseconds. The GPU time can be worse though as the new data structures will probably use more indirections. I didn’t manage to arrive at a consistent conclusion here unfortunately, but also didn’t discover any big performance problems. It’s always true that the bindless model just runs faster overall. Some render passes saw some slight reduction in GPU performance, but some ran faster. The difference between Vulkan and DX12 is also not trivial. I seemed to have more CPU gains with DX12, while the Vulkan GPU times were usually better.

Debugging

All main graphics debuggers that I use (Renderdoc, Nsight, Pix) now support bindless pretty well. Using bindless in these tools can be quite a bit slower than not using them, so there is room for improvement. Enabling the GPU Based Validation layer is very useful as it will report uninitialized descriptors that are accessed by shaders. However this will make the application very slow in my experience, so it’s not a joy to use it.

Closing thoughts

Looking into the future, more systems will be moved to bindless where it makes sense, especially texture atlases. I think texture atlases are now “deprecated” with bindless. There is just so much effort in updating them and ensuring MIP levels are correct. Not to mention the limitation about the texture formats, since you can’t put different texture formats into the same atlas, and it is really difficult to update atlases with block compressed formats. There is also the GLTF model format which adds multiple new textures to be sampled for each extension, which begins to stress the bind slot limits, especially with some material blending shaders. It will be a lot easier to add new textures to materials now.

There will be a lot of places too where bindless doesn’t provide a benefit, for example global per frame resource bindings. I’m still not comfortable just dropping DX11, as it’s very stable now and supports a lot of older GPUs with acceptable performance. For the newer graphics API, I’d rather focus on working with cutting edge features and not worrying so much about backwards compatibility at the moment.

Thank you for reading!