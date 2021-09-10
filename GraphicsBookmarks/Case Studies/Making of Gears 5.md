# The making of Gears 5: how the Coalition hit 60fps - and improved visual quality

## A deep dive tech interview with Digital Foundry.

By [Alex Battaglia](https://www.eurogamer.net/authors/1798). 14/09/2019

The importance of the triple-A first-party blockbuster can't be understated - a spectacular amount of time, money and effort is concentrated into creating experiences that push console hardware to its limits. And in turn, the technological innovations found in these titles are often shared with the development community, improving the technical quality of titles across the board. It's a rising tide that floats all boats, if you like. So it is with Gears 5 from the Coalition - an outstanding release that redefines expectations from Xbox One hardware, while at the same time delivering what we (and many others!) believe is one of the greatest PC ports of the generation.

In this deep dive technical interview with The Coalition's technical art director Colin Penty and studio technical director Mike Rayner, we discuss how the team further iterated and customised key features of the Unreal Engine 4 technology while at the same time integrating the latest and greatest innovations from Epic back into their game. We also discover how the studio's intent in doubling frame-rate from 30fps to 60fps brought about a range of changes that didn't compromise visual quality as such, but rather _improved_ it over Gears of War 4.

ARTICLE CONTINUES BELOW

And then there's the discussion on the importance of native rendering resolution up against dynamic resolution and image reconstruction techniques. Alongside the kind of work we're seeing in titles like the upcoming Call of Duty: Modern Warfare from Infinity Ward, The Coalition implements a flexible approach to rendering resolution to achieve 60 frames per second, evolving and improving from the work carried out in Gears of War 4. But how is this done - and are we nearing the end of the pixel-counting era in terms of using a set figure (or range of figures) to get some idea of overall image quality?

We're entering an exciting time: the end of one console generation and the transition to the next. These end-or-era first-party titles - whether they're from Sony or Microsoft - not only show today's technology pushed to its limits, but also help to set the stage for the generation to come. So grab a coffee, sit down and prepare yourself for a unique insight into the development of one of Xbox One's crowning technological achievements.

![Loading video](https://i.ytimg.com/vi/9BiQaRO2oco/sddefault.jpg#404_is_fine)

**Digital Foundry:** Gears of War 4 was a premiere title utilising many of the high-end rendering features of Unreal Engine 4 at its time of release. UE4 has since evolved during Gears 5's development adding in a volumetric fog, distance field shadows/GI, a faster depth of field, parallax occlusion mapping, greater GPU tessellation support, etc. What newer UE4 features has The Coalition integrated for Gears 5 and where are they best visible?

ARTICLE CONTINUES BELOW

**The Coalition:** Gears 5 is utilising a number of UE4's latest rendering features which gives us access to technology that has come online since Gears of War 4 such as temporal upscaling, volumetric fog, distance fields, compute ray traced shadows, and bent normals for specular occlusion.

We use compute ray traced shadows in the game beyond our cascade shadow maps to allow us to have fully dynamic shadows in our game which helps with our LOD quality, texture memory usage, and disk space, while also of course allowing moveable shadows. The ray traced shadows utilises the distance field representation of each static mesh.

All of our primary characters have bent normal and AO maps to allow them to light in a very 'airtight' way and have no light leaks. It really helps for a third person game with a character using metal armour (shiny)! We use volumetric fog throughout our campaign, versus maps - and our Escape mode leverages it with the 'venom' effect. The new and improved depth of field is used throughout all of our cinematics and front end. It looks superior to the DOF that Gears of War 4 used on PC's insane quality and is 4x faster allowing us to run it on Xbox.

Alongside effects and techniques packaged with UE4, Gears of War 4 notably had many of its bespoke effects - like character shadow capsules, its own volumetric lighting solution, screen-space shadows, virtual point lights from reflective shadow maps for dynamic GI from select spot lights/flashlights, etc. What are some examples of techniques/effects added in by The Coalition now for Gears 5 and which ones have returned? With the move to real-time cascades for Gears 5 (Gears of War 4 used per-object shadows on Xbox One S), we wanted to reduce our reliance on screen-space shadows and use cascades wherever we could.

We made a big push for GPU based particles on Gears 5, so we integrated some GPU spawning technology from Rare which helped ease the particle burden on the CPU and helped us achieve 60fps on Xbox One X. Our Escape venom effect was an amazingly complex effect that flows through a procedural generated map in Escape at 60fps on Xbox One. We developed a custom system that allowed us to dynamically move the volume fog throughout the map, but also had to develop a new particle system that allowed us to run thousands of 'venom' particles in the air and maintain performance. So, we developed a new particle system using vertex shaders that incurred zero CPU performance overhead called Swift Particles which we not only used for Venom but used for all of our environmental VFX.

We further leverage the Swift Particles idea and created a new system called Swift Destruction which allows us to destroy objects using a vertex shader. We had Houdini Engine tools we would run in Maya that would pre-fracture a mesh - we would have to watch the amount of fracture points we had on an object - if we went too high we would really hit our GPU. Overall though this would create very dense destruction moments that would incur no CPU performance overhead.

We developed a volume fog painting system that would allow us to 'paint' volume fog throughout our levels - this allowed us to bypass the standard volume fog injection cost you get on the GPU per light source/material - we would only pay an injection cost once. We also would dynamically generate a height map for all of our levels that we would feed into the volume fog painting system, allowing us to dial in or out the amount of ground fog. Sometimes we would use this height map for fake GI as well in our shaders.

ARTICLE CONTINUES BELOW

We wrote a custom compute tessellation shader which allowed us to generate very high frequency tessellated surfaces and then run the tessellation async to help absorb the cost at 60fps. We chose to not do parallax occlusion mapping due to high cost, and instead implemented relaxed cone step mapping (GPU Gems 3) which actually produces better quality than POM (no stepping) and runs much faster. It also slotted in well with our custom material masking system (MMS for short - presented at Siggraph 2017) which has to bake out textures anyway per-material so we would also pre-compute the cone step map as well.

We retained our custom geocache system, originally developed for Halo 5 and also used in Gears of War 4. This technology allows us to create massive complex offline destruction simulations in tools like Houdini and then play them back in real-time on the GPU. You will see this in use in a number of the real-time large-scale set-pieces destruction moments in the game. We enhanced and improved our material masking system from Gears of War 4 adding in much more aggressive texture packing and shader optimisations which in-turn allowed us to create more complex layered materials with some headroom for tessellation or Cone Step Mapping.

![Loading video](https://i.ytimg.com/vi/TaI2n6KIB-4/sddefault.jpg#404_is_fine)

**Digital Foundry:** Gears of War 4 had excellent character models with realistic skin, hair (Delmont Walker particularly), and animation deformation - Gears 5 looks noticeably improved in this area particularly in regards to the faces themselves. What technical/artistic changes are reflected here?

**The Coalition:** We made a huge up-front investment in our characters for Gears 5 and we are really happy with the results. It was a combination of asset improvements and new technology that really helped up make the noticeable leap. First off, we decided to increase our hero character triangle counts by 50 per cent over Gears of War 4, then went about cleaning up all of our face-shapes, wrinkles, and blood flow regions making them much higher quality in the process. We then went through and improved our skin shader by adding dual lobe specular and improving the backscattering component.

We also greatly enhanced our eye shader with dynamic iris caustics, proper sub-surface scattering, AO maps, and other micro-geometry additions and we switched over to using Faceware for our facial animation capture on Gears 5 - capturing our actors' facial animation, not just on the mo-cap stage but also in the audio booth. This, of course, was then hand tuned by an animator to bring the performance to life. We improved our Hair (including Del's!) by improving the artist tools for authoring, material tuning, and baking out hair AO maps. The core hair shader is unchanged from Gears of War 4.

**Digital Foundry:** With many more biomes, weather types and locations, what technical changes or art pipeline changes had to be made? Watching trailers and looking at the PC benchmark, snow/sand deformation looks present and foliage looks rather dynamic.

**The Coalition:** The real content challenges on Gears 5 has been scale and variety, for sure. Our large overworld maps have a level of scale in them we never had to deal with before in Gears of War - this forced us to generate new workflows and tech to deal with the size (fully dynamic shadows is an example). Having snow and sand in this game was challenging from a content creation point of view, forcing us to create high quality sand and snow materials but also deform them dynamically.

ARTICLE CONTINUES BELOW

We created a custom deformation system that would allow us to deform large areas of the map and not have the deformation disappear for a long time. We would run a tessellation shader on this deformation allowing there to be enough resolution around the player's feet so it looks realistic, as well as do some material blending at the deformation point. Far in the distance we cannot afford to run tessellation on those triangles, so we dynamically generate a normal map in the material and blend it in based on distance.

For things like water ripples and vegetation displacing around the character we were inspired by Uncharted 4's approach to this - so we implemented a similar approach of spawning particles into a buffer which allowed us to influence the materials dynamically. This works best for temporary effects.

Xbox One X uses the equivalent of a range of medium and high presets, with screen-space reflection quality lower than medium.

![Insane SSR](https://d2skuhm0vrry40.cloudfront.net/2019/articles/2019-09-14-11-54/GR_1_000.png/EG11/thumbnail/688x387/format/jpg/quality/80)

Insane SSR

![Ultra SSR](https://d2skuhm0vrry40.cloudfront.net/2019/articles/2019-09-14-11-56/GR_2_000.png/EG11/thumbnail/688x387/format/jpg/quality/80)

Ultra SSR

Screen-space reflections can have a dramatic impact on the games visuals.

![Insane SSR](https://d2skuhm0vrry40.cloudfront.net/2019/articles/2019-09-14-11-55/GR_1_001.png/EG11/thumbnail/688x387/format/jpg/quality/80)

Insane SSR

![Xbox One X](https://d2skuhm0vrry40.cloudfront.net/2019/articles/2019-09-14-11-56/GR_2_001.png/EG11/thumbnail/688x387/format/jpg/quality/80)

Xbox One X

The PC version includes a 4K texture pack, dramatically increasing texture quality over Xbox One X.

![Insane Texture Pack](https://d2skuhm0vrry40.cloudfront.net/2019/articles/2019-09-14-11-55/GR_1_002.png/EG11/thumbnail/688x387/format/jpg/quality/80)

Insane Texture Pack

![Xbox One X](https://d2skuhm0vrry40.cloudfront.net/2019/articles/2019-09-14-11-56/GR_2_002.png/EG11/thumbnail/688x387/format/jpg/quality/80)

Xbox One X

Less performance intensive settings, such as cone-step mapping, are equivalent to PC's high on Xbox One X.

![Ultra Cone Step Mapping](https://d2skuhm0vrry40.cloudfront.net/2019/articles/2019-09-14-11-55/GR_1_003.png/EG11/thumbnail/688x387/format/jpg/quality/80)

Ultra Cone Step Mapping

![High Cone Step Mapping](https://d2skuhm0vrry40.cloudfront.net/2019/articles/2019-09-14-11-56/GR_2_003.png/EG11/thumbnail/688x387/format/jpg/quality/80)

High Cone Step Mapping

Volumetric quality on PC is the most scalable setting for performance with insane delivering extreme quality at a cost to match (PC accidentally with vegetation detail set to off, hence no icicles).

![Insane Volumetric Lighting](https://d2skuhm0vrry40.cloudfront.net/2019/articles/2019-09-14-11-55/GR_1_004.png/EG11/thumbnail/688x387/format/jpg/quality/80)

Insane Volumetric Lighting

![Ultra Volumetric Lighting](https://d2skuhm0vrry40.cloudfront.net/2019/articles/2019-09-14-11-57/GR_2_004.png/EG11/thumbnail/688x387/format/jpg/quality/80)

Ultra Volumetric Lighting

Shadows on PC are also very scalable with high offering great performance and visuals.

![Insane Dynamic Shadows](https://d2skuhm0vrry40.cloudfront.net/2019/articles/2019-09-14-11-55/GR_1_005.png/EG11/thumbnail/688x387/format/jpg/quality/80)

Insane Dynamic Shadows

![Ultra Dynamic Shadows](https://d2skuhm0vrry40.cloudfront.net/2019/articles/2019-09-14-11-57/GR_2_005.png/EG11/thumbnail/688x387/format/jpg/quality/80)

Ultra Dynamic Shadows

**Digital Foundry:** Gears 5 does not adhere to a strict realism in its graphical presentation, having fantastical materials and proportions, does that alter the authoring of assets/textures and materials in comparison to a more photorealistic presentation?

**The Coalition:** It does to a certain extent. Our materials tend to be mostly grounded in reality but our objects are not. So we can't rely on photogrammetry to generate assets for us like props - we tend to concept or gather some rough example assets then 'Gearsify' them. This usually involves making them chunkier, dirtier, worn, with metal trim pieces - though there's an art bible with much more nuanced definition than that.

For our materials we wanted our objects to feel more believable than Gears of War 4, so we moved the content teams to a shared material library for the first time - so this way we didn't get 25 permutations of 'steel' in the database and instead we would just mask and layer our pre-built steel material to generate our assets. This allowed much more consistent and high quality assets, especially when you consider going to scale with our outsource vendors. We found the up-front creation time of an asset increased slightly but the amount of asset iterations we went through decreased dramatically, saving us time.

We also leaned into detail maps on our materials much more on Gears 5 - knowing that this game would be an Xbox One X title running at 4k we knew we had to increase our texel density from Gears of War 4 to hold-up and I'm really happy with the results.

**Digital Foundry:**Alongside the custom layered material model which bakes layers together for performance reasons as used in Gears of War 4, there appears to be a dynamic layer system of sorts now with blood, mud, and other materials dynamically coating character models. How does this exactly work?

ARTICLE CONTINUES BELOW

**The Coalition:** We created a slot system for our character materials that can do various things. We have a single 'dynamic' slot which allows for temporary material effects such as blood splatter or swarm goo. This was great in that we only have to pay for whatever temporary effect we are showing at the time and not all possible permutations at once. We have a wetness layer as well we activate for our rain levels. We also have additional slots for character power-ups in multiplayer as well as numerous other effects.

Overall, it's really tricky to balance new gameplay systems and character GPU performance. Every time a designer would add a new weapon or effect (eg an ice cannon that can freeze characters) the tech art team would have to discuss how to cheaply integrate that new effect, leveraging as much of the existing framework as possible.

![Loading video](https://i.ytimg.com/vi/BG-40AopHdc/sddefault.jpg#404_is_fine)

**Digital Foundry:** Dynamic resolution was key to Gears of War 4's performance on Xbox One previously and an amazing tool on PC - how has this changed in Gears 5? Is it taking advantage of temporal reconstruction? What are the set up differences between Xbox One base systems and Xbox One X?

**The Coalition:** Utilising Epic's temporal upscaling technology was a huge visual improvement from the standard upscaling that Gears of War 4 used. We render up to native 4K Xbox One X (1080p on Xbox One/S) resolution and dynamically scale based on the load on the GPU to ensure we remain locked at 60fps. We always output and render the post-process chain at native resolution (4K Xbox One X and 1080p on Xbox One/S) and only scale anything that happens before the post-process chain. It allows the frame to remain sharp even while resolution is scaling internally.

Philosophically, it's much harder to see resolution scaling on Xbox One X due to the higher resolution at play - so we tended to be a bit more sensitive to resolution scaling swings on Xbox One as that can be a bit more apparent. Ultimately, this technique enables us to increase the overall visual fidelity of the game. It allows us to maintain a higher visual baseline across the board and absorb scene-to-scene fluctuations and even extreme scenarios (eg a heavy VFX battle sequence) without dropping frames.

On PC, we now support temporal reconstruction. We have added a setting called minimum FPS. It allows the user to choose the minimum framerate they want the game to render at (30, 60, 90fps or none). If the game detects it is dropping below that framerate it will dynamically scale before the post-process chain and use temporal upscaling to reconstruct the final image at a higher quality than the dynamic resolution feature in Gears of War 4. The scaling range is determined by the game and based on the screen resolution.

**Digital Foundry:** Asynchronous compute was featured as an option on PC and used on Xbox One X and is key to extracting performance on certain GPU architectures, has this changed in Gears 5?

ARTICLE CONTINUES BELOW

**The Coalition:** On Gears of War 4, we ran only a few passes async such as reflection capture actors. We made a large investment on Gears 5 to run our entire post-process chain on async compute. This made a lot of sense given it is always running at native resolution due to the temporal upscale. This gave us a few more milliseconds to work with and achieve 60fps on Xbox One X, as well as in all MP modes on Xbox One S. There was a non-trivial amount of engineering effort to convert every post-process pass we used on Gears 5 to compute so we could then run it async. We also added a deferred present where the base pass of the next frame is started while the GPU completes the async post process pass of the current frame. These same asynchronous compute optimisations are also supported on PC.

![Loading video](https://i.ytimg.com/vi/A99kon29MbU/sddefault.jpg#404_is_fine)

**Digital Foundry:** Gears of War 4 offered some unique effects to the PC version with the insane quality option, like glossy screen-space reflections. What do the insane quality settings this time do on PC for the various effects they affect?

**The Coalition:** Gears of War 4 supported an insane setting for screen-space reflections and depth of field. On Gears 5, we've refactored depth of field to a higher quality and removed the insane setting in the process. We've retained insane screen-space reflections and added it to dynamic shadows and volumetric fog. Insane screen-space reflections are still glossy, but we've made some adjustments to make the setting scale better at 4K. Insane volumetric fog is one of our favourites which runs volume fog at such a high quality level it's indistinguishable from screen-space light shafts.

Our shadows are more scalable than Gears of War 4's pre-baked shadows, so the Insane shadow quality settings look amazing on Gears 5, with extremely high resolution cascades and spotlight shadows as well as high quality ray traced shadows in the distance. Most noticeable will be the extra detail seen in the distance at the insane setting, with additional cascades and extra distance on the spot lights.

**Digital Foundry:** DirectX 12 supports ray tracing on PC now through DXR and the Project Scarlett has announced hardware ray tracing support, UE4 has also added support for various ray traced effects. Where do you see the usage of ray tracing in Gears 5 or in future titles from The Coalition?

**The Coalition:** On Gears 5 we are using GPU compute-based ray traced distance field shadows which works across all platforms and does not depend on DXR or dedicated ray tracing hardware. We are excited with the possibilities of DXR and Project Scarlett's native hardware support for accelerated ray tracing. At this point, we are not ready to talk about our future plans, however we are following closely the emerging techniques and innovations in this exciting space.

**Digital Foundry:** With image reconstruction techniques becoming more prevalent, what is your opinion of native resolution rendering on systems with set hardware like in consoles?

**The Coalition:** We love running a game at native resolution and the simplicity and purity of that - but we feel like this will become less and less common, especially if we begin to see 8K games next generation. Temporal reconstruction techniques offer cheap performance returns allowing for higher quality visual systems with very hard to perceive visual quality loss - especially at higher resolutions. Put another way: pixel counting was a very useful thing a few years ago with games like Gears of War 4, as it would give you a clear indication of image clarity. With games that use reconstruction now I think pixel counts are an interesting thing to make note of but isn't necessarily that indicative of your final output clarity unless you are seeing some extremely lower-than-native resolutions.

One thing we didn't want to compromise on for Gears 5 was running the output at native resolution. We had all the controls to allow us to scale down from native output resolution if we wanted to for performance reasons but decided against it to maintain the highest possible quality frame 100 per cent of the time. Additionally, we always render the UI and a large number of processing effects at native resolution after the temporal upscale to ensure we have crisp UI and high quality per pixel effects like motion blur, bloom and tone-mapping.

![Loading video](https://i.ytimg.com/vi/t6al1JUYB_c/sddefault.jpg#404_is_fine)

**Digital Foundry:** What technical aspects of the visuals, or experiences in development from Gears 5, you are particularly proud of?

**The Coalition:** We are proud of all the systems and workflow improvements listed above, but overall we are most proud of how we were able to push visual quality noticeably ahead of Gears of War 4 while simultaneously pushing the game to 60fps on Xbox One X and 60fps for all MP modes on Xbox One. It was an immense collaborative effort between engineers, technical artists, and the art team to constantly keep 60fps at the forefront of everything we did while not compromising quality, and in-fact improving quality. We had to adopt many new workflows and techniques - especially around keeping the CPU running at 60fps.

Our CPU optimisation efforts carried over to our dedicated game servers, allowing us to double our server simulation rate from 30Hz to 60Hz. This results in end-to-end lower-latency networking for our multiplayer PvP (player versus player) game modes compared to Gears of War 4.

Targeting 60fps for campaign on Xbox One X turned out to be critical in allowing us to hit our split-screen performance targets. We had to be able to run our entire campaign on Xbox One with three-player split screen at 30fps, which is a non-trivial problem when you consider the burden on the draw thread to render 3x the amount of assets, not to mention the GPU overhead. Had we not targeted 60fps I'm not quite sure how we would have pulled off three-player split screen.

We have improved our HDR support in Gears 5 on Xbox One S/X and added support for HDR on Windows 10. Using a large dataset of pairs of HDR/SDR images from Microsoft's first-party shipped games, we used machine learning to train an inverse tone mapper for colour space conversion (Rec.2020) and optimal tuning of HDR output in Gears 5. Of course, we also have in-game calibration to let players tune the output for their particular monitor's abilities. You can tune against reference images as well as the real-time game.

With Gears 5, all of our cinematics (except for the Gears of War 4 recap and title sequence) are rendered natively in real-time. In the past these would be locked as 1080p videos. In Gears 5, the cinematics will scale to your system and run in 4K on Xbox One X, they will adapt to your PC video options quality settings providing continuity with the gameplay visuals. We feel that the overall visual quality of the real-time cinematics eclipses the off-line rendered cinematic videos we had on Gears of War 4. Our overall push for higher performance along with significant investments in character performance, visual effects and lighting resulted in tools our exceptionally talented cinematics team, animators, VFX, audio and lighting artists were able to use to push further and deliver something really special.

Ultimately seeing the positive reaction to the Technical Test visuals was a great validation of our approach that doubling frame rate doesn't have to impact visual quality if you plan towards that goal and have some great technology to go with it - especially with the power of Xbox One X.