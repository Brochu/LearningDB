# Vulkan tutorial on rendering a fullscreen quad without buffers

 Posted on August 13, 2016  |  Sascha

# Rendering a fullscreen quad* the easy way

_(*) Which is actually just a huge triangle_

Having to render a fullscreen quad (or something that fills the whole screen) is a common task in 3D real time graphics, as many effects rely on rendering a texture over the whole screen with proper uv coordinates in the [0..1] range. This applies to post processing (glow, blur, ssao), deferred shading or procedurally generated output.

The most forward way of rendering such a fullscreen quad is to setup buffers containing vertices (optional texture coordinates) and indices for rendering a quad made up of two triangles, bind these buffers and have these as input attributes for your shaders. While this can be done with only a few lines in OpenGL, this requires lots of boiler plate in Vulkan.

## Technical background

So instead of using vertices (and indices) from a buffer to render something that fills the screen, the vertices and texture coordinates can easily be generated in your vertex shader using the **gl_VertexIndex** (see [GL_KHR_vulkan_GLSL](https://www.khronos.org/registry/vulkan/specs/misc/GL_KHR_vulkan_glsl.txt)) vertex input variable (similar to OpenGL [gl_VertexID](https://www.opengl.org/sdk/docs/man/html/gl_VertexID.xhtml)). Although that’s a bit simplified, it basically holds the index of the current vertex for which the vertex shader is invoked.

We will be using it to generate a **fullscreen triangle** that acts like a fullscreen quad. Why a triangle? Because it only requires 3 vertex shader invocations (instead of 6 for a quad made up of two triangles).

The triangle generated will look like this:

[![fullscreentriangle_clipped](https://www.saschawillems.de/images/2016-08-13-vulkan-tip-for-rendering-a-fullscreen-quad-without-buffers/fullscreentriangle_clipped.png)](https://www.saschawillems.de/images/2016-08-13-vulkan-tip-for-rendering-a-fullscreen-quad-without-buffers/fullscreentriangle_clipped.png)

The triangle contains our whole screen and with that the whole uv range of [0..1] so that we can use it like a normal fullscreen quad for applying a post processing effect. Thanks to the GPU clipping everything outside of the screen boundaries we actually get a quad that only requires 3 vertices. And since clipping is for free, we won’t waste bandwitdh as the clipped parts of the triangle (grayed out parts) are not sampled.

## The Vulkan part

Using this in a Vulkan application is pretty simple, and as we don’t require any buffers we save a lot of boiler plate (including descriptor sets and layouts). So adding this to an existing Vulkan application is pretty straightforward, though there are a few things to consider if you haven’t done any “bufferless” rendering in Vulkan before.

### Graphics pipeline

The **pVertexInputState** member of the **VkPipeline** specifies the vertex input attributes and vertex input bindings. As we don’t pass any vertices to our shaders, we need to setup an empty vertex input state for the fullscreen pipeline that doesn’t contain any input or attribute bindings:

```cpp
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
```

```cpp
VkPipelineVertexInputStateCreateInfo emptyInputState;
emptyInputState.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
emptyInputState.vertexAttributeDescriptionCount = 0;
emptyInputState.pVertexAttributeDescriptions = nullptr;
emptyInputState.vertexBindingDescriptionCount = 0;
emptyInputState.pVertexBindingDescriptions = nullptr;
...
pipelineCreateInfo.pVertexInputState = &emptyInputState;
...
vkCreateGraphicsPipelines(device, pipelineCache, 1, &pipelineCreateInfo;, nullptr, &fullscreenPipeline;));
```

### Culling

Note that the vertices are in clock-wise order (see illustration above), so if you use culling you need to take this into account with e.g the following pipeline setup:

```cpp
1
2
```

```cpp
rasterizationState.cullMode = VK_CULL_MODE_FRONT_BIT;
rasterizationState.frontFace = VK_FRONT_FACE_COUNTER_CLOCKWISE;
```

### Rendering

With the above vertex input state setup for the fullscreen pipeline we’re now able to draw something without having to bind a vertex (and index) buffer upfront like you’d do when rendering geometry from a buffer:

```cpp
1
2
```

```cpp
vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, fullscreenPipeline);
vkCmdDraw(commandBuffer, 3, 1, 0, 0);
```

Just bind the pipeline with the empty vertex input state and issue a draw command (non-indexed) with a vertex count of 3. If you’re using the correct pipeline this will not generate any validation layer errors, even though there are (technically) no vertices to actually render.

### The vertex shader

The interesting part is the vertex shader that will generate the vertices based on the gl_VertexIndex:

```cpp
1
2
3
4
5
6
7
8
9
```

```cpp
#version 450 
...
layout (location = 0) out vec2 outUV;
... 
void main() 
{
    outUV = vec2((gl_VertexIndex << 1) & 2, gl_VertexIndex & 2);
    gl_Position = vec4(outUV * 2.0f + -1.0f, 0.0f, 1.0f);
}
```

This shader does not contain any input attributes and generates the position and the texture coordinate (passed to the fragment shader) solely based on the gl_VertexIndex with the texture coordinates being in the [0..1] range for the visible part of the triangle.

### Example code

While there is no explicit example for this technique in my Vulkan repository, this is used in some of the post processing examples like the [radial blur one](https://github.com/SaschaWillems/Vulkan/tree/master/radialblur) if you want to see this in action.