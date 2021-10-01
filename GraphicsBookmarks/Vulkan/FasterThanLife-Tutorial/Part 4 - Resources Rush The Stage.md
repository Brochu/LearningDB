## I Am Graphics And So Can You :: Part 4 :: Resources Rush The Stage

[Graphics](https://www.fasterthan.life/blog/category/Graphics), [Vulkan](https://www.fasterthan.life/blog/category/Vulkan)

-   ### [PART 3 :: I AM GRAPHICS AND SO CAN YOU :: THE FIRST 1,000](https://www.fasterthan.life/blog/2017/7/12/i-am-graphics-and-so-can-you-part-3-breaking-ground)
    
-   ### [AN ASIDE :: I AM GRAPHICS AND SO CAN YOU :: MOTIVATION & EFFORT](https://www.fasterthan.life/blog/2017/7/15/i-am-graphics-and-so-can-you-effort)
    

In the previous article we went through how VkNeo sets up its Vulkan environment... with a few exceptions.  It's now time we revisited the things we skipped, and it all deals with resources.  Obviously the GPU can't draw anything it doesn't have access to.  That would be silly.  So how do we get data to the GPU and what does it look like?

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500008892596-MJXYNZWTW2CKRXIN0ZD2/image-asset.png?format=750w)

Behold the five data types VkNeo will be sending to the GPU.  We'll talk about four of these and leave pipelines for another article.  If you have some background in graphics, you're aware of the many forms data can take, but you'll also recognize these as the most common elements graphics programmers deal with.  If you're new to graphics, don't worry.  I'll cover the purpose of each of these in just a moment.  But first let's talk about what surrounds us, penetrates us, binds us to... memory it's memory.

# Vulkan Memory

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500010314374-WF9MR0E7K3FHLOM9PJ2M/image-asset.png?format=750w)

Sorry Jesse, it's right over here!!!

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500010420318-VLXE4CF0NTAXHFO44E3S/image-asset.jpeg?format=1000w)

All your data starts off CPU side.  In a typical hardware setup both the CPU and GPU maintain a region of memory which can be mapped by the other device.  This makes sharing data between the two much easier.  Still it's your job ( using Vulkan ) to make it visible to the GPU.  Vulkan provides several different types of memory.   Note that reading memory across the bus is obviously going to be slower so unless warranted it's best to avoid that whenever ( as long as ) possible.  

As always, let's look at some actual data, this time from the extremely useful [gpuinfo.org](http://www.gpuinfo.org/).  The site is written and maintained by [Sascha Willems](https://twitter.com/SaschaWillems2), and is a great database for details on graphics hardware info.  

![Fresh off a GTX 1080. &nbsp;Notice heap indices are shared by multiple memory types.](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500432802923-DJVKPGE7QHMER6ULG47L/image-asset.jpeg?format=750w)

Fresh off a GTX 1080.  Notice heap indices are shared by multiple memory types.

This is how Vulkan organizes memory.  When you call vkGetPhysicalDeviceMemoryProperties you get back what amounts to a list of heaps and memory types.  When allocating memory, you'll look through this and find a best match for your purposes.  Note: There must be at least one memory type with HOST_VISIBLE_BIT and HOST_COHERENT_BIT set.  Also there must be one memory type with just DEVICE_LOCAL_BIT set.  

### HOST_CACHED

Host cached memory means that GPU data is actually cached CPU side.  This is useful for when the CPU needs access to data coming off the GPU.  

### HOST_VISIBLE

Host visible memory can be mapped to the CPU for uploading/updating data.  This is very common for uniform buffers as the data contained therein is often updated every frame.  

### DEVICE_LOCAL

This memory is not visible to the host ( CPU ).  It is GPU RAM. It should be the main target of your allocations.  If it's not visible to the CPU then how do we get data to the GPU?  This is where command buffers come in handy.  Vulkan provides special commands to copy data to device local memory.  vkCmdCopy*.  

### HOST_COHERENT

This flag simply means that you don't need to manually flush writes to the device and likewise nothing "special" has to be done to receive write updates from the device.

Note: If you'd like to know more on this subject, head over to [gpuopen](http://gpuopen.com/vulkan-device-memory/) for a nice write up by [Timothy Lottes](https://twitter.com/TimothyLottes) from AMD.  

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500012262958-63VOG8430DFVGQ0JL832/image-asset.jpeg?format=750w)

So now we see that the CPU and GPU work in concert together, how do we do our part?  This is where I wrote two custom objects ( globals ) to handle memory management.  That would be idVulkanAllocator and idVulkanStagingManager.  idVulkanAllocator manages memory allocations while idVulkanStagingManager handles uploading allocations to the GPU.  

Let me just interject real quick that recently Adam Sawicki over at AMD came out with a Vulkan Allocator you can drag and drop into your game.  As memory management is something "hairy" for a lot of people, just know there are options open to you.  I have not had a chance to evaluate it yet, but it's great to have the contribution!

[AMD Vulkan Allocator](http://gpuopen.com/gaming-product/vulkan-memory-allocator/)

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500013155108-RBLR19J0FAMULLP4ZFE6/image-asset.jpeg?format=500w)

Vulkan can really catch you off guard here.  If you remember in Part 3 I briefly mentioned a VKPhysicalDeviceLimits struct?  Well amongst the 106 limitations there's one very important one that doesn't readily manifest itself when crossed.  This is maxMemoryAllocationCount.  That's right, there's an upper bound on how many allocations you can have!  At the time of writing my system only supported 4096.  That's not much.  So what do you do?  You actually just allocate in large pools, then sub allocate individual buffers, images, etc from them.  Easy peasy right?

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500013181816-MCDPKI25X5L7A93QCQEQ/image-asset.jpeg?format=500w)

# idVulkanAllocator

So after the warning above, you can see why a good allocator can save you and your application's life.  Now I'll introduce you to managing memory on Vulkan.  Primarily this global is a pool allocator.  It keeps separate pools for device local memory and host visible.  The reason for this is that multiple vkAllocation_t's share the same VkDeviceMemory.  Remember we have to sub allocate on our own.  Internally each pool manages a linked list of blocks.  Neighboring free blocks are merged when possible.  If a pool fills up a new one is created.  I won't focus so much on the pooling aspect as I will the Vulkan parts.  

// This struct is what gets passed around outside the allocator. 
// It's properties are used in various API calls where memory is referenced.
struct vkAllocation_t {
    vkAllocation_t() : 
        poolId( 0 ),
        deviceMemory( VK_NULL_HANDLE ), 
        offset( 0 ), 
        size( 0 ),
        data( NULL ) {
    }

    uint32          poolId;         // Which pool did this come from
    uint32          blockId;        // What memory block is this
    VkDeviceMemory  deviceMemory;   // The Vulkan device memory associated with this block.
    VkDeviceSize    offset;         // The offset into the data below.
    VkDeviceSize    size;           // The size of the memory associated with this block.
    byte *          data;           // If the data is host visible we can map it.
};

/*
================================================================================================

idVulkanMemoryPool

================================================================================================
*/

class idVulkanMemoryPool {
friend class idVulkanAllocator;
public:
    idVulkanMemoryPool( const uint32 id, const uint32 memoryTypeBits, const VkDeviceSize size, const bool hostVisible );
    ~idVulkanMemoryPool();

    bool                Init();
    void                Shutdown();

    // Return true/false for success.  If true vkAllocation_t reference is filled.
    bool                Allocate( 
                            const uint32 size, 
                            const uint32 align, 
                            vkAllocation_t & allocation );
    void                Free( vkAllocation_t & allocation );

private:
    // The pool is a simple linked list of allocated blocks.
    // If a block and its neighbor are free, then they are merged
    // back together in one block.
    struct block_t {
        uint32         id;
        VkDeviceSize   size;
        VkDeviceSize   offset;
        block_t *      prev;
        block_t *      next;
        bool           free;
    };
    block_t *           m_head;

    uint32              m_id;
    uint32              m_nextBlockId;
    uint32              m_memoryTypeIndex;
    bool                m_hostVisible;
    VkDeviceMemory      m_deviceMemory;
    VkDeviceSize        m_size;
    VkDeviceSize        m_allocated;
    byte *              m_data;
};

/*
================================================================================================

idVulkanAllocator

================================================================================================
*/

class idVulkanAllocator {
public:
    idVulkanAllocator();

    void                    Init();

    // This is the external allocation call which goes through
    // the private AllocateFromPools below.  This in turn can
    // reach idVulkanMemoryPool::Allocate
    vkAllocation_t          Allocate( 
                                const uint32 size, 
                                const uint32 align, 
                                const uint32 memoryTypeBits, 
                                const bool hostVisible );
    void                    Free( const vkAllocation_t allocation );

    // There are several places in the code base, where I don't
    // immediately free Vulkan resources.  Instead I add them to 
    // a garbage list which I can later cleanup.  This is useful
    // as it gives Vulkan a chance to finish up with resources 
    // bound to command buffers still in flight.
    void                    EmptyGarbage();

private:
    bool                    AllocateFromPools( 
                                const uint32 size, 
                                const uint32 align, 
                                const uint32 memoryTypeBits,
                                const bool hostVisible,
                                vkAllocation_t & allocation );

private:
    int                            m_nextPoolId;
    int                            m_garbageIndex;

    // How big should each pool be when created?
    int                            m_deviceLocalMemoryMB;
    int                            m_hostVisibleMemoryMB;

    idList< idVulkanMemoryPool *>  m_pools;
    idList< vkAllocation_t >       m_garbage[ NUM_FRAME_DATA ];
};

extern idVulkanAllocator vulkanAllocator;

The implementation of idVulkanAllocator is actually just over 400 lines of code.  So apart from the complexity of knowing what's going on with Vulkan, the code is rather straight forward.

/*
=============
idVulkanAllocator::Allocate
=============
*/
vkAllocation_t idVulkanAllocator::Allocate( 
		const uint32 size, 
		const uint32 align, 
		const uint32 memoryTypeBits, 
		const bool hostVisible ) {
	
	vkAllocation_t allocation;

	// First try to allocate from an existing pool
	if ( AllocateFromPools( size, align, memoryTypeBits, hostVisible, allocation ) ) {
		return allocation;
	}

	// If we couldn't allocate from a pool we'll need to create a new one.
	VkDeviceSize poolSize = hostVisible ? m_hostVisibleMemoryMB : m_deviceLocalMemoryMB;

	idVulkanMemoryPool * pool = new idVulkanMemoryPool( m_nextPoolId++, memoryTypeBits, poolSize, hostVisible );
	if ( pool->Init() ) {
		m_pools.Append( pool );
	} else {
		idLib::FatalError( "idVulkanAllocator::Allocate: Could not allocate new memory pool." );
	}

	// Now that we've added the new pool, allocate directly from it.
	pool->Allocate( size, align, allocation );

	return allocation;
}

No surprises there.  Let's continue on with the actual AllocateFromPools.

/*
=============
idVulkanAllocator::AllocateFromPools
=============
*/
bool idVulkanAllocator::AllocateFromPools( 
		const uint32 size, 
		const uint32 align, 
		const uint32 memoryTypeBits,
		const bool needHostVisible,
		vkAllocation_t & allocation ) {

	// Remember in Part 3 we got the memory properties when enumerating devices?  That comes in handy here. (vkcontext.gpu->memProps)
	const VkPhysicalDeviceMemoryProperties & physicalMemoryProperties = vkcontext.gpu->memProps;
	const int num = m_pools.Num();

	// If we need host visible memory set the appropriate bits.  Otherwise we assume device local.
	const VkMemoryPropertyFlags required = needHostVisible ? VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT : 0;
	// Because it offers the best performance we prefer it, but don't require it.
	const VkMemoryPropertyFlags preferred = VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT;

	// Now iterate over the pools looking for a suitable one.
	// This is the first attempt of two. 
	// 1.) We look for both required and preferred flags.
	// 2.) We look for just required flags.
	for ( int i = 0; i < num; ++i ) {
		idVulkanMemoryPool * pool = m_pools[ i ];
		const uint32 memoryTypeIndex = pool->m_memoryTypeIndex;

		// If we need host visible and this pool doesn't support it continue
		if ( needHostVisible && pool->m_hostVisible == false ) {
			continue;
		}

		// vkcontext.gpu->memProps enumerates the number of memory types supported.
		// When we create a pool we assign it a given index.  That's what we try to 
		// match here.
		if ( ( ( memoryTypeBits >> memoryTypeIndex ) & 1 ) == 0 ) {
			continue;
		}

		// Now try to match our required memory flags.
		const VkMemoryPropertyFlags properties = physicalMemoryProperties.memoryTypes[ memoryTypeIndex ].propertyFlags;
		if ( ( properties & required ) != required ) {
			continue;
		}

		// Now try to match our preferred memory flags.
		if ( ( properties & preferred ) != preferred ) {
			continue;
		}

		// If this matches both required and preferred, go ahead and allocate from the pool.
		if ( pool->Allocate( size, align, allocation ) ) {
			return true;
		}
	}

	// On the second attempt we just look for required flags.  Otherwise it's the same as above.
	for ( int i = 0; i < num; ++i ) {
		idVulkanMemoryPool * pool = m_pools[ i ];
		const uint32 memoryTypeIndex = pool->m_memoryTypeIndex;

		if ( needHostVisible && pool->m_hostVisible == false ) {
			continue;
		}

		if ( ( ( memoryTypeBits >> memoryTypeIndex ) & 1 ) == 0 ) {
			continue;
		}

		const VkMemoryPropertyFlags properties = physicalMemoryProperties.memoryTypes[ memoryTypeIndex ].propertyFlags;
		if ( ( properties & required ) != required ) {
			continue;
		}

		if ( pool->Allocate( size, align, allocation ) ) {
			return true;
		}
	}

	return false;
}

Allocating for vulkan centers largely around the memory properties you're looking for.  That's one reason I went over them explicitly.  Now instead of looking at the Allocate function for the pool, we're going to look at the Pool's Init and Shutdown functions.  The reason is because Allocate is akin to any run of the mill block allocator you may have seen.  And you're here to see Vulkan.

/*
=============
idVulkanMemoryPool::idVulkanMemoryPool
=============
*/
idVulkanMemoryPool::idVulkanMemoryPool( const uint32 id, const uint32 memoryTypeBits, const VkDeviceSize size, const bool hostVisible ) : 
	m_id( id ),
	m_nextBlockId( 0 ),
	m_size( size ),
	m_allocated( 0 ),
	m_hostVisible( hostVisible ) {

	// Determine a suitable memory type index for this pool.
	// This is basically the same thing as idVulkanAllocator's AllocateFromPools
	m_memoryTypeIndex = VK_FindMemoryTypeIndex( memoryTypeBits, hostVisible );
}

/*
=============
idVulkanMemoryPool::Init
=============
*/
bool idVulkanMemoryPool::Init() {
	if ( m_memoryTypeIndex == UINT64_MAX ) {
		return false;
	}

	// Yes it really is this easy ( once you get here ).
	// Simply tell Vulkan the size and memory index you want to use.
	VkMemoryAllocateInfo memoryAllocateInfo = {};
	memoryAllocateInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
	memoryAllocateInfo.allocationSize = m_size;
	memoryAllocateInfo.memoryTypeIndex = m_memoryTypeIndex;

	ID_VK_CHECK( vkAllocateMemory( vkcontext.device, &memoryAllocateInfo, NULL, &m_deviceMemory ) )

	if ( m_deviceMemory == VK_NULL_HANDLE ) {
		return false;
	}

	// If this is host visible we map it into our address space using m_data.
	if ( m_hostVisible ) {
		ID_VK_CHECK( vkMapMemory( vkcontext.device, m_deviceMemory, 0, m_size, 0, (void **)&m_data ) );
	}

	// Setup the first block.
	m_head = new block_t();
	m_head->size = m_size;
	m_head->offset = 0;
	m_head->prev = NULL;
	m_head->next = NULL;
	m_head->free = true;

	return true;
}

/*
=============
idVulkanMemoryPool::Shutdown
=============
*/
void idVulkanMemoryPool::Shutdown() {
	// Unmap the memory
	if ( m_hostVisible ) {
		vkUnmapMemory( vkcontext.device, m_deviceMemory );
	}

	// Free the memory
	vkFreeMemory( vkcontext.device, m_deviceMemory, NULL );

	block_t * prev = NULL;
	block_t * current = m_head;
	while ( 1 ) {
		if ( current->next == NULL ) {
			delete current;
			break;
		} else {
			prev = current;
			current = current->next;
			delete prev;
		}
	}

	m_head = NULL;
}

I want to pause a minute and point something out I mentioned from the beginning of this series.  Note how the road to get to allocate/free was long and ridden with details?  ( Yep who can forget )  But at the parts where the rubber meets the road, the code is lean and mean.  This is Vulkan to a T.  

So how are the allocations used?  I don't want to get ahead of myself, but I think an example is warranted.

ID_VK_CHECK( vkCreateImage( vkcontext.device, &imageCreateInfo, NULL, &m_image ) );

VkMemoryRequirements memoryRequirements;
vkGetImageMemoryRequirements( vkcontext.device, m_image, &memoryRequirements );

m_allocation = vulkanAllocator.Allocate( 
	memoryRequirements.size,
	memoryRequirements.alignment,
	memoryRequirements.memoryTypeBits, 
	false );

ID_VK_CHECK( vkBindImageMemory( vkcontext.device, m_image, m_allocation.deviceMemory, m_allocation.offset ) );

Again things are lean and mean.  Vulkan actually helps us out by giving us the size, alignment, and memory type bits it expects for a given resource.  In this case we just created an image.  Every time we create a resource we should ask Vulkan for its memory requirements.  This then gets fed to the allocator.  The results from the allocator are then used to bind the resource to the vulkan memory!!  Pretty exciting eh?

# idVulkanStagingManager

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500018316282-WJ3TG3BEI05XIF7PGC0R/image-asset.jpeg?format=750w)

The staging manager exclusively handles device local memory.  Because we can't map device memory without the host visible bit, we have to use the command buffers to instruct upload that data.  Currently in VkNeo, the only vulkan resources that utilize this global system are idImages.  Why not just allocate inline?  Well this is actually an optimization.  We could allocate everything inline, but to do so we'd need to create and destroy a command buffer for each image.  We'd also have to manage synchronization for that image work being done.  Instead we just allocate a big pool and push multiple images at once.  Let's take a look.

// This looks very similar to vkAllocation_t right?
// The only difference is the addition of a VkCommandBuffer property.
struct stagingBuffer_t {
    stagingBuffer_t() :
        submitted( false ),
        commandBuffer( VK_NULL_HANDLE ),
        buffer( VK_NULL_HANDLE ),
        fence( VK_NULL_HANDLE ),
        offset( 0 ),
        data( NULL ) {}

    bool                submitted;
    VkCommandBuffer     commandBuffer;
    VkBuffer            buffer;
    VkFence             fence;
    VkDeviceSize        offset;
    byte *              data;
};

class idVulkanStagingManager {
public:
    idVulkanStagingManager();
    ~idVulkanStagingManager();

    void            Init();
    void            Shutdown();

    byte *          Stage( const int size, const int alignment, VkCommandBuffer & commandBuffer, VkBuffer & buffer, int & bufferOffset );
    
    // Flush will drain all data for the current staging buffer.
    void            Flush();

private:
    // This waits until the command buffer carrying the copy commands is done.
    void            Wait( stagingBuffer_t & stage );

private:
    // If the maximum size is hit, the staging manager will swap buffers and flush
    int              m_maxBufferSize;
    int              m_currentBuffer;
    byte *           m_mappedData;
    VkDeviceMemory   m_memory;
    VkCommandPool    m_commandPool;

    // Like so many other things, we double buffer so one command buffer can be in flight.
    stagingBuffer_t m_buffers[ NUM_FRAME_DATA ];
};

extern idVulkanStagingManager stagingManager;

The interesting part isn't what's going on inside, but the bits that are using it.  And that brings us to our first actual resource.  Images.

# Image Resources

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500019759869-FBS4JZHQ8PMCIGMT6N5D/image-asset.jpeg?format=750w)

In Part 3 we actually touched on Vulkan images when we got the SwapChain and created views for it.  The thing I didn't go over was actually allocating the image itself, because with the SwapChain the device provides them.  Now we'll actually go over the entire allocation process which demonstrates everything we've learned so far in this article.

### ALLOCATE IMAGE

/*
====================
idImage::AllocImage
====================
*/
void idImage::AllocImage() {
	PurgeImage();

	m_internalFormat = VK_GetFormatFromTextureFormat( m_opts.format );

	// Create Sampler
	CreateSampler();

	// VkNeo's images will always be sampled
	VkImageUsageFlags usageFlags = VK_IMAGE_USAGE_SAMPLED_BIT;
	if ( m_opts.format == FMT_DEPTH ) {
		// Map the idTech depth format to the Vulkan one.
		usageFlags |= VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT;
	} else {
		// For everything else we mark it as a transfer destination.
		// This is because we are copying data to the image.
		usageFlags |= VK_IMAGE_USAGE_TRANSFER_DST_BIT ;
	}

	// Create Image
	// Most of the attributes listed below should make sense. Par for the course.
	VkImageCreateInfo imageCreateInfo = {};
	imageCreateInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
	imageCreateInfo.flags = ( m_opts.textureType == TT_CUBIC ) ? VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT: 0;
	imageCreateInfo.imageType = VK_IMAGE_TYPE_2D;
	imageCreateInfo.format = m_internalFormat;
	imageCreateInfo.extent.width = m_opts.width;
	imageCreateInfo.extent.height = m_opts.height;
	imageCreateInfo.extent.depth = 1;
	imageCreateInfo.mipLevels = m_opts.numLevels;
	imageCreateInfo.arrayLayers = ( m_opts.textureType == TT_CUBIC ) ? 6 : 1;
	imageCreateInfo.samples = VK_SAMPLE_COUNT_1_BIT; // TODO_VK can probably do more
	imageCreateInfo.tiling = VK_IMAGE_TILING_OPTIMAL;
	imageCreateInfo.usage = usageFlags;

	// New Concept: Image Layouts
	// In Vulkan images have layouts which determine what sorts of operations can
	// be performed on it at any given time.  The other reason is it helps the hardware
	// optimize access to the image data for certain tasks.
	// Here we are starting out undefined, simply because we'll be transitioning the
	// image again during upload.
	imageCreateInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;

	// Unless this is a render target we don't need to worry about sharing it.
	imageCreateInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
	
	// Create the images
	ID_VK_CHECK( vkCreateImage( vkcontext.device, &imageCreateInfo, NULL, &m_image ) );

	// As with any Vulkan resource we want to get its memory requirements after creating it.
	// This is because although we've created the resource, we still need to back it with memory.
	VkMemoryRequirements memoryRequirements;
	vkGetImageMemoryRequirements( vkcontext.device, m_image, &memoryRequirements );

	// Utillize the idVulkanAllocator
	m_allocation = vulkanAllocator.Allocate( 
		memoryRequirements.size,
		memoryRequirements.alignment,
		memoryRequirements.memoryTypeBits, 
		false );

	// Bind the Vulkan image to the memory provided via the idVulkanAllocator
	ID_VK_CHECK( vkBindImageMemory( vkcontext.device, m_image, m_allocation.deviceMemory, m_allocation.offset ) );

	// Create Image View
	// As mentioned in Part 3 ( again in the swapchain function ) views are how you interface with a given image.
	// You can think of the VkImage as the actual data and VkImageView as your view into the image.
	VkImageViewCreateInfo viewCreateInfo = {};
	viewCreateInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
	viewCreateInfo.image = m_image;
	viewCreateInfo.viewType = ( m_opts.textureType == TT_CUBIC ) ? VK_IMAGE_VIEW_TYPE_CUBE : VK_IMAGE_VIEW_TYPE_2D;
	viewCreateInfo.format = m_internalFormat;
	viewCreateInfo.components = VK_GetComponentMappingFromTextureFormat( m_opts.format, m_opts.colorFormat );
	viewCreateInfo.subresourceRange.aspectMask = ( m_opts.format == FMT_DEPTH ) ? VK_IMAGE_ASPECT_DEPTH_BIT | VK_IMAGE_ASPECT_STENCIL_BIT : VK_IMAGE_ASPECT_COLOR_BIT;
	viewCreateInfo.subresourceRange.levelCount = m_opts.numLevels;
	viewCreateInfo.subresourceRange.layerCount = ( m_opts.textureType == TT_CUBIC ) ? 6 : 1;
	viewCreateInfo.subresourceRange.baseMipLevel = 0;
	
	ID_VK_CHECK( vkCreateImageView( vkcontext.device, &viewCreateInfo, NULL, &m_view ) );
}

### UPLOAD IMAGE

/*
====================
idImage::SubImageUpload
====================
*/
void idImage::SubImageUpload( int mipLevel, int x, int y, int z, int width, int height, const void * pic, int pixelPitch ) {
	assert( x >= 0 && y >= 0 && mipLevel >= 0 && width >= 0 && height >= 0 && mipLevel < m_opts.numLevels );

	if ( IsCompressed() ) {
		width = ( width + 3 ) & ~3;
		height = ( height + 3 ) & ~3;
	}

	int size = width * height * BitsForFormat( m_opts.format ) / 8;

	// This is where we use the staging manager to "stage" an image upload.
	VkBuffer buffer;
	VkCommandBuffer commandBuffer;
	int offset = 0;
	// Returns a pointer to memory we can copy the image into.
	byte * data = stagingManager.Stage( size, 16, commandBuffer, buffer, offset );
	
	// For most internal formats we simply memcpy into the buffer.
	// However, for RGB565 we reverse the bytes 5=red, 6=green, 5=blue = 16 bits = 2 bytes
	if ( m_opts.format == FMT_RGB565 ) {
		byte * imgData = (byte *)pic;
		for ( int i = 0; i < size; i += 2 ) {
			data[ i ] = imgData[ i + 1 ];
			data[ i + 1 ] = imgData[ i ];
		}
	} else {
		memcpy( data, pic, size );
	}

	// The VkBufferImageCopy struct is what allows us to specify the upload.
	VkBufferImageCopy imgCopy = {};
	imgCopy.bufferOffset = offset;
	imgCopy.bufferRowLength = pixelPitch;
	imgCopy.bufferImageHeight = height;
	imgCopy.imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
	imgCopy.imageSubresource.layerCount = 1;
	imgCopy.imageSubresource.mipLevel = mipLevel;
	imgCopy.imageSubresource.baseArrayLayer = z;
	imgCopy.imageOffset.x = x;
	imgCopy.imageOffset.y = y;
	imgCopy.imageOffset.z = 0;
	imgCopy.imageExtent.width = width;
	imgCopy.imageExtent.height = height;
	imgCopy.imageExtent.depth = 1;

	// As mentioned in the allocate function, images have layouts. 
	// To change layouts we need to transition them using image barriers.
	// This ensures proper access to the image memory for operations trying to use it.
	VkImageMemoryBarrier barrier = {};
	barrier.sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER;
	// We don't care about the queues.
	barrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
	barrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
	barrier.image = m_image;
	barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
	barrier.subresourceRange.baseMipLevel = 0;
	barrier.subresourceRange.levelCount = m_opts.numLevels;
	barrier.subresourceRange.baseArrayLayer = z;
	barrier.subresourceRange.layerCount = 1;
	
	// In the allocate function we set the layout as UNDEFINED. 
	// Now we want to transition to TRANSFER_DST_OPTIMAL so we can write to it.
	barrier.oldLayout = VK_IMAGE_LAYOUT_UNDEFINED;
	barrier.newLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
	barrier.srcAccessMask = 0;
	// Access is limited to transfer writes
	barrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
	
	// Here we're inserting the pipeline barrier.
	// NOTE: We are inserting this into the commandBuffer, and not executing it right away.
	// Because this is the first time we're encountering pipeline barriers, I'll go into it a bit more.
	vkCmdPipelineBarrier(
		// The command buffer we're inserting the barrier into. ( order matters, but it also gives us guarantees )
		commandBuffer,

		// Source Stage Mask: Specifies which stage of the pipeline wrote to the resource last.
		// I haven't optimized this yet so I'm using the safest stage mask which waits for everything.
		VK_PIPELINE_STAGE_ALL_COMMANDS_BIT,	

		// Destination Stage Mask: Specifies which stage of the pipeline writes to the resource next.
		// Essentially this waits till all "graphical" operations have completed.
		VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT,

		// Dependency Flags: This can be ignored for VkNeo
		0, 

		// Memory Barriers: We're only submitting image barriers here, so leave this at a num=0, and data=NULL
		0, 
		NULL, 

		// Buffer Memory Barriers: Same as above
		0, 
		NULL, 

		// Image Barriers: Here we actually use the barrier we created.  Set num=1
		1, 
		&barrier );

	// Now that the barrier transitioning the layout to TRANSFER_DST_OPTIMAL has been inserted into the command buffer
	// we can go ahead and add the copy command.  Remember this is device local memory, that's why we're having to do this.
	vkCmdCopyBufferToImage( commandBuffer, buffer, m_image, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, 1, &imgCopy );

	// Now we just reuse the barrier to insert another one. 
	// This time though we're transitioning from TRANSFER_DST_OPTIMAL to SHADER_READ_ONLY_OPTIMAL
	// Neo is an older game so it does little else with textures besides read them from shaders.
	barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
	barrier.newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
	barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
	barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

	// Insert the final barrier
	vkCmdPipelineBarrier( commandBuffer, VK_PIPELINE_STAGE_ALL_COMMANDS_BIT, VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT, 0, 0, NULL, 0, NULL, 1, &barrier );

	// Record what our current layout is.
	m_layout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
}

SubImageUpload is called for each image in an idImage.  For example a cube image will have six.  

So in just two functions we've handled quite a bit of image management and introduced some interesting topics.  One thing that really sticks out for Vulkan is the idea of layout transitions.  As I mentioned in the code comments, this exists to optimize and restrict how image memory is accessed by various functionality.  This fine grain control is part of what makes Vulkan so powerful.  Digging into the benefits will take study and experimentation on your part though.

# Buffering

![giphy.gif](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500094250499-72OD5D89YQ7LFZY3X0ZM/giphy.gif?format=750w)

Wow we've come a long way already this article.  Time for a drink.  Aaaahhhh delicious.  Now, we've only talked about one type of resource so far.  But just like all things Vulkan, the effort is front loaded.  The remaining three will go fast because they're all pretty much the same on the application side.  These include the Vertex, Index, and Uniform buffers.  

All three work in concert to properly represent things you'll see on the screen.  Instead of stopping to describe them in detail, let's look at an actual example from VkNeo.  Back to RenderDoc!

Let's look at a frame capture of just the shotgun and we'll inspect each of the buffers.

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500097271070-45HC0VK4WP0FX17KFB9Y/image-asset.jpeg?format=750w)

All VkNeo's draw calls are performed with vkCmdDrawIndexed.  ( We'll get to drawing in a later tutorial. )  For the image above the call is made with 2337 indices.  Let's look at some of the data in RenderDoc.

### VERTEX & INDEX BUFFERS

View fullsize

![Click to Enlarge](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500098237083-ORX4XY7N5QXS8SE5URXS/image-asset.jpeg?format=750w)

Click to Enlarge

You'll notice on the top left we have the input data for the vertex buffer.  On the upper right are the vertex shader outputs.  Each line entry is a shader invocation. idDrawVert is 32 bytes with the first 12 being position data.  As I've mentioned before the vertex shader is primarily concerned with positioning things.  In the input window you'll see a column labeled in_Position.  This is the first 12 bytes of an idDrawVert that was uploaded to the GPU.  To the left you'll see two columns VTX and IDX.  VTX is the vertex index for the draw call.  So zero would be the first.  IDX is the actual index number in the index buffer.  DOOM 3 BFG is still a "small enough" game that it uses 16 bit indices ( 2 bytes ).  That means dealing with index buffers is much cheaper than vertex buffers, and that's why draw indexed calls exist in most graphics APIs; Vulkan being no different.  

On the right is the output as I mentioned.  Again it has VTX and IDX.  Instead of in_Position it has gl_PerVertex.gl_Position.  This is the typical output for a vertex location as written by the vertex shader.  The indices on the left and right match up so output at zero will always be input at zero.  ( Baldur did a bang up job with this interface )

I'll show you how to populate these buffers with data, but first... why did the vertex shader change the position data?  Well that's all thanks to a uniform buffer.  

### UNIFORM BUFFER

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500098866561-Y4VWL0OPRXNRUBJRBXGP/image-asset.jpeg?format=750w)

Attached to the same vertex shader invocation that gave us the vertex outputs above are two uniform buffers or UBOs.  Pictured above is the first.  ( The second is a skinning buffer. If you don't know what that is, it's not necessary to understand at the moment. )  The UBO is constant for the length of the draw call.  

Again you'll notice Baldur made things super nice by showing the name of each attribute in the buffer.  I won't go into each, but notice the 4x lines at the bottom.  This is the model-view-projection matrix for the current player's view.  This is what's used to transform the vertex data on the left into the vertex data on the right.  If you're unfamiliar with matrices or linear alegebra in general don't worry, I'll touch on this a bit when it comes time to talk about drawning ( future article ).

For beginners, don't get too hung up on the names of things.  It's all data.  It's just data married to a format/layout.  Like I mentioned at the beginning of this article, your job is to get it to the GPU.  If you know how to allocate, free, map, unmap, and copy memory around, then you're set.  The same principles apply here.  Just because it has a big name, don't be scared of it.  So let's look at how we concretely allocate buffers.

### BUFFER OBJECTS

/*
========================
idUniformBuffer::AllocBufferObject
========================
*/
bool idUniformBuffer::AllocBufferObject( const void * data, int allocSize ) {
	assert( m_apiObject == VK_NULL_HANDLE );
	assert_16_byte_aligned( data );

	if ( allocSize <= 0 ) {
		idLib::Error( "idUniformBuffer::AllocBufferObject: allocSize = %i", allocSize );
	}

	// Store the requested size
	m_size = allocSize;

	bool allocationFailed = false;

	const int numBytes = GetAllocedSize();

	// VkBufferCreateInfo is used to create all three buffer types we've talked about.
	VkBufferCreateInfo bufferCreateInfo = {};
	bufferCreateInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
	bufferCreateInfo.pNext = NULL;
	bufferCreateInfo.size = numBytes;

	// You can simply change the usage to whatever buffer you need.
	// VK_BUFFER_USAGE_INDEX_BUFFER_BIT
	// VK_BUFFER_USAGE_VERTEX_BUFFER_BIT
	// etc
	bufferCreateInfo.usage = VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT;

	// Create the buffer resource
	VkResult ret = vkCreateBuffer( vkcontext.device, &bufferCreateInfo, NULL, &m_apiObject );
	assert( ret == VK_SUCCESS );

	// Like with images we need to get the memory requirements
	VkMemoryRequirements memoryRequirements;
	vkGetBufferMemoryRequirements( vkcontext.device, m_apiObject, &memoryRequirements );

	// Utilize the idVulkanAllocator, again just like with image resources.
	m_allocation = vulkanAllocator.Allocate( 
		memoryRequirements.size, 
		memoryRequirements.alignment, 
		memoryRequirements.memoryTypeBits, 
		true );

	// Back the resource with the given allocation, again just like with image resources.
	ID_VK_CHECK( vkBindBufferMemory( vkcontext.device, m_apiObject, m_allocation.deviceMemory, m_allocation.offset ) );

	if ( r_showBuffers.GetBool() ) {
		idLib::Printf( "joint buffer alloc %p, (%i bytes)\n", this, GetSize() );
	}

	// Copy the data
	// This is really just a memcpy.  If you know how to do that, you're set.
	if ( data != NULL ) {
		Update( data, allocSize );
	}

	return !allocationFailed;
}

This looks awfully similar, yet stunningly simpler than image allocation doesn't it?  That's because it is.  All three buffer types interact with Vulkan in the same way from VkNeo's perspective.  It's application side details that are different, warranting different implementations.  

So once we have the buffers, how are they used?  Well in VkNeo it's rather simple.  Remember vertexCache from Part 3?  Well it double buffers frame data consisting of a vertex buffer, index buffer, and uniform buffer ( for skeleton joints, skinning ).  This data is backed by the buffer classes like the one I just showed you.  Then systems that need to submit data just allocate from the vertex cache and fill on their own.  ( guis, model allocation, skinning data, etc )

/*
==============
MapGeoBufferSet
==============
*/
static void MapGeoBufferSet( geoBufferSet_t &gbs ) {
	if ( gbs.mappedVertexBase == NULL ) {
		gbs.mappedVertexBase = (byte *)gbs.vertexBuffer.MapBuffer( BM_WRITE );
	}
	if ( gbs.mappedIndexBase == NULL ) {
		gbs.mappedIndexBase = (byte *)gbs.indexBuffer.MapBuffer( BM_WRITE );
	}
	if ( gbs.mappedJointBase == NULL && gbs.jointBuffer.GetAllocedSize() != 0 ) {
		gbs.mappedJointBase = (byte *)gbs.jointBuffer.MapBuffer( BM_WRITE );
	}
}

/*
==============
UnmapGeoBufferSet
==============
*/
static void UnmapGeoBufferSet( geoBufferSet_t &gbs ) {
	if ( gbs.mappedVertexBase != NULL ) {
		gbs.vertexBuffer.UnmapBuffer();
		gbs.mappedVertexBase = NULL;
	}
	if ( gbs.mappedIndexBase != NULL ) {
		gbs.indexBuffer.UnmapBuffer();
		gbs.mappedIndexBase = NULL;
	}
	if ( gbs.mappedJointBase != NULL ) {
		gbs.jointBuffer.UnmapBuffer();
		gbs.mappedJointBase = NULL;
	}
}

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500102465392-82SN32AG9Y8BE6L4P8L8/image-asset.gif?format=750w)

Congrats on making it through arguably the most boring part of Vulkan!  But this is all well and good.  It lays the groundwork for the next two articles which will be the most exciting.  In them we'll cover pipelines ( shaders, etc ) and rendering itself.  So sit tight, grab a drink, and cheers to your success.