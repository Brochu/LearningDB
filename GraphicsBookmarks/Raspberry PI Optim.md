# Optimizing RigelEngine’s OpenGL rendering for the Raspberry Pi

 [lethal_guitar](https://lethalguitar.wordpress.com/author/lethalguitar/ "Posts by lethal_guitar")  [programming](https://lethalguitar.wordpress.com/category/programming/), [Rigel Engine](https://lethalguitar.wordpress.com/category/rigel-engine/)  April 22, 2021 16 Minutes

While working on [RigelEngine](https://github.com/lethal-guitar/RigelEngine), performance was mostly a non-issue. Modern computers are so powerful that a [2D game from the early 90s](https://en.wikipedia.org/wiki/Duke_Nukem_II) is no match for them. My gaming PC easily runs the game at well over 3000 FPS when uncapping the frame rate, and a 6 year old MacBook Pro still runs at more than 400 FPS. But it’s a different story on the Raspberry Pi. When I first tried running RigelEngine on a Pi 3 model B, it only managed around 15 to 17 FPS when in-game. After a round of optimizations, it now runs at a stable 60 FPS (v-sync on) at 1080p resolution on both Pi 3 and Pi Zero. In this article, I’ll go through the optimization process and describe the changes that made a big difference in performance. And since the project is open-source, you’ll find links to all the relevant code changes.

[![](https://lethalguitar.files.wordpress.com/2021/04/pxl_20210421_105838816.jpg?w=768)](https://lethalguitar.files.wordpress.com/2021/04/pxl_20210421_105838816.jpg)

RigelEngine running on a Pi Zero, with a Pi 3 and Pi 1 sitting next to it

## Rendering and scene complexity

First, let’s have a look at how complex a typical scene in the game is. This way, we have a frame of reference for how much we are asking of the hardware. At first glance, it may seem odd that a game this old would pose any challenge even for something like the Raspberry Pi. The original ran on machines with less than a megabyte of RAM and CPU speed in the range of 25 to 66 MHz. And the Pi is vastly more powerful than that, even if it pales in comparison to a modern low-end gaming PC. But the way RigelEngine is rendering its graphics is very different from how it originally worked, even if the visual output looks practically the same. The game uses the GPU for rendering, via OpenGL (either 3.0 or 2.0 ES).

  
On the Pi, I’m using GL ES with the proprietary Broadcom driver (aka legacy driver), without an X server running. I’ve found that this gives me the best performance compared to the KMS or FKMS driver. I should note though that I haven’t tested using KMS _without_ X11, so I don’t know how that compares to the legacy driver.

When in-game, there are several elements making up the scene: Parallax background layer, tiles, sprites, particle effects, and hud. Tiles and sprites are further divided into a background and foreground layer.

[![](https://lethalguitar.files.wordpress.com/2021/04/untitled-diagram-1.png?w=1024)](https://lethalguitar.files.wordpress.com/2021/04/untitled-diagram-1.png)

Aside from particle effects, which are rendered using the `GL_POINTS` primitive, all of these elements are drawn as textured quads using `GL_TRIANGLES` and an index buffer. My renderer automatically combines (batches) draw requests that use an identical texture to reduce the number of draw calls. All textures are drawn with `GL_NEAREST` filtering.

RigelEngine runs at whatever resolution the OS reports, which is typically the native resolution of the monitor unless configured otherwise. So that’s 1080p in my case. But the actual rendered content is only 1440×1080, matching the original’s 4:3 aspect ratio (unless [wide-screen mode](https://lethalguitar.wordpress.com/2019/12/29/rigelengine-adding-a-wide-screen-mode/) is enabled, in which case the entire screen is used).

The original game ran at a resolution of 320×200, which (although 16:10 normally) was displayed as 4:3 on CRT monitors of the time (the monitor horizontally stretched the image, resulting in non-square pixels). Textures are accordingly small in size. Many sprites are around 24×24 or 32×32 pixels. A background image is 320×200, and a tileset is 320×240. The sprite sheet containing fonts and many UI elements for the HUD is also 320×200.

In terms of primitives rendered, the workload can range from as low as 340 triangles, to over 1500 triangles plus 500 points. The largest amount of triangles comes from rendering the tiles: There can be between 680 and 710 tiles visible at a time in busy scenes. Sprite count in a busy scene could be around 20 or more, but a simple scene might have only one sprite.

[![](https://lethalguitar.files.wordpress.com/2021/04/screenshot-2021-04-18-at-11.16.57.png?w=1024)](https://lethalguitar.files.wordpress.com/2021/04/screenshot-2021-04-18-at-11.16.57.png)

Scene with many tiles rendered

[![](https://lethalguitar.files.wordpress.com/2021/04/screenshot-2021-04-18-at-11.17.11.png?w=1024)](https://lethalguitar.files.wordpress.com/2021/04/screenshot-2021-04-18-at-11.17.11.png)

Scene with fewer tiles rendered due to more background showing through

To put this in perspective, 1500 is roughly the same amount of triangles as three humanoid characters or a single large enemy from Half-Life 1, or a single (regular) headcrab from Half-Life 2. Even Quake 1 can have more than 2000 triangles in a single scene. So overall, the workload seems very reasonable for the Pi – what made it so slow initially?

## Hunting for the bottleneck

I didn’t use any dedicated profiling tools. My approach was to set up an easily repeatable benchmark that gave me a frame time measurement, and then experiment. I might look more into tools for getting performance data in the future, but for this round of optimizations, the benchmark-driven approach worked great.

As a side note, I highly recommend using frame time (delta time between consecutive frames) instead of FPS when optimizing: The former makes it easy to compare two different values. A gain of 5ms is always the same regardless of what the baseline performance is. “20 FPS faster” on the other hand means different things depending on the baseline. Improving from 20 FPS to 40 means we’ve reduced rendering time by 25 ms, but improving from 200 FPS to 220 is only a reduction of 0.5 ms. You can read a more in-depth argument for frame time here, if you’re interested.

For the benchmark, I made it so that the game would load directly into the first level, run the render loop, and collect frame times for a few seconds, then automatically quit and print out the average frame time. This way, it was easy to make a change in the code and see how it affected frame time. No user input was given during the benchmark, in order to keep the workload consistent.

Experimenting this way, I was able to hone in on the bottlenecks relatively quickly. It still took some time (and asking a question in a [forum post](https://www.raspberrypi.org/forums/viewtopic.php?f=68&t=294631)) to really understand why some things are slow – I initially found it hard to believe how much the Pi struggled with such a seemingly simple workload at 1080p. But finding the slow parts was fairly easy. In the end, it came down to two fairly big issues which accounted for the bulk of the slowdown, plus a couple smaller optimizations to help especially the weaker Pis like Pi Zero and Pi 1 model B.

Before we get into what the bottlenecks were, a bit of context: The average frame time I started out with in my benchmark was 65ms, which corresponds to 15 FPS. In order to hit 60 FPS (16ms), I had to find a way to reduce this by at least 49ms. Now let’s look at the first improvement I found.

## Introducing a sprite texture atlas

I quickly discovered that skipping sprite rendering saved 24ms. That was very surprising: I didn’t expect sprite rendering to be an issue at all, since it generally requires even less triangles than the HUD. And in the benchmark scene, there were only 4 sprites on screen. So something was clearly fishy.  
  
After some digging, I realized that many more sprites were submitted for rendering than what I could see on screen. It turned out that I had introduced a bug long ago, which broke culling (the removal of invisible elements to improve performance) of sprites. In my benchmark scene, this caused 24 sprites to be rendered, instead of just the 4 that are visible on screen. Because each sprite had its own texture at that point, this also resulted in 20 additional draw calls, and as I’ve already learned before, draw call overhead is quite heavy on the Pi. I suspect that all the texture switches also added to the overall cost, though, as 20 draw calls is still not that many.  
  
I could have just fixed the culling, and most likely that would’ve improved things quite a bit already. But I had also been planning to introduce a texture atlas for sprites for a while. So I decided to focus on that first, and deal with the culling issue later. My thinking was that even if I didn’t do any culling, the resulting number of triangles would still be fairly low, since most levels feature only around 200 sprites in total. So I expected reducing draw calls and texture switches to have a much bigger impact than proper culling.  
  
And indeed, once I had the texture atlas in place, it gave me a frame time improvement of 22ms. Later on, I also fixed the culling, but only after I had done some other significant changes (more about that in a sec), so I don’t have a good frame time comparison for the culling specifically. When I tested rendering all the sprites in the level without any culling, the difference was less than 2 ms, so I don’t think 20 additional sprites would have made much of a difference once the texture atlas was in place.

See the [code changes](https://github.com/lethal-guitar/RigelEngine/pull/620).  
  
With the sprite rendering bottleneck now out of the way, I was in a better place, but still had a long way to go. Fortunately, there was another very big win to be found.

## Bandwidth, bandwidth, bandwidth

The next big bottleneck turned out to be bandwidth, although it took me some time to fully understand this. Being a mobile focused GPU, the Pi’s [VideoCore IV](https://en.wikipedia.org/wiki/VideoCore#Variants) has very limited memory bandwidth. This is because high bandwidth also requires a lot of power, and in a mobile device, power efficiency is more important than raw performance. What’s more, the GPU doesn’t even have dedicated video memory aside from a small on-chip buffer – it uses the system’s main memory. The Pi’s GPU is a tile-based architecture, which means that it processes small chunks of data at a time by first fetching necessary input data from main memory, then using the on-chip memory for intermediate storage, and finally writing the end result back to main memory. I highly recommend reading [Performance Tuning for Tile-Based Architectures](https://www.seas.upenn.edu/~pcozzi/OpenGLInsights/OpenGLInsights-TileBasedArchitectures.pdf) to learn more. The authors do a great job explaining the differences to a Desktop GPU, and the tradeoffs that motivated the tile-based design.

Getting back to the Pi, it seems that the Pi 3 has a real-world memory write bandwidth of around 1.6 GB/s, and read of 2.2 GB/s ([source](https://magpi.raspberrypi.org/articles/raspberry-pi-4-specs-benchmarks)). The cited measurements are for the CPU and I don’t know how they compare to GPU memory bandwidth, but I assume that it can’t be far off since the GPU is using the same physical memory modules. Putting this in perspective, even low to mid-range gaming GPUs have bandwidth of more than 100 GB/s, with latest top of the line models like the RTX 3080 reaching more than 700 GB/s ([source](https://www.techpowerup.com/gpu-specs/geforce-rtx-3080.c3621)).

Now if we want to render a 1080p image at 60 FPS, we need roughly 355 MB/s of bandwidth (1920*1080*3*60) just for writing the output to the framebuffer (475 MB/s when including an alpha channel). That’s getting close to a quarter of the available total bandwidth, and we haven’t even factored in reading textures and vertex data yet. And lower-spec Pis like the Pi 1 or Pi Zero have even lower bandwidth. With this in mind, it seems pretty clear that processing multiple 1080p images in a single frame would chew through available bandwidth pretty quickly. But that’s exactly what my renderer was doing, by using two 1080p render targets, plus two additional smaller ones.

## Reducing use of render targets

Before optimization, rendering looked like this:

[![](https://lethalguitar.files.wordpress.com/2021/04/untitled-diagram-2.png?w=771)](https://lethalguitar.files.wordpress.com/2021/04/untitled-diagram-2.png)

Graph of the rendering pipeline. Each gray rectangle represents a render target (FBO)

First, the scene’s background layer (consisting of background image, tiles, and sprites) is rendered into a render target. This is needed to implement the “under water” effect, which works by modifying the colors of an already rendered scene. The easiest way of doing that kind of post-processing is rendering into a texture, and then rendering that texture with a special shader in order to implement the effect. This render pass is only necessary when water is visible on screen, but I had it enabled at all times originally in order to keep the logic simple. 

[![](https://lethalguitar.files.wordpress.com/2021/04/image-5.png?w=746)](https://lethalguitar.files.wordpress.com/2021/04/image-5.png)

The game’s “under water” effect turns all colors into shades of blue

After the background and foreground have been drawn, we need to draw particle effects. These are rendered using `GL_POINTS`. Each individual particle needs to be larger than one pixel however, since we are up-scaling to the native screen resolution. While it’s possible to increase the point size, we cannot accurately recreate the look of non-square pixels. So instead, I’m using another render target, which I call the low-res layer: This target has the same resolution as the original, so particles can be drawn as single pixels, and the rendered texture is then scaled to get the correct end result. Vertically stretching the texture with a larger factor than horizontally is no problem, so the non-square pixel effect is easily achieved this way. But at the cost of adding another render target into the mix. Similarly to particle effects, we need the same technique for dots on the radar shown in the HUD, since it’s also using single pixels for rendering.

[![](https://lethalguitar.files.wordpress.com/2021/04/grafik.png?w=266)](https://lethalguitar.files.wordpress.com/2021/04/grafik.png)

The game’s radar display

Everything we’ve just described is rendered into another texture, and that final top-level render target is then drawn on screen.

This is a multi-pass rendering setup: Things aren’t directly rendered into the frame buffer, but into intermediate buffers (the render targets). The top-level buffer then needs to be drawn onto the frame buffer in a separate step. As we’ve just learned, this is not ideal for tile-based architectures due to the increased demand on memory bandwidth. What happens is that the GPU renders something to a render target and writes the intermediate buffer into main memory (sliced up into tiles). It then has to load that data back into on-chip memory, do more processing, and finally write it back into main memory. This means that each render target causes an additional roundtrip to main memory which eats into our bandwidth. With a 1080p render target, we need almost 1 GB/s of additional bandwidth on top of what we already need for writing out the final framebuffer. That’s a massive difference! I measured roughly 8ms of overhead in frame time when rendering to a 1080p render target vs rendering directly. And if we want to hit 60 FPS, we only have 16ms of overall time budget per frame. Once we have two 1080p render targets, it becomes completely infeasible to run at 60 FPS.

So could we do without all these render targets? In theory yes, but it would require fairly significant reworking of the code. I could render particle effects as small colored rectangles, which would avoid the need for the “low-res layer” render target. That change alone wouldn’t be so bad. But getting rid of the top-level render target is harder. And I have no idea how to avoid the water effect buffer, short of disabling the effect completely.

Why is it hard to get rid of the top-level render target? I’m using it to implement screen-fade effects. Whenever the game transitions between scenes, it does a quick fade-out and fade-in of the screen. With a render target, I can implement that easily by drawing the texture with decreasing/increasing alpha (side note: the original game does the fade by modifying the VGA color palette). It’s possible to implement the same effect without a render target, by drawing a black rectangle on top of the scene, and then increasing/decreasing its alpha. But changing my code to do the fade that way would be a lot of work. Doing it with a render target simplifies a lot of code, since the render target captures whatever was last rendered by the game. This means I can do a fade-out without needing to continually re-render the game, and so the game code largely doesn’t even need to be aware of the fading. If I wanted to switch to the other approach, I would need to change a lot of places in the code to make it work, and do a lot of testing to make sure all possible transitions work properly.

Because of that, I decided to keep the render targets, but make them much smaller. Instead of being as large as the framebuffer, the water effect buffer and top-level target are now 320×200 each. Previously, each individual tile, sprite etc. was up-scaled separately, but now everything is rendered at 1 to 1 scale, and only the render target is stretched to the target size at the end. Because of this, I can now get rid of the low-res layer and the radar buffer, saving some more bandwidth.

Now you might ask, why did I use a 1080p render target in the first place, if the game’s art targets only 320×200? Why go through all the trouble with the low-res layer etc, the new setup seems much simpler? The answer is that I’m planning to enable the use of higher resolution sprite/tile mods in the future. And for these to actually appear hi-res, each element needs to be up-scaled individually, since there might be a mix of original artwork and high-res replacements. So I had put the more complex up-scaling system in place in anticipation of adding modding support in the future, without being aware of the performance implications on weaker devices like the Pi.

The high-res code path actually still exists, it’s just not the default anymore. This means that we get good performance on the Pi, but more powerful systems will be able to take advantage of high-res artwork once I add support for it. Fortunately, thanks to the way I had architected the rendering pipeline, this change was possible without having to touch too much code. For the majority of rendering code it doesn’t make a difference whether the coordinate system is defined by the size of the render target or by viewport transformations – it always just assumes that it’s drawing to a 320×200 “screen”, and doesn’t care how the up-scaling happens.

See the code changes [here](https://github.com/lethal-guitar/RigelEngine/pull/621) and [here](https://github.com/lethal-guitar/RigelEngine/commit/b5ae1e9263289e024bf99624d79f4a3e80eacb42).

Introducing the low-res code path gave another very significant boost to performance. Frame time improves by 12ms in a normal scene, and by 25ms when water effects are visible! This combined with the improvements to sprite rendering already improved performance tremendously, but there were a couple more smaller wins to be had.

## Simplified shader code

Most of the game’s graphics consist of straightforward bitmap drawing, but there are a few color modification effects: When enemies and other objects take damage, they briefly flash white.

[![](https://lethalguitar.files.wordpress.com/2021/04/image-8.png?w=428)](https://lethalguitar.files.wordpress.com/2021/04/image-8.png)

An enemy flashes white as its hit by Duke’s shot

Another color modification happens when drawing text in the menu. The underlying font texture is white, and is then modulated to appear in different colors to indicate the selected menu item, or draw attention to certain menu items.

[![](https://lethalguitar.files.wordpress.com/2021/04/image-9.png?w=1024)](https://lethalguitar.files.wordpress.com/2021/04/image-9.png)

The game’s menus featuring color modulation for menu items

Both of these effects are achieved using a shader. Initially, I had made a single shader that could draw regular textures and optionally apply these effects:

`#version 100`
`precision mediump` `float``;`
`varying highp vec2 texCoordFrag;`
`uniform sampler2D textureData;`
`uniform vec4 overlayColor;`
`uniform vec4 colorModulation;`
`void` `main() {`
 `vec2 texCoords = texCoordFrag;`
 `vec4 baseColor = texture2D(textureData, texCoords);`
 `vec4 modulated = baseColor * colorModulation;`
 `float` `targetAlpha = modulated.a;`
 `gl_FragColor =`
 `vec4(mix(modulated.rgb, overlayColor.rgb, overlayColor.a), targetAlpha);`
`}`

This flexibility didn’t come for free on the Pi’s GPU. Since the majority of objects do not use these effects, I created an additional “simple” shader, and added some code to dynamically switch shaders depending on whether the effects are currently enabled or not. This gave me a boost of 3ms in frame time, which isn’t as big but still needed to maintain 60 FPS at all times (for example when water effects are visible).  
  
This is the simple shader:

`#version 100`
`precision mediump` `float``;`
`varying highp vec2 texCoordFrag;`
`uniform sampler2D textureData;`
`void` `main() {`
 `gl_FragColor = texture2D(textureData, texCoordFrag);`
`}`

See the [code changes](https://github.com/lethal-guitar/RigelEngine/commit/b12a9d3191910a249ab616e472cc26e7eb52b3d2) – but note that the rendering code [changed quite a bit](https://github.com/lethal-guitar/RigelEngine/pull/632) in the meantime.

## Submitting less data

Another optimization I did which helped with the lower-spec Pis involved sending less data to the GPU/OpenGL. More specifically, I was originally using the following code to render a batch of textured quads:

`glBufferData(`
 `GL_ARRAY_BUFFER,`
 `sizeof``(``float``) * mBatchData.size(),`
 `mBatchData.data(),`
 `GL_STREAM_DRAW);`
`glBufferData(`
 `GL_ELEMENT_ARRAY_BUFFER,`
 `sizeof``(GLushort) * mBatchIndices.size(),`
 `mBatchIndices.data(),`
 `GL_STREAM_DRAW);`
`glDrawElements(`
 `GL_TRIANGLES,`
 `GLsizei(mBatchIndices.size()),`
 `GL_UNSIGNED_SHORT,`
 `nullptr``);`

This was submitting both vertex and index data for each batch. While optimizing, I realized that the index data is actually always the same – it’s only the vertex positions that change. So I made a single large index buffer that’s only submitted to OpenGL once, and the batch submission code became:

`glBufferData(`
 `GL_ARRAY_BUFFER,`
 `sizeof``(``float``) * mBatchData.size(),`
 `mBatchData.data(),`
 `GL_STREAM_DRAW);`
`glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, mQuadIndicesEbo);`
`glDrawElements(`
 `GL_TRIANGLES,`
 `mBatchSize,`
 `GL_UNSIGNED_SHORT,`
 `nullptr``);`
`glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);`

Each index is two bytes, and we need 6 indices to draw one quad (two triangles). When drawing 700 tiles, this means we save 8 kB of data. This improved frame time by 6ms on a Pi 1 model B.

See the [code changes](https://github.com/lethal-guitar/RigelEngine/commit/64e3e32b9ed045debf120bac0d81b8d4e162f127).

## What about the Pi 1?

The game runs at a stable locked 60 FPS on Pi 3 and Pi Zero now, but it’s not quite there yet on the Pi 1 model B, where it only reaches 50 FPS. It’s still possible to reach 60 FPS on that hardware by either lowering the resolution to 720p, or overclocking the CPU and GPU to 1 GHz and 400 MHz, respectively (stock settings are 700 MHz and 250 MHz). It’s worth noting that the Pi Zero features the same CPU and GPU as the Pi 1, but clocked higher out of the box (at 1 GHz and 400 MHz).

Because of that, I decided not to invest any more time into the Pi 1 for now. After all, the Pi Zero is cheaper and runs the game without issues out of the box, and it’s easily possible to overclock a Pi 1 to the same speeds as the Zero (and presumably safe to do so, given it’s the same hardware). I might still get back to it at some point, and see if I can push it over the edge – but no concrete plans at the moment.

## Wrap up and main takeaways

So that wraps up my look at OpenGL performance optimization for the Raspberry Pi. The two biggest wins were due to introducing a texture atlas for sprites, and using much smaller (and fewer) render targets to reduce the amount of bandwidth needed. Shader simplification and reusing a single index buffer for multiple draw calls also helped. With all optimizations combined, I was able to go from a measly 15/17 FPS up to 80 on the Pi 3, and the game now runs at a stable 60 on a Pi Zero as well.

My main takeaways from this work are:

-   Even a very simple (by today’s standards) 2D game still requires optimization to run well on the limited hardware of the Pi.
-   The VideoCore IV used in all Pis up to Pi 3 (Pi 4 has a new GPU) is not well suited for real-time rendering at native 1080p.

Regarding the latter: When considering that the GPU was aimed at resolutions like 854×480 originally, this isn’t too surprising. But I wasn’t aware of that at first, and kept wondering what I was doing wrong to get such bad performance with such a simple game. After I learned how limited the GPU really is, I tested [Quake III Arena](https://en.wikipedia.org/wiki/Quake_III_Arena) on my Pi 3 (using the optimized [Q3Lite](https://github.com/cdev-tux/q3lite) version) and found that at highest settings, it also only reached 30 FPS when running at 1080p. So overall, the Pi can only run very simple games at 1080p, and it’s not really surprising given what the hardware was actually designed for. With 1440p and 4k being fairly common resolutions nowadays, it’s easy to forget that 1080p is already a pretty high resolution, requiring quite a bit of processing power. But what the Pi lacks in raw power, it makes up for with a very affordable price, and low power consumption. Plus, this optimization journey was fun and rewarding, and I would’ve missed out on that if the Pi was more powerful ![🙂](https://s0.wp.com/wp-content/mu-plugins/wpcom-smileys/twemoji/2/svg/1f642.svg)

Well, I hope you enjoyed this article, and maybe even got something out of it for your own projects!