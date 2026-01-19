.. warning::

   This document is currently under construction and may be incomplete or subject to significant changes.
   Please check back later for updates, and consult the instructor if you are unsure about any missing parts.

==============================
Lab 4: Exception and Interrupt
==============================

############
Introduction
############

An exception is an event that causes the currently executing program to relinquish the CPU to the corresponding handler.
With the exception mechanism, an operating system can 

1. handle errors properly during execution,
2. allow user programs to request system services,
3. respond to peripheral devices that require immediate attention.

#################
Goals of this lab
#################

* Understand the exception mechanism in RISC-V.
* Understand how interrupt delegation works in the VF2 platform.
* Configure and handle timer interrupts using the CLINT.
* Understand and handle UART interrupts via the PLIC.
* Learn how to multiplex timers and schedule asynchronous tasks.

##########
Background
##########

Official Reference
==================

Exceptions and interrupts in RISC-V are defined in the official privileged specification. For details, see:

- RISC-V Privileged Architecture Manual: https://github.com/riscv/riscv-isa-manual/releases

Exception Levels (Privilege Modes)
==================================

RISC-V defines four privilege modes: User, Supervisor, Hypervisor (optional), and Machine mode.
For simplicity, VF2 boots in **Machine mode (M-mode)**, and most bare-metal kernels are executed in M-mode.

In this lab, you will run both kernel and user-mode programs, using `mret` to switch from M-mode to U-mode,
and configuring trap handling via `mtvec`, `mscratch`, `mepc`, `mcause`, `mstatus`, and `medeleg`/`mideleg`.

Interrupts and the CLINT
========================

The Core Local Interruptor (CLINT) is responsible for managing timer and software interrupts for each core.

Registers include:

- `mtime`: a 64-bit timer that increments at a fixed frequency.
- `mtimecmp[h]`: a 64-bit comparator register that triggers an interrupt when `mtime >= mtimecmp`.

Each hart (hardware thread) has its own `mtimecmp` register.

Interrupt Controllers - PLIC
============================

VF2 uses the **Platform-Level Interrupt Controller (PLIC)** to handle external interrupts from devices such as UART.

Key facts:

- Each device interrupt has an ID (e.g., UART0 is usually ID 10).
- PLIC routes interrupt requests to CPU cores with a priority mechanism.
- Each hart has context-specific registers to claim/complete interrupts.

See JH7110 documentation or DTB for actual interrupt IDs and PLIC base addresses.

Critical Sections
=================

As in all interrupt-driven systems, shared data must be protected from concurrent access during interrupt handling.
In RISC-V, this can be done by disabling interrupts via `csrci mstatus, MSTATUS_MIE` and re-enabling via `csrsi`.

###############
Basic Exercises
###############

Basic Exercise 1 - Exception  - 30%
===================================

Mode Switch: M-mode to U-mode
-----------------------------

After booting in M-mode, configure registers to switch to U-mode and run user-level programs.
Setup includes:

1. Writing user program address to `mepc`
2. Setting `mstatus` to enable interrupts and select U-mode
3. Using `mret` to jump to U-mode

Trap Handling from U-mode
-------------------------

When the user program executes an `ecall`, it traps to the M-mode handler.
You need to:

- Set the trap vector address in `mtvec`
- Save user context (`x1-x31`, `mepc`, `mstatus`)
- Print diagnostic info from `mcause`, `mepc`, `mtval`
- Restore context and return to user using `mret`

Basic Exercise 2 - Core Timer Interrupt - 10%
=============================================

Enable `mtimecmp` to raise a timer interrupt after a specified period.
Steps:

- Read `mtime`
- Add time delta (e.g., equivalent of 1 second)
- Write to `mtimecmp`
- Enable `MIE` in `mstatus` and `MTIE` in `mie`
- In the interrupt handler, acknowledge and reprogram `mtimecmp`

Print boot-time seconds in the handler.

Basic Exercise 3 - VF2 UART0 Interrupt - 30%
=============================================

Enable UART0 interrupt via:

- UART interrupt enable register (check VF2 SoC manual or DTS; likely `UART0.IER`)
- Enable UART interrupt ID (e.g., 10) in the PLIC
- Set `mie.MEIE` and enable external interrupts globally

Steps:

1. Setup read/write buffers.
2. Implement ISR for UART RX and TX.
3. In RX, place incoming bytes in buffer.
4. In TX, send data from buffer when ready.

Make shell non-blocking by using buffer for input/output.

##################
Advanced Exercises
##################

Advanced Exercise 1 - Timer Multiplexing - 20%
==============================================

Use `add_timer(callback, duration)` API to schedule deferred tasks.
Use a software-managed priority queue (e.g., min-heap or sorted list) to keep track of expiration time and reprogram `mtimecmp` accordingly.

Advanced Exercise 2 - Concurrent I/O Devices Handling 20%
=========================================================

Replace blocking UART logic with event queue mechanism:

- ISR enqueues tasks to be handled outside critical path
- Enable nesting and priority-based dispatch
- Protect shared state using interrupt masking (`mstatus`)
- Before returning from handler, process pending tasks with interrupts re-enabled

Preemption can be implemented by checking the event queue for higher-priority tasks before final return from trap.

.. note::

    Use nested interrupt handling and task prioritization to support fair and responsive device scheduling.

