## An overview of technical capabilities and specifications of the Lumen Global Illumination and Reflections system.

Lumen uses multiple ray-tracing methods to solve Global Illumination and Reflections. Screen Traces are done first, followed by a more reliable method.

Lumen uses **Software Ray Tracing** through Signed Distance Fields by default, but can achieve higher quality on supporting video cards when **Hardware Ray Tracing** is enabled.

Lumen's Global Illumination and Reflections primary shipping target is to support large, open worlds running at 60 frames per second (FPS) on next-generation consoles. The engine's **High** scalability level contains settings for Lumen targeting 60 FPS.

Lumen's secondary focus is on clean indoor lighting at 30 FPS on next-generation consoles. The engine's **Epic** scalability level produces around 8 milliseconds (ms) on next-generation consoles for global illumination and reflections at 1080p internal resolution, relying on Temporal Super Resolution to output at quality approaching native 4K.

## Surface Cache

Lumen generates an automatic parameterization of the nearby scene's surface called **Surface Cache**. It is used to quickly look up lighting at ray hit points in the scene. Lumen captures the material properties for each mesh from multiple angles. These capture positions (called **Cards**) are generated offline for each mesh.

Cards can be visualized with the console command `r.Lumen.Visualize.CardPlacement 1`.

![Lumen Card Placement Visualization](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/mesh-card-placement-visualization-alt.jpg)

By default, Lumen only places 12 Cards on a mesh, but you can increase that amount by setting **Max Lumen Mesh Cards** in the **Build Settings** of the Static Mesh Editor. Adjusting the number of cards is useful for more complex interiors or single meshes with irregular shapes.

![Static Mesh Editor Build Setting Max Lumen Mesh Cards](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/static-mesh-editor-max-lumen-mesh-cards-setting.jpg)

Areas that don't have Surface Cache coverage are colored pink in the **Surface Cache** View Mode of the Level Editor.

These areas will not bounce light and will appear black in reflections. Issues like this can be fixed by increasing the number of Cards used with Max Lumen Mesh Cards, but that may not solve all issues. Alternatively, breaking the mesh into less complex pieces can resolve these types of issues as well.

[![View Mode Lumen Surface Cache](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/viewmode-lumen-surface-cache.jpg)](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/viewmode-lumen-surface-cache.png)

[![Lumen Surface Cache Visualization Example](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/lumen-surface-cache-visualization.jpg)](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/lumen-surface-cache-visualization.png)

View Mode > Lumen > Surface Cache

Lumen Surface Cache Visualization of Complex Mesh

_Click image for full size._

_Click image for full size._

Materials with view-dependent logic, such as Pixel Depth, Camera Position, or Camera Vector may appear incorrectly in Lumen Surface Cache view mode. Materials that use these nodes can use the **Ray Tracing Quality Switch** node to provide a version of the Material that works with Lumen Surface Cache, or to optimize Surface Cache captures for complex materials.

![ray tracing quality switch material node](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/ray-tracing-quality-switch-node.jpg)

For more information about using the Ray Tracing Quality Switch node, see [Ray Tracing Performance Guide](https://docs.unrealengine.com/5.0/en-US/ray-tracing-performance-guide-in-unreal-engine).

Nanite accelerates the mesh captures used to keep Surface Cache in sync with the triangle scene. High polygon meshes in particular, need to be using [Nanite](https://docs.unrealengine.com/5.0/en-US/nanite-virtualized-geometry-in-unreal-engine) to have efficient captures. Foliage and Instanced Static Mesh Components can only be supported if the mesh is using Nanite.

After the Surface Cache is populated with material properties, Lumen calculates direct and indirect lighting for these surface positions. These updates are amortized over multiple frames, providing efficient support for many dynamic lights and multiple bounce global illumination.

Unreal Engine provides visualization modes for Surface Cache and Cards representation. See the [Lumen Visualization Options](https://docs.unrealengine.com/5.0/en-US/lumen-technical-details-in-unreal-engine#lumenvisualizationoptions) section of this page for more details.

Only meshes with simple interiors can be supported — walls, floors, and ceilings should all be separate meshes. Importing an entire room, which includes furniture, in a single mesh is not expected to work with Lumen.

## Screen Tracing

Lumen features trace rays against the screen first (called **Screen Tracing** or Screen Space Tracing), before using a more reliable method if no hit is found, or the ray passes behind a surface. Screen tracing supports any geometry type and is useful for covering up mismatches between the Lumen Scene and triangle scene.

The disadvantage in using screen traces is that they greatly limit controls for art direction, which would only apply to indirect lighting, like lighting properties for Indirect Lighting Scale or Emissive Boost. Setting a large Indirect Lighting Scale on a light will cause view-dependent global illumination, as Screen Traces cannot support it correctly.

The example scene below uses Screen Traces first before falling back to other, more costly tracing options. When disabling Screen Traces for global illumination and reflections, it is possible to see only the Lumen Scene produced by Software Ray Tracing. Screen Traces help resolve the mismatch that can happen between the triangle scene and Lumen Scene.

You can perform this type of comparison by disabling Screen Traces from the Level Viewport's **Show > Lumen** menu and removing the check next to **Screen Traces**.

![Screen Traces Enabled | (Default)](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/screen-traces-enabled.jpg)

Screen Traces Enabled

(Default)

![Screen Traces Disabled | Using Surface Cache](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/screen-traces-disabled.jpg)

Screen Traces Disabled

Using Surface Cache

## Lumen Ray Tracing

Lumen provides two methods of ray tracing the scene: Software Ray Tracing and Hardware Ray Tracing.

-   **Software Ray Tracing** uses Mesh Distance Fields to operate on the widest range of hardware and platforms but is limited in the types of geometry, materials, and workflows it can effectively use.
    
-   **Hardware Ray Tracing** supports a larger range of geometry types for high quality by tracing against triangles and to evaluate lighting at the ray hit instead of the lower quality Surface Cache. It requires supported video cards and systems to operate.
    

Software Ray Tracing is the only performant option in scenes with many overlapping instances, while Hardware Ray Tracing is the only way to achieve high quality mirror reflections on surfaces.

### Software Ray Tracing

Lumen uses Software Ray Tracing against Signed Distance Fields by default. This tracing representation is supported on any hardware supporting Shader Model 5 (SM5), and only requires that **Generate Mesh Distance FIelds** be enabled in the Project Settings.

The renderer merges Mesh Distance Fields into a Global Distance Field to accelerate tracing. By default, Lumen traces against each mesh's distance field for the first two meters for accuracy, and the merged Global Distance Field for the rest of each ray.

Projects with extreme overlapping instances can control the method Lumen uses with the project setting **Software Ray Tracing Mode**. Lumen provides two options to choose from:

-   **Detail Tracing** is the default method and involves tracing against the individual mesh's signed distance field for the highest quality. The first two meters are used for accuracy and the Global Distance Field for the rest of each ray.
    
-   **Global Tracing** only traces against the Global Distance Field for each ray for the fastest traces.
    

Mesh Distance Fields are streamed in and out based on distance as the camera moves through the world. They are packed into a single atlas to allow ray tracing.

Use the command `r.DistanceFields.LogAtlasStats 1` to output mesh distance field statistics to the log.

The quality of Lumen's Software Ray Tracing depends on the quality of the mesh's distance field representation. There are two visualization options, one for **Mesh DistanceFields** and the other for **Global DistanceField**. These visualization modes are found under the viewport's **Show > Visualize** menu.

![Show Visualization menu selections for Mesh Distance Fields and Global Distance Field](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/show-visualization-mdf-gdf.jpg)

![Lumen Scene View](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/scene-view.jpg)

![Mesh Distance Field Visualization](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/vis-mesh-distance-fields.jpg)

![Global Distance Field Visualization](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/vis-global-distance-field.jpg)

Scene View

Mesh Distance Fields Visualization

Global Distance Field Visualization

For some meshes, thin surfaces may not have a good distance field representation and could cause issues with light leaking. The Mesh Distance Field visualization can help you spot these types of issues.

![mdfresolution-chandelier.png](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/mdfresolution-chandelier.png)

(Left to Right) Triangle Mesh, Distance Field Resolution Scales 1.0 (Default), 1.5, 2.0

There are two ways to go about improving a meshes distance field representation:

[![projectsettings-voxeldensity.png](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/projectsettings-voxeldensity.jpg)](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/projectsettings-voxeldensity.png)

[![staticmesheditor-proxymeshpercent.png](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/staticmesheditor-proxymeshpercent.jpg)](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/staticmesheditor-proxymeshpercent.png)

Project Settings: Distance Field Voxel Density

Static Mesh Editor: Distance Field Resolution Scale

_Click image for full size._

_Click image for full size._

-   Projects can globally increase mesh distance field quality using the **Distance Field Voxel Density** found in the Project Settings.
    
-   Increase the **Distance Field Resolution Scale** of individual meshes that need more quality from the Static Mesh Editor **Build Settings**.
    

Increasing Distance Field resolution or density will increase the disk size of the project.

#### Limitations of Software Ray Tracing

Software Ray Tracing has some limitations relating to how you should work with it in your projects and what types of geometry and materials it currently supports.

**Geometry Limitations:**

-   Only Static Meshes, Instanced Static Meshes, Hierarchical Instanced Static Meshes, and Landscape terrain are represented in the Lumen Scene.
    
-   Foliage must be enabled with the setting **Affect Distance Field Lighting** found in the Foliage Tool settings.
    

**Material Limitations:**

-   World Position Offset (WPO) is not supported.
    
-   Transparent materials are ignored by distance fields and Masked materials are treated as Opaque.
    
    -   Masked materials can cause significant over-shadowing on foliage where large areas of leaves are masked out.
        
-   Distance fields are built off of properties of the material assigned to the Static Mesh Asset rather than the override component.
    
    -   Overriding with a material that has a different Blend Mode or that has Two-Sided property enabled will cause a mismatch between the triangle representation and the mesh's distance field representation.
        

**Workflow Limitations:**

-   Software Ray Tracing requires that levels be composed of modular geometry. Things like walls, floors, and ceilings should be separate meshes. Large single meshes, such as a Mountain or multi-story building, will have a poor distance field representation that can cause self-occlusion artifacts to appear.
    
-   Walls should be no thinner than 10 centimeters (cm) to avoid light leaking.
    
-   Distance Fields cannot represent extremely thin features, or one-sided meshes seen from behind. Avoid these types of artifacts by ensuring the viewer doesn't see the triangle back faces of one-sided meshes or only use closed geometry.
    
-   Mesh Distance Field resolution is assigned based on the imported scale of the Static Mesh.
    
    -   A mesh that is imported very small and then scaled up on the component **will not** have sufficient distance field resolution. Instead, set the distance field resolution from the Static Mesh Editor's Build Settings if you use scaling on placed instances in a Level.
        

### Hardware Ray Tracing

Hardware Ray Tracing supports a larger range of geometry types than Software Ray Tracing, in particular it supports tracing against skinned meshes. Hardware Ray Tracing also scales up better to higher qualities — it intersects against the actual triangles and has the option to evaluate lighting at the ray hit instead of the lower quality Surface Cache.

To use high quality hit lighting, the following must be enabled in the Project Settings under the **Engine > Rendering** section:

![Lumen Project Settings for High Quality](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/project-setting-lumen-settings.jpg)

-   Support Hardware Ray Tracing: **Enabled**
    
-   Use Hardware Ray Tracing when available: **Enabled**
    
-   Ray Lighting Mode: **Hit Lighting for Reflections**
    

While Hardware Ray Tracing provides the highest quality of the two ray tracing methods, it also has the highest setup cost in large scenes causing tracing to become expensive with many overlapping meshes. Dynamically deforming meshes, like skinned meshes, also incur a large cost to update the Ray Tracing acceleration structures each frame, proportional to the number of skinned triangles. You can learn more about the setup and cost of Hardware Ray Tracing in the [Ray Tracing Performance Guide](https://docs.unrealengine.com/5.0/en-US/ray-tracing-performance-guide-in-unreal-engine).

For Static Meshes that have Nanite enabled, Hardware Ray Tracing can only operate on the **Fallback Mesh**. This is a mesh that is generated by Nanite's **Fallback Relative Error** property to be used when the full detail mesh cannot be. These fallback meshes can be visualized in the Level viewport under the **Show > Advanced > Nanite Meshes**.

Screen Traces are used to cover the mismatch between the full triangle meshes rendered by Nanite and the Fallback Mesh being ray-traced by Lumen. However, there are some cases where the mismatch is too large to be hidden. In these cases, raising the Fallback Relative Error can reduce incorrect self-intersection artifacts.

Full-detail Triangle Mesh

![Nanite Generated Fallback Mesh](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/sme-naniteproxymesh2.jpg)

Nanite Generated Fallback Mesh

Lumen will use Hardware Ray Tracing when:

-   The project has **Support Hardware Ray Tracing** and **Use Hardware Ray Tracing when available** enabled.
    
    -   Changing the first setting requires restarting the engine.
        
-   The project is running on a supported operating system, RHI, and video card. Currently only the following platforms support performant Hardware Ray Tracing:
    
    -   Windows 10 with DirectX 12
        
        -   Video cards must be NVIDIA RTX-2000 series and higher, or AMD RX 6000 series and higher.
            
    -   PlayStation 5
        
    -   Xbox Series S / X
        

A project using Hardware Ray Tracing for Lumen may also want to fallback to Software Ray Tracing when needed, but not pay the memory costs and scene update costs of both at the same time. In these cases, you should add the following console variable to your `DefaultEngine.ini` configuration file:

```
r.DistanceFields.SupportEvenIfHardwareRayTracingSupported=0
```

With this setting disabled, it is no longer possible to toggle **Use Hardware Ray Tracing when available** at runtime.

## Large Worlds

Lumen Scene operates on the world around the camera, enabling large worlds and streaming. Lumen relies on Nanite's Level of Detail (LOD) and Multi-View rasterization for fast scene captures to maintain the Surface Cache, with all operations throttled to prevent hitches from occurring. Lumen does not require Nanite to operate, but Lumen's scene capturing becomes very slow in scenes with lots of high polygon meshes that have not enabled Nanite. This is especially true in scenes if the assets do not have good LODs set up for them.

Fast camera movement will cause Lumen Scene updating to fall behind where the camera is looking, causing indirect lighting to pop in as it catches up.

When Lumen is using Software Ray Tracing, Lumen Scene only covers 200 meters (m) from the camera position, but can be increased up to 800 m with the **Lumen Scene View Distance** setting in the Post Process Volume. Past the highest Lumen Scene distance, only Screen Traces are active for global illumination.

### Far Field

Lumen Hardware Ray Tracing implements **Far Field** traces, which extend Lumen Global Illumination and Reflections to one kilometer distance from the camera by default.

[Far Field](https://docs.unrealengine.com/5.0/en-US/ray-tracing-performance-guide-in-unreal-engine#farfield) traces are enabled by the console variable `r.LumenScene.FarField=1` in the `DefaultEngine.ini` configuration file, and requires the use of [World Partition's Hierarchical Level of Detail](https://docs.unrealengine.com/5.0/en-US/world-partition---hierarchical-level-of-detail-in-unreal-engine) (HLOD). Far Field requires the use of HLOD1 to be built.

When Far Field is enabled, it is traced beginning at the **Max Trace Distance** (default is 200m) and continues to `r.LumenScene.FarField.MaxtraceDistance` (default is 1 kilometer). Since objects can be culled from ray tracing using `r.RayTracing.Culling.Radius`, your projects will likely want the culling radius and max trace distance to match. Otherwise, the near-field (objects from the camera to the Max Trace Distance) may be culled before the far-field traversal point, which will leave a gap in coverage.

![Lumen Far Field Global Illumination and Reflections](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/lumen-far-field.jpg)

_The Unreal Engine 5 technical demo "The Matrix Awakens" demonstrates large-world global illumination using Far Field traces._

## General Limitations of Lumen Features

-   Lumen Global Illumination cannot be used with [Static Lighting](https://docs.unrealengine.com/5.0/en-US/static-light-mobility-in-unreal-engine) in lightmaps. Lumen Reflections should be extended to work with global illumination in lightmaps in a future release of Unreal Engine 5, which will provide a way to further scale up render quality for projects using Static lighting techniques.
    
-   Lumen Reflections do not support multiple specular reflections.
    
-   [Single Layer Water](https://docs.unrealengine.com/5.0/en-US/single-layer-water-shading-model-in-unreal-engine) Materials are currently not supported by Lumen.
    
-   Lumen does not currently support Scene Captures or Split-screen.
    
-   Lumen is **not** compatible with [Forward Shading](https://docs.unrealengine.com/5.0/en-US/forward-shading-renderer-in-unreal-engine).
    

## Performance

Lumen uses the Scalability settings found in the Level Editor under the viewport **Settings > Engine Scalability Settings**. These settings contain general quality settings from **Low** to **Cinematic**, but also allow you to specify individual features settings in the same ranges. The scalability options are not limited to the Editor and can be called through Blueprint to use in the projects you ship.

Lumen is only active when the global **Quality**, or the individual **Global Illumination**, and **Reflections** are set to **High**, **Epic**, **Cinematic** sclabaility settings.

![In-Editor Scalability Settings](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/lumen-scalability-settings.jpg)

Lumen relies heavily on [Temporal Upsampling](https://docs.unrealengine.com/5.0/en-US/screen-percentage-with-temporal-upscale-in-unreal-engine) with the Unreal Engine 5 feature Temporal Super Resolution algorithm for 4K output. The engine scalability options set target fps with Lumen default settings.

-   **Epic** scalability level is set for a 30 fps budget (8 ms for global illumination and reflections) at 1080p on next-generation consoles.
    
-   **High** scalability level is set for a 60 fps budget. However, note that achieving a 60 fps budget with acceptable quality is still a work in progress.
    
-   Lumen is disabled for **Low** and **Medium** scalability levels.
    

Materials with roughness below 0.4 cost the most for Lumen to solve lighting for, as extra rays have to be traced to provide Lumen Reflections.

## Lumen Platform Support

-   Lumen **does not** support previous-generation consoles, such as PlayStation 4 and Xbox One.
    
-   Projects that rely on dynamic lighting can use a combination of [Distance Field Ambient Occlusion](https://docs.unrealengine.com/5.0/en-US/distance-field-ambient-occlusion-in-unreal-engine) and [Screen Space Global Illumination](https://docs.unrealengine.com/5.0/en-US/screen-space-global-illumination-in-unreal-engine) on those platforms and legacy PC hardware.
    
-   Lumen is developed for next-generation consoles (PlayStation 5 and Xbox Series S / X) and high-end PCs. Lumen has two ray tracing modes, each with different requirements.
    
-   Software Ray Tracing Requirements:
    
-   Video cards using DirectX 11 with support for Shader Model 5 (SM5)
    
    Requires an NVIDIA GeForce GTX-1070 or higher card.
    
-   Hardware Ray Tracing Requirements:
    
    -   Windows 10 with DirectX 12
        
    -   Video cards must be NVIDIA RTX-2000 series or higher, or AMD RX-6000 series or higher.
        
-   Lumen **does not** support mobile platforms. There are no plans to develop a dynamic global illumination system for the mobile renderer. Games using dynamic lighting need to use unshadowed [Sky Light](https://docs.unrealengine.com/5.0/en-US/sky-lights-in-unreal-engine) on mobile.
    
-   Lumen **does not** currently support Virtual Reality (VR) systems. While VR can be supported, the high frame rates and resolutions required by VR make dynamic global illumination a poor fit.
    

## Lumen Visualization Options

Lumen provides several ways of visualizing data in the Unreal Editor that can be helpful in inspecting and troubleshooting how Lumen is lighting the scene.

The primary Lumen view modes are found in the Level Viewport under the View Modes menu:

[![lumen viewmodes menu](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/lumen-viewmodes-menu.jpg)](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/lumen-viewmodes-menu.png)

[![lumen overview viewmode](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/lumen-overview-viewmode.jpg)](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/lumen-overview-viewmode.png)

Lumen View Modes Options

Lumen Overview Visualization

_Click image for full size._

_Click image for full size._

-   **Overview** shows all three of Lumen's visualizations as small tiles over the rendered scene.
    
-   **Lumen Scene** shows Lumen's version of the scene, in the highest quality available: Surface Cache or Hardware Ray Tracing.
    
-   **Reflection View** shows the scene as Lumen Reflections see it, including limited tracing distances used by reflections.
    
-   **Surface Cache** is the same as Lumen Scene except that areas of meshes that were too complex to be covered are color highlighted in pink.
    

Additional visualizations are found in the Level Viewport's **Show > Visualize** menu.

![Show Visualization menu selections for Mesh Distance Fields and Global Distance Field](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/show-visualization-mdf-gdf.jpg)

![Lumen Scene View](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/scene-view.jpg)

![Mesh Distance Field Visualization](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/vis-mesh-distance-fields.jpg)

![Global Distance Field Visualization](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/vis-global-distance-field.jpg)

Scene View

Mesh Distance Fields Visualization

Global Distance Field Visualization

-   **Mesh Distance Fields** shows the individual mesh's Signed Distance Field representations that make up the scene.
    
-   **Global Distance Field** shows the lower detail distance field that has been merged of individual mesh distance fields. There are also some console commands you can use to visualize other data used for Lumen:
    
-   Lumen's **Card Placement** looks at how capture positions (called **Cards**) are used in the scene. These Cards are generated offline for each mesh, and are used to capture the material properties of each mesh from multiple angles. Enable this visualization mode using `r.Lumen.Visualize.CardPlacement 1`.
    
-   Hardware Ray Tracing uses the **Nanite Fallback Mesh**, which is generated from the **Fallback Relative Error** property of the Static Mesh Asset. The fallback mesh is used to cover mismatches between Screen Traces and the Nanite ray-traced scene in Lumen Reflections. Use **Show > Advanced > Nanite Meshes** to disable Nanite rendering and see the fallback mesh used by Lumen.
    

## Troubleshooting Topics

When troubleshooting any Lumen-related issues in your scene, a good place to start is with the **Lumen Overview** view mode.

Lumen Scene should match the main scene view in ways that have noticeable impact on indirect lighting. Significant pink areas in Lumen Surface Cache view mode should be solved by raising the **Max Lumen Mesh Cards** in the Static Mesh settings, or splitting the mesh into multiple parts.

![troubleshooting-topic-1.png](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/troubleshooting-topic-1.jpg)

### Problem-Causing Meshes

In cases where you have problem-causing meshes contributing to indirect lighting, they can be removed from Lumen Scene using by using the Level instance's **Details** panel to disable one of the following:

-   For **Software Ray Tracing**, unchecking the box for **Affect Distance Field Lighting** removes them.
    
-   For **Hardware Ray Tracing**, unchecking the box for **Visible in Ray Tracing** removes them.
    

### Mismatch Between Shadow Maps and Ray-Traced Shadows

Lumen reuses the Renderer's shadow maps for shadowing the Lumen Scene. However, these are only available for on-screen positions. For off-screen surfaces, Lumen uses ray tracing for shadowing. When there is significant mismatch between these two techniques, Lumen Global Illumination becomes highly view-dependent, similarly to screen space techniques like [Screen Space Global Illumination](https://docs.unrealengine.com/5.0/en-US/screen-space-global-illumination-in-unreal-engine).

This type of issue can be solved by figuring out what is cursing the mismatch to occur. Disable the **Reuse Shadow Maps** show flag under **Show > Lumen** while looking at the **Lumen Scene** view mode to see the ray-traced shadows.

#### Problem: Splotchy Artifacts seen in Mirror Reflections Indoors

Splotchy artifacts seen in mirror reflections happen because Lumen uses very low quality settings to calculate multi-bounce global illumination since there is not much budget for it. In the Post Process settings, increase the **Lumen Scene Quality** to reduce artifacts.

![lumen-scene-viewmode-default.png](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/lumen-scene-viewmode-default.jpg)

![lumen-scene-lighting-quality-4.png](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/lumen-scene-lighting-quality-4.jpg)

Lumen Scene View Mode

Improved Lumen Scene Lighting Quality to 4

#### Problem: Small Meshes are Black in Mirror Reflections

Small meshes appear black in mirror reflections because Lumen culls small objects from Lumen Scene for performance. In the Post Process Setting, Increase the **Lumen Scene Detail** will capture smaller meshes over farther distances.

![Lumen Scene View Mode](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/troubleshooting-topic-black-meshes-1.jpg)

Lumen Scene View Mode

![Lumen Scene Detail Increased](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/troubleshooting-topic-black-meshes-2.jpg)

Lumen Scene Detail Increased

#### Problem: Sky Occlusion and Global Illumination Disappears at 200 Meters

Sky occlusion and global illumination are only maintained in Lumen Scene for the first 200 meters. In the Post Process settings, increase the **Lumen Scene View Distance** to maintain Lumen Scene over farther distances.

#### Problem: Light Leaking in Large Cave-like Areas

In cave-like, or enclosed areas, if light leaking is happening, it is because Lumen limits how far rays are traced to improve performance, but missing an occluder causes light leaking.

In the Post Process Setting, raise the **Lumen Scene Detail** and the **Max Trace Distance**.

For example, the mesh below is an enclosed box with no holes viewed from the outside.

![troubleshooting-topic-enclosed-box-1.png](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/troubleshooting-topic-enclosed-box-1.jpg)

Inside the box, you can see that skylighting is leaking throughout the scene.

![troubleshooting-topic-enclosed-box-2.png](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/troubleshooting-topic-enclosed-box-2.jpg)

Raise the following values:

-   Lumen Scene View Distance
    
-   Max Trace Distance
    

The light leaking should now be fixed.

![troubleshooting-topic-enclosed-box-3.png](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/troubleshooting-topic-enclosed-box-3.jpg)

#### Problem: Lighting Changes Propagate too Slowly to Global Illumination

When changing or toggling Lumen Global Illumination off, the lighting in the scene changes too slowly causing it to fade rather than quickly turn on or off.

For example, toggling the Sky Light off in the following scene, you see the lighting slowly fade out.

![troubleshooting-topic-lumen-update-speed-1.gif](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/troubleshooting-topic-lumen-update-speed-1.jpg)

You can improve the speed at which Lumen Global Illumination changes by increasing the Post Process Volume setting for **Final Gather Lighting Update Speed**.

Notice how the skylighting turns off more quickly than before.

![troubleshooting-topic-lumen-update-speed-2.gif](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/troubleshooting-topic-lumen-update-speed-2.jpg)

Remember to turn these settings back to defaults after the change to get back to cheaper lighting.

#### Problem: Small Emissive Meshes Do Not Light the Scene Consistently

Small objects are culled from Lumen Scene leaving only the Screen Traces to pick up small emissive meshes. This leads to inconsistency in their lighting in the scene. Set the Level instances **Emissive Light Source** from the **Details** panel.

![Lumen Scene View Mode](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/troubleshooting-topic-emissive-1.jpg)

Lumen Scene View Mode

![Emissive Light Source Property Enabled](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/troubleshooting-topic-emissive-2.jpg)

Emissive Light Source Property Enabled

#### Problem: Wanting Highest Quality Mirror Reflections Even if Not Performant

Lumen uses its Surface Cache for lighting reflection rays by default since it is significantly faster to render. However, lighting can be configured to evaluate at the ray hit by setting one of the following:

-   In the Project Settings, set **Ray Lighting Mode** to **Hit Lighting for Reflections**.
    
-   This changes lighting for the entire project.
    
-   In the Post Process Volume, set **Ray Lighting Mode** to **Hit Lighting for Reflections**.
    
    -   Ideal for individual changes where you only need it for a single shot or viewpoint.
        

In the example scene below, notice how the specular highlights on the wall and stairs are missing in the reflection.

![Lumen Surface Cache](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/troubleshooting-topic-reflection-quality-1.jpg)

Lumen Surface Cache

![Lumen Hit Lighting for Reflections](https://docs.unrealengine.com/5.0/Images/building-virtual-worlds/lighting-and-shadows/global-illumination/lumen/TechOverview/troubleshooting-topic-reflection-quality-2.jpg)

Lumen Hit Lighting for Reflections