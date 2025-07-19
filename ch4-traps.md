# 第 4 章 陷阱和系统调用（Chapter 4 Traps and system call）

> There are three kinds of event which cause the CPU to set aside ordinary execution of instructions and force a transfer of control to special code that handles the event. One situation is a system call, when a user program executes the `ecall` instruction to ask the kernel to do something for it. Another situation is an exception: an instruction (user or kernel) does something illegal, such as divide by zero or use an invalid virtual address. The third situation is a device *interrupt*, when a device signals that it needs attention, for example when the disk hardware finishes a read or write request.

有三种事件会导致 CPU 暂停正常的指令执行，并强制将执行流转移到处理该事件的特殊指令处。第一种情况是 “系统调用（system call）”，即用户程序执行 `ecall` 指令，发起对内核的请求，请求内核代替它执行某些操作。另一种情况是 “异常（exception）”：发生的场景是当一条（用户态或内核态的）指令执行了非法操作，例如除以零或访问了无效的虚拟地址。第三种情况是设备 *中断（interrupt）*，即设备主动发出信号，提醒处理器注意某个事件发生了，例如当磁盘硬件完成读写请求时。

> This book uses *trap* as a generic term for these situations. Typically whatever code was executing at the time of the trap will later need to resume, and shouldn’t need to be aware that anything special happened. That is, we often want traps to be transparent; this is particularly important for device interrupts, which the interrupted code typically doesn’t expect. The usual sequence is that a trap forces a transfer of control into the kernel; the kernel saves registers and other state so that execution can be resumed; the kernel executes appropriate handler code (e.g., a system call implementation or device driver); the kernel restores the saved state and returns from the trap; and the original code resumes where it left off.

在本书中使用 *陷阱（trap）* 作为这些情况的统称。通常，当 trap 发生时（被打断的）正在执行的指令序列稍后都需要恢复，并且从执行指令序列的角度来看并不需要知道发生了什么特殊情况。也就是说，我们通常希望 trap 是透明的；这对于设备中断尤其重要，因为被中断的代码通常对这种情况并没有预期。通常的顺序是：trap 强制将控制权转移到内核态；内核保存寄存器和其他状态，以便将来恢复执行；内核执行适当的处​​理程序代码（例如，系统调用函数或设备驱动程序）；内核恢复保存过的状态并从 trap 返回；原先的代码从被中断处继续恢复执行。

> Xv6 handles all traps in the kernel; traps are not delivered to user code. Handling traps in the kernel is natural for system calls. It makes sense for interrupts since isolation demands that only the kernel be allowed to use devices, and because the kernel is a convenient mechanism with which to share devices among multiple processes. It also makes sense for exceptions since xv6 responds to all exceptions from user space by killing the offending program.

对于 xv6 来说，所有的 trap 处理都在内核中；也就是说，我们不会在用户态下执行 trap 的处理代码。这对于系统调用来说是很自然的事情。对于中断来说，也很有意义，因为这实现了隔离，确保只有在内核态才可以直接访问设备，所以我们可以把内核看成是一种供多个进程共享设备访问的便捷机制。trap 对于异常来说也具有同样的意义，只不过目前对于所有来自用户空间的异常，xv6 的处理方式只是简单地终止（kill）触发异常的进程。

> Xv6 trap handling proceeds in four stages: hardware actions taken by the RISC-V CPU, some assembly instructions that prepare the way for kernel C code, a C function that decides what to do with the trap, and the system call or device-driver service routine. While commonality among the three trap types suggests that a kernel could handle all traps with a single code path, it turns out to be convenient to have separate code for two distinct cases: traps from user space, and traps from kernel space. Kernel code (assembler or C) that processes a trap is often called a handler; the first handler instructions are usually written in assembler (rather than C) and are sometimes called a *vector*.

xv6 的 trap 处理分为四个阶段：最开始是 RISC-V CPU 内部的硬件操作，然后执行一段为内核 C 代码做准备的汇编指令，接下来是决定如何处理 trap 的 C 函数，最终决定是调用系统调用函数还是设备驱动程序服务例程。虽然这三种 trap 类型的共性表明内核可以用同一条代码路径处理所有 trap，但实践证明，将 trap 处理按两种不同的情况分开，分别编写代码更为方便：具体是区分该 trap 来自用户空间还是来自内核空间。处理 trap 的内核代码（汇编语言或 C 语言）通常被称为 “处理程序（handler）”；处理程序中最开始的一段指令通常用汇编语言（而不是 C 语言）编写，所以有时我们也把处理程序叫做 *向量（vector）*。

## 4.1 RISC-V 的陷阱机制（RISC-V trap machinery）

> Each RISC-V CPU has a set of control registers that the kernel writes to tell the CPU how to handle traps, and that the kernel can read to find out about a trap that has occurred. The RISC-V documents contain the full story [3]. `riscv.h` (kernel/riscv.h:1) contains definitions that xv6 uses. Here’s an outline of the most important registers:

每个 RISC-V CPU 都有一组控制寄存器，内核通过写入这些寄存器来告诉 CPU 如何处理 trap，内核也可以读取这些寄存器来了解一个已发生的 trap。RISC-V 文档包含完整的内容 [3]。`riscv.h` (kernel/riscv.h:1) 中定义了 xv6 所使用的和 RISC-V 相关的宏和 inline 函数。以下是一些最重要的寄存器的概述：

> - `stvec`: The kernel writes the address of its trap handler here; the RISC-V jumps to the address in `stvec` to handle a trap.
> - `sepc`: When a trap occurs, RISC-V saves the program counter here (since the `pc` is then overwritten with the value in `stvec`). The `sret` (return from trap) instruction copies `sepc` to the `pc`. The kernel can write `sepc` to control where `sret` goes.
> - `scause`: RISC-V puts a number here that describes the reason for the trap.
> - `sscratch`: The trap handler code uses `sscratch` to help it avoid overwriting user registers before saving them.
> - `sstatus`: The SIE bit in `sstatus` controls whether device interrupts are enabled. If the kernel clears SIE, the RISC-V will defer device interrupts until the kernel sets SIE. The SPP bit indicates whether a trap came from user mode or supervisor mode, and controls to what mode `sret` returns.

- `stvec`：内核在此处写入其 trap 处理程序的地址；RISC-V 会跳转到 `stvec` 中记录的地址来处理 trap。
- `sepc`：当一个 trap 发生时，RISC-V 会在此处保存 “程序计数器（program counter）” 的值（因为当 trap 发生时，`pc` 会被 `stvec` 中的值覆盖）。指令 `sret`（ret 是 return 的缩写，表示从 trap 返回）将 `sepc` 的值复制到 `pc` 中。内核可以设置 `sepc` 的值来控制执行 `sret` 后从哪里开始恢复取指执行。
- `scause`：RISC-V 在此处放置一个数字来描述 trap 发生的原因。
- `sscratch`：trap 处理程序的代码使用 `sscratch` 来帮助其避免在保存用户寄存器之前覆盖它们。
- `sstatus`：`sstatus` 中的 SIE 比特位控制是否启用设备中断。如果内核清除了 SIE 比特位，RISC-V 将屏蔽设备中断，直到内核重新设置 SIE 比特位。SPP 比特位指示发生 trap 时机器处于用户模式还是管理员模式，从而控制执行 `sret` 时返回到哪个模式。
 
> The above registers relate to traps handled in supervisor mode, and they cannot be read or written in user mode.

上述寄存器与在管理员模式下处理 trap 相关，在用户模式下它们无法被读取或写入。

> Each CPU on a multi-core chip has its own set of these registers, and more than one CPU may be handling a trap at any given time.

对于此类寄存器，在多核芯片上的每个 CPU 核都有一组，也就是说在任意给定时刻，每个 CPU 可以独立地处理自己的 trap。

> When it needs to force a trap, the RISC-V hardware does the following for all trap types:
>
> 1. If the trap is a device interrupt, and the `sstatus` SIE bit is clear, don’t do any of the following.
> 2. Disable interrupts by clearing the SIE bit in `sstatus`.
> 3. Copy the `pc` to `sepc`.
> 4. Save the current mode (user or supervisor) in the SPP bit in `sstatus`.
> 5. Set `scause` to reflect the trap’s cause.
> 6. Set the mode to supervisor.
> 7. Copy `stvec` to the `pc`.
> 8. Start executing at the new `pc`.

硬件针对各种 trap 类型的处理都是相同的，当一个 trap 发生时，RISC-V 处理器会按顺序执行以下动作：

1. 如果 trap 是设备中断，并且 `sstatus` 的 SIE 位已清零，则不会执行以下任何操作（译者注：即关中断情况下设备中断即使发生也不会触发 trap）。
2. 清除 `sstatus` 中的 SIE 比特位来禁用中断。
3. 将 `pc` 的值复制到寄存器 `sepc` 中。
4. 将当前模式（用户模式或管理员模式）保存在 `sstatus` 的 SPP 比特位中。
5. 设置寄存器 `scause` 以反映 trap 发生的原因。
6. 将模式设置为管理员模式。
7. 用寄存器 `stvec` 中的值覆盖 `pc`。
8. 根据 `pc` 中新的指令地址取指执行。

> Note that the CPU doesn’t switch to the kernel page table, doesn’t switch to a stack in the kernel, and doesn’t save any registers other than the `pc`. Kernel software must perform these tasks. One reason that the CPU does minimal work during a trap is to provide flexibility to software; for example, some operating systems omit a page table switch in some situations to increase trap performance.

请注意，（当 trap 发生时）CPU 不会自动切换到内核页表，也不会自动切换为使用内核栈，同时也不会保存除 `pc` 之外的任何寄存器。内核程序必须自己执行这些操作。CPU 在 trap 发生期间只执行少量工作的原因之一是为软件实现提供灵活性；例如，某些操作系统在某些情况下可能会省略页表切换以提高执行 trap 的性能。

> It’s worth thinking about whether any of the steps listed above could be omitted, perhaps in search of faster traps. Though there are situations in which a simpler sequence can work, many of the steps would be dangerous to omit in general. For example, suppose that the CPU didn’t switch program counters. Then a trap from user space could switch to supervisor mode while still running user instructions. Those user instructions could break user/kernel isolation, for example by modifying the `satp` register to point to a page table that allowed accessing all of physical memory. It is thus important that the CPU switch to a kernel-specified instruction address, namely `stvec`.

这里我们可以深入思考一下，如果为了寻求更快的 trap 执行速度，（CPU 硬件）是否可以省略上面列出来的步骤中的任何一步。虽然在某些情况下可以使用更简单的执行步骤，但通常情况下，省略某些步骤是危险的。例如，假设 CPU 不切换 program counter。那么当在用户模式下发生 trap 时虽然模式已经切换到管理员模式，可是 CPU 仍然在执行用户指令。这些用户指令可能会破坏我们要保持用户和内核之间隔离的要求，譬如用户指令可以通过修改 `satp` 寄存器使其指向一个允许访问所有物理内存的页表。因此，由 CPU 来确保自动切换到内核指定的指令地址（即寄存器 `stvec` 中设定的值）是非常重要的。

## 4.2 用户空间触发的陷阱（Traps from user space）

> Xv6 handles traps differently depending on whether the trap occurs while executing in the kernel or in user code. Here is the story for traps from user code; Section 4.5 describes traps from kernel code.

在处理 trap 的方式上，xv6 区分 trap 发生在内核态还是在用户态。本小节介绍了当 trap 发生在用户态时的处理；第 4.5 节描述了 trap 发生在内核态时的处理。

> A trap may occur while executing in user space if the user program makes a system call (`ecall` instruction), or does something illegal, or if a device interrupts. The high-level path of a trap from user space is `uservec` (kernel/trampoline.S:22), then `usertrap` (kernel/trap.c:37); and when returning, `usertrapret` (kernel/trap.c:90) and then `userret` (kernel/trampoline.S:101).

如果用户程序在用户空间执行时发生系统调用（执行 `ecall` 指令），或执行了非法操作，或者发生设备中断，都会触发 trap。用户态 trap 的执行路径为：`uservec`（kernel/trampoline.S:22），然后是 `usertrap`（kernel/trap.c:37）；返回时，依次为 `usertrapret`（kernel/trap.c:90）和 `userret`（kernel/trampoline.S:101）。

> A major constraint on the design of xv6’s trap handling is the fact that the RISC-V hardware does not switch page tables when it forces a trap. This means that the trap handler address in `stvec` must have a valid mapping in the user page table, since that’s the page table in force when the trap handling code starts executing. Furthermore, xv6’s trap handling code needs to switch to the kernel page table; in order to be able to continue executing after that switch, the kernel page table must also have a mapping for the handler pointed to by `stvec`.

xv6 中 trap 处理设计的一个主要限制在于，RISC-V 硬件在触发 trap 时不会自动切换页表。也就是说当 trap 处理程序的代码开始执行时当前激活的仍然是用户态的页表，所以这意味着 `stvec` 中所指向的 trap 处理程序的地址必须在用户页表中具备有效的映射。此外，xv6 的 trap 处理代码需要负责将页表切换为内核页表；为了能够在切换后能够继续执行，内核页表中对 `stvec` 所指向的 trap 处理程序也要具备有效的映射。

> Xv6 satisfies these requirements using a trampoline page. The trampoline page contains `uservec`, the xv6 trap handling code that `stvec` points to. The trampoline page is mapped in every process’s page table at address `TRAMPOLINE`, which is at the top of the virtual address space so that it will be above memory that programs use for themselves. The trampoline page is also mapped at address `TRAMPOLINE` in the kernel page table. See Figure 2.3 and Figure 3.3. Because the trampoline page is mapped in the user page table, traps can start executing there in supervisor mode. Because the trampoline page is mapped at the same address in the kernel address space, the trap handler can continue to execute after it switches to the kernel page table.

xv6 使用一个 “蹦床页（trampoline page）” 来满足以上要求。trampoline page 中存放了 `uservec` 函数的指令，即 `stvec` 指向的 xv6 的 trap 处理程序。每个进程的页表（即用户页表）都会将这个 trampoline page 映射为进程地址空间中的虚拟地址 `TRAMPOLINE`，该地址位于虚拟地址空间的顶部，因此它位于程序会使用的内存地址空间的上方。trampoline page 也被映射到内核页表中的虚拟地址 `TRAMPOLINE`。参见图 2.3 和图 3.3。由于用户页表中有效映射了 trampoline page，因此当处理器进入管理员模式开始处理 trap 时可以从 `TRAMPOLINE` 那里开始执行。同时由于 trampoline page 也映射到内核地址空间中的相同地址，因此 trap 处理程序在切换到内核页表后依然可以继续执行。

> The code for the `uservec` trap handler is in `trampoline.S` (kernel/trampoline.S:22). When `uservec` starts, all 32 registers contain values owned by the interrupted user code. These 32 values need to be saved somewhere in memory, so that later on the kernel can restore them before returning to user space. Storing to memory requires use of a register to hold the address, but at this point there are no general-purpose registers available! Luckily RISC-V provides a helping hand in the form of the `sscratch` register. The `csrw` instruction at the start of `uservec` saves `a0` in `sscratch`. Now `uservec` has one register (`a0`) to play with.

`uservec` 函数（译者注：即用户态 trap 处理程序）的代码位于 `trampoline.S` (kernel/trampoline.S:22) 中。当 `uservec` 刚开始执行时，所有 32 个寄存器的值对应着被中断的用户程序的上下文。这 32 个值需要保存在内存中的某个位置，以便内核稍后在返回用户空间之前恢复它们。执行保存的操作需要使用寄存器来存放地址，但目前没有可用的通用寄存器！（译者注：所有的通用寄存器的内容此时都需要保存，此时使用它们中的任何一个，都会修改它们的内容，这显然是不可以的）幸运的是，RISC-V 有一个 `sscratch` 寄存器可以提供帮助。`uservec` 函数中一开始利用 `csrw` 指令将 `a0` 保存在 `sscratch` 中。这样此时 `uservec` 至少有一个寄存器 (`a0`) 可以使用。

> `uservec`’s next task is to save the 32 user registers. The kernel allocates, for each process, a page of memory for a `trapframe` structure that (among other things) has space to save the 32 user registers (kernel/proc.h:43). Because `satp` still refers to the user page table, `uservec` needs the trapframe to be mapped in the user address space. Xv6 maps each process’s trapframe at virtual address `TRAPFRAME` in that process’s user page table; `TRAPFRAME` is just below `TRAMPOLINE`. The process’s `p->trapframe` also points to the trapframe, though at its physical address so the kernel can use it through the kernel page table.

`uservec` 的下一个任务是保存 32 个用户寄存器（译者注：严格说是 31 个，x0/zero 不需要保存和恢复）。内核为每个进程分配一个内存物理页（译者注：下文直接用 trapframe 指代这个物理页），用于存放 `trapframe` 结构体（以及其他一些内容），该结构体的部分空间用于保存 32 个用户寄存器 (kernel/proc.h:43)。由于 `satp` 此时仍然指向用户页表，`uservec` 需要将 trapframe 映射到用户地址空间。xv6 将每个进程的 trapframe 映射到该进程用户页表中的虚拟地址 `TRAPFRAME`；`TRAPFRAME` 就位于 `TRAMPOLINE` 的下方。进程的 `p->trapframe` 也指向 trapframe，虽然 `p->trapframe` 保存的值是 trapframe 的物理地址，但内核同样可以通过内核页表使用它（译者注：即内核代码中可以通过 `p->trapframe` 访问 trapframe，因为内核采用的是 direct mapping，具体参考 3.2 节）。

> Thus `uservec` loads address `TRAPFRAME` into `a0` and saves all the user registers there, including the user’s `a0`, read back from `sscratch`.

因此，`uservec` 将地址 `TRAPFRAME` 加载到 `a0` 中（译者注：接上文，此时 `a0` 中原先的值已经保存在 `sscratch` 中），然后（根据这个地址）将所有用户寄存器保存在 trapframe 中，包括备份在 `sscratch` 中的用户的 `a0`。

> The `trapframe` contains the address of the current process’s kernel stack, the current CPU’s hartid, the address of the `usertrap` function, and the address of the kernel page table. `uservec` retrieves these values, switches `satp` to the kernel page table, and jumps to `usertrap`.

`trapframe` 中还保存了当前进程的内核栈地址、当前 CPU 的 hartid、`usertrap` 函数的地址以及内核页表地址，`uservec` 将读取和使用这些值，然后将 `satp` 切换到内核页表，并跳转到 `usertrap`。

> The job of `usertrap` is to determine the cause of the trap, process it, and return (kernel/trap.c:37). It first changes `stvec` so that a trap while in the kernel will be handled by `kernelvec` rather than `uservec`. It saves the `sepc` register (the saved user program counter), because `usertrap` might call `yield` to switch to another process’s kernel thread, and that process might return to user space, in the process of which it will modify `sepc`. If the trap is a system call, `usertrap` calls `syscall` to handle it; if a device interrupt, `devintr`; otherwise it’s an exception, and the kernel kills the faulting process. The system call path adds four to the saved user program counter because RISC-V, in the case of a system call, leaves the program pointer pointing to the `ecall` instruction but user code needs to resume executing at the subsequent instruction. On the way out, `usertrap` checks if the process has been killed or should yield the CPU (if this trap is a timer interrupt).

`usertrap` 的工作包括确定 trap 发生的原因，进行相应处理然后返回 (kernel/trap.c:37)。它首先修改 `stvec` 的值，以便在内核态中一旦发生 trap 处理器会跳转到 `kernelvec` 而不是依然交给 `uservec` 处理。然后，它将 `sepc` 寄存器的值保存到 trapframe 中（`sepc` 的值是 trap 发生时的用户态下的 program counter），这是因为 `usertrap` 可能会调用 `yield` 切换到另一个进程的内核线程，而该进程可能会返回用户空间，在此过程中 `sepc` 的值会被修改。接下来 `usertrap` 会判断，如果 trap 是系统调用，则调用 `syscall` 来处理；如果是设备中断，则调用 `devintr`；其他情况是异常，内核会终止出错的进程。针对 RISC-V 的系统调用，我们并不会在这里直接修改处理器的 `sepc` 寄存器（即仍然让它指向调用 `ecall` 指令的地址），而是将已保存的用户态 program counter（译者注：即 `p->trapframe->epc`）的值加上 4，这么做的目的是确保系统调用执行完毕后，在返回用户态时，我们能从 `ecall` 指令的下一条指令处开始恢复执行（译者注：考虑到本节前面也说到 `usertrap` 中可能会发生进程切换返回用户态并修改 `sepc`，所以 `p->trapframe->epc` 的值只有在本进程真正确定需要返回用户态之前才会被恢复到 `sepc` 中，具体见下文针对 `usertrapret` 函数的解释）。`usertrap` 在退出时会检查进程是否已被终止或是否应该放弃 CPU（如果触发此 trap 的是定时器中断）。

> The first step in returning to user space is the call to `usertrapret` (kernel/trap.c:90). This function sets up the RISC-V control registers to prepare for a future trap from user space: setting `stvec` to `uservec` and preparing the trapframe fields that `uservec` relies on. `usertrapret` sets `sepc` to the previously saved user program counter. At the end, `usertrapret` calls `userret` on the trampoline page that is mapped in both user and kernel page tables; the reason is that assembly code in `userret` will switch page tables.

当我们开始准备返回用户空间的时候，第一步是调用 `usertrapret` (kernel/trap.c:90)。此函数设置 RISC-V 控制寄存器，为下一次用户空间的 trap 做好准备，这些准备工作包括：将 `stvec` 设置为 `uservec`，并为执行 `uservec` 准备好所需要的 trapframe 。`usertrapret` 将 `sepc` 设置为先前保存的用户 program counter。最后，`usertrapret` 调用 `userret`，这个函数也存放在 trampoline 上，并被用户页表和内核页表共同映射，之所以这么设计同样是因为 `userret` 中的汇编代码会切换页表。

> `usertrapret`’s call to `userret` passes a pointer to the process’s user page table in `a0` (kernel/trampoline.S:101). `userret` switches `satp` to the process’s user page table. Recall that the user page table maps both the trampoline page and `TRAPFRAME`, but nothing else from the kernel. The trampoline page mapping at the same virtual address in user and kernel page tables allows `userret` to keep executing after changing `satp`. From this point on, the only data `userret` can use is the register contents and the content of the trapframe. `userret` loads the `TRAPFRAME` address into `a0`, restores saved user registers from the trapframe via `a0`, restores the saved user `a0`, and executes `sret` to return to user space.

`usertrapret` 在调用 `userret` 时通过 `a0` 传递了一个指向进程用户页表的指针 (kernel/trampoline.S:101)。`userret` 将 `satp` 切换到进程的用户页表。回想一下，用户页表中只映射了 trampoline（映射到 `TRAMPOLINE`）和 trapframe（映射到 `TRAPFRAME`），除此之外，并没有映射内核的其他内容。由于 trampoline 在用户和内核页表中的虚拟地址相同，因此 `userret` 在更改 `satp` 后仍能继续执行。从此时起，`userret` 唯一可以使用的数据是寄存器内容和 trapframe 的内容。`userret` 先将 `TRAPFRAME` 地址加载到 `a0`，然后通过 `a0` 从 trapframe 恢复保存的（除 `a0` 之外其他的）用户寄存器，最后恢复保存的 `a0`，`userret` 的最后执行 `sret` 返回用户空间。

## 4.3 代码讲解：执行系统调用（Code: Calling system calls）

> Chapter 2 ended with `initcode.S` invoking the `exec` system call (user/initcode.S:11). Let’s look at how the user call makes its way to the `exec` system call’s implementation in the kernel.

第 2 章最后（译者注：第 2.6 节）提到了 `initcode.S` 调用 `exec` 系统调用 (user/initcode.S:11)，但没有继续深入。这里让我们看看系统调用 `exec` 是如何进入内核的。

> `initcode.S` places the arguments for `exec` in registers `a0` and `a1`, and puts the system call number in `a7`. System call numbers match the entries in the `syscalls` array, a table of function pointers (kernel/syscall.c:107). The `ecall` instruction traps into the kernel and causes `uservec`, `usertrap`, and then `syscall` to execute, as we saw above.

`initcode.S` 将 `exec` 的参数放在寄存器 `a0` 和 `a1` 中，并将系统调用号放在 `a7` 中。每一个系统调用号在 `syscalls` 数组（一个存放函数指针的列表（kernel/syscall.c:107）中都有对应的匹配项。`ecall` 指令触发 trap，切换到内核态，并顺序执行 `uservec`、`usertrap` 和 `syscall`，正如我们在前面小节所介绍的。

> `syscall` (kernel/syscall.c:132) retrieves the system call number from the saved `a7` in the trapframe and uses it to index into `syscalls`. For the first system call, `a7` contains `SYS_exec` (kernel/syscall.h:8), resulting in a call to the system call implementation function `sys_exec`.

`syscall` (kernel/syscall.c:132) 从 trapframe 中保存的 `a7` 中取到系统调用号，并用它作为索引在 `syscalls` 中找到对应的项。对于（`initcode.S` 中 xv6 发起的）第一个系统调用，`a7` 的值是 `SYS_exec` (kernel/syscall.h:8)，所以最终会调用其对应的系统调用实现函数 `sys_exec`。

> When `sys_exec` returns, `syscall` records its return value in `p->trapframe->a0`. This will cause the original user-space call to `exec()` to return that value, since the C calling convention on RISC-V places return values in `a0`. System calls conventionally return negative numbers to indicate errors, and zero or positive numbers for success. If the system call number is invalid, `syscall` prints an error and returns −1.

当 `sys_exec` 返回时，`syscall` 会将其返回值记录在 `p->trapframe->a0` 中。因为根据 RISC-V 上的 C 函数调用约定，规定将返回值放在 `a0` 中，所以这将导致原先用户空间对 `exec()` 的调用返回该值。系统调用通常返回负数表示错误，返回零或正数表示成功。如​​果系统调用号无效，`syscall` 会打印错误并返回 -1。

## 4.4 代码讲解：系统调用参数（Code: System call arguments）

> System call implementations in the kernel need to find the arguments passed by user code. Because user code calls system call wrapper functions, the arguments are initially where the RISC-V C calling convention places them: in registers. The kernel trap code saves user registers to the current process’s trap frame, where kernel code can find them. The kernel functions `argint`, `argaddr`, and `argfd` retrieve the *n*’th system call argument from the trap frame as an integer, pointer, or a file descriptor. They all call `argraw` to retrieve the appropriate saved user register (kernel/syscall.c:34).

内核中的系统调用实现需要获取用户代码传进来的参数。由于用户代码直接调用的是系统调用的封装函数，因此根据 RISC-V 的 C 函数调用约定，参数最初存放在寄存器中。内核的 trap 处理代码将用户态下的寄存器上下文信息保存到当前进程的 trapframe 中，以便内核代码可以找到它们。内核函数 `argint`、`argaddr` 和 `argfd` 从 trapframe 中找到第 *n* 个系统调用参数，再分别转化成整数、指针或文件描述符的形式返回。这些函数内部都调用 `argraw` 来找到系统调用发生时保存参数的相应寄存器 (kernel/syscall.c:34)。

> Some system calls pass pointers as arguments, and the kernel must use those pointers to read or write user memory. The `exec` system call, for example, passes the kernel an array of pointers referring to string arguments in user space. These pointers pose two challenges. First, the user program may be buggy or malicious, and may pass the kernel an invalid pointer or a pointer intended to trick the kernel into accessing kernel memory instead of user memory. Second, the xv6 kernel page table mappings are not the same as the user page table mappings, so the kernel cannot use ordinary instructions to load or store from user-supplied addresses.

一些系统调用通过参数传递指针，内核需要使用这些指针来读取或写入用户内存。例如，`exec` 系统调用向内核传递一个指向用户空间中字符串参数的指针数组。这些指针带来了两个挑战。首先，用户程序可能存在缺陷或者纯粹怀有恶意，它们可能会向内核传递无效指针，或传递一个指针试图诱骗内核进而访问内核内存而不是用户内存。其次，由于 xv6 内核页表与用户页表的映射方式不同，因此内核无法直接使用普通指令从用户提供的地址读取或存储数据。

> The kernel implements functions that safely transfer data to and from user-supplied addresses. `fetchstr` is an example (kernel/syscall.c:25). File system calls such as `exec` use `fetchstr` to retrieve string file-name arguments from user space. `fetchstr` calls `copyinstr` to do the hard work.

内核实现了安全地根据用户提供的地址传输数据的函数。例如 `fetchstr` (kernel/syscall.c:25)。诸如 `exec` 之类的系统调用通过使用 `fetchstr` 从用户空间获取字符串形式的文件名参数。`fetchstr` 中通过调用 `copyinstr` 来完成这项复杂的工作。

> `copyinstr` (kernel/vm.c:415) copies up to `max` bytes to `dst` from virtual address `srcva` in the user page table `pagetable`. Since `pagetable` is *not* the current page table, `copyinstr` uses `walkaddr` (which calls `walk`) to look up `srcva` in `pagetable`, yielding physical address `pa0`. The kernel’s page table maps all of physical RAM at virtual addresses that are equal to the RAM’s physical address. This allows `copyinstr` to directly copy string bytes from `pa0` to `dst`. `walkaddr` (kernel/vm.c:109) checks that the user-supplied virtual address is part of the process’s user address space, so programs cannot trick the kernel into reading other memory. A similar function, `copyout`, copies data from the kernel to a user-supplied address.

基于参数中给定的用户页表 `pagetable`，函数 `copyinstr` (kernel/vm.c:415) 将最多 `max` 个字节从用户态虚拟地址 `srcva` 复制到 `dst`。由于 `pagetable` *不是* 当前页表，`copyinstr` 使用 `walkaddr`（内部调用 `walk`）在 `pagetable` 中查找 `srcva`，从而得到物理地址 `pa0`。因为内核的页表将所有物理内存的地址映射到与其值相等的虚拟地址处。所以 `copyinstr` 可以直接将字符串字节从 `pa0` 复制到 `dst`。`walkaddr` (kernel/vm.c:109) 会检查用户提供的虚拟地址是否属于进程的用户地址空间，因此程序无法欺骗内核读取其他内存。类似的函数 `copyout` 将数据从内核空间复制到用户提供的地址。

## 4.5 内核空间触发的陷阱（Traps from kernel space）

> Xv6 handles traps from kernel code in a different way than traps from user code. When entering the kernel, `usertrap` points `stvec` to the assembly code at `kernelvec` (kernel/kernelvec.S:12). Since `kernelvec` only executes if xv6 was already in the kernel, `kernelvec` can rely on `satp` being set to the kernel page table, and on the stack pointer referring to a valid kernel stack. `kernelvec` pushes all 32 registers onto the stack, from which it will later restore them so that the interrupted kernel code can resume without disturbance.

xv6 对于处理内核态 trap 的方式与处理用户态 trap 不同。进入内核时，`usertrap` 会将 `stvec` 指向汇编函数 `kernelvec` （kernel/kernelvec.S:12）。由于 `kernelvec` 仅在 xv6 已进入内核后才会执行，因此当 `kernelvec` 被执行时我们知道 `satp` 已经设置为指向内核页表，以及栈指针也已经指向有效的内核栈（译者注：`uservec` 中跳转 `usertrap` 之前已经完成内核页表以及内核栈指针的设置）。`kernelvec` 会将所有 32 个寄存器压入栈，之后再从栈中恢复它们，以便被中断的内核代码能够不受干扰地恢复执行（译者注：原文这里所说的会将所有 32 个寄存器压栈的描述并不准确，实际只会保存 caller-saved 寄存器，具体见代码）。

> `kernelvec` saves the registers on the stack of the interrupted kernel thread, which makes sense because the register values belong to that thread. This is particularly important if the trap causes a switch to a different thread – in that case the trap will actually return from the stack of the new thread, leaving the interrupted thread’s saved registers safely on its stack.

`kernelvec` 将寄存器保存在被中断的内核线程的栈中，这么做是合理的，因为每个线程需要独立地保存一份自己被中断时的寄存器上下文信息。尤其重要的是我们需要意识到 trap 发生时可能导致线程切换，一旦发生线程切换，trap 返回时我们就可以从新线程的栈中恢复寄存器上下文，而同时被中断的线程的寄存器信息则被保存在自己的栈中，这也是安全的。

> `kernelvec` jumps to `kerneltrap` (kernel/trap.c:135) after saving registers. `kerneltrap` is prepared for two types of traps: device interrupts and exceptions. It calls `devintr` (kernel/trap.c:185) to check for and handle the former. If the trap isn’t a device interrupt, it must be an exception, and that is always a fatal error if it occurs in the xv6 kernel; the kernel calls `panic` and stops executing.

保存好寄存器后，`kernelvec` 跳转到 `kerneltrap` (kernel/trap.c:135)。`kerneltrap` 可处理两种类型的 trap：设备中断和异常。它调用 `devintr` (kernel/trap.c:185) 来检查并处理前者。如果 trap 不是设备中断，则一定是异常，而对于 xv6 来说，内核态中发生的异常都被认为是一种致命的错误；内核会调用 `panic` 停止内核的执行。

> If `kerneltrap` was called due to a timer interrupt, and a process’s kernel thread is running (as opposed to a scheduler thread), `kerneltrap` calls `yield` to give other threads a chance to run. At some point one of those threads will yield, and let our thread and its `kerneltrap` resume again. Chapter 7 explains what happens in `yield`.

对于定时器中断触发执行 `kerneltrap` 的情况，如果被打断的是普通的进程的内核线程（而不是调度器线程），`kerneltrap` 会调用 `yield` 让其他线程有机会运行。此后的某个时刻，当其他运行的线程中的某一个让出处理器时，我们这里先前让出处理器的线程会再次恢复运行并完成 `kerneltrap`。第 7 章将解释 `yield` 中发生的细节。

> When `kerneltrap`’s work is done, it needs to return to whatever code was interrupted by the trap. Because a `yield` may have disturbed `sepc` and the previous mode in `sstatus`, `kerneltrap` saves them when it starts. It now restores those control registers and returns to `kernelvec` (kernel/kernelvec.S:38). `kernelvec` pops the saved registers from the stack and executes `sret`, which copies `sepc` to `pc` and resumes the interrupted kernel code.

当 `kerneltrap` 的工作完成后，它需要返回到被 trap 中断的代码。考虑到这期间可能发生过 `yield` 导致 `sepc` 和 `sstatus` 中原先存放的内容被修改，我们在刚进入 `kerneltrap` 时会保存它们的值。在 `kerneltrap` 的最后会恢复这些控制寄存器并返回到 `kernelvec` (kernel/kernelvec.S:38)。`kernelvec` 会从栈中弹出保存的寄存器上下文并执行 `sret`，这条指令会将 `sepc` 复制到 `pc`，并恢复执行被中断的内核代码。

> It’s worth thinking through how the trap return happens if `kerneltrap` called `yield` due to a timer interrupt.

对于定时器中断所导致的 `kerneltrap` 调用 `yield` 的情况，值得深入思考一下 trap 的返回是如何发生的。

> Xv6 sets a CPU’s `stvec` to `kernelvec` when that CPU enters the kernel from user space; you can see this in `usertrap` (kernel/trap.c:29). There’s a window of time when the kernel has started executing but `stvec` is still set to `uservec`, and it’s crucial that no device interrupt occur during that window. Luckily the RISC-V always disables interrupts when it starts to take a trap, and `usertrap` doesn’t enable them again until after it sets `stvec`.

当 CPU 从用户态进入内核时，xv6 会将 CPU 的 `stvec` 设置为 `kernelvec`；你可以在 `usertrap` (kernel/trap.c:29 译者注：原文这里貌似写错了，应该是 37 行) 中看到这一点。在内核开始执行但 `stvec` 仍设置为 `uservec` 的时间段内，至关重要的一点是在此期间不能发生任何设备中断。所幸的是，RISC-V 规定当 trap 发生时硬件会自动禁用中断，而 `usertrap` 直到设置 `stvec` 后才会再次启用中断。

## 4.6 缺页异常（Page-fault exceptions）

> Xv6’s response to exceptions is quite boring: if an exception happens in user space, the kernel kills the faulting process. If an exception happens in the kernel, the kernel panics. Real operating systems often respond in much more interesting ways.

xv6 对异常的处理相当简单粗暴：如果异常发生在用户空间，内核会终止触发异常的进程。如果异常发生在内核，内核则直接 “崩溃 (panic)”。真正的操作系统通常会以更复杂的方式处理异常。

> As an example, many kernels use page faults to implement *copy-on-write (COW) fork*. To explain copy-on-write fork, consider xv6’s `fork`, described in Chapter 3. `fork` causes the child’s initial memory content to be the same as the parent’s at the time of the fork. Xv6 implements fork with `uvmcopy` (kernel/vm.c:313), which allocates physical memory for the child and copies the parent’s memory into it. It would be more efficient if the child and parent could share the parent’s physical memory. A straightforward implementation of this would not work, however, since it would cause the parent and child to disrupt each other’s execution with their writes to the shared stack and heap.

例如，许多内核使用 “缺页异常（page fault）” 来实现 *写时复制 (copy-on-write，缩写 COW) fork*。为了解释 COW fork，我们来看一下 xv6 的 `fork` 函数实现，它在第 3 章中描述过。`fork` 执行的结果会导致子进程的初始内存内容与父进程在 `fork` 时的内容相同。xv6 使用 `uvmcopy` (kernel/vm.c:313) 实现 `fork`，它会为子进程分配物理内存，并将父进程内存中的内容复制到其中。如果子进程和父进程能够共享父进程的物理内存，效率肯定会更高。然而，这种简单的共享方式并不可行，因为这么做会使得父进程和子进程也共用了同一份栈和堆，任何一方改写了栈和堆的内容都会干扰另一方的运行。

> Parent and child can safely share physical memory by appropriate use of page-table permissions and page faults. The CPU raises a *page-fault exception* when a virtual address is used that has no mapping in the page table, or has a mapping whose `PTE_V` flag is clear, or a mapping whose permission bits (`PTE_R`, `PTE_W`, `PTE_X`, `PTE_U`) forbid the operation being attempted. RISC-V distinguishes three kinds of page fault: load page faults (caused by load instructions), store page faults (caused by store instructions), and instruction page faults (caused by fetches of instructions to be executed). The `scause` register indicates the type of the page fault and the `stval` register contains the address that couldn’t be translated.

通过适当设置页表权限和利用缺页异常，父进程和子进程可以安全地共享物理内存。当使用的虚拟地址在页表中没有映射，或者映射的 `PTE_V`标志位设置为清除状态，或者指令尝试执行虚拟地址禁止的操作（体现在映射的权限位 `PTE_R`、`PTE_W`、`PTE_X`、`PTE_U` 上），都会触发 CPU 产生 *缺页异常（page-fault exception）*。RISC-V 区分三种类型的缺页异常：“加载页异常（load page fault）”（由加载指令触发）、“存储页异常（store page fault）”（由存储指令触发）和 “指令页异常（instruction page fault）”（由取指操作触发）。`scause` 寄存器指示缺页异常的类型，`stval` 寄存器会指出异常发生时对应的访问内存地址。

> The basic plan in COW fork is for the parent and child to initially share all physical pages, but for each to map them read-only (with the `PTE_W` flag clear). Parent and child can read from the shared physical memory. If either writes a given page, the RISC-V CPU raises a page-fault exception. The kernel’s trap handler responds by allocating a new page of physical memory and copying into it the physical page that the faulted address maps to. The kernel changes the relevant PTE in the faulting process’s page table to point to the copy and to allow writes as well as reads, and then resumes the faulting process at the instruction that caused the fault. Because the PTE now allows writes, the re-executed instruction will execute without a fault. Copy-on-write requires book-keeping to help decide when physical pages can be freed, since each page can be referenced by a varying number of page tables depending on the history of forks, page faults, execs, and exits. This book-keeping allows an important optimization: if a process incurs a store page fault and the physical page is only referred to from that process’s page table, no copy is needed.

COW fork 的基本方案是父进程和子进程最初共享所有物理页面，但各自只以只读方式映射这些页面（即清除 `PTE_W` 标志）。父进程和子进程可以从共享的物理内存中读取数据。但如果其中任何一方对给定的页执行了写操作，RISC-V CPU 就会触发缺页异常。内核的 trap 处理程序会做出响应，分配一个新的物理内存页，并将出错地址所映射的物理页内容复制到该页中。内核会将出错进程页表中的相关 PTE 更改为指向新的副本，并允许对其写入和读取，然后在导致错误的指令处恢复出错进程。由于 PTE 现在允许写入，因此重新执行的指令将正确执行。因为 fork、page fault 异常处理、exec 和 exit 等操作的发生都在不停地改变一个物理页被不同页表所引用的状态，所以引入 COW 处理后需要记录引用信息来帮助决定何时可以释放这些物理页。利用这些记录的引用信息可以实现重要的优化：譬如某个进程在写入物理内存时发生页错误，但由于该物理页仅被当前此进程的页表所引用，则不需要触发 COW。

> Copy-on-write makes `fork` faster, since `fork` need not copy memory. Some of the memory will have to be copied later, when written, but it’s often the case that most of the memory never has to be copied. A common example is `fork` followed by `exec`: a few pages may be written after the `fork`, but then the child’s `exec` releases the bulk of the memory inherited from the parent. Copy-on-write `fork` eliminates the need to ever copy this memory. Furthermore, COW fork is transparent: no modifications to applications are necessary for them to benefit.

COW 使 `fork` 运行速度更快，因为 `fork` 无需复制内存。部分内存会稍后在发生写入操作时才会被复制，但通常情况下，大部分内存无需复制。一个常见的例子是 `fork` 后跟 `exec`：在 `fork` 之后可能会对少量页执行写操作，但随后子进程因为执行 `exec` 会释放从父进程继承的大部分内存。支持 COW 的 `fork` 则无需复制这些内存。此外，COW fork 是透明的：无需对应用程序进行任何修改即可从中受益。

> The combination of page tables and page faults opens up a wide range of interesting possibilities in addition to COW fork. Another widely-used feature is called *lazy allocation*, which has two parts. First, when an application asks for more memory by calling `sbrk`, the kernel notes the increase in size, but does not allocate physical memory and does not create PTEs for the new range of virtual addresses. Second, on a page fault on one of those new addresses, the kernel allocates a page of physical memory and maps it into the page table. Like COW fork, the kernel can implement lazy allocation transparently to applications.

通过结合页表和缺页异常处理，除了 COW fork 之外，可以尝试更多有趣的可能性。另一个广泛使用的功能称为 *延迟分配（lazy allocation）*，它包含两个部分。首先，当应用程序通过调用 `sbrk` 请求更多内存时，内核会注意到大小的增加，但不会立即分配物理内存，也不会为新的虚拟地址范围创建 PTE。其次，当其中一个新地址发生缺页异常时，内核会分配一页物理内存并将其映射到页表中。与 COW fork 类似，内核可以对应用程序透明地实现 lazy allocation。

> Since applications often ask for more memory than they need, lazy allocation is a win: the kernel doesn’t have to do any work at all for pages that the application never uses. Furthermore, if the application is asking to grow the address space by a lot, then `sbrk` without lazy allocation is expensive: if an application asks for a gigabyte of memory, the kernel has to allocate and zero 262,144 4096-byte pages. Lazy allocation allows this cost to be spread over time. On the other hand, lazy allocation incurs the extra overhead of page faults, which involve a user/kernel transition. Operating systems can reduce this cost by allocating a batch of consecutive pages per page fault instead of one page and by specializing the kernel entry/exit code for such page-faults.

由于应用程序经常请求超出其实际需要的内存，因此 lazy allocation 这个方案有其独到的优势：内核无需为应用程序还未使用的内存页做任何工作。此外，如果应用程序要求大幅增加地址空间，那么不使用 lazy allocation 的 `sbrk` 开销会非常高：如果应用程序请求 1 GB 内存，内核必须分配并清零 262144 个 4096 字节的内存页。lazy allocation 可以将这笔开销延迟并分摊到未来实现。但另一方面，lazy allocation 会引入额外的针对缺页异常处理的开销，这涉及用户态和内核态的转换。操作系统可以通过在缺页异常处理时分配一批连续的页（而不是一页），并为此类缺页异常处理编写专门的入口和退出代码来降低这笔开销。

> Yet another widely-used feature that exploits page faults is *demand paging*. In `exec`, xv6 loads all of an application’s text and data into memory before starting the application. Since applications can be large and reading from disk takes time, this startup cost can be noticeable to users. To decrease startup time, a modern kernel doesn’t initially load the executable file into memory, but just creates the user page table with all PTEs marked invalid. The kernel starts the program running; each time the program uses a page for the first time, a page fault occurs, and in response the kernel reads the content of the page from disk and maps it into the user address space. Like COW fork and lazy allocation, the kernel can implement this feature transparently to applications.

另一个广泛使用的，利用缺页异常的特性是 *按需分页（demand paging）*。在 `exec` 中，xv6 会在启动应用程序之前将应用程序的所有文本和数据加载到内存中。由于应用程序可能很大，并且从磁盘读取数据需要时间，因此用户可能会注意到这种启动成本。为了缩短启动时间，现代内核不会一开始就将可执行文件全部加载到内存中，而是仅创建用户页表，并将所有 PTE 标记为无效。内核启动程序后；每当一个物理页被程序首次访问时，都会发生缺页异常，作为响应，内核会从磁盘读取物理页内容并将其映射到用户地址空间。与 COW fork 和 lazy allocation 类似，内核可以对应用程序透明地实现此功能。

> The programs running on a computer may need more memory than the computer has RAM. To cope gracefully, the operating system may implement *paging to disk*. The idea is to store only a fraction of user pages in RAM, and to store the rest on disk in a *paging area*. The kernel marks PTEs that correspond to memory stored in the paging area (and thus not in RAM) as invalid. If an application tries to use one of the pages that has been *paged out* to disk, the application will incura page fault, and the page must be *paged in*: the kernel trap handler will allocate a page of physical RAM, read the page from disk into the RAM, and modify the relevant PTE to point to the RAM.

计算机上运行的程序所需要的内存可能比计算机实际安装的内存更多。为了妥善应对这种情况，操作系统可能需要实现 *磁盘扩展分页（paging to disk）* 的功能。其原理是只将一小部分用户程序的内容存放在内存中，其余部分则存储在磁盘中专门的 *分页区域（paging area）* 中。内核会将存储在 paging area（而非内存）中的物理页对应的 PTE 标记为无效。如果应用程序尝试使用已 *换出（paged out）* 到磁盘的页，则会触发缺页异常，这将导致内核将这些物理页 *换入（paged in）*：具体操作就是内核的 trap handler 将分配一个物理页，将已经换出到磁盘上的从磁盘读入内存，并修改相关的 PTE 以指向该内存。

> What happens if a page needs to be paged in, but there is no free physical RAM? In that case, the kernel must first free a physical page by paging it out or *evicting* it to the paging area on disk, and marking the PTEs referring to that physical page as invalid. Eviction is expensive, so paging performs best if it’s infrequent: if applications use only a subset of their memory pages and the union of the subsets fits in RAM. This property is often referred to as having good locality of reference. As with many virtual memory techniques, kernels usually implement paging to disk in a way that’s transparent to applications.

如果某个物理页需要换入，但没有可用的物理内存，会发生什么情况呢？在这种情况下，内核必须首先释放一些物理页，方法是将某些内存页上的内容换出（page out）或 “驱逐（evicting）” 到磁盘上的 paging area 中，并将引用该物理页的 PTE 标记为无效。evict 操作的开销很大，因此，如果这个操作不是很频繁的话，那尚可接受，也就是说如果应用程序仅使用其内存页的一部分，并且这部分内容都在内存中。此特性通常被称为具有良好的 “引用局部性（locality of reference）”。与许多虚拟内存技术一样，内核通常以对应用程序透明的方式实现 paging to disk。

> Computers often operate with little or no *free* physical memory, regardless of how much RAM the hardware provides. For example, cloud providers multiplex many customers on a single machine to use their hardware cost-effectively. As another example, users run many applications on smart phones in a small amount of physical memory. In such settings allocating a page may require first evicting an existing page. Thus, when free physical memory is scarce, allocation is expensive.

无论我们给硬件配备多少内存，系统都不会觉得多，这导致计算机通常都只能在极少剩余甚至耗尽物理内存的情况下运行。例如，云服务提供商会在一台机器上为众多客户提供多路复用服务，以经济高效地利用硬件资源。再比如，用户在智能手机上只有少量的物理内存，但需要运行的应用程序却很多。在这种情况下，分配一个物理页往往需要先清除一个现有物理页。因此，当可用物理内存稀缺时，分配成本会非常高。

> Lazy allocation and demand paging are particularly advantageous when free memory is scarce and programs actively use only a fraction of their allocated memory. These techniques can also avoid the work wasted when a page is allocated or loaded but either never used or evicted before it can be used.

当可用内存稀缺但程序仅使用分配给它内存的一小部分时，lazy allocation 和 demand paging 的方案就有很大的优势。这些技术还可以避免因分配或加载物理页后但一直没有使用或在使用前被 evict 而造成的浪费。

> Other features that combine paging and page-fault exceptions include automatically extending stacks and *memory-mapped files*, which are files that a program mapped into its address space using the `mmap` system call so that the program can read and write them using load and store instructions.

基于分页和缺页异常机制还可以实现其他功能，包括自动扩展栈和 *内存映射文件（memory-mapped files）*，所谓 memory-mapped files 是指使用 `mmap` 系统调用将文件映射到进程的地址空间，以便程序可以使用加载和存储指令对文件直接进行读取和写入。

## 4.7 现实世界（Real world）

> The trampoline and trapframe may seem excessively complex. A driving force is that the RISCV intentionally does as little as it can when forcing a trap, to allow the possibility of very fast trap handling, which turns out to be important. As a result, the first few instructions of the kernel trap handler effectively have to execute in the user environment: the user page table, and user register contents. And the trap handler is initially ignorant of useful facts such as the identity of the process that’s running or the address of the kernel page table. A solution is possible because RISC-V provides protected places in which the kernel can stash away information before entering user space: the `sscratch` register, and user page table entries that point to kernel memory but are protected by lack of `PTE_U`. Xv6’s trampoline and trapframe exploit these RISC-V features.

xv6 中引入 trampoline 和 trapframe 的方案看起来可能有点复杂。这么设计的考虑在于，在强制触发 trap 时，能够尽可能少地避免对 RISC-V 处理器的操作，从而实现非常快速的 trap 处理，而这一点至关重要。因此，内核 trap 处理程序的前几条指令实际上必须在用户态上下文中执行，即仍然使用用户页表和用户寄存器的内容。同时 trap handler 程序最初也并不知道一些有用的信息，例如正在运行的进程的 ID或内核页表的地址。RISC-V 提供了一些受保护的位置，内核可以在进入用户空间之前将信息存储在这些位置，这包括：`sscratch` 寄存器，以及指向内核内存但通过不设置 `PTE_U` 而受到保护的用户页表条目。xv6 的 trampoline 和 trapframe 利用了这些 RISC-V 特性。

> The need for special trampoline pages could be eliminated if kernel memory were mapped into every process’s user page table (with `PTE_U` clear). That would also eliminate the need for a page table switch when trapping from user space into the kernel. That in turn would allow system call implementations in the kernel to take advantage of the current process’s user memory being mapped, allowing kernel code to directly dereference user pointers. Many operating systems have used these ideas to increase efficiency. Xv6 avoids them in order to reduce the chances of security bugs in the kernel due to inadvertent use of user pointers, and to reduce some complexity that would be required to ensure that user and kernel virtual addresses don’t overlap.

如果将内核的内存在每个进程的用户页表中都进行映射（同时清除 `PTE_U` 标志位），则可以消除对特殊的 trapmpoline 页的需求。这也将消除从用户空间进入内核时进行页表切换的需要。同时这将允许内核在实现系统调用时利用当前进程映射的用户内存，从而允许内核代码直接利用传入的用户态指针访问内存。许多操作系统已经采用了这些想法来提高效率。xv6 避免了这些想法，以减少由于无意间错误使用用户指针而导致内核出现安全漏洞的可能性，并降低确为了保用户和内核虚拟地址不重叠而引入的复杂性。

> Production operating systems implement copy-on-write fork, lazy allocation, demand paging, paging to disk, memory-mapped files, etc. Furthermore, production operating systems try to store something useful in all areas of physical memory, typically caching file content in memory that isn’t used by processes.

工业级操作系统一般会实现 copy-on-write fork、lazy allocation、demand paging、paging to disk 以及 memory-mapped files 等特性。此外，工业级操作系统会尽可能尝试在物理内存的区域中存储一些有用的内容，譬如将未被进程使用的文件内容缓存在内存中。

> Production operating systems also provide applications with system calls to manage their address spaces and implement their own page-fault handling through the `mmap`, `munmap`, and `sigaction` system calls, as well as providing calls to pin memory into RAM (see `mlock`) and to advise the kernel how an application plans to use its memory (see `madvise`).

工业级操作系统还会为应用程序提供一些系统调用来管理它们的地址空间以及实现自己的缺页异常处理，譬如 `mmap`、`munmap` 和 `sigaction`，此外还会提供系统调用将进程映射的物理内存锁定，避免被换出（譬如 `mlock`）以及提供系统调用告诉内核应用程序计划如何使用其内存，方便内核提前优化对内存的使用（譬如 `madvise`）。

## 4.8 练习（Exercises）

> 1. The functions `copyin` and `copyinstr` walk the user page table in software. Set up the kernel page table so that the kernel has the user program mapped, and `copyin` and `copyinstr` can use `memcpy` to copy system call arguments into kernel space, relying on the hardware to do the page table walk.

1. 函数 `copyin` 和 `copyinstr` 在代码中遍历用户页表。设置内核的页表，使内核映射用户程序，并且使 `copyin` 和 `copyinstr` 可以使用 `memcpy` 将系统调用的参数复制到内核空间，并依赖硬件遍历页表。

> 2. Implement lazy memory allocation.

实现 lazy memory allocation.

> 3. Implement COW fork.

实现 COW fork。

> 4. Is there a way to eliminate the special `TRAPFRAME` page mapping in every user address space? For example, could `uservec` be modified to simply push the 32 user registers onto the kernel stack, or store them in the `proc` structure?

有没有办法消除每个用户地址空间中对特殊的 TRAPFRAME 页的映射？例如，是否可以修改 `uservec`，将 32 个用户寄存器简单地保存到内核栈中，或者将它们存储在 `proc` 结构体中？

> 5. Could xv6 be modified to eliminate the special `TRAMPOLINE` page mapping?

是否可以修改 xv6，消除针对 TRAMPOLINE 页的映射？

> 6. Implement `mmap`.

实现 `mmap`。
