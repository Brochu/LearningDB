# NVIDIA Omniverse™ platform

  

---

  

NVIDIA Omniverse is a multi-GPU real-time simulation and collaboration platform based on [Pixar universal scene description](https://developer.nvidia.com/usd) and NVIDIA RTX™. It has powerful performance and is dedicated to handling 3D production processes.

  

Omniverse is committed **to achieving universal interoperability across different applications and 3D ecosystem vendors** . It provides efficient **real-time scene updates** **and is** designed based on **open standards and protocols** . Omniverse platform acts as a **hub of** role, so that the new service functions as micro- **micro-service** open to all clients and applications connected.

  

[Download the  
public beta](https://www.nvidia.cn/design-visualization/omniverse/)     [Apply to join the developer  
program](https://developer.nvidia.com/nvidia-omniverse-early-access-program)

  

---

  
  

### Real-time collaboration between 3D applications and users

Use Universal Scene Description (USD) and Material Definition Language (MDL) to collaborate in real time between your favorite applications.

### Real-time multi-GPU ray tracing viewport

Supports high-quality multi-GPU ray tracing and path tracing related to USD content.

### simulation

Use the latest NVIDIA technology to efficiently simulate the complex 3D physical world.

  
  

---

  
  

Omniverse contains 5 important components, namely **Omniverse Connect** , **Nucleus** , **Kit** , **Simulation** and **RTX** . These components, together with the connected third-party digital content creation (DCC) tools, and other connected Omniverse microservices, together form the entire Omniverse ecosystem.

  

![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/OV_marketecture_007.png)

  

  
  

---

  

## It all starts with a common format

  

![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/AEC_Exp/Omni_AEC_Exp_009.png)

The main representation of assets in Omniverse adopts Pixar's open source **universal scenario description (USD)** notation. USD is not only a file format, but also a rich scene representation. APIs can be used to support complex attribute inheritance, instantiation, layering, lazy loading, and various other key features. Omniverse uses USD to exchange assets through the Nucleus DB service.

  
  

[![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/Marbles_at_Night_RTX_small.png)](https://developer.nvidia.com/sites/default/files/akamai/omniverse/Marbles_at_Night_RTX.png)

  

The materials in Omniverse are expressed using NVIDIA's open source **MDL (Material Definition Language)** . NVIDIA has developed a custom framework expressed in USD to represent material data and parameters, simplifying the exchange of specific material definitions for different applications. This standard definition makes the materials in many applications look very similar even if they are not exactly the same.

  
  

Learn more about [USD and MDL](https://developer.nvidia.com/usd)

  

  
  

---

  

## The core of all this is Omniverse Nucleus

  

![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/AEC_Exp/Omni_AEC_Exp_008.png)

  

**Omniverse Nucleus** provides a set of basic services that enable various client applications, renderers, and microservices to share and modify the representation of the virtual world.

Nucleus runs in publish/subscribe mode. According to access control, the Omniverse client can publish changes to digital assets and virtual worlds to the Nucleus database (DB), or subscribe to these changes. Changes are transmitted in real time between connected applications. Digital assets include geometric figures, lights, materials, textures, and other data describing the virtual world and its evolution.

  

Learn more about [Nucleus](https://docs.omniverse.nvidia.com/prod_nucleus/prod_nucleus/overview/description.html)

  

  
  

---

  

## Connector opens interconnection portals for various applications

  

![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/Omni_connect_003.png)

  
  

**The Omniverse Connect** library is distributed in the form of plug-ins, enabling client applications to connect to Nucleus and publish and subscribe to individual assets and the entire world.

After completing the necessary synchronization, the DCC plug-in will use the Omniverse Connect library to apply the updates received from the outside and publish internally generated changes when necessary.

When the USD representation of the change scenario is applied, Omniverse Connect will track all local changes since the last launch event. When the application makes a request, the Omniverse Connect library will build a separate file for each difference, publish it to Nucleus, and then forward it to all subscribers.

  

Learn more about [Connect](https://docs.omniverse.nvidia.com/con_connect/con_connect/overview.html)

  

  
  

---

  

## Omniverse process

  

On the left, we can see many popular DCC applications, as well as new applications created specifically for Omniverse using Kit. These applications can export USD file format and support MDL material. Thanks to the Omniverse portal created by the Omniverse Connector plug-in, these applications can connect to the Nucleus database.

  

![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/omni_pipeline_002.png)

  

The Nucleus server can also provide Omniverse functions as a headless microservice, and can provide exquisite rendering results to many visualization clients, including VR and AR devices.

  

---

  

## Create your own application with Omniverse Kit

  
  

Kit is not a single application, but is composed of extensions that can be used as building blocks to be assembled in a variety of ways to help create different types of applications. Because they are all written in Python, all UI elements, workflows and general functions are highly customizable.

**The Omniverse Kit** is a toolkit for building native Omniverse applications and **microservices. It** is built on a basic framework that can provide various functions through a set of lightweight extensions. These independent extension programs are plug-ins written in Python or C++.

![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/omni_kit_parts_002.png)

  

After design, Kit has become a flexible and extensible application and microservice development platform. It can create microservices in headless mode or through the UI. UI applications can be written completely using the UI engine, so as to obtain complete customization.

For better performance or access to certain C++ APIs, lower-level C++ plug-ins can be added to these extensions, and these plug-ins can also be connected to the UI through bindings. These extensions include the icons, images, and configuration required for them to operate individually

  
  

---

  

### Omniverse Kit extension

  

![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/Omni_Ext_viewport.png)

  
  
**RTX viewport extension**

Utilize NVIDIA RTX and MDL materials to represent your data with ultra-high fidelity. The program has amazing scalability, supports a large number of GPUs, and can provide real-time interaction in large scenes, as well as ensuring accuracy through various ray tracing and path tracing options.

  

---

  

**Content browsing extension**

Browse the files on your local or remote Omniverse Nucleus server, organize the data, and find the files you want to process or collaborate on. It contains a rich set of APIs that can help you automate tasks and processes, for example, by using DeepTag, using AI to assign metadata categories, and searching for assets in new ways.

![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/Omni_Ext_browser.png)

  

---

  

![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/Omni_Ext_layers.png)

**USD widget and Window extension**

The Stage Window extension can be used to create a robust Stage data browsing experience. The Stage Window contains all the relevant information of the scene objects, and you can process this information.

With the help of Property Window, you can access all the object properties and other various information contained in the USD file. In addition, the program is fully extensible, and each part of it is derived from a dedicated extension program for each primitive type in the scene.

Finally, you can use USD's powerful layering system through Layer Window to achieve rich composition, and at the same time, you can get Omniverse's layer access management and real-time collaboration functions through the system.

  

---

  

**Omniverse UI**

In order to provide a fast and responsive lightweight, open hardware-accelerated UI, the Omniverse framework is built on the basis of the [Dear ImGui](https://github.com/ocornut/imgui) library.

Main features:

-   Fast modern lightweight UI framework
-   The basics of the Omniverse Kit user interface
-   Declarative syntax and dynamic layout
-   Support full styleable, similar to HTML using "stylesheet-like" workflow
-   Support Omni UI streaming with lossless UI quality
-   Support XR (VR and AR) rendering (3D projection of small components)
-   Includes XR input devices (controller, hands, eyes)

  

![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/Omni_ui.png)

  

---

  

In the final analysis, Create (the sample application included in Omniverse), View (the main application for creating the AEC experience), and other Omniverse applications come from extensions, which together form the atomic building blocks of Omniverse Apps. The number of extensions will increase rapidly, because they are mainly written in Python and are accompanied by complete source code, which can help developers easily create, add, and modify the tools and workflows needed to increase productivity.

  
  

Learn more: [Omniverse Create (built based on Kit)](https://docs.omniverse.nvidia.com/app_create/app_create/overview.html)

  
  

  
  

---

  

### Simulated reality

The  
**simulation** function in Omniverse is provided to Omniverse Kit in the form of plug-ins or microservices by a series of NVIDIA technologies.

As one of the first simulation tools provided by Omniverse, NVIDIA's open source physics simulator **PhysX is** widely used in computer games. The objects participating in the simulation, their properties, any constraints and any solver parameters are specified in the custom USD framework. Kit provides functions such as editing simulation settings, starting and stopping simulation, and adjusting all parameters.

Omniverse physics simulation currently includes simulations of rigid body dynamics, destruction and fracture, vehicle dynamics, and fluid dynamics (Flow). Flow is an Euler fluid simulation of smoke/fire, using sparse voxel grids to realize an unbounded simulation domain.

  

Learn more about [Omniverse physics simulation](https://docs.omniverse.nvidia.com/app_create/prod_extensions/ext_physics.html)

  

---

  

### Visualize and render the beautiful world

  

  

Omniverse supports a variety of renderers compatible with the **Pixar Hydra architecture** . One of them is the new Omniverse RTX renderer, which makes full use of the hardware RT Core in the NVIDIA Turing and Ampere architecture to achieve real-time hardware accelerated ray tracing and path tracing.

This renderer has **no rasterization before ray tracing** , so it can process **large scenes in** **real time** . It contains two modes, one is traditional ray tracing that provides fast performance, and the other is path tracing that provides high-quality results.

The Omniverse RTX renderer **natively supports multiple GPUs** in one system , and will support interactive rendering of multiple systems in the near future.

  

Learn more about the [Omniverse RTX renderer](https://docs.omniverse.nvidia.com/prod_rtx/prod_rtx/overview.html)

  

---

  

## Omniverse Apps

  

NVIDIA Omniverse can now connect to many content creation applications, and NVIDIA has created **Apps** to showcase its functions in different workflows.

**Apps are built using Omniverse Kit** , which is not only a practical tool in itself, but also a starting point for developers to build, extend, or create their own applications on top of it. Apps in Apps not only provide examples for technical artists and developers, but will continue to gain new functions and features in the future.

  
  

[![](https://www.nvidia.com/content/dam/en-zz/Solutions/gtcf20/omniverse/create/nv-omni-create-app-lp-create-static-630x354-D.jpg)**NVIDIA Omniverse Create**](https://www.nvidia.cn/design-visualization/omniverse/create/)

[![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/kaolinapp2.png)**NVIDIA Kaolin App**](https://developer.nvidia.cn/nvidia-kaolin)

  

[![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/Omni_App_aec.png)**NVIDIA Omniverse View**](https://www.nvidia.cn/design-visualization/omniverse/view/)

![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/Omni_App_isaac.png)**[NVIDIA Isaac Sim](https://developer.nvidia.com/isaac-sim)**

  

[![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/Omni_App_machinima.png)**NVIDIA Omniverse Machinima**](https://www.nvidia.cn/geforce/machinima/)

[![](https://developer.nvidia.com/sites/default/files/akamai/omniverse/Omni_App_audio2face.png)**NVIDIA Omniverse Audio2Face**](https://www.nvidia.cn/design-visualization/omniverse/audio2face/)

  
  

More applications are under development...