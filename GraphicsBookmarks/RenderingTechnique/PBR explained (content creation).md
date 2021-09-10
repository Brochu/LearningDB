# Physically-Based Rendering, And You Can Too!

-   [View Larger Image![](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2015/10/pbrandyoucantoo.jpg)](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2015/10/pbrandyoucantoo.jpg)

By [Joe “EarthQuake” Wilson](https://www.artstation.com/artist/earthquake)This tutorial will cover the basics of art content creation, some of the reasoning behind various PBR standards (without getting too technical), and squash some common misconceptions. [Jeff Russell](https://twitter.com/j3ffdr) wrote an excellent tutorial on the [Theory of Physically Based Rendering](https://marmoset.co/posts/basic-theory-of-physically-based-rendering/), which I highly recommend reading first.

Additional help from [Jeff Russell](https://twitter.com/j3ffdr), [Teddy Bergsman](http://www.quixel.se/), and [Ryan Hawkins](https://www.artstation.com/artist/ryanhawkins). Special thanks to [Wojciech Klimas](http://www.wklimas.weebly.com/) and [Joeri Vromman](http://www.joerivromman.com/) for the extra insight and awesome art.

## A New Standard

Fast becoming a standard in the games industry due to increased computing power and the universal need for art content standardization, physically based rendering aims to redefine how we create and render art.

[![](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/header01.jpg)](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/header01.jpg)

Physically based rendering (PBR) refers to the concept of using realistic shading/lighting models along with measured surface values to accurately represent real-world materials.

PBR is more of a concept than a strict set of rules, and as such, the exact implementations of PBR systems tend to vary. However, as every PBR system is based on the same principal idea (render stuff as accurately as possible) many concepts will transfer easily from project to project or engine to engine. Marmoset Toolbag supports most of the common inputs that you would expect to find in a PBR system.

Beyond rendering quality, consistency is the biggest reason to use measured values. Having consistent base materials takes the guess work out of material creation for individual artists. It also makes it easier from an art direction perspective to ensure that content created by a team of artists will look great in every lighting condition.

## PBR FAQs

Before we get started, it’s important to cover common questions that usually pop up when people talk about PBR.

#### 1) I don’t know how to use a PBR system, will I need to re-learn how to create art content?

In most cases, no. If you have experience with previous generation shaders which use dynamic per-pixel lighting you already possess much of the knowledge necessary to create content for a PBR system. Terminology tends to be one of the biggest stumbling blocks for artists, so I have written a section on various terms and translations below. Most of the concepts here are simple and easy to pick up.

If your experience lies mostly with hand painted/mobile work, learning the new techniques and workflows outlined here may be more of a challenge. However, likely not more difficult than picking up a traditional normal map based workflow.

#### 2) Will artists need to capture photographic reference with a polarized camera system for every material they wish to create?

No, generally you will be provided with reference for common materials by your studio. Alternatively, you can find known values from various 3rd party sources, like [Quixel’s Megascans](http://www.quixel.se/megascans/) service. Creating your own scan data is a very technical and time consuming process, and in most cases not necessary.

#### 3) If I use a PBR shader does that mean my artwork is physically accurate?

Not necessarily; simply using a PBR shader does not make your artwork physically accurate. A PBR system is a combination of physically accurate lighting, shading, and properly calibrated art content.

#### 4) Do I need to use a metalness map for it to be PBR?

No, a metalness map is just one method of determining reflectivity and is generally not more or less physically accurate than using a specular color/intensity map.

#### 5) Do I need to use index of refraction (IOR) for it to be PBR?

No, similar to the metalness map input, IOR is simply an alternate method to define reflectivity.

#### 6) Is specular no longer a thing?

Not quite. Specular reflection intensity, or reflectivity is still a very important parameter in PBR systems. You may not have a map to directly set reflectivity (e.g. with a metalness workflow) but it is still required in a PBR system.

#### 7) Do gloss maps replace specular maps?

No, gloss or roughness maps define the microsurface of the material (how rough or smooth it is), and do not replace a specular intensity map. However, if you’re not used to working with gloss maps, it may be somewhat of an adjustment to put certain detail in the gloss map that you would otherwise add to the specular map.

#### 8) Can a PBR system be used to create stylized art?

Yes, absolutely. If your goal is to create a fantastical, stylized world, having accurate material definition is still very important. Even if you’re creating a unicorn that farts rainbows, you still generally want that unicorn to obey the physics of light and matter.

A great example of this is Pixar’s work, which is very stylized, yet often on the cutting edge of material accuracy. Here is a great article about PBR in _Monsters University_: [fxguide feature on Monsters University](http://www.fxguide.com/featured/monsters-university-rendering-physically-based-monsters/)

## Inputs and Terminology

Artists who are unfamiliar with the concept of PBR systems often assume that content creation is drastically different, usually because of the terminology that is used. If you’ve worked with modern shaders and art creation techniques you already have experience with many of the concepts of a physically based rendering system.

Figuring out what type of content to create, or how to plug your content into a PBR shader can be confusing, so here are some common terms and concepts to get started.

### Energy Conservation

The concept of energy conservation states that an object can not reflect more light than it receives.

[![](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/energycompare03.jpg)](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/energycompare03.jpg)

For practical purpose, more diffuse and rough materials will reflect dimmer and wider highlights, while smoother and more reflective materials will reflect brighter and tighter highlights.

### Albedo

Albedo is the base color input, commonly known as a diffuse map.

[![albedocompare02](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/albedocompare03.jpg)](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/albedocompare03.jpg)

An albedo map defines the color of diffused light. One of the biggest differences between an albedo map in a PBR system and a traditional diffuse map is the lack of directional light or ambient occlusion. Directional light will look incorrect in certain lighting conditions, and ambient occlusion should be added in the separate AO slot.

The albedo map will sometimes define more than the diffuse color as well, for instance, when using a metalness map, the albedo map defines the diffuse color for insulators (non-metals) and reflectivity for metallic surfaces.

### Microsurface

Microsurface defines how rough or smooth the surface of a material is.

[![](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/microcompare06.jpg)](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/microcompare06.jpg)

Here we see the how the principles of energy conservation are affected by the microsurface of the material, rougher surfaces will show wider, but dimmer specular reflections while smoother surfaces will show brighter, but sharper specular reflections.

Depending on what engine you’re authoring content for, your texture may be called a roughness map instead of a gloss map. In practice there is little difference between these two types, though a roughness map may have an inverted mapping, ie: dark values equal glossy/smooth surfaces while bright values equal rough/matte surfaces. By default, Toolbag expects white to define the smoothest surfaces while black defines roughest surfaces, if you’re loading a gloss/roughness map with an inverted scale, click the invert check box in the gloss module.

### Reflectivity

Reflectivity is the percentage of light a surface reflects. All types of reflectivity (aka base reflectivity or F0) inputs, including specular, metalness, and IOR, define how reflective a surface is when viewing head on, while Fresnel defines how reflective a surface is at grazing angles.

[![](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/reflcompare11.jpg)](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/reflcompare11.jpg)

Its important to note how narrow the range of reflectivity is for insulative materials. Combined with the concept of energy conservation it’s easy to conclude that surface variation should generally be represented in the microsurface map, not the reflectivity map. For a given material type, reflectivity tends to remain fairly constant. Reflection color tends to be neutral/white for insulators, and colored only for metals. Thus, a map specifically dedicated to reflectivity intensity/color (commonly called a specular map) may be dropped in favor of a metalness map.

[![](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/metalness02.jpg)](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/metalness02.jpg)

When using a metalness map, insulative surfaces – pixels set to 0.0 (black) in the metalness map – are assigned a fixed reflectance value (linear: 0.04 sRGB: 0.22) and use the albedo map for the diffuse value. For metallic surfaces – pixels set to 1.0 (white) in the metalness map – the specular color and intensity is taken from the albedo map, and the diffuse value is set to 0 (black) in the shader. Gray values in the metalness map will be treated as partially metallic and will pull the reflectivity from the albedo and darken the diffuse proportionally to that value (partially metallic materials are uncommon).

Again, a metalness map is not more or less physically accurate than a standard specular map. It is, however, a concept that may be easier to understand, and a metalness map can be packed into a grayscale slot to save memory. The drawback to using a metalness map over a specular map is a loss of control over the exact values for insulative materials

Traditional specular maps offer more control over the the specular intensity and color, and allow greater flexibility when trying to reproduce certain complex materials. The main drawback to a specular map is that it generally will be saved as a 24 bit file resulting in more memory use. It also requires artists to have a very good understanding of physical material properties to get the values right, which can be a positive or negative depending on your perspective.

**PROTIP**: Metalness maps should use values of 0 or 1 ( some gradation can be okay for transitions). Materials like painted metal should not be set to metallic as paint is an insulator. The metalness value should represent the top layer of the material.

IOR is another way to define reflectivity, and is equivalent to the specular and metalness inputs. The biggest difference from the specular input is that IOR values are defined with a difference scale. The IOR scale determines how fast light travels through a material in relation to a vacuum. An IOR value of 1.33 (water) means that light travels 1.33 times slower through water than it does the empty vacuum of space. You can find more measured values in the [Filmetrics Refractive Index Database](http://www.filmetrics.com/refractive-index-database "Filmetrics Refractive Index Database").

For insulators, IOR values do not require color information, and can be entered into the index field directly, while the extinction field should be set to 0. For metals that have color reflections,you will need to enter a value for the red, green and blue channels. This can be done with an image map input (where each channel of the map contains the correct value). The extinction value will also need to be set for metals, which you can usually find in libraries that contain IOR values.

Using IOR as opposed to specular or metalness input is generally not advised, as it is not typically used in games, and getting the correct value in a texture with multiple material types is difficult. IOR input is supported in Toolbag more for scientific purposes than practical.

### Fresnel

Fresnel is the percentage of light that a surface reflects at grazing angles.

[![](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/fresnelcompare03.jpg)](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/fresnelcompare03.jpg)

Fresnel generally should be set to 1 (and is locked to a value of 1 with the metalness reflectivity module) as all types of materials become 100% reflective at grazing angles. Variances in microsurface which result in a brighter or dimmer Fresnel effect are automatically accounted for via the gloss map content.

_Note:_ Toolbag does not currently support a texture map to control Fresnel intensity.

Fresnel, in Toolbag and most PBR systems, is approximated automatically by the BRDF, in this case Blinn-Phong or GGX, and usually does not need an additional input. However, there is an extra control for Fresnel for the Blinn-Phong BRDF, which is meant for legacy use as it can result in non physically accurate results.

### Ambient Occlusion

[![ao01](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/ao01.jpg)](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/ao01.jpg)

Ambient occlusion(AO) represents large scale occluded light and is generally baked from a 3d model.

Adding AO as a separate map as opposed to baking it into the albedo and specular maps allows the shader to use it in a more intelligent way. For instance, the AO function only occludes ambient diffuse light (the diffuse component of the image based lighting system in Toolbag), not direct diffuse light from dynamic lights or specular reflections of any kind.

AO should generally not be multiplied on to specular or gloss maps. Multiplying AO onto the specular map may have been a common technique in the past to reduce inappropriate reflections (e.g. the sky reflecting on an occluded object) but these days local screen space reflections do a much better job of representing inter-object reflections.

### Cavity

[![cavity01](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/cavity01.jpg)](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/cavity01.jpg)

A cavity map represents small scale occluded light and is generally baked from a 3d model or a normal map.

A cavity map should only contain the concave areas (pits) of the surface, and not the convex areas, as the cavity map is multiplied. The content should be mostly white with darker sections to represent the recessed areas of the surface where light would get trapped. The cavity map affects both diffuse and specular from ambient and dynamic light sources.

Alternatively, a reflection occlusion map can be loaded into the cavity slot, but be sure to set the diffuse cavity value to 0 when doing this.

## Finding Material Values

One of the most difficult challenges when working with a PBR system is finding accurate and consistent values. There are a variety of sources for measured values on the internet, however, it can be a real pain to find a library with enough information to rely on.

Quixel’s Megascan’s service is really useful here, as they provide a large library of calibrated tiling textures scanned from real world data.

[![materialref02](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/materialref03.png)](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/materialref03.png)

Material values from most libraries tend to be measured from raw materials in laboratory conditions, of which you rarely see in real life. Factors like pureness of material, age, oxidization, and wear may cause variation in the real world reflectance value for a given object.

While Quixel’s scans are measured from real world materials, there is often variation even within the same material type depending on the various conditions described above, especially when it comes to gloss/roughness. The values in the chart above should be thought of as more of a beginning point, not a rigid/absolute reference.

## Creating Texture Content

[![layering01](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/layering01.jpg)](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/layering01.jpg)

There are many ways to create texture content for PBR systems; the exact method you choose will depend on your personal preferences and what software you have available to you. Here is a quick recap of the method I used to create the lens above:

[![lenstextures01](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/lenstextures01.jpg)](https://mk0marmosandboxoagtf.kinstacdn.com/wp-content/uploads/2014/01/lenstextures01.jpg)First, basic materials were created in Toolbag for each surface type using a combination of tiling textures from Megascans, measured data from known materials, and where lacking appropriate reference, logic and observation, to determine the values. Creating the base materials in Toolbag allows me to quickly adjust values and offers a very accurate preview of the end result. Often I assign base materials directly to my high poly model to get a clear idea of how the texture will come together before doing the final bakes.

After setting up my base materials I brought the values and textures into Photoshop and started layering them in a logical manner. Brass at the bottom, nickel plating, matte primer, semi-gloss textured paint, paint for the lettering, and finally the red glossy plastic. This layering setup provides an easy way to reveal the various materials below with simple masks.

Similar layering functionality can be achieved with applications such as [dDo](http://quixel.se/ddo/), [Mari](http://www.thefoundry.co.uk/products/mari/) and [Substance Designer](http://www.allegorithmic.com/).

After I have my base layers set up and blended together to represent various stages of wear, I added some extra details. First I used dDo to generate a dust and dirt pass, and then I finished it off with fine surface variation in the gloss map.

The exact method you use to create content for a PBR system is much less important than the end result, so feel free to experiment and figure out what works best for your needs. However, you should void tweaking materials values to look more interesting in a specific lighting environment. Using sound base values for your materials can greatly simplify the process, increase consistency and asset reuse on larger projects, and will ensure that your assets always look great no matter how you light them.