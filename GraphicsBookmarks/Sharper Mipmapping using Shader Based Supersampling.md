# Sharper Mipmapping using Shader Based Supersampling

Mipmapping is ubiquitous in real time rendering, but has limitations. This article is going to go into what mipmapping is, how it’s used, and how it can be made better.

**_Warning: this page has about 72MB of Gifs! Medium also has a tendency to not load them properly, so if there’s a large gap or overly blurry image, try reloading the page. Ad blockers may cause problems, no script may make it better. It’s also broken in the official Medium app. Sorry, Medium is just weird._**

This article is Unity and MSAA / no-FSAA focused. If you’re intending to use some form of post transparency TAA, this article may not be for you.

# Inscrutable Text

Okay, story time! On one of the first VR projects I worked on, [Wayward Sky](https://www.uberent.com/waywardsky/), we ran into a curious problem. We had a sign in the game with a character’s name written across it. Not an unusual thing in itself. The problem was the name was almost completely illegible when playing the game.

![](https://miro.medium.com/max/1024/1*o1XRauTdrNObLXR4wPoQ7A.gif)

Bilinear Filtering (200% Pixel Scale)

The first guess by the artist was it was a texture resolution issue, but increasing that didn’t change anything. Moving the camera closer solved the problem, but that wasn’t an option for this particular case. The camera needed to be where it was and the rest of the area couldn’t be significantly changed either. But it proved the base texture resolution wasn’t the issue.

The second guess was it was the display resolution itself. This was an early PSVR title which has a lower panel resolution than the competition. We also weren’t rendering at a very high resolution scale compared to the recommended target. However when the artist disabled the mipmaps the text _was_ legible. It was mip mapping, and not the display or rendering resolution!

What was happening was the texture was on a surface just far enough away and slightly turned away from the camera such that it got dropped to too low of a mip level. The result was a blurred mess that didn’t even appear to be text.

Some of you probably figured out this was the problem in the first paragraph. I mean, it’s in the title of the article. Mipmapping causing blurring a common issue. It is magnified by resolution limited situations like VR, or even mobile and home consoles. Disabling mip mapping on the texture is one common “solution” you’ll find online for this problem. But this is a bad solution! Disabling mip mapping means the texture gets extremely aliased (especially bad for VR), and there are performance implications with having the full resolution texture rendering at all times. Ask anyone tasked with optimizing game performance about disabling mip mapping and they’ll have some “fun” stories to tell …

Ultimately we solved the issue with a small amount of mip LOD biasing, and forcing anisotropic filtering on for that texture. But while it was now clearer, it aliased a little.

![](https://miro.medium.com/freeze/max/60/1*f15AsFztAFHQt3dqzzV6NA.gif?q=20)

![](https://miro.medium.com/max/512/1*f15AsFztAFHQt3dqzzV6NA.gif)

Anisotropic 8x Bias -1.0 (200% Pixel Scale)

It was good enough for the project which was close to shipping. For a bit of text that only showed up for a few moments it wasn’t worth the time to investigate solutions more. But it still bugged me.

Cut to our next game, [Dino Frontier](https://www.uberent.com/dinofrontier/). This is a light sim game with a lot of in world UI, text, and icons. Sure enough the same issue of blurred images immediately appeared as we started implementing the UI and trying it out. The first solution the team came up with was to scale the UI up to much larger sizes to ensure text and icons remained legible. For some UI elements this meant making them so large as to be abusive to the player’s comfort.

I decided to take the time to solve the issue properly. And here it is:

![](https://miro.medium.com/freeze/max/60/1*LhPwE1oyD8EZYAnGr5eNOQ.gif?q=20)

![](https://miro.medium.com/max/512/1*LhPwE1oyD8EZYAnGr5eNOQ.gif)

Anisotropic 8x Bias -1.0 2x2 RGSS (200% Pixel Scale)

Don’t worry, I’ll explain all that stuff in the little text under the image.

# MIP Mapping

## Multum in Parvo

First a bit of discussion about what [mip mapping](https://en.wikipedia.org/wiki/Mipmap) is, and why it’s a good thing. It exists to solve the problem of texture aliasing during minification. In simpler terms, reduce flickering, jaggies, and complete loss of some image details when an image is displayed smaller than its original resolution.

A GPU renders a texture by sampling it once at the center of each screen pixel. Since the texture is only being sampled once at each pixel, when the texture is being shown at a higher resolution than there are screen pixels, some details won’t be “seen”. This results in parts of the image effectively disappearing.

Now it’s possible to figure out how many texels are within the bounds of each pixel, sample all of those, and then output the average color. This would be a form of Supersampling. Offline rendering tools often do this, and it yields extremely high quality results. But it is also _stupendously_ expensive. The below image which I’ll be using as a ground truth reference was generated using **_6400_** anisotropic samples per pixel.

![](https://miro.medium.com/freeze/max/54/1*PnAX0MGlBxf4oq2jbvzTWQ.gif?q=20)

![](https://miro.medium.com/max/512/1*PnAX0MGlBxf4oq2jbvzTWQ.gif)

Extreme Supersampling (200% Pixel Scale)

If rendered at 1920x1080 on an Nvidia GTX 970, this scene takes roughly 350 ms on the GPU (as measured with Nvidia Nsight), or about a third of a second per frame. And even this isn’t quite perfect. Really I’d need to use closer to 15,000 samples per pixel, but that brings the rendering time to multiple seconds per frame. That is the kind of quality that we’re aiming for, and it’s way too expensive for real time. So what if we skip that and just sample the full resolution texture once per pixel? It can’t be that bad, right?

![](https://miro.medium.com/freeze/max/54/1*uZXjZ_9bYO-blnI6S-gmsQ.gif?q=20)

![](https://miro.medium.com/max/512/1*uZXjZ_9bYO-blnI6S-gmsQ.gif)

No mipmaps, Bilinear Filtering (200% Pixel Scale)

Ouch.

Lines are popping in and out of existence in the foreground, and in the background it’s just a noisy mess. It would be even worse if the texture had more than just some white lines. This is what we mean when we say aliasing in the context of image sampling.

Mip mapping seeks to avoid both the problems of aliasing and the cost of Supersampling by prefiltering the image with multiple successively half resolution versions of the previous sized image. These are the mipmaps. Each image’s pixel is the average of the 4 pixels of the larger mip level. The idea is that as long as you pick the correct mip level, all of the original image details are accounted for. This successive half sizing is also why power of 2 texture sizes are a thing; it’s harder to make a mipmap for when a half size might not end up with a whole number value for the resolution.

![](https://miro.medium.com/freeze/max/58/1*_v8pT4KAL9qJwwAjuCnSLQ.gif?q=20)

![](https://miro.medium.com/max/600/1*_v8pT4KAL9qJwwAjuCnSLQ.gif)

The other benefit of mip mapping is memory bandwidth usage. While the overall texture memory usage is now roughly 33% larger due to the inclusion of the mip maps, when the GPU goes to read the texture, it only has to read the data from the appropriate mip level. If you’ve got a very large texture that’s being displayed very small, the GPU doesn’t have to read the full texture, only the smaller mip, reducing the amount of data being passed around. This is especially important on mobile devices where memory bandwidth is in very short supply. Technically a GPU never really loads the full resolution version of the texture at once out of its memory. Instead only a small chunk of it depending on what’s needed by the shader. The full resolution textures exists in the GPU’s main memory, and when a shader tries to sample from that texture the GPU pulls a small section of that texture into the L1 Cache for the TMU (Texture Mapping Unit, the physical part of the GPU that does the work of sampling textures) to read from. If multiple pixels all sample from a small region, then the GPU doesn’t have to fetch another part of the texture later and can reuse the one chunk already loaded. That’s what saves memory bandwidth.

So now, all a GPU has to do to prevent aliasing is pick the best mip level to display. It does this by calculating the expected texel to pixel ratio for both the horizontal and vertical screen axis at each pixel[¹](https://bgolus.medium.com/sharper-mipmapping-using-shader-based-supersampling-ed7aadb47bec#a030), and then using the largest ratio picks the closest mip level that would keep it as close to 1 texel to 1 pixel as possible.

The below image is a 256x256 texture with custom colored mipmaps being rendered at 256x256.

![](https://miro.medium.com/freeze/max/54/1*1dV7jIJHxOklsSb50SFCCA.gif?q=20)

![](https://miro.medium.com/max/512/1*1dV7jIJHxOklsSb50SFCCA.gif)

Colored Mipmaps, Bilinear Filtering (200% Pixel Scale)

The only time the full 256x256 top mip is sampled is when the quad is very close to the camera. Notice the floor is _never_ showing the top mip. If this was rendered at a higher resolution, or the texture was a lower resolution, the top mip would continue to be shown until further away. Again, this is due to that 1:1 texel to pixel ratio the mip level calculations are trying to achieve. Those game analysis videos online that complain about texture filtering quality being reduced when they lower the screen resolution or a game uses dynamic resolution … the filtering quality is _exactly_ the same, it’s just the resolution ratio that’s changing.

## Isotropic Filtering

Lets go over the basics of texture filtering. Really there’s two main kinds of texture filtering, point and linear. There’s also anisotropic, but we’ll come back to that. When sampling a texture you can tell the GPU what filter to use for the “MinMag” filter and the “Mip” filter.

The “MinMag” filter is how to handle blending between texels themselves. Point sampling chooses the closest texel to the position being sampled and returns that color. Linear finds the 4 closest texels and returns a bilinear interpolation of the four colors.

The “Mip” filter determines how to blend between mip levels. Point simply picks the closest mip level and uses that. Linear blends between the colors of the two closest mip levels.

Most people reading this are likely familiar with Point, Bilinear, and Trilinear filtering. Point filtering is using a point filter for both MinMag and Mip. Bilinear is using a linear filter for MinMag, and a point filter for Mip, as you probably guessed.

![](https://miro.medium.com/freeze/max/54/1*88BbPuG2NSyfOlFdjWICYg.gif?q=20)

![](https://miro.medium.com/max/512/1*88BbPuG2NSyfOlFdjWICYg.gif)

Bilinear Filtering (200% Pixel Scale)

As you can see, Bilinear shows clear jumps between the mip levels, both on the floor and as the quad moves forward and back. The point at which the changes happen is important, as the texture isn’t fully scaled down to the next mip size, but roughly 40% larger. This leads to the changes not only being abrupt, but for the texture to be obviously blurry when the change occurs.

Trilinear is the same linear filter for MinMag as Bilinear filtering with the addition of a linear filter for Mip, hence the name. This hides the harsh transitions between mip levels, but the blurring still remains as the next mip still starts being faded in early.

![](https://miro.medium.com/freeze/max/54/1*RirtKCMoqHDsh9vrx9FjZA.gif?q=20)

![](https://miro.medium.com/max/512/1*RirtKCMoqHDsh9vrx9FjZA.gif)

Trilinear Filtering (200% Pixel Scale)

This is the downside of mip mapping. Mip maps only accurately represent the image at exactly those half sizes. In between those perfectly scaled mip levels, the GPU has to pick which mip level to display, or blend between two levels. But remember, mip mapping’s goals are to reduce aliasing and rendering cost. It has technically achieved that, even if it’s at the expense of clarity.

The most obvious issue is how blurry the floor gets. This is because mipmaps are _isotropic_. In simple terms, each mipmap is scaled down uniformly in both the horizontal and vertical axis, and thus can only accurately reproduce a uniformly scaled surface. This works well enough for camera facing surfaces, like the rotating quad in the examples. But when viewing a surface that isn’t perfectly facing the camera, like the ground plane, one axis of the displayed texture is scaling down faster than the other in screen space. The ground plane has non-uniform, or _anisotropic_, scaling. Mipmaps alone don’t handle that case well. As I mentioned above, GPUs pick the mip level based on the worst case as the alternative would cause aliasing.

## Anisotropic Filtering

Anisotropic filtering exists to try to get around the blurry ground problem. Roughly speaking it works by using the mip level of the smaller texel to pixel ratio, then sampling the texture multiple times along the non-uniform scale’s orientation. The mip level used is still limited by the number of samples allowed, so low Anisotropic levels will still become blurry in the distance to prevent aliasing.

![](https://miro.medium.com/freeze/max/60/1*HMi0_WQIiNE1TwKyLxSUTA.gif?q=20)

![](https://miro.medium.com/max/512/1*HMi0_WQIiNE1TwKyLxSUTA.gif)

Anisotropic Filtering quality level comparison (200% Pixel Scale)

However the overall result is _much_ sharper ground textures even at lower settings. To my eyes, 4x or 8x are good options for improving the sharpness over a significant portion of the ground at this viewing angle and resolution.

![](https://miro.medium.com/freeze/max/54/1*-shl3Gi0Gd3TXtsUVeM5lg.gif?q=20)

![](https://miro.medium.com/max/512/1*-shl3Gi0Gd3TXtsUVeM5lg.gif)

Anisotropic Filtering 8x (200% Pixel Scale)

Looks pretty good, right? So I guess we’re done? Well, not so fast. Let's compare back to the “ground truth”.

![](https://miro.medium.com/freeze/max/54/1*PnAX0MGlBxf4oq2jbvzTWQ.gif?q=20)

![](https://miro.medium.com/max/512/1*PnAX0MGlBxf4oq2jbvzTWQ.gif)

“Ground Truth” Supersampling (200% Pixel Scale)

Notice how much sharper it is still? Especially the quad? No? Wait, I know it’s hard to compare those two when they’re not next to each other. How about this.

![](https://miro.medium.com/freeze/max/60/1*IKrVVgzgXwqaD5ml2z-sMQ.gif?q=20)

![](https://miro.medium.com/max/512/1*IKrVVgzgXwqaD5ml2z-sMQ.gif)

Anisotropic Filtering 8x vs “Ground Truth” (200% Pixel Scale)

Anisotropic filtering helps a lot with textures that are angled away from the camera, but directly facing is actually no different than Trilinear! Even the near ground plane isn’t quite right. Look closely at that center line and you’ll see there’s a little bit of blurring there still with Anisotropic filtering.[²](https://bgolus.medium.com/sharper-mipmapping-using-shader-based-supersampling-ed7aadb47bec#ce28)

So now what?

# Going Sharper-ish

## The Modest Proposal

So now we know that mip mapping can cause a texture to be displayed a little blurry, even with the best texture filtering enabled. The “obvious solution” is to disable mip mapping all together. We already saw how effective _that_ is. But this is often the “solution” artist are likely to end up with, mainly because it may be the only option the have immediately available to them. For small amounts of scaling, this is actually quite effective. The problem is of course past that it quickly goes south. It also usually results in some programmer shouting at said artist some time later when it comes time to do performance optimizations. I’ve shown this to you before near the start of the article as the “just sample the full resolution image” example. To remind you of what that looks like…

![](https://miro.medium.com/freeze/max/54/1*uZXjZ_9bYO-blnI6S-gmsQ.gif?q=20)

![](https://miro.medium.com/max/512/1*uZXjZ_9bYO-blnI6S-gmsQ.gif)

No mipmaps, aka “The Horror” (200% Pixel Scale)

So, again, don’t do this, unless you’re doing a pixel art game where you know for sure things aren’t going to be scaled, or are trying to capture the look of PS1 or PS2 game.

## Leveraging Conservative Bias

A better solution is to use mip _biasing_. Mip biasing tells the GPU to adjust what mip level to use one way or another. For example a mip bias of -0.5 pushes the mip changes slightly further away. Specifically it would move the transition back half way between the transition points it would use normally. A mip bias of -1 would push the mip level one full mip back, so where the GPU would originally be displaying mip level N, it’s now N-1. For some engines you can set the bias on the texture settings directly, and no modifications to shaders are needed. But direct texture asset mip biasing isn’t available on all platforms, for example it’s not supported on Apple devices running Metal, so custom shaders may be needed depending on your target platform(s). It can also be annoying to tweak as it’s not value exposed in the Unity editor by default. And the LOD Bias setting in Unreal Engine 4 isn’t the same thing. Luckily biasing in a shader is easy to add, and well supported across a wide range of hardware.

[https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-tex2dbias](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-tex2dbias)

half4 col = tex2Dbias(_MainTex, float4(i.uv.xy, 0.0, _Bias));

Alternatively you can get the same bias using `tex2Dgrad()` by multiplying the UV derivatives by 2^bias. This is useful for situations where you don’t have access to a `tex2Dbias()` function, like Unreal Engine 4’s node based materials.

[https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-tex2dgrad](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-tex2dgrad)

// per pixel screen space partial derivatives  
float2 dx = ddx(i.uv);  
float2 dy = ddy(i.uv);// bias scale  
float bias = pow(2, _Bias);half4 col = tex2Dgrad(_MainTex, i.uv.xy, dx * bias, dy * bias);

![](https://miro.medium.com/freeze/max/54/1*kKOTZeUvA-GMrSinfj_QCw.gif?q=20)

![](https://miro.medium.com/max/512/1*kKOTZeUvA-GMrSinfj_QCw.gif)

Anisotropic Filtering 8x Bias -0.5 (200% Pixel Scale)

![](https://miro.medium.com/freeze/max/54/1*4B2OGD3_DX38OXqicupbvw.gif?q=20)

![](https://miro.medium.com/max/512/1*4B2OGD3_DX38OXqicupbvw.gif)

Anisotropic Filtering 8x Bias -1.0 (200% Pixel Scale)

But biasing introduces some aliasing again. Notice the horizontal lines on the floor blinking and the diagonals looking like dashed lines. This is especially obvious with a bias of -1.0, so it’s not a perfect solution. As we push the mip levels to change later we’re missing entire texels again. That’s the whole reason why GPUs swap between mip levels when they do. Also this does increase memory bandwidth usage slightly, but significantly less than not using mip mapping at all. The overall impact on performance is likely minimal to non-existent if only used on problem textures.

I’ve shipped games using this technique, and it’s totally fine in a pinch. If your game is using some form of temporal anti-aliasing, this is actually recommended! To the best of my knowledge both Unreal Engine 4 and Unity’s new High Definition Render Pipeline do this!

# The Super Solution

So how do we increase the image clarity without introduce aliasing? Well, we might look to other anti-aliasing techniques. Specifically back at Super-Sample Anti-Aliasing (SSAA), and Multi-Sample Anti-Aliasing (MSAA).

## Supersampling

Supersampling for rendering means rendering or “sampling” at a higher resolution than the screen displays, then averaging those values. This step of averaging the values is called downsampling. Since we’re sampling an existing texture there’s no higher resolution rendering, just sampling the texture multiple times. As mentioned earlier, the “ground truth” example was rendered using Supersampling with a crazy high sample count.

But that downsampling is a big part of what mip mapping was created to avoid! Doing a downsample from 2x or even 3x the resolution isn’t too bad, but the number of samples required increases _quadratically_ as the texel to screen pixel ratio increases. A 256x256 texture that’s being displayed on screen at 16x16 pixels is a 16:1 ratio. Downsampling in a way that wouldn’t introduce aliasing would require at least _64_ bilinear samples of the texture (256 samples if point sampling)! This is why the ground truth example needed so many samples, because it had to handle that 256x256 texture getting down to 1 pixel. Luckily we’re not going to need that many samples, and GPUs have gotten really fast.

We can use mip biasing, anisotropic filtering, and Supersample the texture with only a few samples to remove the added aliasing introduced by the bias. This is the best of both world as we’re still using mip mapping and getting the benefits of its reduced aliasing through prefiltering and memory bandwidth usage. Plus if you sample one texture multiple times within only a small pixel range, it’s actually quite cheap. Like I mentioned above, GPUs load textures in chunks. So if a single pixel samples a texture multiple times in a small area, it might not need to load a new chunk making successive samples much faster. It’s not _free_, it’s just not as expensive as the initial texture sample alone.

So now we need to sample the texture multiple times with an offset on each sample and average the results. For this we’re going to use a simple 2x2 screen aligned grid, also called Ordered Grid Super-Sampling (OGSS). The offset needs to be small, ideally roughly within the bounds of the pixel we’re rendering. Luckily my old friend the per pixel screen space partial derivatives are back to rescue us! The derivatives of the texture UVs will give us both the offset magnitude and direction between one pixel and the next in the form of a vector.

// per pixel screen space partial derivatives  
float2 dx = ddx(i.uv.xy) * 0.25; // horizontal offset  
float2 dy = ddy(i.uv.xy) * 0.25; // vertical offset// supersampled 2x2 ordered grid  
half4 col = 0;  
col += tex2Dbias(_MainTex, float4(i.uv.xy + dx + dy, 0.0, _Bias));  
col += tex2Dbias(_MainTex, float4(i.uv.xy - dx + dy, 0.0, _Bias));  
col += tex2Dbias(_MainTex, float4(i.uv.xy + dx - dy, 0.0, _Bias));  
col += tex2Dbias(_MainTex, float4(i.uv.xy - dx - dy, 0.0, _Bias));  
col *= 0.25;

![](https://miro.medium.com/freeze/max/54/1*mn6P6PsiT3o6ozjzZzvi0g.gif?q=20)

![](https://miro.medium.com/max/512/1*mn6P6PsiT3o6ozjzZzvi0g.gif)

Anisotropic Filtering 8x Bias -1.0 2x2 OGSS (200% Pixel Scale)

The result is a slight increase in the shader cost. But the aliasing introduced by the mip biasing is greatly removed. How much cost? That will depend on the use case. For a similar scene as the above with the ground mirrored as a ceiling, running at 1920x1080 on an Nvidia GTX 970, it increases the overall frame time by about 0.15 ms to 0.5~0.6 ms per frame compared to anisotropic filtering alone. That’s a significant percentage increase due to how little is being rendered, but in a more complex scene it should only add a similar _total_ ms increase.

## Multi Sample Anti-Aliasing

MSAA, or more specifically 4x MSAA, has another trick. MSAA is similar to Supersampling in that it’s rendering at a higher resolution than the screen can necessarily display, but different in _what_ it renders at a higher resolution isn’t necessarily the scene color, but rather the scene depth. The difference is inconsequential for this topic, and I’ve gone into more detail in another post, [Anti-aliased Alpha Test](https://medium.com/@bgolus/anti-aliased-alpha-test-the-esoteric-alpha-to-coverage-8b177335ae4f), so we’ll skip that for now. What is important is 4x MSAA uses a Rotated Grid pattern, sometimes called 4 rooks.

![](https://miro.medium.com/max/60/1*7zkCcfpjwrJebUl48F21Ww.png?q=20)

![](https://miro.medium.com/max/306/1*7zkCcfpjwrJebUl48F21Ww.png)

from [A Quick Overview of MSAA](https://mynameismjp.wordpress.com/2012/10/24/msaa-overview/)

The benefit of this pattern is vertical and horizontal lines get 4 evenly spaced positions to be sampled on as they sweep through the pixel, increasing the granularity of anti-aliasing for those kinds of edges. This means higher quality lines for the type that are usually the most affected by aliasing.

// per pixel partial derivatives  
float2 dx = ddx(i.uv.xy);  
float2 dy = ddy(i.uv.xy);// rotated grid uv offsets  
float2 uvOffsets = float2(0.125, 0.375);  
float4 offsetUV = float4(0.0, 0.0, 0.0, _Bias);// supersampled using 2x2 rotated grid  
half4 col = 0;  
offsetUV.xy = i.uv.xy + uvOffsets.x * dx + uvOffsets.y * dy;  
col += tex2Dbias(_MainTex, offsetUV);  
offsetUV.xy = i.uv.xy - uvOffsets.x * dx - uvOffsets.y * dy;  
col += tex2Dbias(_MainTex, offsetUV);  
offsetUV.xy = i.uv.xy + uvOffsets.y * dx - uvOffsets.x * dy;  
col += tex2Dbias(_MainTex, offsetUV);  
offsetUV.xy = i.uv.xy - uvOffsets.y * dx + uvOffsets.x * dy;  
col += tex2Dbias(_MainTex, offsetUV);  
col *= 0.25;

![](https://miro.medium.com/freeze/max/54/1*wtxpNGvKqafKzuLg4XvgcQ.gif?q=20)

![](https://miro.medium.com/max/512/1*wtxpNGvKqafKzuLg4XvgcQ.gif)

Anisotropic Filtering 8x Bias -1.0 2x2 RGSS (200% Pixel Scale)

The difference in quality between the Ordered Grid and Rotated Grid Supersampling in this case is very minor. It can be seen in the minor amount of aliasing reduction on the ground plane’s horizontal lines, and slightly worse aliasing on a few diagonal lines. You can read up on the benefits and limitations of the two techniques here: [Super-sampling Anti-aliasing Analyzed](https://pdfs.semanticscholar.org/ebd9/ddb08c4244fc7df00672cacb420212cdde54.pdf)

Performance wise, the difference isn’t even measurable comparing OGSS and RGSS. So this is an essentially free upgrade over the previous technique.

But the main test is comparing 2x2 Rotated Grid Super-Sampling to the “Ground Truth” image from before.

![](https://miro.medium.com/freeze/max/60/1*Tzz9KbQ-gZVT6fcCZq5jBg.gif?q=20)

![](https://miro.medium.com/max/512/1*Tzz9KbQ-gZVT6fcCZq5jBg.gif)

2x2 RGSS vs “Ground Truth” (200% Pixel Scale)

Not exactly the same, but _extremely_ close. Certainly far better than Anisotropic filtering alone. The ground plane is also now slightly higher quality than even 16x Anisotropic filtering when only using 8x due to the addition of Supersampling.

# Closing Thoughts

And there we go! We’ve solved the problem! For a modest increase in shader cost we have near ground truth quality texture filtering! This means noticeably clearer text and images with little to no aliasing at significantly less performance impact than the ground truth.

![](https://miro.medium.com/freeze/max/60/1*LXI_fRDMXkHwvT49EURxmA.gif?q=20)

![](https://miro.medium.com/max/512/1*LXI_fRDMXkHwvT49EURxmA.gif)

Bilinear vs RGSS (200% Pixel Scale)