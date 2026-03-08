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
* Understand how interrupt delegation works in the OrangePi RV2 platform.
* Configure and handle core timer interrupts using the SBI Timer Extension.
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

RISC-V defines privilege modes to isolate different system components.
In our OS design, the kernel executes in **Supervisor mode (S-mode)**, while user applications execute in **User mode (U-mode)**.

.. image:: /images/RISC_privilege.png
.. :align: left

In this lab, you will run both kernel and user-mode programs, using `sret` to switch from S-mode to U-mode, and configuring trap handling via the following CSRs: `stvec`, `sscratch`, `sepc`, `scause`, and `sstatus`.

Supervisor Control and Status Registers (CSRs)
==============================================

RISC-V provides dedicated CSRs to manage and observe the state of traps (exceptions and interrupts). To implement a robust trap handler in S-mode, you must understand and manipulate the following key registers:

- `sstatus` (Supervisor Status): Keeps track of the processor's current operating state. Key bits include `SIE` (Supervisor Interrupt Enable) for global interrupt control, `SPIE` (Previous `SIE`) to save the interrupt state prior to the trap, and `SPP` (Previous Privilege mode) to record whether the trap originated from U-mode or S-mode.
- `stvec` (Supervisor Trap Vector Base Address): Holds the base address of your kernel's trap handler function. When a trap occurs, the CPU automatically jumps to the address specified here.
- `sepc` (Supervisor Exception Program Counter): When a trap is taken, the hardware automatically saves the memory address of the interrupted instruction (or the instruction that caused the exception) into this register. The `sret` instruction later uses `sepc` to return to the correct execution point.
- `scause` (Supervisor Cause): Contains a code indicating the exact reason for the trap (e.g., timer interrupt, `ecall` from U-mode, or memory fault). The highest bit indicates whether the event is an asynchronous interrupt (1) or a synchronous exception (0).
- `stval` (Supervisor Trap Value): Provides additional, trap-specific information. For instance, if a page fault or memory access error occurs, `stval` will hold the faulting memory address.
- `sscratch` (Supervisor Scratch): A temporary register typically used to safely swap the user stack pointer with the kernel context pointer at the very beginning of a trap handler, before any general-purpose registers are modified.
- `sie` (Supervisor Interrupt Enable): Used for fine-grained control and observation of specific S-mode interrupts. They contain bits for Software (`SSIE`/`SSIP`), Timer (`STIE`/`STIP`), and External (`SEIE`/`SEIP`) interrupts.

Core Timer and SBI
==================

In S-mode, the kernel relies on the Supervisor Binary Interface (SBI) to manage timer interrupts.

Key concepts for S-mode timers:

- `time` CSR: A 64-bit read-only register that reflects the current timer value (accessible via the `rdtime` instruction).
- SBI Timer Extension: To schedule a timer interrupt, the S-mode kernel must call `sbi_set_timer(uint64_t stime_value)`. The SBI implementation will configure the underlying hardware and trigger a timer interrupt to S-mode when the specified time is reached.

Interrupt Controllers - PLIC
============================

OrangePi RV2 uses the **Platform-Level Interrupt Controller (PLIC)** to handle external interrupts from devices such as UART.

Key facts:

- Each device interrupt has an ID (e.g., UART0 is usually ID 10).
- PLIC routes interrupt requests to CPU cores with a priority mechanism.
- Supervisor Context: Because the kernel runs in S-mode, you must configure and access the PLIC using the registers specific to the S-mode context (Claim/Complete registers, Priority Threshold registers, etc.).

See documentation or DTB for actual interrupt IDs and PLIC base addresses.

Critical Sections
=================

As in all interrupt-driven systems, shared data must be protected from concurrent access during interrupt handling.
In RISC-V, this can be done by disabling interrupts via `csrci sstatus, SSTATUS_SIE` and re-enabling via `csrsi`.

###############
Basic Exercises
###############

Basic Exercise 1 - Exception  - 30%
===================================

Mode Switch: S-mode to U-mode
-----------------------------

After booting in S-mode, configure registers to switch to U-mode and run user-level programs.
Setup includes:

1. Writing user program address to `sepc`
2. Setting `sstatus` to enable interrupts and select U-mode
3. Using `sret` to jump to U-mode

.. admonition:: Todo

   Add a command that can load a user program in the initramfs. Then, run it in U-mode by steps mentioned above.

Trap Handling from U-mode
-------------------------

When the user program executes an `ecall`, it traps to the S-mode handler.
You need to:

- Set the trap vector address in `stvec`
- Save user context (`x1-x31`, `sepc`, `sstatus`)
- Print diagnostic info from `scause`, `sepc`, `stval`
- Restore context and return to user using `sret`

.. admonition:: Todo

   Set the vector table and implement the exception handler.

Basic Exercise 2 - Core Timer Interrupt - 10%
=============================================

Enable a supervisor timer interrupt after a specified period using SBI.
Steps:

- Read the current time using `rdtime`
- Add time delta (e.g., equivalent of 1 second based on the CPU's timebase frequency)
- Program the next timer event using an SBI call (e.g., `sbi_set_timer`)
- Enable `SIE` in `sstatus` and `STIE` in `sie`
- In the interrupt handler, acknowledge and reprogram the next timer event via the SBI call

.. Print boot-time seconds in the handler.

.. admonition:: Todo

   Enable the core timer’s interrupt. The interrupt handler should print the seconds after booting and set the next timeout to 2 seconds later.

Basic Exercise 3 - OrangePi RV2 UART0 Interrupt - 30%
=============================================

Enable UART0 interrupt via:

- UART interrupt enable register (check OrangePi RV2 SoC manual or DTS; likely `UART0.IER`)
- Enable UART interrupt ID (e.g., 10) in the PLIC
- Set `sie.SEIE` and enable external interrupts globally

Steps:

1. Setup read/write buffers.
2. Implement ISR for UART RX and TX.
3. In RX, place incoming bytes in buffer.
4. In TX, send data from buffer when ready.
5. In the PLIC, read the Claim register to get the IRQ number, handle it, and write the IRQ number back to the Complete register.

.. Make shell non-blocking by using buffer for input/output.

.. admonition:: Todo

   Implement the asynchronous UART read/write by interrupt handlers.

##################
Advanced Exercises
##################

Advanced Exercise 1 - Timer Multiplexing - 20%
==============================================

Use `add_timer(callback, duration)` API to schedule deferred tasks.
Use a software-managed priority queue (e.g., min-heap or sorted list) to keep track of expiration time and reprogram the next timer event using an SBI call accordingly.

Advanced Exercise 2 - Concurrent I/O Devices Handling 20%
=========================================================

Replace blocking UART logic with event queue mechanism:

- ISR enqueues tasks to be handled outside critical path
- Enable nesting and priority-based dispatch
- Protect shared state using interrupt masking (`sstatus.SIE`)
- Before returning from handler, process pending tasks with interrupts re-enabled

Preemption can be implemented by checking the event queue for higher-priority tasks before final return from trap.

.. note::

   Use nested interrupt handling and task prioritization to support fair and responsive device scheduling.

