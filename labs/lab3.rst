.. warning::

   This document is currently under construction and may be incomplete or subject to significant changes.
   Please check back later for updates, and consult the instructor if you are unsure about any missing parts.


=======================
Lab 3: Memory Allocator
=======================

############
Introduction
############

A kernel allocates physical memory for maintaining its internal states and user programs' use.
Without memory allocators, you need to statically partition the physical memory into several memory pools for
different objects.
It's sufficient for some systems that run known applications on known devices.
Yet, general-purpose operating systems that run diverse applications on diverse devices determine the use and amount
of physical memory at runtime. On VF2, available physical memory and reserved regions are described by the Device Tree,
so you must read those descriptions at boot.

In Lab 3, you need to implement memory allocators that rely on VF2 Device Tree for memory layout.
They'll be used in all later labs.


#################
Goals of this lab
#################

* Implement a Startup Allocator (bump-pointer) that uses the addresses retrieved by Lab 2's DT parser:
  * DTB Blob
  * Kernel Image
  * Initramfs
  * Any implementer-defined reserved regions
  to reserve all those areas before setting up the Page Frame Array.

* Implement a Page Frame Allocator that uses Lab 2's `mem_regions[]` array for all physical RAM segments.

* Implement a Dynamic Memory Allocator that calls `p_alloc(1)` (from the Page Frame Allocator) and partitions pages into chunk pools.



##########
Background
##########

Reserved Memory
================

After VF2 is booted, the following memory regions must be reserved before any allocation:
  1. Device Tree Blob (DTB)
  2. Kernel image
  3. Initramfs (ramdisk)
  4. Any additional platform-specific reserved areas (e.g., spin tables, CLINT/PLIC buffers)

Since Lab 2 already performed DT parsing, use the parsed data directly:
  • For the DTB: use the ``fdt_ptr`` value (passed in a1) and ``fdt_totalsize(fdt_ptr)`` to obtain start and size.
  • For the Kernel image: use linker symbols (e.g., `&_phys_start` and `&_phys_end`) to obtain its physical range.
  • For the Initramfs: use ``/chosen`` properties ``linux,initrd-start`` and ``linux,initrd-end`` to obtain its range.
  • For any other reserved areas defined in the DTS: they should be listed by the implementer and retrieved from the parsed DT data.

Before initializing any allocator, call:
    memory_reserve(dtb_start, dtb_size);
    memory_reserve(kernel_start, kernel_size);
    memory_reserve(initrd_start, initrd_size);
    // plus any other implementer‐defined reserved regions from DT parsing
to mark those frames as reserved.


Page Frame Allocator
======================

You may be wondering why you are asked to implement another memory allocator for page frames.
Isn't a well-designed dynamic memory allocator enough for all dynamic memory allocation cases?
Indeed, you don't need it if you run applications in kernel space only.

However, if you need to run user space applications with virtual memory,
you'll need a lot of 4KB memory blocks with 4KB memory alignment called page frames.
That's because 4KB is the unit of virtual memory mapping.
A regular dynamic memory allocator that uses a size header before memory block turns out half of the physical memory
can't be used as page frames.
Therefore, a better approach is representing available physical memory as page frames.
The page frame allocator reserves and uses an additional page frame array to bookkeep the use of page frames.
The page frame allocator should be able to allocate contiguous page frames for allocating large buffers.
For fine-grained memory allocation, a dynamic memory allocator can allocate a page frame first then cut it into chunks.


Dynamic Memory Allocator
========================

Given the allocation size,
a dynamic memory allocator needs to find a large enough contiguous memory block and return the pointer to it.
Also, the memory block would be released after use.
A dynamic memory allocator should be able to reuse the memory block for another memory allocation.
Reserved or allocated memory should not be allocated again to prevent data corruption.


Observability of Allocators
============================

It's hard to observe the internal state of a memory allocator and hence hard to demo.
To check the correctness of your allocator on VF2, you need to **print the log of each allocation and free**
over the VF2 UART driver. Assume Lab 2 already initialized UART from the Device Tree's `stdout-path`.

.. note::
  TAs will verify correctness by these logs during the demo. Prefix each log message with “[VF2 Allocator] ”
  to differentiate from other kernel output.


###############
Basic Exercises
###############

In the basic part, your allocator can focus on a single allocable memory region provided by the VF2 Device Tree's <memory> node
and does not need to handle reserved-memory entries. Choose one available memory segment from the Device Tree
(for example, the first <memory> region) and manage that part of memory only.

Basic Exercise 1 - Buddy System - 40%
=====================================

Buddy system is a well-known and simple algorithm for allocating contiguous memory blocks.
It has an internal fragmentation problem, but it's still suitable for page frame allocation 
because the problem can be reduced with the dynamic memory allocator.
We provide one possible implementation in the following part.
You can still design it yourself as long as you follow the specification of the buddy system.

.. admonition:: Todo

    Implement the buddy system for contiguous page frame allocation on VF2. Your maximum order must be larger than 5.
    Use Lab 2's `mem_regions[]` array (each entry contains base and size of a RAM segment)
    to compute total page count and physical frame addresses, ignoring any gaps between segments.

.. note::

  You don't need to handle the case of out-of-memory.

Data Structure
----------------

**The Frame Array** (or *"The Array"*, so to speak)

*The Array* represents the allocation status of the memory by constructing a 1-1 relationship between the physical memory frame and *The Array*'s entries.
For example, if VF2 Device Tree indicates two allocable regions totaling 200 KiB with each frame being 4 KiB,
then The Array would consist of 50 entries. Use the base addresses from the <memory> regions to compute
each frame's physical address (e.g., if the first region begins at 0x8000_0000, its first entry represents 0x8000_0000,
next 0x8000_1000, etc.).

However, to describe a living Buddy system with *The Array*, we need to provide extra meaning to items in *The Array* by assigning values to them, defined as followed:

For each entry in *The Array* with index :math:`\text{idx}` and value :math:`\text{val}`
  (Suppose the framesize to be ``4kb``)

  if :math:`\text{val} \geq 0`:
    There is an allocable, contiguous memory that starts from the :math:`\text{idx}`'th frame with :math:`\text{size} = 2^{\text{val}}` :math:`\times` ``4kb``.

  if :math:`\text{val} = \text{<F>}`: (user defined value)
    The :math:`\text{idx}`'th frame is free, but it belongs to a larger contiguous memory block. Hence, buddy system doesn't directly allocate it.

  if :math:`\text{val} = \text{<X>}`: (user defined value)
    The :math:`\text{idx}`'th frame is already allocated, hence not allocable.

.. image:: /images/buddy_frame_array.svg

Below is the generalized view of **The Frame Array**:

.. image:: /images/buddy.svg


You can calculate the address and the size of the contiguous block by the following formula.

+ :math:`\text{block's physical address} = \text{block's index} \times 4096 +  \text{base address}`
+ :math:`\text{block's size} = 4096 \times 2^\text{block's exponent}`

Linked-lists for blocks with different size (VF2 Device Tree <memory> node)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
You can set a maximum contiguous block size and create one linked-list for each size.
The linked-list links free blocks of the same size.
The buddy allocator's search starts from the specified block size list.
If the list is empty, it tries to find a larger block in a larger block list

.. _release_redu:

Release redundant memory block
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The above algorithm may allocate one block far larger than the required size.
The allocator should cut off the bottom half of the block and put it back to the buddy system until the size equals the required size.

.. note::
  You should print the log of releasing redundant memory block (via VF2 UART) for the demo

Free and Coalesce Blocks
--------------------------
To allow the buddy system to reconstruct larger contiguous memory blocks on VF2,
when the user frees an allocated block, the buddy allocator should not naively place it back on the free list.
Instead, it must call find_buddy() and merge_iter(), using page frame indices computed from
VF2 Device Tree's <memory> base addresses.

.. _find_buddy:

Find the buddy
^^^^^^^^^^^^^^

On VF2, compute each block's page frame index relative to the <memory> region base address.
Then use index XOR exponent to find its buddy's index. If the buddy lies within the same <memory> region,
merge them into a larger block.

.. _merge_iter:

Merge iteratively
^^^^^^^^^^^^^^^^^
There is still a possible buddy for the merged block.
You should use the same way to find the buddy of the merge block.
When you can't find the buddy of the merged block or the merged block size is maximum-block-size, 
the allocator stops and put the merged block to the linked-list.

.. note::
  You should print the log of merge iteration for the demo.

Basic Exercise 2 - Dynamic Memory Allocator - 30%
=================================================

Your Page Frame Allocator (from Basic Exercise 1) provides 4 KB-aligned page frames via `p_alloc(1)`.
The Dynamic Memory Allocator must call `p_alloc(1)` to obtain one page and use its physical base address.
For small allocations (< 4 KB), maintain multiple chunk pools (sizes 16, 32, 48, 96 bytes, etc.),
partitioning each page into fixed-size chunks for the corresponding pool.

On each allocation request:
  1. Round up the requested size to the nearest pool size.
  2. If a free chunk exists in the corresponding pool, return it; otherwise, request a new page from the Page Frame Allocator.
  3. Slice the new page frame into chunks and add them to the pool's free list, then return one chunk.
When freeing a chunk, use its base page frame address to identify which pool it belongs to,
and place it back onto that pool's free list.

.. admonition:: Todo

    Implement a dynamic memory allocator.
    

##################
Advanced Exercises
##################

.. _startup_alloc:

Advanced Exercise 1 - Efficient Page Allocation on VF2 Device Tree - 10%
=====================================================

Basically, when you dynamically assign or free a page on VF2, your buddy system's response time should be as quick as possible.
In the basic part, your allocator can focus on one of the parsed memory segments from Lab 2's `mem_regions[]` array
and does not need to handle reserved regions. Simply pick a single `mem_regions[i]` (e.g., the first entry)
and manage that contiguous block of RAM only.


.. admonition:: Todo

   You should allocate and free a page in O(log n), while ensuring any page frame lookup is O(1).

Advanced Exercise 2 - Reserved Memory via VF2 Device Tree - 10%
===========================================

As previously noted in the background, when VF2 is booted, specific memory regions must be reserved.  
Since Lab 2 already parsed the Device Tree, use the parsed values directly to reserve:

  1. DTB Blob:
     • `dtb_start = (uint64_t)fdt_ptr;`
     • `dtb_size  = fdt_totalsize(fdt_ptr);`
     Call `memory_reserve(dtb_start, dtb_size);`
  2. Kernel Image:
     • `kernel_start = (uint64_t)&_phys_start;`
     • `kernel_end   = (uint64_t)&_phys_end;`
     • `kernel_size  = kernel_end - kernel_start;`
     Call `memory_reserve(kernel_start, kernel_size);`
  3. Initramfs:
     • `initrd_start = fdt_getprop_u64(fdt_ptr, "/chosen", "linux,initrd-start");`
     • `initrd_end   = fdt_getprop_u64(fdt_ptr, "/chosen", "linux,initrd-end");`
     • `initrd_size  = initrd_end - initrd_start;`
     Call `memory_reserve(initrd_start, initrd_size);`
  4. Any additional reserved regions (e.g., spin tables, DMA buffers, platform-specific):
     • These must be defined by the implementer (e.g., hardcoded in C or provided via `/memreserve/` in DTS)
     • Retrieve their `(base, size)` pairs from the parsed DT data
     • Call `memory_reserve(base, size);` for each.

.. code:: c

  void memory_reserve(uint64_t start, uint64_t size) {
      // Mark all 4 KB page frames in [start, start + size) as reserved
  }

.. admonition:: Todo

   Use the values already obtained by Lab 2's DT parser and call `memory_reserve()` for each of the four categories above.


Advanced Exercise 3 - Startup Allocation with VF2 Device Tree - 20%
==============================================
In general purpose operating systems, the amount of physical memory is determined at runtime. Hence, a kernel needs to dynamically allocate its page frame array for its page frame allocator. The page frame allocator then depends on dynamic memory allocation. The dynamic memory allocator depends on the page frame allocator. This introduces the chicken or the egg problem. To break the dilemma, you need a dedicated startup allocator during startup time.

The design of the startup allocator is quite simple. Implement a minimal bump-pointer allocator at startup that does not rely on the Page Frame Allocator.  
Lab 2 has already parsed all `<memory>` nodes and retrieved:
  • An array `mem_regions[]` of `{ base, size }` pairs for all available RAM segments.

This bump allocator must:
  1. Iterate over `mem_regions[]` (from Lab 2) to compute the combined usable memory segments.
  2. Before allocating any frame array or other structures, reserve:
     • DTB Blob: call memory_reserve(dtb_start, dtb_size).
     • Kernel Image: call memory_reserve(kernel_start, kernel_size).
     • Initramfs:   call memory_reserve(initrd_start, initrd_size).
     • Any implementer-defined reserved regions (from Lab 2's parsed DT data): call memory_reserve(base, size).
  3. From the first usable `mem_regions[i]` after those reservations, allocate a contiguous region (4KB-aligned)
     of length `frame_array_size = total_page_count × sizeof(struct frame_struct)`:
       • `frame_array_start = align_up(mem_regions[i].base, 4096);`
       • Advance `bump_ptr` by `frame_array_size`.
       • Call `memory_reserve(frame_array_start, frame_array_size)` to mark it as reserved.
  4. Initialize the Page Frame Array data structures in that region.
  5. Handover control to the Buddy System, which will manage all remaining free 4 KB page frames.

This bump allocator is used only during early boot (before Buddy System is ready).

.. admonition:: Todo

   Use Lab 2's `mem_regions[]`, plus the four categories of reservation from above, to implement `startup_alloc()`.

.. note::
  * Your buddy system must handle VF2's total physical memory and any holes reported by the Device Tree.
    Read all <memory> regions from the VF2 Device Tree to determine usable segments.
  * All usable memory regions must be used to build the Page Frame Array dynamically via the Startup Allocator.
    Allocate the Page Frame Array out of those ranges, skipping reserved areas.
  * Reserved memory block detection is not part of the Startup Allocator itself.
    Instead, Startup Allocator must call `memory_reserve(start, size)` for each entry in <reserved-memory>.
  * Do not hardcode any physical addresses. All memory ranges (usable or reserved) must be obtained from
    the VF2 Device Tree:

    1. Kernel image region
    2. Initramfs region
    3. Device Tree blob itself
    4. Any additional platform-specific reserved areas (OpenSBI, Framebuffer, etc.)
