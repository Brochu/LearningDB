### Hallucinations re: the rendering of Cyberpunk 2077

**Introduction**

Two curses befall rendering engineers. First, we lose the ability to look at reality without being constantly reminded of how fascinatingly hard it is to solve light transport and model materials.

Second, when you start playing any game, you cannot refrain from trying to reverse its rendering technology (which is particularly infuriating for multiplayer titles - stop shooting at me, I'm just here to look how rocks cast shadows!).

So when I bought Cyberpunk 2077 I had to look at how it renders a frame. It's very simple to take RenderDoc captures of it, so I had really no excuse.

The following are speculations on its rendering techniques, observations made while skimming captures, and playing a few hours.

It's by no means a serious attempt at reverse engineering. For that, I lack both the time and the talent. I also rationalize doing a bad job at this by the following excuse: it's actually better this way. 

I think it's better to dream about how rendering (or anything really) could be, just with some degree of inspiration from external sources (in this case, RenderDoc captures), rather than exactly knowing what is going on.

If we know, we know, there's no mystery anymore. It's what we do not know that makes us think, and sometimes we exactly guess what's going on, but other times we do one better, we hallucinate something new... Isn't that wonderful?

The following is mostly a read-through of a single capture. I did open a second one to try to fill some blanks, but so far, that's all.

[![](https://1.bp.blogspot.com/-Accm4KmCjcQ/X90IqbPkniI/AAAAAAAACO4/Y30F2-QUQuEN3oa44WvOhq4zrALKMpAHwCLcBGAsYHQ/w400-h224/Capture.PNG)](https://1.bp.blogspot.com/-Accm4KmCjcQ/X90IqbPkniI/AAAAAAAACO4/Y30F2-QUQuEN3oa44WvOhq4zrALKMpAHwCLcBGAsYHQ/s957/Capture.PNG)

This is the frame we are going to look at.

I made the captures at high settings, without RTX or DLSS as RenderDoc does not allow these (yet?). I disabled motionblur and other uninteresting post-fx and made sure I was moving in all captures to be able to tell a bit better when passes access previous frame(s) data.

I am also not relying on insider information for this. Makes everything easier and more fun.

**The basics**

At a glance, it doesn't take long to describe the core of Cyberpunk 2077 rendering.

It's a classic deferred renderer, with a fairly vanilla g-buffer layout. We don't see the crazy amount of buffers of say, Suckerpunch's PS4 launch Infamous:Second Son, nor complex bit-packing and re-interpretation of channels.

[![](https://1.bp.blogspot.com/-Hfi1VZdBR7k/X9ukF1t2liI/AAAAAAAACMI/YfEXc8ELO8knX4zW4L0pfzHyckezaFVfACLcBGAsYHQ/w640-h360/001.png)](https://1.bp.blogspot.com/-Hfi1VZdBR7k/X9ukF1t2liI/AAAAAAAACMI/YfEXc8ELO8knX4zW4L0pfzHyckezaFVfACLcBGAsYHQ/s1690/001.png)

Immediately recognizable g-buffer layout

-   10.10.10.2 Normals, with the 2-bit alpha reserved to mark hair
-   10.10.10.2 Albedo. Not clear what the alpha is doing here, it seems to just be set to one for everything drawn, but it might be only the captures I got
-   8.8.8.8 Metalness, Roughness, Translucency and Emissive, in this order (RGBA)
-   Z-buffer and Stencil. The latter seems to isolate object/material types. Moving objects are tagged. Skin. Cars. Vegetation. Hair. Roads. Hard to tell / would take time to identify the meaning of each bit, but you get the gist...

If we look at the frame chronologically, it starts with a bunch of UI draws (that I didn't investigate further), a bunch of copies from a CPU buffer into VS constants, then a shadowmap update (more on this later), and finally a depth pre-pass.

[![](https://1.bp.blogspot.com/-Ysj98gZ695I/X9ukNnRwAiI/AAAAAAAACMU/71spZTnarM0XbAk1lEuI9Bis-ptl50CMgCLcBGAsYHQ/w640-h358/002.png)](https://1.bp.blogspot.com/-Ysj98gZ695I/X9ukNnRwAiI/AAAAAAAACMU/71spZTnarM0XbAk1lEuI9Bis-ptl50CMgCLcBGAsYHQ/s1056/002.png)

Some stages of the depth pre-pass.

This depth pre-pass is partial (not drawing the entire scene) and is only used to reduce the overdraw in the subsequent g-buffer pass.

Basically, all the geometry draws are using instancing and some form of bindless textures. I'd imagine this was a big part of updating the engine from The Witcher 3 to contemporary hardware. 

Bindless also makes it quite annoying to look at the capture in renderDoc unfortunately - by spot-checking I could not see too many different shaders in the g-buffer pass - perhaps a sign of not having allowed artists to make shaders via visual graphs? 

Other wild guesses: I don't see any front-to-back sorting in the g-buffer, and the depth prepass renders all kinds of geometries, not just walls, so it would seem that there is no special authoring for these (brushes, forming a BSP) - nor artists have hand-tagged objects for the prepass, as some relatively "bad" occluders make the cut. I imagine that after culling a list of objects is sorted by shader and from there instanced draws are dynamically formed on the CPU.

The opening credits do not mention Umbra (which was used in The Witcher 3) - so I guess CDPr rolled out their own visibility solution. Its effectiveness is really hard to gauge, as visibility is a GPU/CPU balance problem, but there seem to be quite a few draws that do not contribute to the image, for what's worth. It also looks like that at times the rendering can display "hidden" rooms, so it looks like it's not a cell and portal system - I am guessing that for such large worlds it's impractical to ask artists to do lots of manual work for visibility.

  

[![](https://1.bp.blogspot.com/-KRaRIyMFtME/X9ukTego-UI/AAAAAAAACMY/4U1NZpnI0ukFGI5v2xdt1QfIRWi8x7cnwCLcBGAsYHQ/w640-h240/003.png)](https://1.bp.blogspot.com/-KRaRIyMFtME/X9ukTego-UI/AAAAAAAACMY/4U1NZpnI0ukFGI5v2xdt1QfIRWi8x7cnwCLcBGAsYHQ/s1300/003.png)

A different frame, with some of the pre-pass.  
Looks like some non-visible rooms are drawn then covered by the floor - which might hint at culling done without old-school brushes/BSP/cell&portals?

Lastly, I didn't see any culling done GPU side, with depth pyramids and so on, no per-triangle or cluster culling or predicated draws, so I guess all frustum and occlusion culling is CPU-side.

_Note: people are asking if "bad" culling is the reason for the current performance issues, I guess meaning on ps4/xb1. This inference cannot be done, nor the visibility system can be called "bad" - as I wrote already. FWIW - it seems mostly that consoles struggle with memory and streaming more than anything else. Who knows..._

Let's keep going... After the main g-buffer pass (which seems to be always split in two - not sure if there's a rendering reason or perhaps these are two command buffers done on different threads), there are other passes for moving objects (which write motion vectors - the motion vector buffer is first initialized with camera motion).

This pass includes avatars, and the shaders for these objects do not use bindless (perhaps that's used only for world geometry) - so it's much easier to see what's going on there if one wants to.

Finally, we're done with the main g-buffer passes, depth-writes are turned off and there is a final pass for decals. Surprisingly these are pretty "vanilla" as well, most of them being mesh decals.

Mesh decals bind as inputs (a copy of) the normal buffer, which is interesting as one might imagine the 10.10.10 format was chosen to allow for easy hardware blending, but it seems that some custom blend math is used as well - something important enough to pay for the price of making a copy (on PC at least).

[![](https://1.bp.blogspot.com/-IOHdlY5caH0/X9ukf5owjLI/AAAAAAAACMk/0SBXr0kOnlg7a4BUqAwdOxZdoixcZRfNACLcBGAsYHQ/w640-h358/004.png)](https://1.bp.blogspot.com/-IOHdlY5caH0/X9ukf5owjLI/AAAAAAAACMk/0SBXr0kOnlg7a4BUqAwdOxZdoixcZRfNACLcBGAsYHQ/s1086/004.png)

A mesh decal - note how it looks like the original mesh with the triangles that do not map to decal textures removed.

It looks like only triangles carrying decals are rendered, using special decal meshes, but other than that everything is remarkably simple. It's not bindless either (only the main static geometry g-buffer pass seems to be), so it's easier to see what's going on here.

At the end of the decal pass we see sometimes projected decals as well, I haven't investigated dynamic ones created by weapons, but the static ones on the levels are just applied with tight boxes around geometry, I guess hand-made, without any stencil-marking technique (which would probably not help in this case) to try to minimize the shaded pixels.

Projected decals do bind depth-stencil as input as well, obviously as they need the scene depth, to reconstruct world-space surface position and do the texture projection, but probably also to read stencil and avoid applying these decals on objects tagged as moving.

[![](https://1.bp.blogspot.com/-lAQrsXP7oD0/X9ukuT7u_JI/AAAAAAAACMs/YvxJzcvupaokmi3Y_WqNd6bxqcw8Rz1fACLcBGAsYHQ/w640-h184/005.png)](https://1.bp.blogspot.com/-lAQrsXP7oD0/X9ukuT7u_JI/AAAAAAAACMs/YvxJzcvupaokmi3Y_WqNd6bxqcw8Rz1fACLcBGAsYHQ/s1520/005.png)

A projected decal, on the leftmost wall (note the decal box in yellow)

As for the main g-buffer draws, many of the decals might end up not contributing at all to the image, and I don't see much evidence of decal culling (as some tiny ones are draws) - but it also might depend on my chosen settings.

The g-buffer pass is quite heavy, but it has lots of detail and it's of course the only pass that depends on scene geometry, a fraction of the overall frame time. E.g. look at the normals on the ground, pushed beyond the point of aliasing. At least on this PC capture, textures seem even biased towards aliasing, perhaps knowing that temporal will resolve them later (which absolutely does in practice, rotating the camera often reveals texture aliasing that immediately gets resolved when stopped - not a bad idea, especially as noise during view rotation can be masked by motion blur).

[![](https://1.bp.blogspot.com/-FMoGzCIfuWA/X90LGa88O6I/AAAAAAAACPE/bKQroTcaEcQ1H-fhxouSXGkEy4RTZ0qAgCLcBGAsYHQ/w320-h245/Capture.PNG)](https://1.bp.blogspot.com/-FMoGzCIfuWA/X90LGa88O6I/AAAAAAAACPE/bKQroTcaEcQ1H-fhxouSXGkEy4RTZ0qAgCLcBGAsYHQ/s1010/Capture.PNG)

1:1 crop of the final normal buffer

**A note re:Deferred vs Forward+**

Most state-of-the-art engines are deferred nowadays. Frostbite, Guerrilla's Decima, Call of Duty BO3/4/CW, Red Dead Redemption 2, Naughty Dog's Uncharted/TLOU and so on.

On the other hand, the amount of advanced trickery that Forward+ allows you is unparalleled, and it has been adopted by a few to do truly incredible rendering, see for example the latest Doom games or have a look at the mind-blowing tricks behind Call of Duty: Modern Warfare / Warzone (and the previous Infinity Warfare which was the first time that COD line moved from being a crazy complex forward renderer to a crazy complex forward+).

I think the jury is still out on all this, and as most thing rendering (or well, coding!) we don't know anything about what's optimal, we just make/inherit choices and optimize around them. 

That said, I'd wager this was a great idea for CP2077 - and I'm not surprised at all to see this setup. As we'll see in the following, CP2077 does not seem to have baked lighting, relying instead on a few magic tricks, most of which operating in screen-space.

For these to work, you need before lighting to know material and normals, so you need to write a g-buffer anyways. Also you need temporal reprojection, so you want motion vectors and to compute lighting effects in separate passes (that you can then appropriately reproject, filter and composite).

I would venture to say also that this was done not because of the need for dynamic GI - there's very little from what I've seen in terms of moving lights and geometry is not destructible. I imagine instead, this is because the storage and runtime memory costs of baked lighting would be too big. Plus, it's easier to make lighting interactive for artists in such a system, rather than trying to write a realtime path-tracer that accurately simulates what your baking system results would be...

Lastly, as we're already speculating things, I'd imagine that CDPr wanted really to focus on artists and art. A deferred renderer can help there in two ways. First, it's performance is less coupled with the number of objects and vertices on screen, as only the g-buffer pass depends on them, so artists can be a smidge less "careful" about these. 

Second, it's simpler, overall - and in an open-world game you already have to care about so many things, that having to carefully tune your gigantic foward+ shaders for occupancy is not a headache you want to have to deal with...

**Lighting part 1: Analytic lights**

Obviously, no deferred rendering analysis can stop at the g-buffer, we split shading in two, and we have now to look at the second half, how lighting is done.

Here things become a bit dicier, as in the modern age of compute shaders, everything gets packed into structures that we cannot easily see. Even textures can be hard to read when they do not carry continuous data but pack who-knows-what into integers.

[![](https://1.bp.blogspot.com/-akYpe-UlOyM/X9ulOwOPPXI/AAAAAAAACNA/-HtNJRvFXmQUng7aSSy0_iLR5eWNr2W7gCLcBGAsYHQ/w640-h262/006B.png)](https://1.bp.blogspot.com/-akYpe-UlOyM/X9ulOwOPPXI/AAAAAAAACNA/-HtNJRvFXmQUng7aSSy0_iLR5eWNr2W7gCLcBGAsYHQ/s1614/006B.png)

Normal packing and depth pyramid passes.

  

Regardless, it's pretty clear that after all the depth/g-buffer work is said and done, a uber-summarization pass kicks in taking care of a bunch of depth-related stuff.

  

[![](https://1.bp.blogspot.com/-zvaNqDpea10/X90MzaKCXJI/AAAAAAAACPQ/9pgbEMY5YCALgjJXWuw_89lzz582HzHnQCLcBGAsYHQ/w400-h214/Capture.PNG)](https://1.bp.blogspot.com/-zvaNqDpea10/X90MzaKCXJI/AAAAAAAACPQ/9pgbEMY5YCALgjJXWuw_89lzz582HzHnQCLcBGAsYHQ/s734/Capture.PNG)

RGBA8 packed normal (&roughness). Note the speckles that are a tell-tale of best-fit-normal encoding.  
Also, note that this happens after hair rendering - which we didn't cover.

It first packs normal and roughness into a RGBA8 using Crytek's lookup-based best-fit normal encoding, then it creates a min-max mip pyramid of depth values.

[![](https://lh3.googleusercontent.com/-MVF93QzrjPc/X9sOrGmmZzI/AAAAAAAACHI/2wOQhr621FkrvdYtxrkxqyMetItZTzOGwCLcBGAsYHQ/w640-h176/006C.png)](https://lh3.googleusercontent.com/-MVF93QzrjPc/X9sOrGmmZzI/AAAAAAAACHI/2wOQhr621FkrvdYtxrkxqyMetItZTzOGwCLcBGAsYHQ/006C.png)

The pyramid is then used to create what looks like a volumetric texture for clustered lighting.

[![](https://1.bp.blogspot.com/-xCRd5MouyBg/X9ulXpg7NxI/AAAAAAAACNI/Mnlhw8tsoT4khzkGsJZdTWbiWBdqdmNPACLcBGAsYHQ/w400-h236/007.png)](https://1.bp.blogspot.com/-xCRd5MouyBg/X9ulXpg7NxI/AAAAAAAACNI/Mnlhw8tsoT4khzkGsJZdTWbiWBdqdmNPACLcBGAsYHQ/s1242/007.png)

A slice of what looks like the light cluster texture, and below one of the lighting buffers partially computed. Counting the pixels in the empty tiles, they seem to be 16x16 - while the clusters look like 32x32?

  

So - from what I can see it looks like a clustered deferred lighting system. 

The clusters seem to be 32x32 pixels in screen-space (froxels), with 64 z-slices. The lighting though seems to be done at a 16x16 tile granularity, all via compute shader indirect dispatches.

I would venture this is because CS are specialized by both the materials and lights present in a tile, and then dispatched accordingly - a common setup in contemporary deferred rendering systems (e.g. see Call of Duty Black Ops 3 and Uncharted 4 presentations on the topic).

Analytic lighting pass outputs two RGBA16 buffers, which seems to be diffuse and specular contributions. Regarding the options for scene lights, I would not be surprised if all we have are spot/point/sphere lights and line/capsule lights. Most of Cyberpunk's lights are neons, so definitely line light support is a must.

You'll also notice that a lot of the lighting is unshadowed, and I don't think I ever noticed multiple shadows under a single object/avatar. I'm sure that the engine does not have limitations in that aspect, but all this points at lighting that is heavily "authored" with artists carefully placing shadow-casting lights. I would also not be surprised if the lights have manually assigned bounding volumes to avoid leaks.

[![](https://1.bp.blogspot.com/-9mBcJwANHls/X9ulet6mFUI/AAAAAAAACNU/sIAxNMz0fxMbggAenggd0xZQLJH1_XXuQCLcBGAsYHQ/w640-h178/008.png)](https://1.bp.blogspot.com/-9mBcJwANHls/X9ulet6mFUI/AAAAAAAACNU/sIAxNMz0fxMbggAenggd0xZQLJH1_XXuQCLcBGAsYHQ/s1588/008.png)

Final lighting buffer (for analytic lights) - diffuse and specular contributions.

  
**Lighting part 2: Shadows**

But what we just saw does not mean that shadows are unsophisticated in Cyberpunk 2077, quite the contrary, there are definitely a number of tricks that have been employed, most of them not at all easy to reverse!

First of all, before the depth-prepass, there are always a bunch of draws into what looks like a shadowmap. I suspect this is a CSM, but in the capture I have looked at, I have never seen it used, only rendered into. This points to a system that updates shadowmaps over many frames, likely with only static objects?

[![](https://lh3.googleusercontent.com/-cbhGJg48_Zo/X9sO0otQApI/AAAAAAAACHU/6qqPjiQznwYXo-RU2euIsWlhjcVKcwLPQCLcBGAsYHQ/w640-h306/009.png)](https://lh3.googleusercontent.com/-cbhGJg48_Zo/X9sO0otQApI/AAAAAAAACHU/6qqPjiQznwYXo-RU2euIsWlhjcVKcwLPQCLcBGAsYHQ/009.png)

Is this a shadowmap? Note that there are only a few events in this capture that write to it, none that reads - it's just used as a depth-stencil target, if RenderDoc is correct here...

These multi-frame effects are complicated to capture, so I can't say if there are further caching systems (e.g. see the quadtree compressed shadows of Black Ops 3) at play. 

One thing that looks interesting is that if you travel fast enough through a level (e.g. in a car) you can see that the shadows take some time to "catch up" and they fade in incrementally in a peculiar fashion. It almost appears like there is a depth offset applied from the sun point of view, that over time gets reduced. Interesting!

[![](https://1.bp.blogspot.com/-RNRoYSAGoAI/X9ulnO7qUDI/AAAAAAAACNc/_Ge0dypU3UwdHN2srCXN4smNRN_j1m9oQCLcBGAsYHQ/w640-h224/010.png)](https://1.bp.blogspot.com/-RNRoYSAGoAI/X9ulnO7qUDI/AAAAAAAACNc/_Ge0dypU3UwdHN2srCXN4smNRN_j1m9oQCLcBGAsYHQ/s1726/010.png)

This is hard to capture in an image, but note how the shadow in time seems to crawl "up" towards the sun.

  

Sun shadows are pre-resolved into a screen-space buffer prior to the lighting compute pass, I guess to simplify compute shaders and achieve higher occupancy. This buffer is generated in a pass that binds quite a few textures, two of which look CSM-ish. One is clearly a CSM, with in my case five entries in a texture array, where slices 0 to 3 are different cascades, but the last slice appears to be the same cascade as slice 0 but from a slightly different perspective. 

There's surely a lot to reverse-engineer here if one was inclined to do the work!

[![](https://1.bp.blogspot.com/-38tlP29gG0M/X9ulrq40zQI/AAAAAAAACNg/QRyZf5ma0RoO0jIryhm_sfn02fSVFoHhQCLcBGAsYHQ/w640-h252/Capture.PNG)](https://1.bp.blogspot.com/-38tlP29gG0M/X9ulrq40zQI/AAAAAAAACNg/QRyZf5ma0RoO0jIryhm_sfn02fSVFoHhQCLcBGAsYHQ/s1499/Capture.PNG)

The slices of the texture on the bottom (in red) are clearly CSM. The partially rendered slices in gray are a mystery. The yellow/green texture is, clearly, resolved screen-space sun shadows, I've never, so far, seen the green channel used in a capture.

  
All other shadows in the scene are some form of VSMs, computed again incrementally over time. I've seen 512x512 and 256x256 used, and in my captures, I can see five shadowmaps rendered per frame, but I'm guessing this depends on settings. Most of these seem only bound as render targets, so again it might be that it takes multiple frames to finish rendering them. One gets blurred (VSM) into a slice of a texture array - I've seen some with 10 slices and others with 20.

[![](https://lh3.googleusercontent.com/-HIs4TfNLeAY/X9sPfvI0-xI/AAAAAAAACHw/o-mxU7OOhgsfOA-dhNCIZjOduJV9WppGQCLcBGAsYHQ/w640-h212/Capture1.PNG)](https://lh3.googleusercontent.com/-HIs4TfNLeAY/X9sPfvI0-xI/AAAAAAAACHw/o-mxU7OOhgsfOA-dhNCIZjOduJV9WppGQCLcBGAsYHQ/Capture1.PNG)

A few of the VSM-ish shadowmaps on the left, and artefacts of the screen-space raymarched contact shadows on the right, e.g. under the left arm, the scissors and other objects in contact with the plane...

  
Finally, we have what the game settings call "contact shadows" - which are screen-space, short-range raymarched shadows. These seem to be computed by the lighting compute shaders themselves, which would make sense as these know about lights and their directions...

Overall, shadows are both simple and complex. The setup, with CSMs, VSMs, and optionally raymarching is not overly surprising, but I'm sure the devil is in the detail of how all these are generated and faded in. It's rare to see obvious artifacts, so the entire system has to be praised, especially in an open-world game!

**Lighting part III: All the rest...**

Since booting the game for the first time I had the distinct sense that most lighting is actually not in the form of analytic lights - and indeed looking at the captures this seems to not be unfounded. At the same time, there are no lightmaps, and I doubt there's anything pre-baked at all. This is perhaps one of the most fascinating parts of the rendering.

[![](https://1.bp.blogspot.com/-bbC6CTELyWg/X9ulxSo8mUI/AAAAAAAACNo/Q1Qqth8zrNcZkQHZ699GUtENhQoK78dQwCLcBGAsYHQ/w640-h232/011.png)](https://1.bp.blogspot.com/-bbC6CTELyWg/X9ulxSo8mUI/AAAAAAAACNo/Q1Qqth8zrNcZkQHZ699GUtENhQoK78dQwCLcBGAsYHQ/s1652/011.png)

First pass highlighted is the bent-cone AO for this frame, remaining passes do smoothing and temporal reprojection.

  

First of all, there is a very good half-res SSAO pass. This is computed right after the uber-depth-summarization pass mentioned before, and it uses the packed RGBA8 normal-roughness instead of the g-buffer one. 

It looks like it's computing bent normals and aperture cones - impossible to tell the exact technique, but it's definitely doing a great job, probably something along the lines of HBAO-GTAO. First, depth, normal/roughness, and motion vectors are all downsampled to half-res. Then a pass computes current-frame AO, and subsequent ones do bilateral filtering and temporal reprojection. The dithering pattern is also quite regular if I had to guess, probably Jorge's Gradient noise?

It's easy to guess that the separate diffuse-specular emitted from the lighting pass is there to make it easier to occlude both more correctly with the cone information.

[![](https://lh3.googleusercontent.com/-kL-t2u0d_B8/X9sPl0DJMVI/AAAAAAAACH4/43dZuZS7yWI7oEddbuLWljHzDPlZOIDegCLcBGAsYHQ/w400-h362/012.png)](https://lh3.googleusercontent.com/-kL-t2u0d_B8/X9sPl0DJMVI/AAAAAAAACH4/43dZuZS7yWI7oEddbuLWljHzDPlZOIDegCLcBGAsYHQ/012.png)

One of many specular probes that get updated in an array texture, generating blurred mips.

  
Second, we have to look at indirect lighting. After the light clustering pass there are a bunch of draws that update a texture array of what appear to be spherically (or dual paraboloid?) unwrapped probes. Again, this is distributed across frames, not all slices of this array are updated per frame. It's not hard to see in captures that some part of the probe array gets updated with new probes, generating on the fly mipmaps, presumably GGX-prefiltered. 

[![](https://lh3.googleusercontent.com/-uGNwRe4lM4g/X9sPoe8-cqI/AAAAAAAACH8/INPEcs7WMWoo7uuXs98yWUiRtIlEd6yagCLcBGAsYHQ/w400-h175/013.png)](https://lh3.googleusercontent.com/-uGNwRe4lM4g/X9sPoe8-cqI/AAAAAAAACH8/INPEcs7WMWoo7uuXs98yWUiRtIlEd6yagCLcBGAsYHQ/013.png)

A mysterious cubemap. It looks like it's compositing sky (I guess that dynamically updates with time of day) with some geometry. Is the red channel an extremely thing g-buffer?

The source of the probe data is harder to find though, but in the main capture I'm using there seems to be something that looks like a specular cubemap relighting happening, it's not obvious to me if this is a different probe from the ones in the array or the source for the array data later on. 

Also, it's hard to say whether or not these probes are hand placed in the level, if the relighting assumption is true, then I'd imagine that the locations are fixed, and perhaps artist placed volumes or planes to define the influence area of each probe / avoid leaks.

[![](https://lh3.googleusercontent.com/-REAm7lCte88/X9sPqlfNwsI/AAAAAAAACIA/lAZTrqsZh6Ut4O3PShY9CHDINl7COxcXwCLcBGAsYHQ/w640-h238/014.png)](https://lh3.googleusercontent.com/-REAm7lCte88/X9sPqlfNwsI/AAAAAAAACIA/lAZTrqsZh6Ut4O3PShY9CHDINl7COxcXwCLcBGAsYHQ/014.png)

A slice of the volumetric lighting texture, and some disocclusion artefacts and leaks in a couple of frames.

  
We have your "standard" volumetric lighting, computed in a 3d texture, with both temporal reprojection. The raymarching is clamped using the scene depth, presumably to save performance, but this, in turn, can lead to leaks and reprojection artifacts at times. Not too evident though in most cases.

[![](https://1.bp.blogspot.com/-1-jMMhWcUi8/X9ul26ng4SI/AAAAAAAACNw/1cxKd1Rvn1k4fJaVwrOkwC6f7sh_hQUSwCLcBGAsYHQ/w640-h360/015.png)](https://1.bp.blogspot.com/-1-jMMhWcUi8/X9ul26ng4SI/AAAAAAAACNw/1cxKd1Rvn1k4fJaVwrOkwC6f7sh_hQUSwCLcBGAsYHQ/s1378/015.png)

Screen-Space Reflections

Now, things get very interesting again. First, we have an is an amazing Screen-Space Reflection pass, which again uses the packed normal/roughness buffer and thus supports blurry reflections, and at least at my rendering settings, is done at full resolution. 

It uses previous-frame color data, before UI compositing for the reflection (using motion vectors to reproject). And it's quite a lot of noise, even if it employs a blue-noise texture for dithering!

[![](https://lh3.googleusercontent.com/-DhE56d2JeXg/X9sPvWQpVKI/AAAAAAAACIM/ptQVfB6_3ykLsQrfnMqjblvhj9xmVYvtACLcBGAsYHQ/w640-h346/016.png)](https://lh3.googleusercontent.com/-DhE56d2JeXg/X9sPvWQpVKI/AAAAAAAACIM/ptQVfB6_3ykLsQrfnMqjblvhj9xmVYvtACLcBGAsYHQ/016.png)

Diffuse/Ambient GI, reading a volumetric cube, which is not easy to decode...

  
Then, a indirect diffuse/ambient GI. Binds the g-buffer and a bunch of 64x64x64 volume textures that are hard to decode. From the inputs and outputs one can guess the volume is centered around the camera and contains indices to some sort of computed irradiance, maybe spherical harmonics or such. 

The lighting is very soft/low-frequency and indirect shadows are not really visible in this pass. This might even by dynamic GI!

Certainly is volumetric, which has the advantage of being "uniform" across all objects, moving or not, and this coherence shows in the final game.

[![](https://1.bp.blogspot.com/-yh1PjDZP01I/X9ul7882KTI/AAAAAAAACN4/Hm4NIeWdIekrhtU8qZTgAvmwKqOkwog7wCLcBGAsYHQ/w640-h180/017.png)](https://1.bp.blogspot.com/-yh1PjDZP01I/X9ul7882KTI/AAAAAAAACN4/Hm4NIeWdIekrhtU8qZTgAvmwKqOkwog7wCLcBGAsYHQ/s1708/017.png)

Final lighting composite, diffuse plus specular, and specular-only.

  

And finally, everything gets composited together: specular probes, SSR, SSAO, diffuse GI, analytic lighting. This pass emits again two buffers, one which seems to be final lighting, and a second with what appears to be only the specular parts.

And here is where we can see what I said at the beginning. Most lighting is not from analytic lights! We don't see the usual tricks of the trade, with a lot of "fill" lights added by artists (albeit the light design is definitely very careful), instead indirect lighting is what makes most of the scene. This indirect lighting is not as "precise" as engines that rely more heavily on GI bakes and complicated encodings, but it is very uniform and regains high-frequency effects via the two very high-quality screen-space passes, the AO and reflection ones.

[![](https://1.bp.blogspot.com/-bdgbmkY6UIM/X9umBi5aKBI/AAAAAAAACN8/FwHp6pBmjLEI1E3NdCuFNoL21iHWl5I3gCLcBGAsYHQ/w640-h238/Capture2.PNG)](https://1.bp.blogspot.com/-bdgbmkY6UIM/X9umBi5aKBI/AAAAAAAACN8/FwHp6pBmjLEI1E3NdCuFNoL21iHWl5I3gCLcBGAsYHQ/s1519/Capture2.PNG)

  

[![](https://1.bp.blogspot.com/-38jST1klN0s/X9umGWHCiXI/AAAAAAAACOA/W_A1KdeUCtUTOiCDEpBCaxu7gbWZWW-2ACLcBGAsYHQ/w640-h236/Capture3.PNG)](https://1.bp.blogspot.com/-38jST1klN0s/X9umGWHCiXI/AAAAAAAACOA/W_A1KdeUCtUTOiCDEpBCaxu7gbWZWW-2ACLcBGAsYHQ/s1522/Capture3.PNG)

The screen-space passes are quite noisy, which in turn makes temporal reprojection really fundamental, and this is another extremely interesting aspect of this engine. Traditional wisdom says that reprojection does not work in games that have lots of transparent surfaces. The sci-fi worlds of Cyberpunk definitely qualify for this, but the engineers here did not get the news and made things work anyway!

And yes, sometimes it's possible to see reprojection artifact, and the entire shading can have a bit of "swimming" in motion, but in general, it's solid and coherent, qualities that even many engines using lightmaps cannot claim to have. Light leaks are not common, silhouettes are usually well shaded, properly occluded.

**All the rest**

There are lots of other effects in the engine we won't cover - for brevity and to keep my sanity. Hair is very interesting, appearing to render multiple depth slices and inject itself partially in the g-buffer with some pre-lighting and weird normal (fake anisotropic?) effect. Translucency/skin shading is surely another important effect I won't dissect.

[![](https://1.bp.blogspot.com/-EuTVj_sfL5E/X9umMNhR2_I/AAAAAAAACOM/wPmGc_QtDB4KChTGl_kWmh9W9WLLUQxyACLcBGAsYHQ/w640-h244/018.png)](https://1.bp.blogspot.com/-EuTVj_sfL5E/X9umMNhR2_I/AAAAAAAACOM/wPmGc_QtDB4KChTGl_kWmh9W9WLLUQxyACLcBGAsYHQ/s1302/018.png)

Looks like charts caching lighting...

Before the frame is over though, we have to mention transparencies - as more magic is going on here for sure. First, there is a pass that seems to compute a light chart, I think for all transparencies, not just particles.

Glass can blur whatever is behind them, and this is done with a specialized pass, first rendering transparent geometry in a buffer that accumulates the blur amount, then a series of compute shaders end up creating three mips of the screen, and finally everything is composited back in the scene.

[![](https://1.bp.blogspot.com/-hlgY9S_NmkA/X9umRhzIEmI/AAAAAAAACOU/zpr4fBrKMQ872Yxck3R2k_OSagfjRnPaQCLcBGAsYHQ/w640-h277/Capture4.PNG)](https://1.bp.blogspot.com/-hlgY9S_NmkA/X9umRhzIEmI/AAAAAAAACOU/zpr4fBrKMQ872Yxck3R2k_OSagfjRnPaQCLcBGAsYHQ/s1431/Capture4.PNG)

  
After the "glass blur", transparencies are rendered again, together with particles, using the lighting information computed in the chart. At least at my rendering settings, everything here is done at full resolution.

[![](https://1.bp.blogspot.com/-kYuFo70VMp8/X9umXTlWppI/AAAAAAAACOc/zKU17uWhNswP-etH5929wAhpE1RWyFrlgCLcBGAsYHQ/w640-h390/Capture5.PNG)](https://1.bp.blogspot.com/-kYuFo70VMp8/X9umXTlWppI/AAAAAAAACOc/zKU17uWhNswP-etH5929wAhpE1RWyFrlgCLcBGAsYHQ/s1052/Capture5.PNG)

Scene after glass blur (in the inset) and with the actual glass rendered on top (big image)

  
Finally, the all-mighty temporal reprojection. I would really like to see the game without this, the difference before and after the temporal reprojection is quite amazing. There is some sort of dilated mask magic going on, but to be honest, I can't see anything too bizarre going on, it's astonishing how well it works. 

Perhaps there are some very complicated secret recipes lurking somewhere in the shaders or beyond my ability to understand the capture.

[![](https://1.bp.blogspot.com/-gO2R-e5TmHs/X9umb0DXKFI/AAAAAAAACOk/1h6cMKV0PJcwb1Msz7rd_l-Z-pJoh8XgACLcBGAsYHQ/w640-h266/Capture6.PNG)](https://1.bp.blogspot.com/-gO2R-e5TmHs/X9umb0DXKFI/AAAAAAAACOk/1h6cMKV0PJcwb1Msz7rd_l-Z-pJoh8XgACLcBGAsYHQ/s1499/Capture6.PNG)

On the left, current and previous frame, on the right, final image after temporal reprojection.

[![](https://lh3.googleusercontent.com/-rrvMn-e22vo/X9sSMk8zvcI/AAAAAAAACJ0/7YSc9F--T242NeQs1tL6WvS-XoMziocYgCLcBGAsYHQ/w400-h226/Capture7.PNG)](https://lh3.googleusercontent.com/-rrvMn-e22vo/X9sSMk8zvcI/AAAAAAAACJ0/7YSc9F--T242NeQs1tL6WvS-XoMziocYgCLcBGAsYHQ/Capture7.PNG)

This is from a different frame, a mask that is used for the TAA pass later on...

I wrote "finally" because I won't look further, i.e. the details of the post-effect stack, things here are not too surprising. Bloom is a big part of it, of course, almost adding another layer of indirect lighting, and it's top-notch as expected, stable, and wide. 

Depth of field, of course, tone-mapping and auto-exposure... There are of course all the image-degradation fixings you'd expect and probably want to disable: film grain, lens flares, motion blur, chromatic aberration... Even the UI compositing is non-trivial, all done in compute, but who has the time... Now that I got all this off my chest, I can finally try to go and enjoy the game! Bye!