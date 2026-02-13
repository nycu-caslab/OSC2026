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
of physical memory at runtime. On OrangePi RV2, available physical memory and reserved regions are described by the Device Tree,
so you must read those descriptions at boot.

In Lab 3, you need to implement memory allocators that rely on OrangePi RV2's Device Tree for memory layout.
They'll be used in all later labs.


#################
Goals of this lab
#################

- Implement a **Page Frame Allocator**.
- Implement a **Dynamic Memory Allocator** that requests pages from the previous Page Frame Allocator and partitions pages into chunk pools.
- Implement a **Startup Allocator** (bump-pointer).

##########
Background
##########

Reserved Memory
================

After OrangePi RV2 is booted, some physical memory is already in use.
For example:

1. Device Tree Blob (DTB)
2. Kernel image
3. Initramfs (ramdisk)
4. Any additional platform-specific reserved areas (e.g., spin tables, CLINT/PLIC buffers)

Your memory allocator should not allocate these memory blocks.
Thus, you need to reserve these memory blocks before initializing any allocator.

Dynamic Memory Allocator
========================

Given the allocation size,
a dynamic memory allocator needs to find a large enough contiguous memory block and return the pointer to it.
Also, the memory block would be released after use.
A dynamic memory allocator should be able to reuse the memory block for another memory allocation.
Reserved or allocated memory should not be allocated again to prevent data corruption.

Page Frame Allocator
======================

You may be wondering why you are asked to implement another memory allocator for page frames.
Isn't a well-designed dynamic memory allocator enough for all dynamic memory allocation cases?
Indeed, you don't need it if you run applications in kernel space only.

However, if you need to run user space applications with virtual memory,
you'll need a lot of 4KB memory blocks with 4KB memory alignment called page frames.
That's because 4KB is the unit of virtual memory mapping.
A regular dynamic memory allocator that store metadata (like size headers) within the memory block are unsuitable for managing page frames.
Since page frames must be strictly 4KB-aligned, an inline header would shift the page boundary, forcing the allocator to waste significant space to maintain alignment.

Therefore, a better approach is representing available physical memory as page frames.
The page frame allocator reserves and uses an additional page frame array to bookkeep the use of page frames.
The page frame allocator should be able to allocate contiguous page frames for allocating large buffers.
For fine-grained memory allocation, a dynamic memory allocator can allocate a page frame first then cut it into chunks.

Observability of Allocators
============================

It's hard to observe the internal state of a memory allocator and hence hard to demo.
To check the correctness of your allocator, you need to **print the log of each allocation and free**.

.. note::

  TAs will verify correctness by these logs during the demo.

###############
Basic Exercises
###############

In the basic part, your allocator can focus on a single allocable memory region provided by the Device Tree's ``<memory>`` node
and does not need to handle reserved-memory entries. Choose one available memory segment from the Device Tree
(for example, the first ``<memory>`` region) and manage that part of memory only.

.. note::

  There may be many memory regions in the Device Tree. For example, OrangePi RV2 has two memory regions:

  - ``0x0000_0000_0000_0000`` - ``0x0000_0000_7FFF_FFFF`` (2 GiB)
  - ``0x0000_0001_0000_0000`` - ``0x0000_0001_7FFF_FFFF`` (2 GiB)

  You can only use the first memory region for the basic part.
  However, you are also encouraged to handle all memory regions in the Device Tree.

.. admonition:: Todo

  Retrieve the memory regions from the Device Tree ``<memory>`` node.
  Please do not hardcode them, since they may depend on the platform.

Basic Exercise 1 - Buddy System - 40%
=====================================

Buddy system is a well-known and simple algorithm for allocating contiguous memory blocks.
It has an internal fragmentation problem, but it's still suitable for page frame allocation 
because the problem can be reduced with the dynamic memory allocator.
We provide one possible implementation in the following part.
You can still design it yourself as long as you follow the specification of the buddy system.

.. admonition:: Todo

  Implement the buddy system for contiguous page frame allocation.
  Your maximum order must be larger than 5.

.. note::

  You don't need to handle the case of out-of-memory.

Data Structure
----------------

**The Frame Array** (or *"The Array"*, so to speak)

*The Array* represents the allocation status of the memory by constructing a 1-1 relationship between the physical memory frame and *The Array*'s entries.
For example, if OrangePi RV2 Device Tree indicates two allocable regions,
totaling 200 KiB with each frame being 4 KiB, then The Array would consist of 50 entries.
Use the base addresses from the ``<memory>`` regions to compute each frame's physical address
(e.g., if the first region begins at ``0x8000_0000``, its first entry represents ``0x8000_0000``, next ``0x8000_1000``, etc.).

However, to describe a living Buddy system with *The Array*, we need to provide extra meaning to items in *The Array* by assigning values to them, defined as followed:

For each entry in *The Array* with index :math:`\text{idx}` and value :math:`\text{val}`
(Suppose the framesize to be ``4kb``)

- if :math:`\text{val} \geq 0`:

  There is an allocable, contiguous memory that starts from the :math:`\text{idx}`'th frame with :math:`\text{size} = 2^{\text{val}}` :math:`\times` ``4kb``.

- if :math:`\text{val} = \text{<F>}`: (user defined value)

  The :math:`\text{idx}`'th frame is free, but it belongs to a larger contiguous memory block. Hence, buddy system doesn't directly allocate it.

- if :math:`\text{val} = \text{<X>}`: (user defined value)

  The :math:`\text{idx}`'th frame is already allocated, hence not allocable.

.. image:: /images/buddy_frame_array.svg

\

Below is the generalized view of **The Frame Array**:

.. image:: /images/buddy.svg

\

You can calculate the address and the size of the contiguous block by the following formula.

+ :math:`\text{block's physical address} = \text{block's index} \times 4096 +  \text{base address}`
+ :math:`\text{block's size} = 4096 \times 2^\text{block's exponent}`

Linked-lists for free blocks with different size (Free List)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
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

  You should print the log of releasing redundant memory block for the demo

Free and Coalesce Blocks
--------------------------
To allow the buddy system to reconstruct larger contiguous memory blocks,
when the user frees an allocated block, the buddy allocator should not naively place it back on the free list.
It should try to :ref:`find_buddy` and :ref:`merge_iter`.

.. _find_buddy:

Find the buddy
^^^^^^^^^^^^^^

You can use the block's index xor with its exponent to find its buddy.
If its buddy is in the page frame array, then you can merge them to a larger block.

.. _merge_iter:

Merge iteratively
^^^^^^^^^^^^^^^^^
There is still a possible buddy for the merged block.
You should use the same way to find the buddy of the merge block.
When you can't find the buddy of the merged block or the merged block size is maximum-block-size, 
the allocator stops and put the merged block to the free list.

.. note::

  You should print the log of merge iteration for the demo.

Basic Exercise 2 - Dynamic Memory Allocator - 30%
=================================================

Your Page Frame Allocator (from Basic Exercise 1) provides 4 KB-aligned page frames.
The Dynamic Memory Allocator must request page frames from the Page Frame Allocator and use their physical base address.
For small allocations (< 4 KB), you may maintain multiple chunk pools (sizes 16, 32, 48, 96 bytes, etc.),
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

Advanced Exercise 1 - Efficient Page Allocation - 10%
=====================================================

Basically, when you dynamically assign or free a page, your buddy system's response time should be as quick as possible.
In the basic part, we only care about the correctness of your implementation, but in this section, you must devise a technique to access the page frame node in constant time.

.. admonition:: Todo

  You should allocate and free a page in O(log n), while ensuring any page frame lookup is O(1).

Advanced Exercise 2 - Reserved Memory via VF2 Device Tree - 10%
===============================================================

As previously noted in the background, when VF2 is booted, specific memory regions must be reserved.  
Since Lab 2 already parsed the Device Tree, use the parsed values directly to reserve:

1. DTB Blob: use the ``fdt_ptr`` and ``fdt_header.totalsize`` to obtain start and size.
2. Kernel Image: use linker symbols (e.g., ``&_phys_start`` and ``&_phys_end``) to obtain its physical range.
3. Initramfs: use ``/chosen`` properties ``linux,initrd-start`` and ``linux,initrd-end`` to obtain its range.
4. Any additional reserved regions (if have, e.g., spin tables, DMA buffers, platform-specific):

   - You may hardcode in C, or retrieve via ``/reserved-memory`` node in the device tree.

The following code is a brief example, you may design your own:

.. code:: c

  void memory_reserve(uint64_t start, uint64_t size) {
    // Mark all 4 KB page frames in [start, start + size) as reserved
  }

.. admonition:: Todo

   Design an API to reserve memory blocks.
   Use the values already obtained by Lab 2's DT parser and call ``memory_reserve()`` for each of the four categories above.

.. _startup_alloc:

Advanced Exercise 3 - Startup Allocation - 20%
==============================================
In general purpose operating systems, the amount of physical memory is determined at runtime.
Hence, a kernel needs to dynamically allocate its page frame array for its page frame allocator (e.g., *The Frame Array*).
The page frame allocator then depends on dynamic memory allocation.
The dynamic memory allocator depends on the page frame allocator.
This introduces the chicken or the egg problem.
To break the dilemma, you need a dedicated startup allocator during startup time.

The design of the startup allocator is quite simple. Implement a minimal bump-pointer allocator at startup that does not rely on the Page Frame Allocator.

This startup allocator must:

1. Before allocating The Frame Array or other structures, mark reserved regions such as DTB blob, kernel image, etc.
2. From the first usable memory region after those reservations, allocate a contiguous region (4KB-aligned)
   of length ``frame_array_size = total_page_count × sizeof(struct frame_struct)``.
3. Initialize the Page Frame Array data structures in that region.
4. Handover control to the Buddy System, which will manage all remaining free 4 KB page frames.

This startup allocator is used only during early boot (before Buddy System is ready).

.. note::
  "Mark reserved regions" is to detect the reserved regions and prevent the startup allocator from allocating them.

.. admonition:: Todo

  Implement the startup allocation.

.. admonition:: Note for this lab

  - Your buddy system must handle OrangePi RV2's total physical memory and any holes reported by the Device Tree.
    Read all ``<memory>`` regions from the OrangePi RV2 Device Tree to determine usable segments.
  - All usable memory regions must be used to build the Page Frame Array dynamically via the Startup Allocator.
    Allocate the Page Frame Array out of those ranges, skipping reserved areas.
  - Reserved memory block detection is not part of the Startup Allocator itself.
    Instead, Startup Allocator must call `memory_reserve(start, size)` for each entry in <reserved-memory>.
  - Do not hardcode any physical addresses. All memory ranges (usable or reserved) must be obtained from
    the OrangePi RV2 Device Tree:

    1. Kernel image region
    2. Initramfs region
    3. Device Tree blob itself
    4. Any additional platform-specific reserved areas (OpenSBI, Framebuffer, etc.)
