# API without Secrets: Introduction to Vulkan

by Pawel Lapinski

Source code examples for "API without Secrets: Introduction to Vulkan" tutorial which can be found at:

[https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-preface](https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-preface)

Special thanks to Slawomir Cygan for help and for patiently answering my many, many questions!

## [](https://github.com/GameTechDev/IntroductionToVulkan#drivers)Drivers:

Vulkan drivers and other related resources can be found at [https://www.khronos.org/vulkan/](https://www.khronos.org/vulkan/)

## [](https://github.com/GameTechDev/IntroductionToVulkan#tutorials)Tutorials:

### [](https://github.com/GameTechDev/IntroductionToVulkan#01---the-beginning)[01 - The Beginning](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Project/Tutorials/01)

[![](https://github.com/GameTechDev/IntroductionToVulkan/raw/master/Document/Images/01%20-%20The%20Beginning.png)](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Document/Images/01%20-%20The%20Beginning.png)

#### [](https://github.com/GameTechDev/IntroductionToVulkan#introduction-to-a-vulkan-world)Introduction to a Vulkan world

##### [](https://github.com/GameTechDev/IntroductionToVulkan#httpssoftwareintelcomen-usarticlesapi-without-secrets-introduction-to-vulkan-part-1)[https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-1](https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-1)

Tutorial presents how to create all resources necessary to use Vulkan inside our application: function pointers loading, Vulkan instance creation, physical device enumeration, logical device creation and queue set up.

---

### [](https://github.com/GameTechDev/IntroductionToVulkan#02---swap-chain)[02 - Swap chain](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Project/Tutorials/02)

[![](https://github.com/GameTechDev/IntroductionToVulkan/raw/master/Document/Images/02%20-%20Swap%20Chain.png)](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Document/Images/02%20-%20Swap%20Chain.png)

#### [](https://github.com/GameTechDev/IntroductionToVulkan#integrating-vulkan-with-os)Integrating Vulkan with OS

##### [](https://github.com/GameTechDev/IntroductionToVulkan#httpssoftwareintelcomen-usarticlesapi-without-secrets-introduction-to-vulkan-part-2)[https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-2](https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-2)

This lesson focuses on a swap chain creation. Swap chain enables us to display Vulkan-generated image in an application window. To display anything simple command buffers are allocated and recorded.

---

### [](https://github.com/GameTechDev/IntroductionToVulkan#03---first-triangle)[03 - First triangle](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Project/Tutorials/03)

[![](https://github.com/GameTechDev/IntroductionToVulkan/raw/master/Document/Images/03%20-%20First%20Triangle.png)](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Document/Images/03%20-%20First%20Triangle.png)

#### [](https://github.com/GameTechDev/IntroductionToVulkan#graphics-pipeline-and-drawing)Graphics pipeline and drawing

##### [](https://github.com/GameTechDev/IntroductionToVulkan#httpssoftwareintelcomen-usarticlesapi-without-secrets-introduction-to-vulkan-part-3)[https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-3](https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-3)

Here I present render pass, framebuffer and pipeline objects which are necessary to render arbitrary geometry. It is also shown how to convert GLSL shaders into SPIR-V and create shader modules from it.

---

### [](https://github.com/GameTechDev/IntroductionToVulkan#04---vertex-attributes)[04 - Vertex Attributes](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Project/Tutorials/04)

[![](https://github.com/GameTechDev/IntroductionToVulkan/raw/master/Document/Images/04%20-%20Vertex%20Attributes.png)](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Document/Images/04%20-%20Vertex%20Attributes.png)

#### [](https://github.com/GameTechDev/IntroductionToVulkan#buffers-memory-objects-and-fences)Buffers, memory objects and fences

##### [](https://github.com/GameTechDev/IntroductionToVulkan#httpssoftwareintelcomen-usarticlesapi-without-secrets-introduction-to-vulkan-part-4)[https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-4](https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-4)

This tutorial shows how to set up vertex attributes and bind buffer with a vertex data. Here we also create memory object (which is used by buffer) and fences.

---

### [](https://github.com/GameTechDev/IntroductionToVulkan#05---staging-resources)[05 - Staging Resources](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Project/Tutorials/05)

[![](https://github.com/GameTechDev/IntroductionToVulkan/raw/master/Document/Images/05%20-%20Staging%20Resources.png)](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Document/Images/05%20-%20Staging%20Resources.png)

#### [](https://github.com/GameTechDev/IntroductionToVulkan#copying-data-between-buffers)Copying data between buffers

##### [](https://github.com/GameTechDev/IntroductionToVulkan#httpssoftwareintelcomen-usarticlesapi-without-secrets-introduction-to-vulkan-part-5)[https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-5](https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-5)

In this example staging resources are presented. They are used as an intermediate resources for copying data between CPU and GPU. This way, resources involved in rendering can be bound only to a device local (very fast) memory.

---

### [](https://github.com/GameTechDev/IntroductionToVulkan#06---descriptor-sets)[06 - Descriptor Sets](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Project/Tutorials/06)

[![](https://github.com/GameTechDev/IntroductionToVulkan/raw/master/Document/Images/06%20-%20Descriptor%20Sets.png)](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Document/Images/06%20-%20Descriptor%20Sets.png)

#### [](https://github.com/GameTechDev/IntroductionToVulkan#using-textures-in-shaders)Using textures in shaders

##### [](https://github.com/GameTechDev/IntroductionToVulkan#httpssoftwareintelcomen-usarticlesapi-without-secrets-introduction-to-vulkan-part-6)[https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-6](https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-6)

This tutorial shows what resources are needed and how they should be prepared to be able to use textures (or other shader resources) in shader programs.

---

### [](https://github.com/GameTechDev/IntroductionToVulkan#07---uniform-buffers)[07 - Uniform Buffers](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Project/Tutorials/07)

[![](https://github.com/GameTechDev/IntroductionToVulkan/raw/master/Document/Images/07%20-%20Uniform%20Buffers.png)](https://github.com/GameTechDev/IntroductionToVulkan/blob/master/Document/Images/07%20-%20Uniform%20Buffers.png)

#### [](https://github.com/GameTechDev/IntroductionToVulkan#using-buffers-in-shaders)Using buffers in shaders

##### [](https://github.com/GameTechDev/IntroductionToVulkan#httpssoftwareintelcomen-usarticlesapi-without-secrets-introduction-to-vulkan-part-7)[https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-7](https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-7)

Here it is shown how to add uniform buffer to descriptor sets, how to provide data for projection matrix through it and how to use it inside shader.