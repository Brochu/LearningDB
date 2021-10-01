# Render graphs and Vulkan — a deep dive

Modern graphics APIs such as Vulkan and D3D12 bring new challenges to engine developers. While the CPU overhead has dramatically been reduced by these APIs, it’s clear that it is difficult to bridge the gap in terms on GPU performance when we are hitting the “good” paths of the driver, and we are GPU bound. OpenGL and D3D11 drivers (clearly) go to extreme lengths in order to improve GPU performance using all sorts of trickery. The cost we pay for this as developers is unpredictable performance and higher CPU overhead. Writing graphics backends has become more interesting again, as we are still figuring out how to build great rendering backends for these APIs which balance flexibility, performance and ease of use.

Last week I released my side-project, [Granite](https://github.com/Themaister/Granite), which is my take on a Vulkan rendering engine. While there are plenty of such projects out in the wild, all with their own merits, I would like to discuss my render graph implementation in particular.

The render graph implementation is inspired by Yuriy O’Donnells GDC 2017 presentation: “[FrameGraph: Extensible Rendering Architecture in Frostbite](https://www.ea.com/frostbite/news/framegraph-extensible-rendering-architecture-in-frostbite).” While this talk focuses on D3D12, I’ve implemented my own for Vulkan.

_(Note: render graphs and frame graphs mean the same thing here. Also, if I mention Vulkan, it probably also applies to D3D12 as well … maybe)_

## The problem

Render graphs fundamentally solve a very annoying problem in modern APIs. How do we deal with manual synchronization? Let’s go over the obvious alternatives.

### Just-in-time synchronization

The most straight forward approach is basically doing synchronization at the last minute. Whenever we start rendering to a texture, bind a resource or similar, we need to ask ourselves, “does this resource have pending work which needs to be synchronized?” If so, we need to somehow at the very last minute deal with it. This kind of tracking clearly becomes very painful because we might read a resource 1000+ times, while we only write to it once. Multithreading becomes very painful, what if two threads discover a barrier is needed? One thread needs to “win”, and now we have a lot of useless cross-thread synchronization hassles to deal with.

It’s also not just execution itself we need to track, we also have the problem of image layouts and memory access in Vulkan. Using a resource in a particular way will require a specific image layout (or just GENERAL, but you might lose framebuffer compression!).

Essentially, if what we want is just-in-time automatic sync, we basically want OpenGL/D3D11 again. Drivers have already been optimized to death for this, so why do we want to reimplement it in a half-assed way?

### Fully explicit synchronization

On the other side of the spectrum, the API abstraction we choose completely removes automatic synchronization, and the application needs to deal with every synchronization point manually. If you make a mistake, prepare for some “interesting” debugging sessions.

For simpler applications, this is fine, but once you start going down this route you quickly realize what a mess it turns into. Typically your rendering pipeline will be compartmentalized into blocks — maybe you have the forward/deferred/whatever-is-cool-now renderer in one module, some post-processing passes scattered around in other modules, maybe you drag in some feedbacks for reprojection steps, you add a new technique here and there and you realize you have to redo your synchronization strategy — again, and things turn sour.

Why does this happen?

Let’s write some pseudo-code for a dead-simple post-processing pass and think about it.

// When was the last time I read from this image? Probably last frame later in the post-chain ... // We want to avoid write-after-read hazards. // We're going to write the whole image, // so we might as well transition from UNDEFINED to "discard" the previous content ... // Ideally I would keep careful track of VkEvents from earlier frames, but that got so messy ... // Where was this render target allocated from? BeginRenderPass(RT = BloomThresholdBuffer) // This image was probably written to in the previous pass, but who knows anymore. BindTexture(HDR) DrawMyQuad() EndRenderPass()

These kinds of problems are typically solved with a big fat pipeline barrier. Pipeline barriers let you reason **locally** about **global** synchronization issues, but they’re not always the optimal way to do it.

// To be safe, wait for all fragment execution to complete, this takes care of write-after-read and syncing the HDR render pass ... // Assuming they are never used in async compute ... hm, this will probably work fine for now. PipelineBarrier(FRAGMENT -> FRAGMENT, RT layout: UNDEFINED -> COLOR_ATTACHMENT_OPTIMAL, RT srcAccess: 0 (write-after-read) RT dstAccess: COLOR_ATTACHMENT_WRITE_BIT, HDR layout: COLOR_ATTACHMENT_OPTIMAL -> SHADER_READ_ONLY, HDR srcAccess: COLOR_ATTACHMENT_WRITE_BIT, HDR dstAccess: SHADER_READ_BIT) BeginRenderPass(...)

So we transitioned the HDR image, because we assumed it was the previous pass, but maybe in the future you add a different pass in between which also transitions … So now you still need to keep track of image layouts, bleh, but not the end of the world.

If you’re only dealing with FRAGMENT -> FRAGMENT workloads, this is probably not so bad, there isn’t all that much overlap which happens between render passes anyways. When you start throwing compute into the mix is when you start pulling your hair out, because you just **can’t** slap pipeline barriers like this all over the place, you need some non-local knowledge about your frame in order to achieve optimal execution overlap. Plus, you might even need semaphores because you’re doing async compute now in a different queue.

## Render graph implementation

_I’ll be mostly referring to these files: [render_graph.hpp](https://github.com/Themaister/Granite/blob/master/renderer/render_graph.hpp) and [render_graph.cpp.](https://github.com/Themaister/Granite/blob/master/renderer/render_graph.cpp)_

_Note: This is a huge brain dump. Try to follow along in the code while reading this, I’ll go through things in order._

_Note #2: I use the terms “flush” and “invalidate” in the implementation. This is not Vulkan spec lingo. Vulkan uses the terms “make available” and “make visible” respectively. Flush refers to cache flushing, invalidate refers to cache invalidation._

The basic idea is that we have a “global” render graph. All components in the system which need to render stuff need to register with this render graph. We specify which passes we have, which resources go in, which resources are written and so on. This could be done once on application startup, once every frame, or however often you need. The main idea is that we form global knowledge of the entire frame and we can optimize accordingly at a higher level. Modules can reason locally about their inputs and outputs while allowing us to see the bigger picture, which solves a major issue we face when the backend API does not schedule automatically and deal with dependencies for us. The render graph can take care of barriers, layout transitions, semaphores, scheduling, etc.

Outputs from a render pass need some dimensions, fairly straight forward.

Images:

struct AttachmentInfo { SizeClass size_class = SizeClass::SwapchainRelative; float size_x = 1.0f; float size_y = 1.0f; VkFormat format = VK_FORMAT_UNDEFINED; std::string size_relative_name; unsigned samples = 1; unsigned levels = 1; unsigned layers = 1; bool persistent = true; };

Buffers:

struct BufferInfo { VkDeviceSize size = 0; VkBufferUsageFlags usage = 0; bool persistent = true; };

These resources are then added to render passes.

// A deferred renderer setup AttachmentInfo emissive, albedo, normal, pbr, depth; // Default is swapchain sized. emissive.format = VK_FORMAT_B10G11R11_UFLOAT_PACK32; albedo.format = VK_FORMAT_R8G8B8A8_SRGB; normal.format = VK_FORMAT_A2B10G10R10_UNORM_PACK32; pbr.format = VK_FORMAT_R8G8_UNORM; depth.format = device.get_default_depth_stencil_format(); auto &gbuffer = graph.add_pass("gbuffer", VK_PIPELINE_STAGE_ALL_GRAPHICS_BIT); gbuffer.add_color_output("emissive", emissive); gbuffer.add_color_output("albedo", albedo); gbuffer.add_color_output("normal", normal); gbuffer.add_color_output("pbr", pbr); gbuffer.set_depth_stencil_output("depth", depth); auto &lighting = graph.add_pass("lighting", VK_PIPELINE_STAGE_ALL_GRAPHICS_BIT); lighting.add_color_output("HDR", emissive, "emissive"); lighting.add_attachment_input("albedo"); lighting.add_attachment_input("normal"); lighting.add_attachment_input("pbr")); lighting.add_attachment_input("depth"); lighting.set_depth_stencil_input("depth"); lighting.add_texture_input("shadow-main"); // Some external dependencies lighting.add_texture_input("shadow-near");

Here we see three ways which a resource can be used in a render pass.

-   Write-only, the resource is fully written to. For render targets, loadOp = CLEAR or DONT_CARE.
-   Read-write, preserves some input, and writes on top, for render targets, loadOp = LOAD.
-   Read-only, duh.

The story is similar for compute, here’s an adaptive luminance update pass, done in async compute

auto &adapt_pass = graph.add_pass("adapt-luminance", VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT); adapt_pass.add_storage_output("average-luminance-updated", buffer_info, "average-luminance"); adapt_pass.add_texture_input("bloom-downsample-3");

The luminance buffer gets a RMW here for example.

We also need some callbacks which can be called every frame to actually do some work, for gbuffer …

gbuffer.set_build_render_pass([this, type](Vulkan::CommandBuffer &cmd) { render_main_pass(cmd, cam.get_projection(), cam.get_view()); }); gbuffer.set_get_clear_depth_stencil([](VkClearDepthStencilValue *value) -> bool { if (value) { value->depth = 1.0f; value->stencil = 0; } return true; // CLEAR or DONT_CARE? }); gbuffer.set_get_clear_color([](unsigned render_target_index, VkClearColorValue *value) -> bool { if (value) { value->float32[0] = 0.0f; value->float32[1] = 0.0f; value->float32[2] = 0.0f; value->float32[3] = 0.0f; } return true; // CLEAR or DONT_CARE? });

The render graph is responsible for allocating the resources and driving these callbacks, and finally submitting this to the GPU in the proper order. To terminate this graph, we promote a particular resource as the “backbuffer”.

// This is pretty handy for ad-hoc debugging :P const char *backbuffer_source = getenv("GRANITE_SURFACE"); graph.set_backbuffer_source(backbuffer_source ? backbuffer_source : "tonemapped");

Now let’s get into the actual implementation.

## Time to bake!

Once we’ve set up the structures, we need to bake the render graph. This goes through a bunch of steps, each completing one piece of the puzzle …

### Validate

Pretty straight forward, a quick sanity check to ensure that the data in the RenderPass structures makes sense.

One interesting thing here, is that we can check if color input dimensions match color outputs. If they differ, we don’t do straight loadOp = LOAD, but we can do a scaled blit instead on start of the render pass. This is super convenient for things like game rendering at lower-res -> UI at native res. The loadOp in this case becomes DONT_CARE.

### Traverse dependency graph

We have an acyclic graph (I hope … :D) of render passes now, which we need to flatten down into an array of render passes. The list we create will be a valid submission order if we were to submit every pass one after the other. This submission order might not be the most optimal, but we’ll get close later.

The algorithm here is straight forward. We traverse the tree bottom-up. Using recursion, push the pass index of all the passes which write to backbuffer, then, for all those passes, push the writes for the resources in those passes … and so on until we reach the top leaves. This way, we ensure that if a pass A depends on pass B, pass B will always be found later than A in the list. Now, reverse the list, and prune duplicates.

We also register if a pass is a good “merge candidate” with another pass. For example, the lighting pass uses input attachments from gbuffer pass, and it shares some color/depth attachments … On tile-based architectures we can actually merge those passes without going to main memory using Vulkan’s multipass feature, so we keep this in mind for the reordering pass which comes after.

### Render pass reordering

This is the first interesting step of the process. Ideally, we want a submission order which has optimal overlap between passes. If pass A writes some data, and pass B reads it, we want the maximum number of passes between A and B in order to minimize the number of “hard barriers”. This becomes our optimization metric.

The algorithm implemented is probably very inoptimal in terms of CPU time, but it gets the job done. It looks through the list of passes not yet scheduled in, and tries to figure out the best one based on three criteria:

-   Do we have a merge candidate as determined by the dependency graph traveral step earlier? (Score: infinite)
-   What is the latest pass in the list of already scheduled passes which we need to wait for? (Score: number of passes which can overlap in-between)
-   Does scheduling this pass break the dependency chain? (If so, skip this pass).

Reading the code is probably more instructive, see RenderGraph::reorder_passes().

Another sneaky consideration which should be included is when the lighting pass depends on some resources, while the G-buffer pass doesn’t. This can break subpass merging, because we go through this scheduling process:

-   Schedule in G-buffer pass, it has no dependencies
-   Try to schedule in lighting pass, but whoops, we haven’t scheduled the shadow passes which we depend on yet … Oh well ![🙂](https://s.w.org/images/core/emoji/13.1.0/svg/1f642.svg)

The dirty solution to this was to lift dependencies from merge candidates to the first pass in the merge chain. Thus, the G-buffer pass will be scheduled after shadow passes, and it’s all good. A more clever scheduling algorithm might help here, but I’d like to keep it as simple as possible.

### Logical-to-physical resource assignment

When we build our graph, we might have some read-modify-writes. For lighting pass, emissive goes in, HDR result goes out, but clearly, it’s really the same resource, we just have this abstraction to figure out the dependencies in a sensible way, give some descriptive names to resources, and avoid cycles. If we had multiple passes, all doing emissive -> emissive for example, we have no idea which pass comes first, they all depend on each other (?), and I’d rather not deal with potential cycles.

What we do now is assign a physical resource index to all resources, and alias resources which do read-modify-write. If we cannot alias for some reason, it’s a sign we have a very wonky submission order which tries to do reads concurrently with writes. The implementation just throws its hands in the air in that case. I don’t think this will happen with an acyclic graph, but I cannot prove it.

### Logical-to-physical render pass assignment

Next, we try to merge adjacent render passes together. This is particularly important on tile-based renderers. We try to merge passes together if:

-   They are both graphics passes
-   They share some color/depth/input attachments
-   Not more than one unique depth/stencil attachment exists
-   Their dependencies can be implemented with BY_REGION_BIT, i.e. no “texture” dependency, which allows sampling for arbitrary locations.

### Transient or physical image storage

Similar story as subpass merging, tile-based renderers can avoid allocating physical memory for the attachment if you never actually write to it (with storeOp = STORE)! This can save a lot of memory for deferred especially, but also for depth buffers if they are not used later in post for example.

A resource can be transient if:

-   It is used in a single physical render pass (i.e. it never needs to storeOp = STORE)
-   It is invalidated at the start of the render pass (no loadOp = LOAD needed)

### Build RenderPassInfo structures

Now, we have a clear view of all the passes, their dependencies and so on. It is time to make some render pass info structures.

This part of the implementation is very tied into how Granite’s Vulkan backend does things, but it closely mirrors the Vulkan API, so it shouldn’t be too weird. VkRenderPasses are generated on demand in the Vulkan backend, so we don’t do that here, but we could potentially bake that right now.

The actual image views will be assigned later (every frame actually), but subpass infos, number of color attachments, inputs, resolve attachments for MSAA, and so on can be done up front at least. We also build a list of which physical resource indices should be pulled in as attachments as well.

We also figure out which attachments need loadOp = CLEAR or DONT_CARE now by calling some callbacks. For attachments which have an input, just use loadOp = LOAD (or use scaled blits!). For storeOp we just say STORE always. Granite recognizes transient attachments internally, and forces storeOp = DONT_CARE for those attachments anyways.

### Build barriers

It is time to start looking at barriers. For each pass, each resource goes through three stages:

-   Transition to the appropriate layout, caches need to be invalidated
-   Resource is used (read and/or writes happen)
-   The resource ends up in a new layout, with potential writes which need to be flushed later

For each pass we build a list of “invalidates” and “flushes”.

Inputs to a pass are placed in the invalidate bucket, outputs are placed in the flush bucket. Read-modify-write resources will get an entry in both buckets.

For example, if we want to read a texture in a pass we might add this invalidate barrier:

-   stages = FRAGMENT (or well, VERTEX, but I’d have to add extra stage flags to resource inputs)
-   access = SHADER_READ
-   layout = SHADER_READ_ONLY_OPTIMAL

For color outputs, we might say:

-   stages = COLOR_ATTACHMENT_OUTPUT
-   access = COLOR_ATTACHMENT_WRITE
-   layout = COLOR_ATTACHMENT_OPTIMAL

This tells the system that “hey, there are some pending writes in this stage, with this memory access which needs to be flushed with srcAccessMask. If you want to use this resource, sync with these things!”

We can also figure out a particular scenario here with render passes. If a resource is used as both input attachment and read-only depth attachment, we can set the layout to DEPTH_STENCIL_READ_ONLY_OPTIMAL. If color attachment is used also as an input attachment we can use GENERAL (programmable blending yo!), and similar for read-write depth/stencil with input attachment.

### Build physical render pass barriers

Now, we have a complete view of each pass’ barriers, but what happens when we start to merge passes together? Multipass will likely perform some barriers internally as part of the render pass execution (think deferred shading), so we can omit some barriers here. These barriers will be resolved internally with VkSubpassDependency when we build the VkRenderPass later, so we can forget about all barriers which need to happen between subpasses.

What we are interested in is building invalidation barriers for the first pass a resource is used. For flush barriers we care about the last use of a resource.

Now, there are two cases we need to cover here to ensure that every pass can deal with synchronization before and after the pass executes.

#### Only invalidation barrier, no flush barrier

This is the case for read-only resources. We still need to guard ourselves against write-after-read hazards later. For example, what if the next pass starts to write to this resource? Clearly, we need to let other passes know that this pass needs to complete before we can start scribbling on a resource. The way this is implemented is by injecting a fake flush barrier with access = 0. access = 0 basically means: “there are no pending writes to be seen here!” This way we can have multiple passes back to back which all just read a resource. If the image layout stays the same and srcAccessMask is 0, we don’t need barriers.

#### Only flush barrier, no invalidation barrier

This is typically the case for passes which are “write only”. This lets us know that before the pass begins we can discard the resource by transitioning from UNDEFINED. We still need an invalidation barrier however, because we need a layout transition to happen before we start the render pass and caches need to be invalidated, so we just inject an invalidate barrier here with same layout and access as the flush barrier.

#### Ignore barriers for transients/swapchain

You might notice that barriers for transients are just “dropped” for some reason. Granite internally uses external subpass dependencies to perform layout transitions on transient attachments, although this might be kind of redundant now with the render graph. The swapchain is similar. Granite internally uses subpass dependencies to transition the swapchain image to finalLayout = PRESENT_SRC_KHR when it is used in a render pass.

### Render target aliasing

The final step in our baking process is to figure out if we can temporally alias resources in the graph. For example, we might have two or more resources which exist at completely different times in a frame. Consider a separable blur:

-   Render a frame (Buffer #0)
-   Blur horiz (Buffer #1)
-   Blur vert (Should ping-pong back to buffer #0)

When we specify this in the render graph we have 3 distinct resources, but clearly, the vertical blur render target can alias with the initial render target. I suggest looking at Frostbite’s presentation here on their results with aliasing, it’s quite massive.

We could technically alias actual VkDeviceMemory here, but this implementation just tries to reuse VkImages and VkImageViews directly. I’m not sure if there is much to be gained by trying to suballocate directly from the dead corpses of other images and hope that it will work out. Something to look at if you’re really starved for memory I guess. The merit of aliasing image memory might be questionable, as VK_*_dedicated_allocation is a thing, so some implementation might prefer that you don’t alias. Some numbers and IHV guidance on this is clearly needed.

The algorithm is fairly straight forward. For each resource we figure out the first and last physical render pass where a resource is used. If we find another resource with the same dimensions/format, and their pass range does not overlap, presto, we can alias! We inject some information where we can transition “ownership” between resources.

For example, if we have three resources:

-   Alias #0 is used in pass #1 and #2
-   Alias #1 is used in pass #5 and #7
-   Alias #2 is used in pass #8 and #11

At the end of pass #2, the barriers associated with Alias #0 are copied over to Alias #1, and the layout is forced to UNDEFINED. When we start pass #5, we will magically wait for pass #2 to complete before we transition the image to its new layout. Alias #1 hands over to alias #2 after pass #7 and so on. Pass #11 hands over control back to alias #0 in the next frame in a “ring”-like fashion.

Some caveats apply here. Some images might have “history” or “feedback” where each image actually has two instances of itself, one for current frame, and one for previous frame. These images should never alias with anything else. Also, transient images do not alias. Granite’s internal transient image allocator takes care of this aliasing internally, but again, with the render graph in place, that is kind of redundant now …

Another consideration is that adding aliasing might increase the number of barriers needed and reduce GPU throughput. Maybe the aliasing code needs to take extra barrier cost into consideration? Urk … At least if you know your VRAM size while baking, you have a pretty good idea if aliasing is actually worth it based on all the resources in the graph. Optimizing the dependency graph for maximum overlap also greatly reduces the oppurtunities for aliasing, so if we want to take memory into consideration, this algorithm could easily get far more involved …

### Preparing resources for async compute

For async compute, resources might be accessed by both a graphics and a compute queue. If their queue families differ (ohai AMD), we have to decide if we want EXCLUSIVE or CONCURRENT queue access to these resources. For buffers, using CONCURRENT seems like an obvious choice, but it’s a bit more complicated with images. In the name of not making this horribly complicated, I went with CONCURRENT, but only for the resources which are truly needed in both compute and graphics passes. Dealing with EXCLUSIVE will be brutal, because now we have to consider read-after-read barriers as well and ping-pong ownership between two queue families ![😀](https://s.w.org/images/core/emoji/13.1.0/svg/1f600.svg) (Oh dear)

### Summary

A lot of stuff to consider to go through, but now we have all the data structures in place to start pumping out frames.

## The runtime

While baking is a very involved process, executing this is reasonably simple, we just need to track the state of all resources we know about in the graph.

Each resource stores:

-   The last VkEvent. If we need to ask ourselves, “what do I need to wait for before I touch this resource”, this is it. I opted for VkEvent because it can express execution overlap, while pipeline barriers cannot.
-   The last VkSemaphore for graphics queue. If the resource is used in async compute, we use semaphores instead of VkEvents. Semaphores cannot be waited on multiple times, so we have a semaphore which can be waited on **once** in the graphics queue if needed.
-   The last VkSemaphore for compute queue. Same story, but for waiting in the compute queue **once**.
-   Flush stages (VkPipelineStageFlags), this contains the stages which we need to wait for (srcStageMask) if we need to wait for the resource.
-   Flush access (VkAccessFlags), this contains the srcAccessMask of memory we need to flush before we can use the resource.
-   Per-stage invalidation flags (VkAccessFlag for each pipeline stage). These bitmasks keep track of in which pipeline stages and access flags it is safe to use the resource. If we figure out that we have an invalidation barrier, but all the relevant stages and access bits are already good to go, we can drop the barrier altogether. This is great for cases where we read the same resource over and over, all in SHADER_READ_ONLY_OPTIMAL layout.
-   The current layout of the resource. This is currently stored inside the image handles themselves, but this might be a bit wonky if I add multithreading later …

For each frame, we assign resources. At the very least we have to replace the swapchain image, but some images might have been assigned as “not persistent”, in which case we allocate a fresh resource every frame. This is useful for scenarios where we trade more memory usage (more copies in flight on the GPU) for removal of all cross-frame barriers. This is probably a terrible idea for large render targets, but small compute buffers of a few kB each? Duh. If we can kick off GPU work earlier, that’s probably a good thing.

If we allocate a new resource, all barrier state is cleared to its initial state.

Now, we get into pushing render passes out. The current implementation loops through all the passes and deal with barriers as they come up. If you interleave this loop hard enough, I’m sure you’ll see some multithreading potential here ![🙂](https://s.w.org/images/core/emoji/13.1.0/svg/1f642.svg)

### Check conditional execution

Some render passes do not need to be run this frame, and might only need to run if something happened (think shadow maps). Each pass has a callback which can determine this. If a pass is not executed, it does not need invalidation/flush barriers. We still need to hand over aliasing barriers, so just do that and go to next pass.

### Handle discard barriers

If a pass has discard barriers, just set the current layout of the image to UNDEFINED. When we actually do the layout transition, we will have oldLayout = UNDEFINED.

### Handle invalidate barriers

This part comes down to figuring out if we need to invalidate some caches, and potentially flush some caches as well. There are some things we have to check here:

-   Are there pending flushes?
-   Does the invalidate barrier need a different image layout than the current one?
-   Are there some caches which have not been flushed yet?

If the answer to either question is yes, we need some kind of barrier. We implement this barrier in one of three ways:

-   vkCmdWaitEvents – If the resource has a pending VkEvent, along with appropriate VkBufferMemoryBarrier/VkImageMemoryBarrier.
-   vkQueueSubmit w/ semaphore wait. Granite takes care of adding semaphores at submit time. We push in a wait semaphore along with dstWaitStageMask which matches our invalidate barrier. If we also need a layout transition, we can add a vkCmdPipelineBarrier with srcStageMask = dstStageMask to latch onto the dstWaitStageMask … and keep the pipeline going. We generally do not need to deal with srcAccessMask if we waited on a semaphore, so usually this will just be forced to 0.
-   vkCmdPipelineBarrier(srcStage = TOP_OF_PIPE_BIT). This is used if the resource hasn’t been used before, and we just need to transition away from UNDEFINED layout.

The barriers are batched up as appropriate and submitted. Buffers are much simpler as they do not have layouts.

After invalidation we mark the appropriate stages as properly invalidated. If we changed the layout or flushed memory access as part of this step, we clear everything to 0 before this step.

### Execute render passes

This is the easy part, just call begin/nextsubpass/end and fire off some callbacks to push the real graphics work. For compute, just drop the begin/end.

For graphics we might do some scaled blits at the beginning and some automatic mipmap generation at the end.

### Handle flush barriers

This part is simpler. If there is at least one resource which is only used in a single queue, we signal an VkEvent here and assign it to all relevant resources. If we have at least one resource which is used cross-queue, we also signal two semaphores here (one for graphics, one for compute later …)

We also update the current layout, and mark flush stages/flush access for later use.

### Alias handoff

If the resource is aliased, we now copy the barrier state of a resource over to its next alias, and force the layout to UNDEFINED.

### Submission

The command buffer for each pass is now submitted to Granite. Granite tries to batch up command buffers until it needs to wait for a semaphore or signal one.

### Scale to swapchain

After all the passes are done, we can inject a final blit to swapchain if the backbuffer resource dimensions do not match the actual swapchain. Otherwise, we alias those resources anyways, so no need for useless blitting passes.

## Conclusion

Hopefully this was interesting. The word count of this post is close to 5K at this point, and the render graph is a 3 ksloc behemoth (sigh). I’m sure there are bugs (actually I found two in async compute while writing this), but I’m quite happy how this turned out.

Future goals might be trying to see if this can be made into a reusable, standalone library and getting some actual numbers.