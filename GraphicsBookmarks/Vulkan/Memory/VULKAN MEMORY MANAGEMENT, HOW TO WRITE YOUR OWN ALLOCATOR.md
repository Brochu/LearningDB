Hi ! This article will deal with the memory management in Vulkan. But first, I am going to tell you what happened in my life.

# State of my life

Again, it has been more than one month I did not write anything. So, where am I? I am in the last year of Télécom SudParis. I am following High Tech Imaging courses. It is the image specialization in my school. The funny part of it is : in parallel, I am a lecturer in a video games specialization. I taught OpenGL (3.3 because I cannot make an OpenGL 4 courses (everyone does not have a good hardware for that)). I got an internship in Dassault Systemes (France). It will begin the first February. I will work on the soft shadow engine (OpenGL 4.5).

# Vulkan

To begin, some articles that I wrote before this one can contain mistakes, or some things are not well explained, or not very optimized.

## Why came back to Vulkan?

I came back to Vulkan because I wanted to make one of the first “amateur” renderer using Vulkan. Also, I wanted to have a better improvement of memory management, memory barrier and other joys like that. Moreover, I made a repository with “a lot” of Vulkan Example : [Vulkan example repository](https://github.com/qnope/Vulkan-Example).  
I did not mean to replace the [Sascha Willems](https://github.com/SaschaWillems/Vulkan) ones. But I propose my way to do it, in C++, using [Vulkan HPP](https://github.com/KhronosGroup/Vulkan-Hpp).

## Memory Management with Vulkan

### Different kind of memory

#### Heap

One graphic card can read memory from different heap. It can read memory from its own heap, or the system heap (RAM).

#### Type

It exists a different kind of memory type. For example, it exists memories that are host cached, or host coherent, or device local and other.

##### Host and device

###### Host

This memory resides in the RAM. This heap should have generally one (or several) type that own the bit “HOST_VISIBLE”. It means to Vulkan that it could be mapped persistently. Going that way, you get the pointer and you can write from the CPU on it.

###### Device Local

This memory resides on the graphic card. It is freaking fast and is not generally _host_visible_. That means you have to use a _staging resource_ to write something to it or use the GPU itself.

### Allocation in Vulkan

In Vulkan, the number of allocation per heap is _driver limited_. That means you can not do a lot of allocation and you must not use one allocation by buffer or image but one allocation for several buffers and images.  
In this article, I will not take care about the CPU cache or anything like that, I will only focus my explanations on how to have the better from the GPU-side.  
![Memory Managements : good and bad](https://i1.wp.com/cpp-rendering.io/wp-content/uploads/2016/11/Dessin-sans-titre.png?resize=474%2C245&ssl=1)

### How will we do it?

![Memory Managements : device allocator](https://i2.wp.com/cpp-rendering.io/wp-content/uploads/2016/11/Dessin-sans-titre-1.png?resize=474%2C356&ssl=1)  
As you can see, we have a block, that could represent the memory for one buffer, or for one image, we have a chunk that represents one allocation (via vkAllocateMemory) and we have a DeviceAllocator that _manages_ all chunks.

#### Block

I defined a block as follow :

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

11

12

13

14

15

16

17

18

struct Block {

 vk::DeviceMemory memory;

 vk::DeviceSize offset;

 vk::DeviceSize size;

 bool free;

 void *ptr = nullptr; // Useless if it is a GPU allocation

 bool operator==(Block const &block);

};

bool Block::operator==(Block const &block) {

 if(memory == block.memory &&

 offset == block.offset &&

 size == block.size &&

 free == block.free &&

 ptr == block.ptr)

 return true;

 return false;

}

A block, as it is named, defines a little region within one allocation.  
So, it has an offset, one size, and a boolean to know if it is used or not.  
It **may** own a ptr if it is an

#### Chunk

A chunk is a _memory region_ that contains a list of blocks. It represents a single allocation.  
What a chunk could let us to do?

1.  Allocate a block
2.  Deallocate a block
3.  Tell us if the block is inside the chunk

That gives us:

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

11

12

13

14

15

16

17

18

19

20

21

22

#pragma once

#include "block.hpp"

class Chunk : private NotCopyable {

public:

 Chunk(Device &device, vk::DeviceSize size, int memoryTypeIndex);

 bool allocate(vk::DeviceSize size, Block &block);

 bool isIn(Block const &block) const;

 void deallocate(Block const &block);

 int memoryTypeIndex() const;

 ~Chunk();

protected:

 Device mDevice;

 vk::DeviceMemory mMemory = VK_NULL_HANDLE;

 vk::DeviceSize mSize;

 int mMemoryTypeIndex;

 std::vector<Block> mBlocks;

 void *mPtr = nullptr;

};

One chunk allocates its memory inside the constructor.

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

11

12

13

14

15

16

17

Chunk::Chunk(Device &device, vk::DeviceSize size, int memoryTypeIndex) :

 mDevice(device),

 mSize(size),

 mMemoryTypeIndex(memoryTypeIndex) {

 vk::MemoryAllocateInfo allocateInfo(size, memoryTypeIndex);

 Block block;

 block.free = true;

 block.offset = 0;

 block.size = size;

 mMemory = block.memory = device.allocateMemory(allocateInfo);

 if((device.getPhysicalDevice().getMemoryProperties().memoryTypes[memoryTypeIndex].propertyFlags & vk::MemoryPropertyFlagBits::eHostVisible) == vk::MemoryPropertyFlagBits::eHostVisible)

 mPtr = device.mapMemory(mMemory, 0, VK_WHOLE_SIZE);

 mBlocks.emplace_back(block);

}

Since a deallocation is really easy (only to put the block to free), one allocation requires a bit of attention. You need to check if the block is free, and if it is free, you need to check for its size, and, if necessary, create another block if the size of the allocation is less than the available size. You also need take care about memory alignment !

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

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

29

30

31

32

33

34

35

36

37

38

39

40

41

42

43

44

45

46

47

48

49

50

51

52

53

54

55

56

57

void Chunk::deallocate(const Block &block) {

 auto blockIt(std::find(mBlocks.begin(), mBlocks.end(), block));

 assert(blockIt != mBlocks.end());

 // Just put the block to free

 blockIt->free = true;

}

bool Chunk::allocate(vk::DeviceSize size, vk::DeviceSize alignment, Block &block) {

 // if chunk is too small

 if(size > mSize)

 return false;

 for(uint32_t i = 0; i < mBlocks.size(); ++i) {

 if(mBlocks[i].free) {

 // Compute virtual size after taking care about offsetAlignment

 uint32_t newSize = mBlocks[i].size;

 if(mBlocks[i].offset % alignment != 0)

 newSize -= alignment - mBlocks[i].offset % alignment;

 // If match

 if(newSize >= size) {

 // We compute offset and size that care about alignment (for this Block)

 mBlocks[i].size = newSize;

 if(mBlocks[i].offset % alignment != 0)

 mBlocks[i].offset += alignment - mBlocks[i].offset % alignment;

 // Compute the ptr address

 if(mPtr != nullptr)

 mBlocks[i].ptr = (char*)mPtr + mBlocks[i].offset;

 // if perfect match

 if(mBlocks[i].size == size) {

 mBlocks[i].free = false;

 block = mBlocks[i];

 return true;

 }

 Block nextBlock;

 nextBlock.free = true;

 nextBlock.offset = mBlocks[i].offset + size;

 nextBlock.memory = mMemory;

 nextBlock.size = mBlocks[i].size - size;

 mBlocks.emplace_back(nextBlock); // We add the newBlock

 mBlocks[i].size = size;

 mBlocks[i].free = false;

 block = mBlocks[i];

 return true;

 }

 }

 }

 return false;

}

#### Chunk Allocator

Maybe it is bad-named, but the chunk allocator let us to separate the creation of one chunk from the chunk itself. We give it one size and it operates all the verifications we need.

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

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

29

30

31

32

33

34

35

36

37

38

39

40

class ChunkAllocator : private NotCopyable

{

public:

 ChunkAllocator(Device &device, vk::DeviceSize size);

 // if size > mSize, allocate to the next power of 2

 std::unique_ptr<Chunk> allocate(vk::DeviceSize size, int memoryTypeIndex);

private:

 Device mDevice;

 vk::DeviceSize mSize;

};

vk::DeviceSize nextPowerOfTwo(vk::DeviceSize size) {

 vk::DeviceSize power = (vk::DeviceSize)std::log2l(size) + 1;

 return (vk::DeviceSize)1 << power;

}

bool isPowerOfTwo(vk::DeviceSize size) {

 vk::DeviceSize mask = 0;

 vk::DeviceSize power = (vk::DeviceSize)std::log2l(size);

 for(vk::DeviceSize i = 0; i < power; ++i)

 mask += (vk::DeviceSize)1 << i;

 return !(size & mask);

}

ChunkAllocator::ChunkAllocator(Device &device, vk::DeviceSize size) :

 mDevice(device),

 mSize(size) {

 assert(isPowerOfTwo(size));

}

std::unique_ptr<Chunk> ChunkAllocator::allocate(vk::DeviceSize size,

 int memoryTypeIndex) {

 size = (size > mSize) ? nextPowerOfTwo(size) : mSize;

 return std::make_unique<Chunk>(mDevice, size, memoryTypeIndex);

}

#### Device Allocator

I began to make an abstract class for Vulkan allocation :

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

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

/**

* @brief The AbstractAllocator Let the user to allocate or deallocate some blocks

*/

class AbstractAllocator : private NotCopyable

{

public:

 AbstractAllocator(Device const &device) :

 mDevice(std::make_shared<Device>(device)) {

 }

 virtual Block allocate(vk::DeviceSize size, vk::DeviceSize alignment, int memoryTypeIndex) = 0;

 virtual void deallocate(Block &block) = 0;

 Device getDevice() const {

 return *mDevice;

 }

 virtual ~AbstractAllocator() = 0;

protected:

 std::shared_ptr<Device> mDevice;

};

inline AbstractAllocator::~AbstractAllocator() {

}

As you noticed, it is really easy. You can allocate or deallocate from this allocator. Next, I created a DeviceAllocator that inherits from AbstractAllocator.

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

11

12

13

class DeviceAllocator : public AbstractAllocator

{

public:

 DeviceAllocator(Device device, vk::DeviceSize size);

 Block allocate(vk::DeviceSize size, vk::DeviceSize alignment, int memoryTypeIndex);

 void deallocate(Block &block);

private:

 ChunkAllocator mChunkAllocator;

 std::vector<std::shared_ptr<Chunk>> mChunks;

};

This allocator contains a list of chunks, and contains one ChunkAllocator to allocate chunks.  
The allocation is really easy. We have to check if it exists a “good chunk” and if we can allocate from it. Otherwise, we create another chunk and it is over !

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

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

DeviceAllocator::DeviceAllocator(Device device, vk::DeviceSize size) :

 AbstractAllocator(device),

 mChunkAllocator(device, size) {

}

Block DeviceAllocator::allocate(vk::DeviceSize size, vk::DeviceSize alignment, int memoryTypeIndex) {

 Block block;

 // We search a "good" chunk

 for(auto &chunk : mChunks)

 if(chunk->memoryTypeIndex() == memoryTypeIndex)

 if(chunk->allocate(size, alignment, block))

 return block;

 mChunks.emplace_back(mChunkAllocator.allocate(size, memoryTypeIndex));

 assert(mChunks.back()->allocate(size, alignment, block));

 return block;

}

void DeviceAllocator::deallocate(Block &block) {

 for(auto &chunk : mChunks) {

 if(chunk->isIn(block)) {

 chunk->deallocate(block);

 return ;

 }

 }

 assert(!"unable to deallocate the block");

}

# Conclusion

Since I came back to Vulkan, I really had a better understanding of this new API. I can write article in better quality than in march.  
I hope you enjoyed this remake of memory management.  
My next article will be about buffer, and staging resource. It will be a little article. I will write as well an article that explains how to load textures and their mipmaps.