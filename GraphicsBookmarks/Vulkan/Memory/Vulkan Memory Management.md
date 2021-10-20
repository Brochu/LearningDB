Vulkan offers another key difference to OpenGL with respect to memory allocation. When it comes to managing memory allocations as well as assigning it to individual resources, the OpenGL driver does most of the work for the developer. This allows applications to be developed, tested and deployed very quickly. In Vulkan however, the programmer takes responsibility meaning that many operations that OpenGL orchestrates heuristically can be orchestrated based on an absolute knowledge of the resource lifecycle.

Software developers use custom memory management for various reasons:

-   Making allocations often involves the operating system which is rather costly.
-   It is usually faster to re-use existing allocations rather than to free and reallocate new ones.
-   Objects that live in a continuous chunk of memory can enjoy better cache utilization.
-   Data that is aligned well for the hardware can be processed faster.

In previous blog posts we already hinted at some of the systems that Vulkan provides for memory management, which consider these factors.

-   **Device Memory**: This memory is used for buffers and images and the developer is responsible for their content.
-   **Resource Pools**: Objects such as CommandBuffers and DescriptorSets are allocated from pools, the actual content is indirectly written by the driver.
-   **Custom Host Allocators**: Depending on your control-freak level you may also want to provide your own host allocator that the driver can use for the api objects.

We focus on image and buffer memory in this post as it is the central concept to using the Vulkan API effectively.

### Memory Hierarchy

![memory hierarchy](https://developer.nvidia.com/sites/default/files/akamai/gameworks/blog/Vulkan_Memory_Management/vulkan_memory_heap.png)

-   **Heap**: Depending on the hardware and platform, the device will expose a fixed number of heaps, from which you can allocate certain amount of memory in total. Discrete GPUs with dedicated memory will be different to mobile or integrated solutions that share memory with the CPU. Heaps support different memory types which must be queried from the device.
-   **Memory type**: When creating a resource such as a buffer, Vulkan will provide information about which memory types are compatible with the resource. Depending on additional usage flags, the developer must pick the right type, and based on the type, the appropriate heap.
-   **Memory property flags**: These flags encode caching behavior and whether we can map the memory to the host (CPU), or if the GPU has fast access to the memory.  
    
-   **Memory**: This object represents an allocation from a certain heap with a user-defined size.
-   **Resource (Buffer/Image)**: After querying for the memory requirements and picking a compatible allocation, the memory is associated with the resource at a certain offset. This offset must fulfill the provided alignment requirements. After this we can start using our resource for actual work.
-   **Sub-Resource (Offsets/View)**: It is not required to use a resource only in its full extent, just like in OpenGL we can bind ranges (e.g. varying the starting offset of a vertex-buffer) or make use of views (e.g. individual slice and mipmap of a texture array).

Memory is a precious resource, and it can involve several indirect costs by the operating systems. For example some operating systems have a linear cost over the number of allocations for each submission to a Vulkan Queue. Another scenario is that the operating system also handles the paging state of allocations depending on other proceses, we therefore encourage not using too many allocations and organizing them “wisely”.

### Staging Buffers

When memory cannot be mapped to the host or requires format conversions such as packing texels for image memory, a copy of the data from a temporary buffer into the final resource may be required. This process is referred to as ‘staging’ and is common practice for DirectX users already. In OpenGL, the driver implicity managed the temporary copy and potentially batched copying to the device for you. In Vulkan (and with ARB_buffer_storage in OpenGL as well), the use of persistent mapped buffers can achieve similar results.

On devices with dedicated device memory, it is most likely that all resources that are used for many frames are filled through staging buffers. When updating image data we recommend the use of staging buffers, rather than staging images for our hardware. For a small data buffer, updates via the CommandBuffer provide an alternative approach by inlining the data directly.

### Alignment and Aliasing

![memory bind](https://developer.nvidia.com/sites/default/files/akamai/gameworks/blog/Vulkan_Memory_Management/vulkan_memory_bind.png)

The fact that we can manually bind resources to actual memory addresses, gives rise to the following points:

-   Resources may alias (share) the same region of memory.
-   Alignment requirements for offsets into an allocation must be manually managed.

While OpenGL does allow a bit of aliasing for buffers, by just allocating a big buffer and using offsets, Vulkan allows the same for all resources now. One of the simpler use-cases for this capability is overlapping many different-sized framebuffer images into the same memory allocation when the application knows only one size is active in the frame. Some developers use this type of varying resolution rendering to guarantee constant frame-rates. More complex cases could be re-use memory within the same frame for different purposes and achieve an overall lower upper memory bound. A warning from our driver developers is that this feature can put you quickly in the danger zone, where there will be dragons. Memory contents are not defined when aliasing between resources of different types, and care must be taken to (re-)initialize memory contents every time memory is used with a new object.

A far safer scenario is just re-using memory allocations of pre-allocated chunks. Every time a new resource is required we check if we have available chunks from which we can sub-allocate, or whether we should create a new chunk. Memory can be recycled after a few frames when the GPU is not accessing it anymore.

### Buffer Offset Usage

![memory buffer](https://developer.nvidia.com/sites/default/files/akamai/gameworks/blog/Vulkan_Memory_Management/vulkan_memory_buffer.png)

For Buffer memory we recommend making use of the offset mechanism the API provides. Just like in OpenGL, Vulkan allows to binding a range of a buffer. The benefit is that we avoid CPU memory costs for lots of tiny buffers, as well as cache misses by using just the same buffer object and varying the offset.

This optimization applies to all buffers, but in the previous blog post on [shader resource binding](https://developer.nvidia.com/vulkan-shader-resource-binding) it was mentioned that the offsets are particularly good for uniform buffers.

### Summary

![memory strategy](https://developer.nvidia.com/sites/default/files/akamai/gameworks/blog/Vulkan_Memory_Management/vulkan_memory_strategy.png)

Sub-allocation is considered to be a first class approach when working in Vulkan. There are certainly cases where one doesn’t need to use it (e.g. very large 3D volume textures). However, the fact that Vulkan provides us with this flexible and granular approach to memory management, means that with careful consideration to the requirements of our application, one can craft improved memory management schemes.

When this level of performance control is less important than maintainability, OpenGL will continue to deliver. Vulkan on the other hand will find its place where highly dynamic scenes containing many elements require this type of orchestration to achieve peak performance across platforms.

  

**Authors:** Chris Hebert and Christoph Kubisch (special thanks to Daniel Koch)