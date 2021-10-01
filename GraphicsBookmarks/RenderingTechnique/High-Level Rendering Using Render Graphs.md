# High-Level Rendering Using Render Graphs

AUG 28, 2017

I’ve hyped and talked a lot about data-driven rendering architectures before, where the full flow of a rendered frame is defined in some kind of human readable data-format (see: [“Stingray Renderer Walkthrough #7: Data-driven rendering”](https://bitsquid.blogspot.se/2017/03/stingray-renderer-walkthrough-7-data.html) for an overview how this worked in Bitsquid/Stingray).

In my opinion one of the main benefits is the flexibility it gives developers to easily tweak and adjust the rendering pipe to better suit their projects and target platforms, as well as how accessible and encouraging it becomes to experiment with new rendering techniques. Tweak the configuration file, hot-reload it and get instant visual feedback. Rinse and repeat. Having a short iteration cycle in any creative process is golden, and is just as important for programmers as it is for artists.

Over the last five years a lot has happened within real-time rendering that really challenges the level of configurability required from a modern engine; here are a few examples:

-   Introduction of asynchronous compute and new graphic APIs that supports explicit multi-GPU programming. This forces us away from thinking about graphics workloads as something that can be scheduled serially through a single command queue.
    
-   Insane hardware scalability requirements. From phones to high-end PCs with multiple GPUs, from 720p to 4K HDR, from mono- to low-latency stereo-rendering (for XR).
    
-   Increased interest in new high-level pipeline algorithms that drastically change how the final rendered frame is built, e.g: forward and clustered techniques, GPU-based scene traversal and culling systems, etc.
    
-   New types of none-game content. Up until recently it was mainly the games industry that had an interest in high-fidelity real-time rendering. Today there are a lot of industries interested, but often with the perception that it will be trivial to get their (usually highly unoptimized) content rendering efficiently on a broad variety of hardware. In VR… Looking photorealistic… _Automagically!_
    

And while having a data-driven rendering architecture can help tremendously to combat at least some of these challenges it still boils down to a lot of hard work.

I’m currently experimenting with a new high-level rendering API for The Machinery that I call _Render Graphs_ which I hope will simplify authoring of very complicated rendering pipelines by providing high modularity, resource management and some kind of sensible semi-automatic multi-GPU/async compute scheduling. It builds on top of our platform agnostic rendering API that I blogged about in early May: [“A Modern Rendering Architecture”](https://ourmachinery.com/post/a-modern-rendering-architecture/), and the core idea is heavily inspired by Yuriy O’Donnell’s excellent GDC 2017 presentation: [“FrameGraph: Extensible Rendering Architecture In Frostbite”](https://www.ea.com/frostbite/news/framegraph-extensible-rendering-architecture-in-frostbite).

In today’s post I will do a high-level pitch of this system. While I have most of the API in place today I still haven’t reached a point where I feel comfortable that it all makes sense. Later on when the system has matured a bit and been battle tested I will follow up with a more in-depth post on the actual implementation details.

## Render Graphs

I opted to use the name “Render Graph” instead of mimicking Frostbite’s terminology “Frame Graph” as I imagine we will execute multiple of these graphs within a frame. I anticipate most stuff that relates to any kind of graphics workload (e.g., cubemap convolutions, procedural scattering of objects, various paint tools, etc.) will execute as Render Graphs. The name “Frame Graph” feels more like a “global” concept, something that runs once per frame. Most of the core concepts are the very similar though.

The system is built around three main building blocks: _Passes, Modules_ and the _Graph Builder._ I will walk you through them one by one.

### Passes

The user authors _Render Graph Passes,_ a pass implements two functions:

```
struct tm_render_graph_pass_api
{
    void (*setup_pass)(void *inst,
        struct tm_render_graph_builder_o *graph_builder);
    void (*execute_pass)(void *inst, uint64_t sort_key,
        struct tm_renderer_command_buffer_i *commands,
        struct tm_render_graph_o *graph);
};
```

**Setup pass**

In `setup_pass()` the user defines named resources it wants to _create_, _read_ from and _write_ to. For _reads_ the user specifies the resource and which shader stages the resource will be accessed from. For _writes_ the user specifies the resource, if it will be bound as a render target or a UAV and if the current content of the resource matters for this pass (load, discard or clear).

In addition to GPU resources the user can also _create_, _read_ and _write_ to CPU buffers. This allows us to encapsulate pure CPU work (such as view frustum culling or scene traversal) within the same graph. By implicitly guaranteeing that `execute_pass()` won’t touch any CPU data not listed for _reads/writes_ during the `setup_pass()` we can later use the _read/write_ dependency information to automatically derive scheduling for parallel execution of the render passes. More on this later.

**Multi-queue scheduling**

In `setup_pass()` the user also specifies if the pass desires to run on the async-compute queue (if present). This is indicated by setting a boolean to true, by doing so the user guarantees that the pass will only do compute work.

On top of that the user can also specify if the pass feels comfortable getting its GPU workload scheduled to be executed on another GPU. Note that doing so requires copying all resources that the pass _reads_ from and _writes_ to to the executing GPU, i.e there might be a significant overhead in just shuffling data between GPUs.

Theoretically we could expose a hooking mechanism that can flip both the async compute bool as well as the `mGPU` scheduling flag from the outside world and run experiments to determine the optimal scheduling of passes. While that might not be what you want to ship to end customers I think it could be very cool during development.

**Scheduling the pass**

The last thing the user has to do before exiting `setup_pass()` is adding itself to the _Graph Builder_. By keeping this as an explicit step we get support for doing lazy opt-out of a pass if it determines it doesn’t need to run during `setup_pass()`.

When registering the pass to the _Graph Builder_ the user can also choose to expose the pass as a rendering _layer_ to the material system. This allows shaders assigned to various scene primitives (meshes, vfxs, etc.) to be targeted to render into the render targets bound by the pass.

**Execute Pass**

In `execute_pass()` the user creates any graphics commands needed and adds them to the `tm_render_command_buffer_i`. As discussed in [“A Modern Rendering Architecture”](https://ourmachinery.com/post/a-modern-rendering-architecture/) this command buffer holds graphic API agnostic rendering commands paired with a sort key. Final ordering of the graphics API commands is not determined until all `tm_render_command_buffer_i` have been collected and their sort key arrays have been merged and sorted.

This separation completely decouples the execution order of `execute_pass()` from the scheduling of GPU commands. This is a nice and powerful feature of the render graph system as CPU and GPU workloads can be expressed together in a self-contained way but still become scheduled completely differently.

Before entering `execute_pass()` any resource barriers, synchronization primitives, and resource copies across different GPUs have been taken care of by the system. The user does not have to worry about that and can focus on more fun high-level stuff.

### Modules and Sub-Modules

One _Pass_ in itself is usually not enough to express something interesting. Typically you want to chain multiple passes together (like e.g multiple full screen passes). The way to do that is to add multiple passes to a _Render Graph Module._

The order you add passes to a _Module_ acts as the main ordering mechanism for the final GPU command submission order. A pass added prior to another pass will typically get a lower sort key passed to its `execute_pass()` function. There are exception to this though, if the user adds passes that depends on reading data from passes not already added to the _Module_ the passes will get reordered by the _Graph Builder._

_Modules_ can also be embedded into each other. Either by simply appending a _Module_ to another _Module_ in which case the passes in the appended module will behave as if they were added individually to the destination _Module._ Or by inserting a module to be executed at a particular e_xtension point_.

_Extension points_ are added in the same way as passes but are simply named labels where other _Modules_ can be appended. This provides a way for the author of a _Module_ to expose carefully scheduled points where users can choose to inject custom GPU workloads as a form of extension mechanism to the _Module._ The extension in itself is simply another _Module (w_hich in turn can expose more extension points, and so on).

**Sub-Modules**

When rendering regular scene geometry such as meshes there are occasions where you need to _read_ from the same render target you are currently _writing_ to. A typical example of this is for doing refraction-style effects. In those scenarios you need to copy the whole or a portion of the currently bound render target to an another buffer that can be bound as input to the pixel shader. But how do you schedule that in a system like this?

Well, you might get away with just doing this copy once as a “prepass” to the layer rendering the mesh with the refraction shader. But then you won’t be able to see multiple refracting surfaces through each other. To support that we need to ad hoc insert the copy operation right before the mesh is rendered. It is also not uncommon that a simple copy operation won’t do, nowadays people often want to support things like frosted glass which typically requires running some convolution filter after the copy operation. Things gets hairy very quickly.

![Frosted glass (courtesy of Imgur](https://ourmachinery.com/images/frosted-glass.jpg)

#### Frosted glass (courtesy of Imgur

So my current idea how to solve this is to support adding something I call _Sub-Modules._ Basically they are the same regular _Modules_ except that they are unscheduled when the graph is executed. Instead these _Sub-Modules_ can be dynamically scheduled and executed on demand by the material system. They rely on the knowledge that they are executing embedded “within” another layer (i.e a pass exposed to the material system) and therefore has all the resource state information needed for the system to inject the correct resource barriers.

### Graph Builder

The final piece to the puzzle is the _Graph Builder._ Roughly speaking you could say its responsibility is to look at all dependencies between the passes in a _Module_ and from that figure out the optimal execution order for them, both on the CPU and GPU.

Before it can do that we need to register all passes to the _Graph Builder_. This is done by calling `setup_passes()` on the _Module_ passing in the _Graph Builder_ as a parameter. `setup_passes()`loops over all passes in the order they were added to the _Module_ and in turn call the `setup_pass()` function for each pass. This is a serial process and the idea is that no heavy-lifting tasks should be done within `setup_pass()`, it should pretty much only list the pass’s read/write resource dependencies and potentially register a few new resources for creation. As mentioned earlier, if the pass determines it wants to run it has to register itself to the _Graph Builder._ When doing so it can also be flagged to become a _root pass._ This indicates that the pass will output something that is requested from outside the render graph system and hence must never be culled, even if no other pass consumes its output.

With all active passes registered we can call `validate_and_build()` on the _Graph Builder_. This will start by doing some basic sanity checking (resource name collisions and similar) and if everything looks peachy we are ready to build the graphs; one for CPU and one for GPU.

First thing we do is to find all root passes and from there we follow the resource dependencies for each root pass to figure out the execution order. It is important to remember that the GPU execution order will be determined by the values of the sort keys for each rendering command the pass produces in its `execute_pass()` function. Hence we have completely decouple the CPU execution order of the passes from the GPU execution order. We do this by traversing the passes in two stages, first we look at all the read/write dependencies for the GPU and then we repeat the process again to look at the read/write dependencies for the CPU.

We can from this extract a lot of useful information, such as:

-   Passes that can execute in parallel on CPU.
-   Passes that potentially can execute in parallel on GPU.
    -   Automatic scheduling of these passes across one or many GPUs.
    -   Automatic scheduling of passes running async compute.
-   Life scope of transient resources (i.e resources created in `setup_pass()`).
-   Passes that can be culled due to no other pass consuming their output.
-   All needed resources barriers, synchronization primitives and GPU to GPU copies.

I won’t go into how this is implemented as I’m still in the process of testing out a few different approaches trying to reach something that feels clean and simple. I will share more details when I’ve settled on something that I like. In the mean time I encourage you to go check out Hans-Kristian’s recent blog post about [“Render graphs and Vulkan — a deep dive”](http://themaister.net/blog/2017/08/15/render-graphs-and-vulkan-a-deep-dive/). Hans describes a similar system and while our graphics API abstraction level is very different it still covers a lot of highly related and relevant stuff (pass reordering among other things).

**Execute**

The final step is to run `execute()` on the _Graph Builder._ This will run `execute_pass()` on each pass using the job system trying to go as wide as possible. Each pass can in turn spawn more jobs and creating more `tm_render_command_buffer_i` as needed. Typical use case for going wide within a pass is for more heavy-weight tasks such as view frustum culling or traversing renderable objects that survived view frustum culling.

# Wrap up

That’s it. I hope that a system like this will make it easier to build advanced, flexible and scalable rendering pipelines. Using _Modules_ it becomes simple to piece together advanced rendering pipes in a “Lego”-style fashion.

But how is this data-driven you say? Well, while the system described above is a pure C API for setting up a Render Graph there’s nothing stopping us from adding a data-driven layer on top of it. With that said, I’m not sure it is a smart thing to do as we would then loose the ability to describe more complex behaviors (such as e.g. loops) in a clean way. We can still achieve really fast iteration times by building our rendering pipe as a plugin, and since we have a really nice plugin architecture that supports hot-reloading (see Niklas’s post: [“DLL Hot Reloading in Theory and Practice”](https://ourmachinery.com/post/dll-hot-reloading-in-theory-and-practice/)) this gives us the same benefits as a data-driven architecture but much more powerful.

As a proof of concept I’m building a somewhat advanced rendering pipe in parallel with the development of the _Render Graphs_ system. As soon this work has matured a bit I will share more in-depth implementation details.

Stay tuned.