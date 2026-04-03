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

RISC-V provides dedicated CSRs to manage and observe the state of traps (exceptions and interrupts). To implement a robust trap handler in S-mode, you are expected to independently consult the RISC-V Privileged Specification to understand the precise roles and hardware behaviors of the following key registers: `sstatus`, `stvec`, `sepc`, `scause`, `stval`, `sscratch`, and `sie`.

.. hint::
   Before diving into the code, ensure you clearly understand what information the hardware automatically writes to these registers when a trap occurs, and which registers are read by the hardware when the `sret` instruction is executed.

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

To run a user program safely, your kernel must set up an environment that allows jumping into U-mode and successfully catching the exception when the user program wants to return or execute a system call.

Mode Switch: S-mode to U-mode
-----------------------------

After booting in S-mode, configure registers to switch to U-mode and run user-level programs.
Setup includes:

1. Writing user program address to `sepc`
2. Setting `sstatus` to enable interrupts and select U-mode
3. Using `sret` to jump to U-mode

.. admonition:: Todo

   Add command ``exec`` that can load the user program in the initramfs. Then, run it in U-mode by steps mentioned above.

Trap Handling from U-mode
-------------------------

When the user program executes an `ecall`, it traps to the S-mode handler.
You need to:

- Before entering U-mode, ensure ``stvec`` is pointing to your trap handler assembly routine.
- Save user context (``x1-x31``, ``sepc``, ``sstatus``)
- Print diagnostic info from ``scause``, ``sepc``, ``stval``
- Restore context and return to user using ``sret``

.. admonition:: Todo

   Set the vector table and implement the exception handler.

The result would be like this:

.. image:: /images/lab4_b1.png



Basic Exercise 2 - Core Timer Interrupt - 10%
=============================================

Timer interrupts are essential for OS scheduling. You will use the Supervisor Binary Interface (SBI) to program the timer.

1. Read the current time using the ``rdtime`` instruction.
2. Calculate the target time by adding the CPU's frequency to the current time (this represents 1 second).
3. Call ``sbi_set_timer(target_time)`` to schedule the interrupt.
4. Set the ``STIE`` bit in the ``sie`` register to enable timer interrupts.
5. Set the ``SIE`` bit in ``sstatus`` to enable global interrupts.
6. When the interrupt triggers (checked via ``scause``), print the number of seconds passed since boot.
7. Reprogram the timer for the next 2 seconds using the SBI call again.

.. Print boot-time seconds in the handler.

.. admonition:: Todo

   Enable the core timer’s interrupt. The interrupt handler should print the seconds after booting and set the next timeout to 2 seconds later.

The result would be like this:

.. image:: /images/lab4_b2.png

Basic Exercise 3 - OrangePi RV2 UART0 Interrupt - 30%
=============================================

Currently, your ``uart_getc`` and ``uart_puts`` are likely blocking (busy-waiting). You must make them asynchronous using PLIC interrupts and ring buffers.

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

Timers can be used to do periodic jobs such as scheduling and journaling and one-shot executing such as sleeping and timeout.
However, the number of hardware timers is limited.
Therefore, the kernel needs a software mechanism to multiplex the timer.

One simple way is using a periodic timer.
The kernel can use the tick period as the time unit and calculate the corresponding timeout tick.
For example, suppose the periodic timer's frequency is 1000HZ and a process sleeps for 1.5 seconds.
The kernel can add a wake-up event at the moment that 1500 ticks after the current tick.

However, when the tick frequency is too low, the timer has a bad resolution.
Then, it can't be used for time-sensitive jobs.
When the tick frequency is too high, it introduces a lot of overhead for redundant timer interrupt handling.

Another way is using a one-shot timer.
When someone needs a timeout event, a timer is inserted into a timer queue.
If the timeout is earlier than the previous programed expired time, the kernel reprograms the hardware timer to the earlier one.
In the timer interrupt handler, it executes the expired timer's callback function.

In this advanced part, you need to implement the timer API that a user can register the callback function when the
timeout using the one-shot timer(the core timer is a one-shot timer).
The API and its use case should look like the below pseudo code. 

.. code:: c

    //An example API
    void add_timer(void (*callback)(void*), void* arg, int sec){
        ...
    }

    //An example use case
    void sleep(int duration){
        add_timer(wakeup, current_process, duration);
    }

To test the API, you need to implement the shell command ``setTimeout SECONDS MESSAGE``.
It prints MESSAGE after SECONDS with the current time and the command executed time.

.. admonition:: Todo

    Implement the ``setTimeout`` command with the timer API.

.. important::
    ``setTimeout`` is non-blocking. Users can set multiple timeouts. 
    The printing order is determined by the command executed time and the user-specified SECONDS.

This is an example:

.. image:: /images/lab4_adv1.png



Advanced Exercise 2 - Concurrent I/O Devices Handling 20%
=========================================================

Currently, interrupts are disabled while executing an Interrupt Service Routine (ISR). If a slow device (like UART printing a long string) triggers an interrupt, crucial events like Timer ticks might be dropped.
Implement an Event Task Queue:

1. The hardware ISR merely acknowledges the hardware, masks that specific interrupt, and enqueues a "Task" into a software queue.
2. Before returning from the trap (``sret``), the kernel re-enables global interrupts (``sstatus.SIE``) and starts processing the Task queue.
3. This allows nested interrupts: a high-priority Timer interrupt can now preempt a low-priority UART task currently being processed.

.. note::

   Use nested interrupt handling and task prioritization to support fair and responsive device scheduling.

