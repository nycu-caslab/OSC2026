.. warning::

   This document is currently under construction and may be incomplete or subject to significant changes.
   Please check back later for updates, and consult the instructor if you are unsure about any missing parts.

========================
Lab 2: Booting
========================

*************
Introduction
*************

Booting is the process of initializing the system environment to run various user programs after a computer reset. This includes loading the kernel, initializing subsystems, matching device drivers, and launching the initial user program to bring up remaining services in user space.

In Lab 2, you'll implement a bootloader for the VF2 board that loads kernel images through UART. Additionally, you'll develop a simple allocator and gain an understanding of initial ramdisk and device trees.

*****************
Goals of this lab
*****************

- Implement a bootloader that loads kernel images through UART.
- Implement a simple memory allocator.
- Understand the concept and usage of initial ramdisk.
- Understand the structure and purpose of device trees.

************
Background
************

The boot process on the VF2 involves multiple stages, including the execution of the bootloader, loading of the kernel, and initialization of system components. Understanding this process is crucial for developing low-level system software.

In this lab, you'll implement a bootloader that loads the actual kernel through UART, providing flexibility during development and debugging.

*****************
Basic Exercises
*****************

Basic Exercise 1 - UART Bootloader - 30%
#########################################

Loading the kernel image onto the VF2 board can be streamlined by implementing a bootloader that receives the kernel over UART. This eliminates the need to physically move storage media between the host and the VF2 during development.

.. admonition:: Todo

    Develop a bootloader that listens for a kernel image transmitted over UART and loads it into memory for execution. Design a simple protocol for data transmission to ensure reliability.

The bootloader is typically loaded and executed from address ``0x40200000``, which corresponds to the on-chip SRAM on the VF2 board and is the standard entry point used by the SoC after reset.

To avoid overwriting itself, the bootloader **must not** load the kernel into this same region. A common practice is to load the kernel into DDR memory, such as at address ``0x80000000``, while keeping the bootloader resident in SRAM.
A safe practice is to consult the device tree (``/memory`` node) to determine the available RAM layout. Also see *Basic Exercise 3: Device Tree* for more details.

Relocation is covered in *Advanced Exercise* and is optional. If you choose to implement self-relocation, the bootloader can move itself to another region (e.g., higher DRAM) before loading the kernel. Otherwise, you must ensure the kernel and initial ramdisk are loaded into memory regions that do not overlap with the bootloader's footprint.

.. note::

   If you have extended the shell from the previous lab, your bootloader may already support basic device tree parsing and provide a simple command interface. In that case, you can implement a ``load_kernel`` command to receive the kernel image over UART, allowing your shell to act as a convenient interface for development.

   This bootloader can then serve as a stable "fork point" from which both kernel development and system-level testing can proceed. Since the same shell is reused, you avoid duplicating early-stage code and can leverage its existing device tree parsing functionality. 


Basic Exercise 2 - Initial Ramdisk - 30%
#########################################

An initial ramdisk (initrd) is a temporary root file system loaded into memory during the boot process. It provides essential files and drivers needed to mount the actual root file system.

.. admonition:: Todo

    Create an initial ramdisk containing necessary utilities and drivers for the VF2. Modify the bootloader to load this initrd into memory during the boot process.


.. code-block:: dts
   :caption: Modified ``kernel.its`` with ramdisk section (simplified)

   /dts-v1/;

   / {
       description = "FIT Image with Kernel, Initrd, and DTB";
       images {
           kernel { ... };
           fdt { ... };
           ramdisk {
               data = /incbin/("initramfs.cpio");
               type = "ramdisk";
               arch = "riscv";
               os = "linux";
               compression = "none";
           };
       };

       configurations {
           default = "conf";
           conf {
               kernel = "kernel";
               fdt = "fdt";
               ramdisk = "ramdisk";
           };
       };
   };


Basic Exercise 3 - Device Tree - 30%
#######################################

A device tree is a data structure that describes the hardware components of a system. It provides the operating system with information about the available hardware without hardcoding details into the kernel.

.. admonition:: Todo

    Integrate device tree support into your bootloader. Parse the device tree to initialize hardware components appropriately during the boot process.

.. TODO: explain in detail how to parse the device tree and use it to initialize hardware components; 
   include both structural explanation and sample code or reference. 
   Also mention how to retrieve memory layout and related properties from the DT.

********************
Advanced Exercise
********************

Advanced Exercise - Bootloader Self-Relocation - 20%
#######################################################

To accommodate different memory layouts and ensure compatibility, a bootloader may need to relocate itself during execution.

.. admonition:: Todo

    Modify the bootloader to support self-relocation, allowing it to move to a different memory region
    according to the memory layout obtained from the devicetree.
    
.. note::

   When relocating the bootloader, make sure the destination region does not overlap with 
   the currently loaded kernel or the initrd image.
   A typical memory layout might consider the addresses and sizes of 
   `Bootloader`, `Kernel Image` to be loaded, 
   `Device Tree Blob` (DTB) and `Initial Ramdisk` (``initrd``). 
   You need to consult the devicetree to decide the memory layout and adjust the linker script
   to properly decide the relocation process, especially if the ``initramfs.cpio`` you prepared is very large.
   

..    - ``0x40200000`` – Bootloader (original location)
..    - ``0x40100000`` – Bootloader relocated copy
..    - ``0x40200000`` – Kernel image
..    - ``0x46000000`` – Device Tree Blob (DTB)
..    - ``0x46100000`` – Initrd

