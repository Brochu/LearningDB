# Memory Allocation Strategies - Part 1

## Thinking About Memory and Allocation

Series: [Memory Allocation Strategies](https://www.gingerbill.org/series/memory-allocation-strategies)

2019-02-01

Memory allocation seems to be something many people struggle with. Many languages try to automatically handle memory for you using different strategies: garbage collection (GC), automatic reference counting (ARC), resource acquisition is initialization (RAII), and ownership semantics. However, trying to abstract away memory allocation comes at a higher cost than most people realize.

Most people are taught to think of memory in terms of the stack and the heap, where the stack is automatically grown for a procedure call, and the heap is some magical thing that you can use to get memory that needs to live longer than the stack. This dualistic approach to memory is the wrong way to think about it. It gives the programmer the mental model that the stack is a special form of memory[1](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#fn:1) and that the heap is magical in nature.

Modern operating systems virtualize memory on a per-process basis. This means that the addresses used within your program/process are specific to that program/process only. Due to operating systems virtualizing the memory space for us, this allows us to think about memory in a completely different way. Memory is not longer this dualistic model of _the stack_ and _the heap_ but rather a monistic model where everything is virtual memory. Some of that virtual address space is reserved for procedure stack frames, some of it is reserved for things required by the operating system, and the rest we can use for whatever we want. This may sound similar to original dualistic model that I stated previously, however, the biggest difference is realizing that the memory is virtually-mapped and linear, and that you can split that linear memory space in sections.

![Virtual Memory](https://www.gingerbill.org/images/memory-allocation-strategies/virtual_memory.svg#center)

# [Thinking About Allocation](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#thinking-about-allocation)

When it comes to allocation, there are three main aspects to think about[2](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#fn:2):

-   The size of the allocation
-   The lifetime of that memory
-   The usage of that memory

I usually imagine the first two aspects in the following table, for most problem domains, where the percentages signify what proportion of allocations fall into that category:

Size Known

Size Unknown

**Lifetime Known**

95%

~4%

**Lifetime Unknown**

~1%

<1%

In the top-left category (Size Known + Lifetime Known), this is the area in which I will be covering the most in this series. Most of the time, you do know the size of the allocation, or the upper bounds at least, and the lifetime of the allocation in question.

In the top-right category (Size Unknown + Lifetime Known), this is the area in which you may not know how much memory you require but you do know how long you will be using it. The most common examples of this are loading a file into memory at runtime and populating a hash table of unknown size. You may not know the amount of memory you will need a priori and as a result, you may need to “resize/realloc” the memory in order to fit all the data required. In C, `malloc` et al is a solution to this domain of problems.

In the bottom-left category (Size Known + Lifetime Unknown), this is the area in which you may not know how long that memory needs to be around but you do know how much memory is needed. In this case, you could say that the “ownership” of that memory across multiple systems is ill-defined. A common solution for this domain of problems is reference counting or ownership semantics.

In the bottom-right category (Size Unknown + Lifetime Unknown), this is the area in which you have literally no idea how much memory you need nor how long it will be needed for. In practice, this is quite rare and you _ought_ to try and avoid these situations when possible. However, the general solution for this domain of problems is garbage collection[3](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#fn:3).

Please note that in domain specific areas, these percentages will be completely different. For instance, a web server that may be handling an unknown amount of requests may require a form of garbage collection if the memory is limited or it may be cheaper to just buy more memory.

# [Generations of Lifetimes](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#generations-of-lifetimes)

For the common category, the general approach that I take is to think about memory lifetimes in terms of generations. An _allocation generation_ is a way to organize memory lifetimes into a hierarchical structure[4](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#fn:4).

-   **Permanent Allocation**: Memory that is never freed until the end of the program. This memory is persistent during program lifetime.
    
-   **Transient Allocation**: Memory that has a cycle-based lifetime. This memory only persists for the “cycle” and is freed at the end of this cycle. An example of a cycle could be a frame within a graphical program (e.g. a game) or an update loop.
    
-   **Scratch/Temporary Allocation**: Short lived, quick memory that I just want to allocate and forget about. A common case for this is when I want to generate a string and output it to a log.
    

## [Memory Hierarchies](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#memory-hierarchies)

As I previously stated, the monistic model of memory is the preferred model of memory (on modern systems). This generational approach to memory orders the lifetime of memory in a hierarchical fashion. You could still have pseudo-permanent memory within a transient allocator or a scratch allocator, as the difference is thinking about the relative usage of that memory with respect to its lifetime. Thinking locally about how memory is used aids with conceptualizing and managing memory — the human brain can only hold so much.

The same localist thought process can be applied to the memory-space/size of which I will be discussing in later articles in this series.

# [The Compiler’s Knowledge of the Program](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#the-compilers-knowledge-of-the-program)

In languages with automatic memory management, many people assume that the compiler knows a lot about the usage and lifetimes of your program. **This is false**. You know much more about your program than the compiler could ever know. In the case of languages with ownership semantics (e.g. Rust, C++11), the language may aid you in certain cases, but it struggles to know (if it is at all possible) when it should pre-allocate or free in bulk. This is compiler ignorance can lead to a lot of performance issues.

My personal issue with regards to ownership semantics is that it naturally focuses on the ownership of single objects rather than in systems[5](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#fn:5). Such languages also have the tendency to couple the concept of ownership with the concept of lifetime, which are not necessarily linked.

# [Coming Next](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#coming-next)

In this series, I will discuss the different kinds of memory models and allocation strategies that can be used. These are the topics that will be covered:

-   Sequential (Contiguous) Allocations
-   Virtual Memory
-   Out of Order Allocators and Fragmentation
-   `malloc`
-   Hierarchies of Allocators
-   Automatic Lifetime Allocations
-   Allocation Grouping and Mental Models

---

1.  Most architectures have register dedicated as a pointer to the stack, that is added because it is used frequently and pragmatically makes sense to do so. [↩︎](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#fnref:1)
    
2.  Memory safety is another aspect to think about but I will not cover that in this series as that requires a separate set of solutions and trade-offs. In the domains that I deal with, memory safety is not a huge concern. [↩︎](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#fnref:2)
    
3.  Garbage collection is one of the only terms in computers science where the term actually reflects its real world counter part. [↩︎](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#fnref:3)
    
4.  These generations are not cut-and-dry and allocations can span across this spectrum of lifetimes (like in real life). Memory within these generations usually get allocated and freed at the same time (born, live, and die together). [↩︎](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#fnref:4)
    
5.  I know in languages such as Rust, you can describe the lifetime of an object to be linked to a system however, with the memory allocation strategies I will be discussing later, the Rust code that would be required pretty much acts as if you will bypass the ownership semantics entirely and have a liberal use of `unsafe`. [↩︎](https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/#fnref:5)


# Memory Allocation Strategies - Part 2

## Linear/Arena Allocators

Series: [Memory Allocation Strategies](https://www.gingerbill.org/series/memory-allocation-strategies)

2019-02-08

# [Linear Allocation](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#linear-allocation)

The first memory allocation strategy that I will cover is also one of the simplest ones: linear allocation. As the name suggests, memory is allocated linearly. Throughout this series, I will be using the concept of an _allocator_ as a means to allocate this memory. A linear allocator, is also known by other names such as an Arena or Region-based allocator. In this article, I will refer to this allocator an a _Arena_.

## [Basic Logic](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#basic-logic)

The arena’s logic only requires an offset (or pointer) to state the end of the last allocation[1](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#fn:1).

![Linear Allocator Layout](https://www.gingerbill.org/images/memory-allocation-strategies/linear_allocator.svg#center)

To allocate some memory from the arena, it is as simple as moving the offset (or pointer) forward. In [Big-O notation](https://wikipedia.org/wiki/Big_O_notation), the allocation has complexity of _**O(1)**_ (constant).

![Linear Allocator Alloc](https://www.gingerbill.org/images/memory-allocation-strategies/linear_allocator_alloc.svg#center)

Due to being the simplest allocator possible, the arena allocator does not allow the user to free certain blocks of memory. The memory is usually freed all at once.

## [Basic Implementation](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#basic-implementation)

**Note:** The following code examples will be in written in C.

The simplest arena allocator _could_ look like this:

```c
static unsigned char *arena_buffer;
static size_t arena_buffer_length;
static size_t arena_offset;

void *arena_alloc(size_t size) {
	// Check to see if the backing memory has space left
	if (arena_offset+size <= arena_buffer_length) {
		void *ptr = &arena_buffer[arena_offset];
		arena_offset += size;
		// Zero new memory by default
		memset(ptr, 0, size);
		return ptr;
	}
	// Return NULL if the arena is out of memory
	return NULL;
}
```

As you may be able to tell, this is as simple as it gets. There are two issues with this basic approach:

-   You cannot reuse this procedure for different arenas
-   The pointer returned may not be aligned correctly for the data you need

The first issue can be easily solved by coupling that global data into a structure and passing that to the procedure `arena_alloc`. As for the second issue, this requires understanding about the basic issues of _unaligned memory_.

# [Memory Alignment](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#memory-alignment)

Modern computer architectures will always read memory at its “word size” (4 bytes on a 32 bit machine, 8 bytes on a 64 bit machine). If you have an unaligned memory access (on a processor that allows for that), the processor will have to read multiple “words”. This means that an unaligned memory access _may_ be much slower than an aligned memory access. I will not write too much about memory alignment in this series. If you would like to learn more, I recommend the following articles:

-   [IBM - Data alignment: Straighten up and fly right](https://developer.ibm.com/technologies/systems/articles/pa-dalign/)
-   [Gallery of Processor Cache Effects](http://igoro.com/archive/gallery-of-processor-cache-effects/)
-   [x86 Protected Mode Basics](http://www.rcollins.org/articles/pmbasics/tspec_a1_doc.html)

## [Aligning a Memory Address](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#aligning-a-memory-address)

On virtually all architectures, the amount of bytes that something must be aligned by must be a power of two (1, 2, 4, 8, 16, etc). This means we should create procedure to assert that the alignment is a power of two:

```c
bool is_power_of_two(uintptr_t x) {
	return (x & (x-1)) == 0;
}
```

To align a memory address to the specified alignment is simple modulo arithmetic. You are looking to find how many bytes forward you need to go in order for the memory address is a multiple of the specified alignment.

```c
uintptr_t align_forward(uintptr_t ptr, size_t align) {
	uintptr_t p, a, modulo;

	assert(is_power_of_two(align));

	p = ptr;
	a = (uintptr_t)align;
	// Same as (p % a) but faster as 'a' is a power of two
	modulo = p & (a-1);

	if (modulo != 0) {
		// If 'p' address is not aligned, push the address to the
		// next value which is aligned
		p += a - modulo;
	}
	return p;
}
```

Now that we know how to align memory, that means we can update our original `arena_alloc` to support alignment properly and store the arena data within a structure.

```c
typedef struct Arena Arena;
struct Arena {
	unsigned char *buf;
	size_t         buf_len;
	size_t         prev_offset; // This will be useful for later on
	size_t         curr_offset;
};

void *arena_alloc_align(Arena *a, size_t size, size_t align) {
	// Align 'curr_offset' forward to the specified alignment
	uintptr_t curr_ptr = (uintptr_t)a->buf + (uintptr_t)a->curr_offset;
	uintptr_t offset = align_forward(curr_ptr, align);
	offset -= (uintptr_t)a->buf; // Change to relative offset

	// Check to see if the backing memory has space left
	if (offset+size <= a->buf_len) {
		void *ptr = &a->buf[offset];
		a->prev_offset = offset;
		a->curr_offset = offset+size;

		// Zero new memory by default
		memset(ptr, 0, size);
		return ptr;
	}
	// Return NULL if the arena is out of memory (or handle differently)
	return NULL;
}

#ifndef DEFAULT_ALIGNMENT
#define DEFAULT_ALIGNMENT (2*sizeof(void *))
#endif

// Because C doesn't have default parameters
void *arena_alloc(Arena *a, size_t size) {
	return arena_alloc_align(a, size, DEFAULT_ALIGNMENT);
}
```

# [Implementing the Rest](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#implementing-the-rest)

The arena allocator is usable for basic things now but it is missing a few features that would make it practical for everyday use. The complete arena allocator will have the following procedures:

-   `arena_init` initialize the arena with a pre-allocated memory buffer
-   `arena_alloc` simply increments an offset to indicate the current buffer offset
-   `arena_free` does absolutely nothing (just there for completeness)
-   `arena_resize` first checks to see if the allocation being resized was the previously performed allocation and if so, the same pointer will be returned and the buffer offset is changed. Otherwise, `arena_alloc` will be called instead.
-   `arena_free_all` is used to free all the memory within the allocator by setting the buffer offsets to zero.

## [Init](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#init)

The `arena_init` procedure just initializes the parameters for the given arena.

```c
void arena_init(Arena *a, void *backing_buffer, size_t backing_buffer_length) {
	a->buf = (unsigned char *)backing_buffer;
	a->buf_len = backing_buffer_length;
	a->curr_offset = 0;
	a->prev_offset = 0;
}
```

## [Free](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#free)

As I previously stated, `arena_free` does absolutely nothing. It exists purely for completeness.

```c
void arena_free(Arena *a, void *ptr) {
	// Do nothing
}
```

## [Resize](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#resize)

Resizing an allocation is sometimes useful in an arena. To reduce waste of memory, it is useful to track the `prev_offset` and if the `old_memory` passed in equals the offset provided, just resize that memory block.

```c
void *arena_resize_align(Arena *a, void *old_memory, size_t old_size, size_t new_size, size_t align) {
	unsigned char *old_mem = (unsigned char *)old_memory;

	assert(is_power_of_two(align));

	if (old_mem == NULL || old_size == 0) {
		return arena_alloc_align(a, new_size, align);
	} else if (a->buf <= old_mem && old_mem < a->buf+buf_len) {
		if (a->buf+a->prev_offset == old_mem) {
			a->curr_offset = a->prev_offset + new_size;
			if (new_size > old_size) {
				// Zero the new memory by default
				memset(&a->buf[a->curr_offset], 0, new_size-old_size);
			}
			return old_memory;
		} else {
			void *new_memory = arena_alloc_align(a, new_size, align);
			size_t copy_size = old_size < new_size ? old_size : new_size;
			// Copy across old memory to the new memory
			memmove(new_memory, old_memory, copy_size);
			return new_memory;
		}

	} else {
		assert(0 && "Memory is out of bounds of the buffer in this arena");
		return NULL;
	}

}

// Because C doesn't have default parameters
void *arena_resize(Arena *a, void *old_memory, size_t old_size, size_t new_size) {
	return arena_resize_align(a, old_memory, old_size, new_size, DEFAULT_ALIGNMENT);
}
```

## [Free All](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#free-all)

Finally, `arena_free_all` is used to free all the memory within the allocator by setting the buffer offsets to zero. This is very useful for when you want to reset on a per cycle/frame basis.

```c
void arena_free_all(Arena *a) {
	a->curr_offset = 0;
	a->prev_offset = 0;
}
```

# [Using the Allocator](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#using-the-allocator)

To use the allocator, you need to provide some backing memory. A simple approach is to provide an array:

```c
unsigned char backing_buffer[256];
Arena a = {0};
arena_init(&a, backing_buffer, 256);
```

Another approach is to use `malloc`:

```c
void *backing_buffer = malloc(256);
Arena a = {0};
arena_init(&a, backing_buffer, 256);
```

# [Conclusion](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#conclusion)

You have now implemented your very first custom allocator! The full source code is [available here](https://www.gingerbill.org/code/memory-allocation-strategies/part002.c).

In the next article, I will be talking about the basic evolution of the _arena allocator_ in to a [_stack allocator_](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/).

# [Extra Features](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#extra-features)

One extra feature you could add is a temporary arena memory _savepoint_. This is extremely useful for when you just want to use some memory in an arena for a very short period and then reset to the previously saved point.

```c
typedef struct Temp_Arena_Memory Temp_Arena_Memory;
struct Temp_Arena_Memory {
	Arena *arena;
	size_t prev_offset;
	size_t curr_offset;
};

Temp_Arena_Memory temp_arena_memory_begin(Arena *a) {
	Temp_Arena_Memory temp;
	temp.arena = a;
	temp.prev_offset = a->prev_offset;
	temp.curr_offset = a->curr_offset;
	return temp;
}

void temp_arena_memory_end(Temp_Arena_Memory temp) {
	temp.arena->prev_offset = temp.prev_offset;
	temp.arena->curr_offset = temp.curr_offset;
}
```

---

1.  Other data may stored such as the offset of the beginning of the previous allocation or allocation count. [↩︎](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/#fnref:1)


# Memory Allocation Strategies - Part 3

## Stack Allocators

Series: [Memory Allocation Strategies](https://www.gingerbill.org/series/memory-allocation-strategies)

2019-02-15

# [Stack-Like (LIFO) Allocation](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#stack-like-lifo-allocation)

In the previous article, we looked at the [linear/arena allocator](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/), which is the simplest of all memory allocators. In this article, I will cover the fixed-sized stack-like allocator. Throughout this article, I will refer to this allocator as a _stack allocator_.

**Note:** A stack-like allocator means that the allocator acts like a data structure following the _last-in, first-out_ (LIFO) principle. This has nothing to do with _the stack_ or the _stack frame_.

The stack allocator is the natural evolution from the arena allocator. The approach with the stack allocator is to manage memory in a stack-like fashion following the LIFO principle. Therefore, like a stack of books, if you put something on the top of the stack earlier, you need to pick that book up first before you get one underneath.

As with the arena allocator, an offset into the memory block will be stored and will be moved forwards on every allocation. The difference is that the offset can also be moved backwards when memory is _freed_. With an arena, you could only free all the memory all at once.

## [Basic Logic](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#basic-logic)

As with the extended arena in the [previous article](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/), the offset of the previous allocation needs to be tracked. This is required in order to free memory on a _per-allocation_ basis. One approach is to store a _header_ which stores information about that allocation. This _header_ means that the allocator can know how far back it should move the offset to free that memory.

![Stack Allocator Layout](https://www.gingerbill.org/images/memory-allocation-strategies/stack_allocator.svg#center)

To allocate some memory from the stack allocator, as with the arena allocator, it is as simple as moving the offset forward whilst accounting for the header. In [Big-O notation](https://wikipedia.org/wiki/Big_O_notation), the allocation has complexity of _**O(1)**_ (constant).

![Stack Allocator Alloc](https://www.gingerbill.org/images/memory-allocation-strategies/stack_allocator_alloc.svg#center)

To free a block, the header that is stored before the block of memory can be read in order to move the offset backwards. In Big-O notation, the freeing of this memory has complexity of _**O(1)**_ (constant).

![Stack Allocator Free](https://www.gingerbill.org/images/memory-allocation-strategies/stack_allocator_free.svg#center)

## [Header Storage](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#header-storage)

You may have noticed that I never actually state what to store in the allocation header. The reason for this is because there are numerous approaches to stack allocators which store different data. There are three main approaches[1](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#fn:1):

-   Store the padding from the previous offset
-   Store the previous offset
-   Store the size of the allocation

In this article, I will cover the first two approaches, the first of which I will refer to as a _loose stack_ or _small stack_ as it stores very little information in the header. The third approach is useful if you want to query the size of an allocation dynamically[2](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#fn:2).

# [Implementation of the Loose/Small Stack Allocator](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#implementation-of-the-loosesmall-stack-allocator)

The stack allocator will act like an arena allocator in many regards except for the ability to free memory on a per-allocation basis. The complete stack allocator will have the following procedures:

-   `stack_init` initialize the stack with a pre-allocated memory buffer
-   `stack_alloc` increments the offset to indicate the current buffer offset whilst taking into account the allocation header
-   `stack_free` frees the memory passed to it and decrements the offset to _free_ that memory
-   `stack_resize` first checks to see if the allocation being resized was the previously performed allocation and if so, the same pointer will be returned and the buffer offset is changed. Otherwise, stack_alloc will be called instead.
-   `stack_free_all` is used to free all the memory within the allocator by setting the buffer offsets to zero.

## [Data Structures](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#data-structures)

The (loose/small) stack data structure contains the same information as an arena.

```c
typedef struct Stack Stack;
struct Stack {
	unsigned char *buf;
	size_t buf_len;
	size_t offset;
};
```

The allocation header for this particular stack implementation uses an integer to encode the padding.

```c
typedef struct Stack_Allocation_Header Stack_Allocation_Header;
struct Stack_Allocation_Header {
	uint8_t padding;
};
```

This padding stores the amount of bytes that has to be placed before the header in order to have the new allocation correctly aligned.

![Stack Allocator Header](https://www.gingerbill.org/images/memory-allocation-strategies/stack_allocator_header.svg#center)

**Note**: Storing the padding as a byte does limit the the maximum alignment that can used with this stack allocator to 128 bytes. If you require a higher alignment, increase the size of integer used to store the padding. To calculate the maximum alignment that the padding can be used for, use this equation:

Maximum Alignment in Bytes=28×sizeof(padding)−1Maximum Alignment in Bytes=28×sizeof(padding)−1

## [Init](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#init)

The `stack_init` procedure just initializes the parameters for the given stack.

```c
void stack_init(Stack *s, void *backing_buffer, size_t backing_buffer_length) {
	s->buf = (unsigned char *)backing_buffer;
	s->buf_len = backing_buffer_length;
	s->offset = 0;
}
```

## [Alloc](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#alloc)

Unlike an arena, a stack allocator requires a header alongside the allocation. As previously stated above, the `calc_padding_with_header` procedure is similar to the `align_forward` procedure from the previous article, however it determines how much space is needed with respect to the header and the requested alignment. In the header, the amount padding needs to be stored and the address after that header needs to be returned.

```c
size_t calc_padding_with_header(uintptr_t ptr, uintptr_t alignment, size_t header_size) {
	uintptr_t p, a, modulo, padding, needed_space;

	assert(is_power_of_two(alignment));

	p = ptr;
	a = alignment;
	modulo = p & (a-1); // (p % a) as it assumes alignment is a power of two

	padding = 0;
	needed_space = 0;

	if (modulo != 0) { // Same logic as 'align_forward'
		padding = a - modulo;
	}

	needed_space = (uintptr_t)header_size;

	if (padding < needed_space) {
		needed_space -= padding;

		if ((needed_space & (a-1)) != 0) {
			padding += a * (1+(needed_space/a));
		} else {
			padding += a * (needed_space/a);
		}
	}

	return (size_t)padding;
}

void *stack_alloc_align(Stack *s, size_t size, size_t alignment) {
	uintptr_t curr_addr, next_addr;
	size_t padding;
	Stack_Allocation_Header *header;


	assert(is_power_of_two(alignment));

	if (alignment > 128) {
		// As the padding is 8 bits (1 byte), the largest alignment that can
		// be used is 128 bytes
		alignment = 128;
	}

	curr_addr = (uintptr_t)s->buf + (uintptr_t)s->offset;
	padding = calc_padding_with_header(curr_addr, (uintptr_t)alignment, sizeof(Stack_Allocation_Header));
	if (s->offset + padding + size > s->buf_len) {
		// Stack allocator is out of memory
		return NULL;
	}
	s->offset += padding;

	next_addr = curr_addr + (uintptr_t)padding;
	header = (Stack_Allocation_Header *)(next_addr - sizeof(Stack_Allocation_Header));
	header->padding = (uint8_t)padding;

	s->offset += size;

	return memset((void *)next_addr, 0, size);
}

// Because C does not have default parameters
void *stack_alloc(Stack *s, size_t size) {
	return stack_alloc_align(s, size, DEFAULT_ALIGNMENT);
}
```

## [Free](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#free)

For `stack_free`, the pointer passed needs to be checked as to whether it is valid (i.e. it was allocated by this allocator). If it is valid, this means it is possible to acquire the header of this allocation. Using a little _pointer arithmetic_, we can reset the offset to the allocation previous to the passed pointer.

```c
void stack_free(Stack *s, void *ptr) {
	if (ptr != NULL) {
		uintptr_t start, end, curr_addr;
		Stack_Allocation_Header *header;
		size_t prev_offset;

		start = (uintptr_t)s->buf;
		end = start + (uintptr_t)s->buf_len;
		curr_addr = (uintptr_t)ptr;

		if (!(start <= curr_addr && curr_addr < end)) {
			assert(0 && "Out of bounds memory address passed to stack allocator (free)");
			return;
		}

		if curr_addr >= start+(uintptr_t)s->offset {
			// Allow double frees
			return;
		}

		header = (Stack_Allocation_Header *)(curr_addr - sizeof(Stack_Allocation_Header));
		prev_offset = (size_t)(curr_addr - (uintptr_t)header->padding - start);

		s->offset = prev_offset;
	}
}
```

## [Resize](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#resize)

Resizing an allocation is sometimes useful in an stack allocator. As we don’t store the previous offset for this particular version, we will just reallocate new memory if there needs to be a change in allocation size[3](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#fn:3).

```c
void *stack_resize_align(Stack *s, void *ptr, size_t old_size, size_t new_size, size_t alignment) {
	if (ptr == NULL) {
		return stack_alloc_align(s, new_size, alignment);
	} else if (new_size == 0) {
		stack_free(s, ptr);
		return NULL;
	} else {
		uintptr_t start, end, curr_addr;
		size_t min_size = old_size < new_size ? old_size : new_size;
		void *new_ptr;

		start = (uintptr_t)s->buf;
		end = start + (uintptr_t)s->buf_len;
		curr_addr = (uintptr_t)ptr;
		if (!(start <= curr_addr && curr_addr < end)) {
			assert(0 && "Out of bounds memory address passed to stack allocator (resize)");
			return NULL;
		}

		if (curr_addr >= start + (uintptr_t)s->offset) {
			// Treat as a double free
			return NULL;
		}

		if old_size == size {
			return ptr;
		}

		new_ptr = stack_alloc_align(s, new_size, alignment);
		memmove(new_ptr, ptr, min_size);
		return new_ptr;
	}
}

void *stack_resize(Stack *s, void *ptr, size_t old_size, size_t new_size) {
	return stack_resize_align(s, ptr, old_size, new_size, DEFAULT_ALIGNMENT);
}
```

## [Free All](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#free-all)

Finally, `stack_free_all` is used to free all the memory within the allocator by setting the buffer offsets to zero. This is very useful for when you want to reset on a per cycle/frame basis. This acts identically to an arena in this case.

```c
void stack_free_all(Stack *s) {
	s->offset = 0;
}
```

# [Improving the Stack Allocator](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#improving-the-stack-allocator)

This loose/small stack allocator above is already very useful but it does not enforce the LIFO principle for frees. It allows the user to free any block of memory in any order but frees everything that was allocated after it. In order to enforce the LIFO principle, data about the previous offset needs to be stored in the header and the general data structure.

```c
struct Stack_Allocation_Header {
	size_t prev_offset;
	size_t padding;
};

struct Stack {
	unsigned char *buf;
	size_t buf_len;
	size_t prev_offset;
	size_t curr_offset;
};
```

This new header is a lot larger compared to the simple padding approach[4](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#fn:4), but it does mean that LIFO for frees can be enforced. There only needs to be a few adjustments to the code. The resize procedure is left as an exercise for the reader.

### [`stack_alloc_align`](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#stack_alloc_align)

```c
...

s->prev_offset = s->offset; // Store the previous offset
s->offset += padding;

next_addr = curr_addr + (uintptr_t)padding;
header = (Stack_Allocation_Header *)(next_addr - sizeof(Stack_Allocation_Header));
header->padding = padding;
header->prev_offset = s->prev_offset; // store the previous offset in the header

s->offset += size;
```

### [`stack_free`](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#stack_free)

```c
...

header = (Stack_Allocation_Header *)(curr_addr - sizeof(Stack_Allocation_Header));

// Calculate previous offset from the header and its address
prev_offset = (size_t)(curr_addr - (uintptr_t)header->padding - start);

if (prev_offset != header->prev_offset) {
	assert(0 && "Out of order stack allocator free");
	return;
}

// Reset the offsets to the previous allocation
s->curr_offset = s->prev_offset;
s->prev_offset = header->prev_offset;
```

# [Comments and Conclusion](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#comments-and-conclusion)

The stack allocator is the first of many allocators that will use the concept of a _header_ for allocations. In this basic form of a stack allocator, unless you want the LIFO behaviour enforced, I would personally recommend using an arena allocator with the `Temp_Arena_Memory` construct instead. However if you require something like C++'s constructors and destructors, a stack allocator will be more friendly to that framework (RAII)[5](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#fn:5).

You can extend the stack allocator even further by having two different offsets: one that starts at the beginning and increments forwards, and another that starts at the end and increments backwards. This is called a double-ended stack and allows for the maximization of memory usage whilst keeping fragmentation extremely low (as long as the offset never overlap).

In the next article, I will discuss _pool allocators_ and how they are extremely useful for creating and destroying things in completely random order of the same size.

---

1.  These approaches can be combined and are not mutually exclusive. [↩︎](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#fnref:1)
    
2.  I rarely need this as I usually track the size of the allocation manually. [↩︎](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#fnref:2)
    
3.  It is an exercise for the reader to figure out how to make this more efficient by not allocating more memory if it is was the previous allocation. [↩︎](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#fnref:3)
    
4.  You can reduce the size of the header by using smaller integers but this does reduce the size of the allocations that can be used. [↩︎](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#fnref:4)
    
5.  [https://wikipedia.org/wiki/Placement_syntax](https://wikipedia.org/wiki/Placement_syntax) [↩︎](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#fnref:5)


# Memory Allocation Strategies - Part 4

## Pool Allocators

Series: [Memory Allocation Strategies](https://www.gingerbill.org/series/memory-allocation-strategies)

2019-02-16

# [Pool-Based Allocation](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#pool-based-allocation)

In the previous article, we looked at the [stack allocator](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/), which was the natural evolution of the [linear/arena allocator](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/). In this article, I will cover the fixed-sized _pool allocator_.

A pool allocator is a bit different from the previous allocation strategies that I have covered. A pool splits the supplied backing buffer into _chunks_ of equal size and keeps track of which of the chunks are free. When an allocation is wanted, a free chunk is given. When a chunk is wanted to be freed, it adds that chunk to the list of free chunks.

Pool allocators are extremely useful when you need to allocate chunks of memory of the same size which need are created and destroy dynamically, especially in a random order. Pools also have the benefit that arenas and stacks have in that they provide very little fragmentation and allocate/free in constant time _**O(1)**_.

Pool allocators are usually used to allocate _groups_ of “things” at once which share the same lifetime. An example could be within a game that creates and destroys entities in batches where each entity within a batch share the same lifetime.

# [Basic Logic](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#basic-logic)

A pool allocator takes a backing buffer and divides that buffer into pools/slots/bins/chunks[1](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#fn:1) of all the same size.

![Pool Allocator Layout](https://www.gingerbill.org/images/memory-allocation-strategies/pool_allocator.svg#center)

The question is how are these allocations and frees determined? And how do they provide very little fragmentation with allocations that can be made in any order?

## [Free Lists](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#free-lists)

A [free list](https://wikipedia.org/wiki/Free_list) is a data structure that internally stores a [linked-list](https://wikipedia.org/wiki/Linked_list) of the free slots/chunks within the memory buffer. The nodes of the list are stored in-place as this means that there does not need to be another data structure (e.g. array, list, etc) to keep track of the free slots. The data is _only_ stored _within_ the backing buffer of the pool allocator.

![Pool Allocator List](https://www.gingerbill.org/images/memory-allocation-strategies/pool_allocator_list.svg#center)

The general approach is to store a header at the beginning of the chunk (not before the chunk like with the stack allocator) which _points_ to the next available free chunk[2](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#fn:2).

![Pool Allocator List In-Place](https://www.gingerbill.org/images/memory-allocation-strategies/pool_allocator_list_inplace.svg#center)

## [Allocate and Free](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#allocate-and-free)

To allocate a chunk, just pop off the head (first element) from the free list. In [Big-O notation](https://wikipedia.org/wiki/Big_O_notation), the allocation has complexity of _**O(1)**_ (constant).

![Pool Allocator Alloc](https://www.gingerbill.org/images/memory-allocation-strategies/pool_allocator_alloc.svg#center)

**Note**: The free list does not need to be ordered as its order is determined by how chunks are allocated and freed.

![Pool Allocator Alloc Unordered](https://www.gingerbill.org/images/memory-allocation-strategies/pool_allocator_alloc_unordered.svg#center)

To free a chunk, just push the freed chunk as the head of the free list. In Big-O notation, the freeing of this memory has complexity of _**O(1)**_ (constant).

# [Implementation](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#implementation)

The pool allocator requires less code than the arena and stack allocator as there is no logic required for different sized/aligned allocations and resize allocations. The complete pool allocator will have the following procedures:

-   `pool_init` initialize the pool with a pre-allocated memory buffer
-   `pool_alloc` pops off the head from the free list
-   `pool_free` pushes on the freed chunk as the head of the free list
-   `pool_free_all` pushes every chunk in the pool onto the free list

## [Data Structures](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#data-structures)

The pool data structure contains a backing buffer, the size of each chunk, and the head to the free list.

```c
typedef struct Pool Pool;
struct Pool {
	unsigned char *buf;
	size_t buf_len;
	size_t chunk_size;

	Pool_Free_Node *head; // Free List Head
};
```

Each node in the free list just contains a pointer to the next free chunk, which could be `NULL` if it is the _tail_ (last element).

```c
typedef struct Pool_Free_Node Pool_Free_Node;
struct Pool_Free_Node {
	Pool_Free_Node *next;
};
```

## [Init](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#init)

Initializing a pool is pretty simple however, because each chunk has the same size and alignment, this logic can be done now rather than later.

```c
void pool_free_all(Pool *p); // This procedure will be covered later in this article

void pool_init(Pool *p, void *backing_buffer, size_t backing_buffer_length,
               size_t chunk_size, size_t chunk_alignment) {
	// Align backing buffer to the specified chunk alignment
	uintptr_t initial_start = (uintptr_t)backing_buffer;
	uintptr_t start = align_forward_uintptr(initial_start, (uintptr_t)chunk_alignment);
	backing_buffer_length -= (size_t)(start-initial_start);

	// Align chunk size up to the required chunk_alignment
	chunk_size = align_forward_size(chunk_size, chunk_alignment);

	// Assert that the parameters passed are valid
	assert(chunk_size >= sizeof(Pool_Free_Node) &&
	       "Chunk size is too small");
	assert(backing_buffer_length >= chunk_size &&
	       "Backing buffer length is smaller than the chunk size");

	// Store the adjusted parameters
	p->buf = (unsigned char *)backing_buffer;
	p->buf_len = backing_buffer_length;
	p->chunk_size = chunk_size;
	p->head = NULL; // Free List Head

	// Set up the free list for free chunks
	pool_free_all(p);
}
```

## [Alloc](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#alloc)

The `pool_alloc` procedure is a lot simpler than other allocators as each chunk has the same size and alignment and thus these parameters do not need to be passed to the procedure. The latest free chunk from the free list is popped off and is then used as the new allocation.

```c
void *pool_alloc(Pool *p) {
	// Get latest free node
	Pool_Free_Node *node = p->head;

	if (node == NULL) {
		assert(0 && "Pool allocator has no free memory");
		return NULL;
	}

	// Pop free node
	p->head = p->head->next;

	// Zero memory by default
	return memset(node, 0, p->chunk_size);
}
```

## [Free](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#free)

Freeing an allocation is pretty much the opposite of an allocation. The chunk to be freed is pushed onto the free list.

```c
void pool_free(Pool *p, void *ptr) {
	Pool_Free_Node *node;

	void *start = p->buf;
	void *end = &p->buf[p->buf_len];

	if (ptr == NULL) {
		// Ignore NULL pointers
		return;
	}

	if (!(start <= ptr && ptr < end)) {
		assert(0 && "Memory is out of bounds of the buffer in this pool");
		return;
	}

	// Push free node
	node = (Pool_Free_Node *)ptr;
	node->next = p->head;
	p->head = node;
}
```

## [Free All](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#free-all)

Freeing all the memory is equivalent of pushing all the chunks onto the free list.

```c
void pool_free_all(Pool *p) {
	size_t chunk_count = p->buf_len / p->chunk_size;
	size_t i;

	// Set all chunks to be free
	for (i = 0; i < chunk_count; i++) {
		void *ptr = &p->buf[i * p->chunk_size];
		Pool_Free_Node *node = (Pool_Free_Node *)ptr;
		// Push free node onto thte free list
		node->next = p->head;
		p->head = node;
	}
}
```

# [Conclusion](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#conclusion)

The pool allocator is a very useful allocator for when you need to allocator “things” in _chunks_ and the things within those chunks share the same lifetime. The full source code is [available here](https://www.gingerbill.org/code/memory-allocation-strategies/part004.c).

In the next article, I will discuss [free list memory allocators](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/).

---

1.  What’s in a name? That which we call a rose. By any other name would smell as sweet. [Romeo and Juliet (II, ii, 1-2)](https://www.owleyes.org/text/romeo-and-juliet/read/act-ii-scene-ii) [↩︎](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#fnref:1)
    
2.  If there is not an available free chunk, it will point to nothing (`NULL`). [↩︎](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/#fnref:2)


# Memory Allocation Strategies - Part 5

## Free List Allocators

Series: [Memory Allocation Strategies](https://www.gingerbill.org/series/memory-allocation-strategies)

2021-11-30

# [Free List Based Allocation](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/#free-list-based-allocation)

In the previous article, we looked at the [pool allocator](https://www.gingerbill.org/article/2019/02/16/memory-allocation-strategies-004/), which splits the supplied backing buffer into _chunks_ of equal size and keeps track of which of the chunks are free. Pool allocators are fast allocators that allow for out of order free in constant time _**O(1)**_ whilst keeping very little fragmentation. The main restriction of a pool allocator is that every memory allocation must be of the same size.

A free list is a general purpose allocator which, compared to the other allocators that we previously looked at, does not impose any restrictions. It allows allocations and deallocations to be out of order and of any size. Due to its nature, the allocator’s performance is not as good as the others previously discussed in this series.

There are two common approaches to implementing a free list allocator: one using a [linked list](https://wikipedia.org/wiki/Linked_list) and one use a [red black tree](https://wikipedia.org/wiki/Red%E2%80%93black_tree). Using a linked list approach is the most common approach and what we’ll look at first.

# [Linked List Approach](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/#linked-list-approach)

As the title of this section suggests, we’ll be using a linked list to store the address of free contiguous blocks in the memory along with its size. When the user requests memory, it searches in the linked list for a block where the data can fit. It then removes the element from the linked list and places an allocation header (which is required on free) just before the data (similar to what we used in the article on [stack allocators](https://www.gingerbill.org/article/2019/02/15/memory-allocation-strategies-003/#data-structures)).

For freeing memory, we recover the allocation header (stored before the allocation) to know the size of the block we want to free. Once that block has been freed, it is inserted into the linked list, and then we try to _coalescence_ contiguous blocks of memory together to create larger blocks.

![Free List Allocator](https://www.gingerbill.org/images/memory-allocation-strategies/free_list_allocator.svg#center)

# [Linked List Free List Implementation](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/#linked-list-free-list-implementation)

**n.b.** The following implementation does provide some constraints on the size and alignment of requested allocations with this particular allocator. The minimum size of an allocation must be at least the size of the free list node data structure, and the alignment has similar requirements.

## [Data Structures](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/#data-structures)

These are the data structures that are required to implement the linked list based free list allocator.

```c
// Unlike our trivial stack allocator, this header needs to store the 
// block size along with the padding meaning the header is a bit
// larger than the trivial stack allocator
typedef struct Free_List_Allocation_Header Free_List_Allocation_Header;
struct Free_List_Allocation_Header {
    size_t block_size;
    size_t padding;
};

// An intrusive linked list for the free memory blocks
typedef struct Free_List_Node Free_List_Node;
struct Free_List_Node {
    Free_List_Node *next;
    size_t block_size;
};

typedef Placement_Policy Placement_Policy;
enum Placement_Policy {
    Placement_Policy_Find_First,
    Placement_Policy_Find_Best
};

typedef struct Free_List Free_List;
struct Free_List {
    void *           data;
    size_t           size;
    size_t           used;
    
    Free_List_Node * head;
    Placement_Policy policy;
};
```

## [Initialization](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/#initialization)

```c
void free_list_free_all(Free_List *fl) {
    fl->used = 0;
    Free_List_Node *first_node = (Free_List_Node *)fl->data;
    first_node->block_size = fl->size;
    first_node->next = NULL;
    f->head = first_node;
}

void free_list_init(Free_List *fl, void *data, size_t size) {
    fl->data = data;
    fl->size = size;
    free_list_free_all(fl);
}
```

## [Allocation](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/#allocation)

To allocate a block of memory within this allocator, we need to look for a block in the memory in which to fit our data. This means iterating across our linked list of free memory blocks until a block has at least the size requested, and then remove it from the linked list of free memory. Finding the first block is called a _first-fit_ placement policy as it stops at the _first_ block which fits the requested memory size. Another placement policy is called the _best-fit_ which looks for a free block of memory which is the smallest available which fits the memory size. The latter option reduces memory fragmentation within the allocator.

In the diagram there three free memory blocks, but not all are appropriate for the size of the memory allocation that is requested (plus the header)![Free List Allocator Allocate Search](https://www.gingerbill.org/images/memory-allocation-strategies/free_list_allocator_alloc.svg#center)

When an allocation has been made, the free list will then be corrected to remove the used node.![Free List Allocator Allocate Store](https://www.gingerbill.org/images/memory-allocation-strategies/free_list_allocator_alloc2.svg#center)

This algorithm has a time complexity of _**O(N)**_, where N is the number of free blocks in the free list.

```c
// Defined Memory Allocation Strategies Part 3: /article/2019/02/15/memory-allocation-strategies-003/#alloc
size_t calc_padding_with_header(uintptr_t ptr, uintptr_t alignment, size_t header_size);

Free_List_Node *free_list_find_first(Free_List *fl, size_t size, size_t alignment, size_t *padding_, Free_List_Node **prev_node_) {
    // Iterates the list and finds the first free block with enough space 
    Free_List_Node *node = fl->head;
    Free_List_Node *prev_node = NULL;
    
    size_t padding = 0;
    
    while (node != NULL) {
        padding = calc_padding_with_header((uintptr_t)node, (uintptr_t)alignment, sizeof(Free_List_Allocation_Header));
        size_t required_space = size + padding;
        if (node->block_size >= required_space) {
            break;
        }
        prev_node = node;
        node = node->next;
    }
    if (padding_) *padding_ = padding;
    if (prev_node_) *prev_node_ = prev_node;
    return node;
}
Free_List_Node *free_list_find_best(Free_List *fl, size_t size, size_t alignment, size_t *padding_, Free_List_Node **prev_node_) {
    // This iterates the entire list to find the best fit
    // O(n)
    size_t smallest_diff = ~(size_t)0;
    
    Free_List_Node *node = fl->head;
    Free_List_Node *prev_node = NULL;
    Free_List_Node *best_node = NULL;
    
    size_t padding = 0;
    
    while (node != NULL) {
        padding = calc_padding_with_header((uintptr_t)node, (uintptr_t)alignment, sizeof(Free_List_Allocation_Header));
        size_t required_space = size + padding;
        if (node->block_size >= required_space && (it.block_size - required_space < smallest_diff)) {
            best_node = node;
        }
        prev_node = node;
        node = node->next;
    }
    if (padding_) *padding_ = padding;
    if (prev_node_) *prev_node_ = prev_node;
    return best_node;
}
```

```c
void *free_list_alloc(Free_List *fl, size_t size, size_t alignment) {
    size_t padding = 0;
    Free_List_Node *prev_node = NULL;
    Free_List_Node *node = NULL;
    size_t alignment_padding, required_space, remaining;
    Free_List_Allocation_Header *header_ptr;
    
    if (size < sizeof(Free_List_Node)) {
        size = sizeof(Free_List_Node);
    }
    if (alignment < 8) {
        alignment = 8;
    }
    
    
    if (fl->policy == Placement_Policy_Find_Best) {
        node = free_list_find_best(fl, size, alignment, &padding, &prev_node);
    } else {
        node = free_list_find_first(fl, size, alignment, &padding, &prev_node);
    }
    if (node == NULL) {
        assert(0 && "Free list has no free memory");
        return NULL;
    }
    
    alignment_padding = padding - sizeof(Free_List_Allocation_Header);
    required_space = size + padding;
    remaining = node->block_size - required_space;
    
    if (remaining > 0) {
        Free_List_Node *new_node = (Free_List_Node *)((char *)node + required_space);
        new_node->block_size = rest;
        free_list_node_insert(&fl->head, node, new_node);
    }
    
    free_list_node_remove(&fl->head, prev_node, node);
    
    header_ptr = (Free_List_Allocation_Header *)((char *)node + alignment_padding);
    header_ptr->block_size = required_space;
    header_ptr->padding = alignment_padding;
    
    fl->used += required_space;
    
    return (void *)((char *)header_ptr + sizeof(Free_List_Allocation_Header));
}
```

## [Free and Coalescence](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/#free-and-coalescence)

When freeing a memory block that was allocated with our free list allocator, we need to retrieve the allocation header and that memory block to be treated as a free memory block now. We then need to iterate across the linked list of free memory blocks until will get to the right position in memory order (as the link list is sorted), and then insert new at that position. This can be achieved by looking at the previous and next nodes in the list since they are already sorted by address.

When the insert of the free list, we want to coalescence any free memory blocks which are contiguous. When we were iterating across linked list we had to store both the previous and next free nodes, this means that we may be able to merge these blocks together if possible.

This algorithm has a time complexity of _**O(N)**_, where N is the number of free blocks in the free list.

![Free List Allocator Free and Coalescence](https://www.gingerbill.org/images/memory-allocation-strategies/free_list_allocator_free.svg#center)

```c
void free_list_coalescence(Free_List *fl, Free_List_Node *prev_node, Free_List_Node *free_node);

void *free_list_free(Free_List *fl, void *ptr) {
    Free_List_Allocation_Header *header;
    Free_List_Node *free_node;
    Free_List_Node *node;
    Free_List_Node *prev_node = NULL;
    
    if (ptr == NULL) {
        return;
    }
    
    header = (Free_List_Allocation_Header *)((char *)ptr - sizeof(Free_List_Allocation_Header));
    free_node = (Free_List_Node *)header;
    free_node->block_size = header->block_size + header->padding;
    free_node->next = NULL;
    
    node = fl->head;
    while (node != NULL) {
        if (ptr < node) {
            free_list_node_insert(&fl->head, prev_node, free_node);
            break;
        }
        prev_node = node;
        node = node->next;
    }
    
    fl->used -= free_node->block_size;
    
    free_list_coalescence(fl, prev_node, free_node);
}

void free_list_coalescence(Free_List *fl, Free_List_Node *prev_node, Free_List_Node *free_node) {
    if (free_node->next != NULL && (void *)((char *)free_node + free_node->block_size) == free_node->next) {
        free_node->block_size += free_node->next->block_size;
        free_list_node_remove(&fl->head, free_node, free_node->next);
    }
    
    if (prev_node->next != NULL && (void *)((char *)prev_node + prev_node->block_size) == free_node) {
        prev_node->block_size += free_node->next->block_size;
        free_list_node_remove(&fl->head, prev_node, free_node);
    }
}
```

## [Utilities](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/#utilities)

General utilities needed for free list insertion, removal, and calculating the padding required for the header.

```c
void free_list_node_insert(Free_List_Node **phead, Free_List_Node *prev_node, Free_List_Node *new_node) {
    if (prev_node == NULL) {
        if (*phead != NULL) {
            new_node->next = *phead;
        } else {
            *phead = new_node;
        }
    } else {
        if (prev_node->next == NULL) {
            prev_node->next = new_node;
            new_node->next  = NULL;
        } else {
            new_node->next  = prev_node->next;
            prev_node->next = new_node;
        }
    }
}

void free_list_node_remove(Free_List_Node **phead, Free_List_Node *prev_node, Free_List_Node *del_node) {
    if (prev_node == NULL) { 
        *phead = del_node->next; 
    } else { 
        prev_node->next = del_node->next; 
    } 
}
```

# [Red Black Tree Approach](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/#red-black-tree-approach)

The other way of implementing a free list is with a [red black tree](https://wikipedia.org/wiki/Red%E2%80%93black_tree); the purpose of which is to improve the speed at which allocations and deallocations can be done in. The linked list from above, any operation made is needed to be iterated across linearly (_**O(N)**_). A red black reduces its time complexity to _**O(log(N))**_, whilst keeping the space complexity relatively low (using the same trick as before by storing the tree data within the free memory blocks). And as a consequence of this data-structure approach, a _best-fit_ algorithm may be taken always (in order to reduce fragmentation whilst keeping the allocation/deallocation speed).

The minor increase in space complexity is due to instead of using a singly linked list, a (sorted) doubly linked is required, but as a consequence, it allows coalescence operations in **_O(1)_** time.

This implementation is a common aspect in many `malloc` implementations, but note that most `malloc`s utilize multiple different memory allocation strategies that complement each other.

I will not demonstrate how to implement this approach in this article and leave it as an small exercise for the reader. The following diagram may help:

![Free List Red-Black Tree](https://www.gingerbill.org/images/memory-allocation-strategies/free_list_allocator_red_black_tree.svg#center)

# [Conclusion](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/#conclusion)

The free list allocator is a very useful allocator for when you need to general purpose allocator that requires allocations of arbitrary size and out of order deallocations.

In the next article, I will discuss the [buddy memory allocator](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/).


# Memory Allocation Strategies - Part 6

## Buddy Allocators

Series: [Memory Allocation Strategies](https://www.gingerbill.org/series/memory-allocation-strategies)

2021-12-02

# [Buddy Memory Allocation](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#buddy-memory-allocation)

In the previous article, we discussed the [free list allocator](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/) and how it is commonly implemented with a [linked list](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/#linked-list-approach) or a [red-black tree](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/#red-black-tree-approach). In this article, the Buddy Algorithm and how it applies to memory allocation strategies.

In the previous article, the red black tree approach was briefly discussed as a way to improve the time complexity for searching for a free memory block, and get _best-fit_ as a consequence. One of the big problems with free lists is that they are _very_ susceptible to [internal memory fragmentation](https://wikipedia.org/wiki/Fragmentation_(computing)) due to allocations being of any size. If we still require the properties of free lists but want to reduce internal memory fragmentation, the [Buddy algorithm](https://wikipedia.org/wiki/Buddy_memory_allocation)[1](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#fn:1) works in a similar principle.

# [The Algorithm](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#the-algorithm)

The _Buddy Algorithm_ assumes that the backing memory block is a power-of-two in bytes. When an allocation is requested, the allocator looks for a block whose size is at least the size of the requested allocation (similar to a free list). If the requested allocation size is less than half of the block, it is split into two (left and right), and the two resulting blocks are called “buddies”[2](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#fn:2). If this requested allocation size is still less than half the size of the left buddy, the buddy block is recursively split until the resulting buddy is as small as possible to fit the requested allocation size.

When a block is released, we can try to performance coalescence on buddies (contiguous neighbouring blocks). Similar to [free lists](https://www.gingerbill.org/article/2021/11/30/memory-allocation-strategies-005/#free-and-coalescence), there are particular conditions that are needed. Coalescence cannot be performed if a block is has no (free) buddy, the block is still in use, or the buddy block is partially used.

![Buddy Allocator Splitting Algorithm](https://www.gingerbill.org/images/memory-allocation-strategies/buddy_allocator.svg#center)

# [The Implementation](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#the-implementation)

## [Buddy Block Data Structure](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#buddy-block-data-structure)

Each block in the buddy allocator will have a header (similar to our free list in the previous article) which stores information about it _inline_. It stores its size and whether it is free.

```c
typedef struct Buddy_Block Buddy_Block;
struct Buddy_Block { // Allocation header (metadata)
    size_t size; 
    bool   is_free;
};
```

We do not need store a pointer to the next buddy block as we can calculate it directly from the stored size.

```c
Buddy_Block *buddy_block_next(Buddy_Block *block) {
    return (Buddy_Block *)((char *)block + block->size);
}
```

n.b. Many implementations of a buddy allocator use a doubly linked list here and store explicit pointers, which allows for easier coalescence of neighbouring buddies and forward and backwards traversal. However this does add an extra cost of increasing the size of the allocation header for the memory block.

## [Recursive Split](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#recursive-split)

As described above, to get the best fitting block a recursive splitting algorithm is required. We need to continually split a block until it is the optimal size for the allocation of the requested size.

```c
Buddy_Block *buddy_block_split(Buddy_Block *block, size_t size) {
    if (block != NULL && size != 0) {
        // Recursive split
        while (size < block->size) {
            size_t sz = block->size >> 1;
            block->size = sz;
            block = buddy_block_next(block);
            block->size = sz;
            block->is_free = true;
        }
        
        if (size <= block->size) {
            return block;
        }
    }
    
    // Block cannot fit the requested allocation size
    return NULL;
}
```

## [Finding the Best Block](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#finding-the-best-block)

Searching for a free block that matches the requested allocation size can be achieved by traversing an (implicit) linked list bounded by `head` and `tail` pointers[3](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#fn:3). If a block for the requested allocation size cannot be found, but there is a larger free block, the above splitting algorithm is used. If there is no free block available, the following procedure with return `NULL` to represent that the allocator is (possibly) out of memory[4](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#fn:4).

```c
Buddy_Block *buddy_block_find_best(Buddy_Block *head, Buddy_Block *tail, size_t size) {
    // Assumes size != 0
    
    Buddy_Block *best_block = NULL;
    Buddy_Block *block = head;                    // Left Buddy
    Buddy_Block *buddy = buddy_block_next(block); // Right Buddy
     
    // The entire memory section between head and tail is free, 
    // just call 'buddy_block_split' to get the allocation
    if (buddy == tail && block->is_free) {
        return buddy_block_split(block, size);
    }
    
    // Find the block which is the 'best_block' to requested allocation sized
    while (block < tail && buddy < tail) { // make sure the buddies are within the range
        // If both buddies are free, coalesce them together
        // NOTE: this is an optimization to reduce fragmentation
        //       this could be completely ignored
        if (block->is_free && buddy->is_free && block->size == buddy->size) {
            block->size <<= 1
            if (size <= block->size && (best_block == NULL || block->size <= best_block->size)) {
                best_block = block;
            }
            
            block = buddy_block_next(buddy);
            if (block < tail) {
                // Delay the buddy block for the next iteration
                buddy = buddy_block_next(block);
            }
            continue;
        }
        
                
        if (block->is_free && size <= block->size && 
            (best_block == NULL || block->size <= best_block->size)) {
            best_block = block;
        }
        
        if (buddy->is_free && size <= buddy->size &&
            (best_block == NULL || buddy->size < best_block->size)) { 
            // If each buddy are the same size, then it makes more sense 
            // to pick the buddy as it "bounces around" less
            best_block = buddy;
        }
        
        if (block->size <= buddy->size) {
            block = buddy_block_next(buddy);
            if (block < tail) {
                // Delay the buddy block for the next iteration
                buddy = buddy_block_next(block);
            }
        } else {
            // Buddy was split into smaller blocks
            block = buddy;
            buddy = buddy_block_next(buddy);
        }
    }
    
    if (best_block != NULL) {
        // This will handle the case if the 'best_block' is also the perfect fit
        return buddy_block_split(best_block, size);
    }
    
    // Maybe out of memory
    return NULL;
}
```

This algorithm can suffer from undue internal fragmentation. As an exercise for the reader, you can coalesce on neighbouring free buddies[5](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#fn:5) as you iterate.

## [Initialization](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#initialization)

Initialization of the buddy allocator itself is relatively simple. The allocator itself stores three pieces of information: the `head` block (same the backing memory data), a sentinel pointer `tail` which represents the upper memory boundary of the backing memory data (`(char *)head + size)`, which means it is not a “real” block), and the alignment for each allocation. The procedure `buddy_allocator_init` below does some minor checks for the data itself with `assert`ions.

n.b. This implementation of a buddy allocator does require that all allocations must have the same alignment in order to simplify the code a lot. Buddy allocators are usually a single strategy as part of a more complicated allocator and thus the assumption of alignment is less of an issue in practice.

```c
typedef struct Buddy_Allocator Buddy_Allocator;
struct Buddy_Allocator {
    Buddy_Block *head; // same pointer as the backing memory data
    Buddy_Block *tail; // sentinel pointer representing the memory boundary
    size_t alignment; 
};

void buddy_allocator_init(Buddy_Allocator *b, void *data, size_t size, size_t alignment) {
    assert(data != NULL);
    assert(is_power_of_two(size) && "size is not a power-of-two");
    assert(is_power_of_two(alignment) && "alignment is not a power-of-two");
    
    // The minimum alignment depends on the size of the `Buddy_Block` header
    assert(is_power_of_two(sizeof(Buddy_Block)) == 0);
    if (alignment < sizeof(Buddy_Block)) {
        alignment = sizeof(Buddy_Block);
    }
    assert((uintptr_t)data % alignment == 0 && "data is not aligned to minimum alignment");
    
    b->head          = (Buddy_Block *)data;
    b->head->size    = size;
    b->head->is_free = true;
    
    // The tail here is a sentinel value and not a true block
    b->tail = buddy_block_next(b->head);
    
    b->alignment = alignment;
}
```

## [Allocation](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#allocation)

Allocation is relatively straightforward since we have set everything else up already now. We first need increase requested allocation size to a fit the header and align forward before we find a best fitting block. If one is found, we then need to offset from the header to the usable data. If a block cannot be found, we can keep coalescing any free blocks until we cannot coalesce any more and then try to look for a usable block again. If not block is found, we return `NULL` to signify that we are out of memory with this particular allocator.

```c
size_t buddy_block_size_required(Buddy_Allocator *b, size_t size) {
    size_t actual_size = b->alignment;
    
    size += sizeof(Buddy_Block);
    size = align_forward_size(size, b->alignment); 
    
    while (size > actual_size) {
        actual_size <<= 1;
    }
    
    return actual_size;
}

void buddy_block_coalescence(Buddy_Block *head, Buddy_Block *tail) {
    for (;;) { 
        // Keep looping until there are no more buddies to coalesce
        
        Buddy_Block *block = head;   
        Buddy_Block *buddy = buddy_block_next(block);   
        
        bool no_coalescence = true;
        while (block < tail && buddy < tail) { // make sure the buddies are within the range
            if (block->is_free && buddy->is_free && block->size == buddy->size) {
                // Coalesce buddies into one
                block->size <<= 1;
                block = buddy_block_next(block);
                if (block < tail) {
                    buddy = buddy_block_next(block);
                    no_coalescence = false;
                }
            } else if (block->size < buddy->size) {
                // The buddy block is split into smaller blocks
                block = buddy;
                buddy = buddy_block_next(buddy);
            } else {
                block = buddy_block_next(buddy);
                if (block < tail) {
                    // Leave the buddy block for the next iteration
                    buddy = buddy_block_next(block);
                }
            }
        }
        
        if (no_coalescence) {
            return;
        }
    }
}

void *buddy_allocator_alloc(Buddy_Allocator *b, size_t size) {
    if (size != 0) {    
        size_t actual_size = buddy_block_size_required(b, size);
        
        Buddy_Block *found = buddy_block_find_best(b->head, b->tail, actual_size);
        if (found == NULL) {
            // Try to coalesce all the free buddy blocks and then search again
            buddy_block_coalescence(b->head, b->tail);
            found = buddy_block_find_best(b->head, b->tail, actual_size);
        }
            
        if (found != NULL) {
            found->is_free = false;
            return (void *)((char *)found + b->alignment);
        }
        
        // Out of memory (possibly due to too much internal fragmentation)
    }
    
    return NULL;
}
```

The general time-complexity of this allocation algorithm is _**O(N)**_ on average but a space complexity of _**O(log N)**_.

n.b. As buddy allocators are still susceptible to internal fragmentation; it is less than a normal free list allocator but because of the power-of-two restriction, it is less severe in practice.

## [Freeing Memory](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#freeing-memory)

Freeing memory is very trivial with this algorithm since all we need to do is mark the header (which is stored before the passed pointer) as being free.

```c
void buddy_allocator_free(Buddy_Allocator *b, void *data) {
    if (data != NULL) {
        Buddy_Block *block;
        
        assert(b->head <= data);
        assert(data < b->tail);
        
        block = (Buddy_Block *)((char *)data - b->alignment);
        block->is_free = true;
        
        // NOTE: Coalescence could be done now but it is optional
        // buddy_block_coalescence(b->head, b->tail);
    }
}
```

The general time-complexity of freeing memory is _**O(1)**_. If you wanted to, `buddy_block_coalescence` could be performed straight after this free to aid in minimizing internal fragmentation.

# [Conclusion](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#conclusion)

The buddy allocator is a powerful allocator and a conceptually simple algorithm but implementing it efficiently is a lot harder than all of the previous allocators that have been discussed in this series.

In the next set of articles, I will discuss the a lot about virtual memory; how it works, how we can utilize it, and its benefits.

---

1.  The Wikipedia article is not that easy to understand, especially from the basic table diagram given in the _Example_ section. (Accessed 2021-12-01) [↩︎](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#fnref:1)
    
2.  Just like Jackie Chan and Chris Tucker in [Rush Hour](https://www.imdb.com/title/tt0120812/). [↩︎](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#fnref:2)
    
3.  The tail is just `(Buddy_Block *)((char *)data + size)` of the backing memory buffer, representing a sentinel value of the memory boundary, it is not a true block. [↩︎](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#fnref:3)
    
4.  The allocator may have enough memory left but none of it is contiguous due to too much internal fragmentation. [↩︎](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#fnref:4)
    
5.  All becoming a single buddy, trying to be someone else: [https://www.imdb.com/title/tt0120601/](https://www.imdb.com/title/tt0120601/) [↩︎](https://www.gingerbill.org/article/2021/12/02/memory-allocation-strategies-006/#fnref:5)