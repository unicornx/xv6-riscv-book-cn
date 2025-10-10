# 第 5 章 缺页异常（Chapter 5 Page faults）

> The RISC-V CPU raises a page-fault exception when a virtual address is used that has no mapping in the page table, or has a mapping whose `PTE_V` flag is clear, or a mapping whose permission bits (`PTE_R`, `PTE_W`, `PTE_X`, `PTE_U`) forbid the operation being attempted. RISC-V distinguishes three kinds of page fault: load page faults (caused by load instructions), store page faults (caused by store instructions), and instruction page faults (caused by fetches of instructions to be executed). The `scause` register indicates the type of the page fault and the `stval` register contains the address that couldn’t be translated.

在以下情况下会触发 RISC-V CPU 产生 “缺页（page-fault）” 异常：当使用的虚拟地址在页表中没有被映射，或者映射的 `PTE_V` 标志为 0（清除状态），或者指令尝试执行针对该虚拟地址禁止的操作（体现在映射的权限位 `PTE_R`、`PTE_W`、`PTE_X`、`PTE_U` 上）。RISC-V 区分三种类型的缺页异常：“加载页异常（load page fault）”（由加载指令触发）、“存储页异常（store page fault）”（由存储指令触发）和 “指令页异常（instruction page fault）”（由取指操作触发）。`scause` 寄存器指示缺页异常的类型，`stval` 寄存器会指出异常发生时当前访问的内存地址。

> The combination of page tables and page faults is a powerful tool. Page tables give the kernel a level of indirection between virtual and physical addresses, so that the kernel can control the structure and content of address spaces. Page faults allow the kernel to intercept loads and stores and, by modifying the page table, specify on the fly what data those references refer to. The kernel can use these capabilities to increase efficiency: for example, copy-on-write fork allows the kernel to transparently share memory between parent and child, avoiding the cost of copying pages that neither write. Application programmers can also benefit. One possibility is memory-mapped files, where the kernel uses paging to cause a file’s content to appear in an application’s address space, transparently reading file blocks in response to page faults. Another is lazy memory allocation, which allows a program to ask for a huge virtual address space, but only to pay the cost of allocating physical memory for the pages the program actually reads and writes. xv6 uses page faults for only one purpose: lazy allocation.

组合利用页表和缺页异常可以实现强大的功能。页表为内核在虚拟地址和物理地址之间实现了一层封装（译者注，即内核也不会直接访问物理地址，而是在页表和 MMU 的帮助下通过虚拟地址访问物理内存），这使得内核可以控制地址空间的布局结构和内容。缺页异常使得内核可以拦截加载和存储操作，并通过修改页表，动态指定这些引用（即虚拟地址）指向的数据（物理内存）。内核可以利用这些功能来提高运行效率：例如，基于 “写时复制（copy-on-write）” 实现 fork 允许内核在父进程和子进程之间 “透明地（transparently）” 共享内存，从而避免在父子进程都没有发生写内存操作之前复制物理页的开销。编写应用程序得程序员也可以从中受益。譬如利用 *内存映射文件（memory-mapped files）* 技术，该技术在内核中利用分页将文件内容映射到应用程序的地址空间中（译者注，内存映射后虽然文件的内容可能还在磁盘上不在内存中，但给应用程序的感觉是可以使用访问内存的加载和存储指令对文件直接进行读取和写入），并在实际发生缺页异常时才将文件内容从磁盘上读入内存。另一种可能性是 “延迟内存分配（lazy memory allocation,）”，它允许程序请求一段巨大的虚拟地址空间，但只需为程序实际读写的页分配物理内存。在 xv6 中利用缺页异常只实现了一个特性，即 “延迟分配（lazy allocation）”。

> Before proceeding, please read the functions `sys_sbrk()` in `kernel/sysproc.c`, and `vmfault` in `kernel/vm.c`. Search for calls to `vmfault` in `kernel/trap.c` and `kernel/vm.c`.

在继续之前，请阅读 `kernel/sysproc.c` 中的 `sys_sbrk()` 函数，以及 `kernel/vm.c` 中的 `vmfault` 函数。在`kernel/trap.c` 和 `kernel/vm.c` 中搜索哪里调用了函数 `vmfault`。

## 5.1 延迟分配（Lazy allocation）

> xv6’s *lazy allocation* has two parts. First, when an application asks for memory by calling `sbrk` with the flag `SBRK_LAZY`, the kernel notes the increase in size, but does not allocate physical memory and does not create PTEs for the new range of virtual addresses. Second, on a page fault on one of those new addresses, the kernel allocates a page of physical memory and maps it into the page table. The kernel implements lazy allocation transparently to applications: no modifications to applications are necessary for them to benefit.

xv6 的 *延迟分配* 实现由两部分组成。首先，应用程序通过调用 `sbrk` 并在请求分配内存时传入 `SBRK_LAZY` 标志，内核只会记录内存大小的增加，但不会分配物理内存，也不会为新的虚拟地址范围创建页表项 (PTE)。其次，当涉及到的任何一个新地址上发生缺页异常时，内核会分配一页物理内存并将其映射到页表中。内核的延迟分配实现对应用程序来说是无感的：应用程序无需任何修改即可从中受益。

> Lazy allocation is convenient for applications because they don’t have to accurately predict how much memory they will need. For example, an application may process input, but not know in advance how large the input will be. With lazy allocations an application can ask for memory for the worst case, but not have to pay for this worst case: the kernel doesn’t have to do any work at all for pages that the application never uses.

延迟分配对于应用程序来说非常方便，因为对于应用程序来说无需准确预测所需的（实际）内存量。例如，应用程序可能会处理输入，但事先并不知道输入的大小。基于延迟分配，应用程序可以按照最极端的情况申请尽可能大的内存，而无需为此付出代价，因为内核并不需要为应用程序从未使用过的页分配实际的内存。

> Furthermore, if the application is asking to grow the address space by a lot, then `sbrk` without lazy allocation is expensive: if an application asks for a gigabyte of memory, the kernel has to allocate and zero 262,144 4096-byte physical pages. Lazy allocation allows this cost to be spread over time. On the other hand, lazy allocation incurs the extra overhead of page faults, which involve a user/kernel transition. Operating systems can reduce this cost by allocating a batch of consecutive pages per page fault instead of one page and by specializing the kernel entry/exit code for such page-faults (though xv6 does neither).

此外，如果应用程序要求大幅增加地址空间，那么如果 `sbrk` 不支持延迟分配将会非常耗时：因为假设应用程序请求 1GB 内存，内核必须（一次性）分配并清零 262144 个大小为 4096 字节的物理页。延迟分配机制可以将这部分开销分摊到多个时间点上去。但另一方面，延迟分配机制会引入缺页异常处理所带来的额外开销，这包括用户态和内核态之间切换等处理。操作系统可以通过在缺页异常发生时分配一批连续的物理页（而不只是一页），并专门为此类缺页异常编写内核的入口和出口代码来抵消这部分开销（然而 xv6 并没有做这些优化）。

> On the other hand, when taking a page fault for a lazily-allocated page, the kernel may find that it has not free memory to allocate. In this case, the kernel has no easy way of returning an out-of-memory error to the application and instead kills the application. For applications that prefer an error on a failed allocation, xv6 allows an application to allocate memory eagerly by calling `sbrk` with the flag `SBRK_EAGER`.

另一方面，当延迟分配物理页时，内核可能会发现没有可用的内存可供分配了。在这种情况下，由于内核无法方便地给应用程序返回内存不足的错误（译者注，因为此时错误发生在异步的异常处理过程中，而不是源自应用程序发起的系统调用），所以内核选择直接终止该应用程序。对于那些希望在分配时就明确知道是否成功的应用程序，xv6 允许应用程序在调用 `sbrk` 时传入 `SBRK_EAGER` 标志来指明这一点。

## 5.2 代码讲解（Code）

> The system call `sbrk(n)` grows (or shrinks if `n` is negative) a process’s memory size by n bytes, and returns the start of the newly allocated region (i.e., the old size). The kernel implementation is `sys_sbrk` (3801).

系统调用 `sbrk(n)` 会将进程的内存大小增加 `n` 个字节（如果 `n` 为负，则减少），并返回新分配区域的起始位置（即原先的大小）。内核中的实现对应为 `sys_sbrk` (3801)。

> If the application specifies `SBRK_EAGER`, the system call is implemented by the function `growproc` (2353). `growproc` calls `uvmalloc`. `uvmalloc` (1628) allocates physical memory with `kalloc`, zeros the allocated memory, and adds PTEs to the user page table with `mappages`.

如果应用程序指定了 `SBRK_EAGER`，则系统调用由函数 `growproc`（2353）实现。`growproc` 调用 `uvmalloc`。`uvmalloc`（1628）使用 `kalloc` 分配物理内存，将分配的内存清零，并使用 `mappages` 将 PTE 添加到用户页表中。

> If the applications allocates memory lazily, `sys_sbrk` just increments the process’s size (`myproc()->sz`) by `n` and returns the old size; it does not allocate physical memory or add PTEs to the process’s page table.

如果应用程序选择延迟分配内存，`sys_sbrk` 只会将进程的大小（`myproc()->sz`）增加 `n` 并返回原先的大小；它不会分配物理内存或为进程的页表添加 PTE。

> When a process loads or stores to a virtual address that lacks a valid page-table mapping, the CPU will raise *page-fault* exception. `usertrap` checks for this case (3372) and calls `vmfault` (1879) to handle the page fault. `vmfault` checks that the faulting address is within the region previously granted by `sbrk`, allocates a page of physical memory with `kalloc`, zeros the allocated page, and adds a PTE to the user page table with `mappages`. Xv6 sets the `PTE_W`, `PTE_R`, `PTE_U`, and `PTE_V` flags in the PTE for the new page. Then, `usertrap` resumes the process at the instruction that caused the fault. Because the PTE is now valid, the re-executed load or store instruction will execute without a fault.

当进程执行加载或存储操作时访问到没有有效映射的虚拟地址时，CPU 将触发 *缺页（page-fault）* 异常。`usertrap` 中针对这种情况（3372）会调用 `vmfault`（1879）。`vmfault` 首先检查触发异常涉及的虚拟地址是否落在 `sbrk` 先前分配的区域内，然后使用 `kalloc` 分配一个物理内存页并将分配的页清零，最后调用 `mappages` 向用户页表添加 PTE。xv6 为新页的 PTE 设置 `PTE_W`、`PTE_R`、`PTE_U` 和 `PTE_V` 标志。`usertrap` 返回后 CPU 从导致异常指令处恢复进程执行（译者注，即重新执行导致缺页异常的指令）。由于 PTE 现在有效，重新执行的加载或存储指令将不会出错。

> If an application frees memory using `sbrk`, `sys_sbrk` calls `shrinkproc`, which calls `uvmdealloc`. The real work is done by `uvmunmap` (1604), which uses `walk` to find PTEs. Since some pages may never have been used by the process and thus never have been allocated by `vmfault`, `uvmunmap` skips PTEs without the `PTE_V` flag. If a PTE mapping is valid, `uvmunmap` calls `kfree` to free the physical memory it refers to.

如果应用程序使用 `sbrk` 释放内存，`sys_sbrk` 会调用 `shrinkproc`，而 `shrinkproc` 又会调用 `uvmdealloc`。真正的工作由 `uvmunmap` (1604) 完成，它使用 `walk` 来查找 PTE。由于某些页（译者注：虚拟的）可能从未被进程使用过，因此 `vmfault` 也从未为其分配过物理内存，`uvmunmap` 会跳过没有 `PTE_V` 标志的 PTE。如果 PTE 映射有效，`uvmunmap` 会调用 `kfree` 来释放它所引用的物理内存。

> Note that Xv6 uses a process’s page table not just to tell the hardware how to map user virtual addresses, but also as the only record of which physical memory pages are allocated to that process. That is the reason why freeing user memory (in `uvmunmap`) requires examination of the user page table.

注意 xv6 使用进程的页表不仅是为了告诉硬件如何映射用户虚拟地址，而且也正是在这份页表中唯一记录了为该进程分配了哪些物理内存页。这就是为什么释放用户内存（在 `uvmunmap` 中）需要检查用户页表的原因。

## 5.3 现实世界：写时拷贝 fork（Real world: Copy-On-Write (COW) fork）

> Many kernels (though not xv6) use page faults to implement *copy-on-write (COW) fork*. The `fork` system call promises that the child sees memory whose initial content is the same as the parent’s memory at the time of the fork. One way to implement this is to copy the entire memory of the parent to newly allocated physical memory for the child; this is what xv6 does. Copying can be slow, and it would be more efficient if the child could share the parent’s physical memory. A straightforward implementation of this would not work, however, since it would cause the parent and child to disrupt each other’s execution with their writes to the shared stack and heap.

许多内核（xv6 除外）使用缺页异常来实现 *写时复制 (copy-on-write，简称 COW) fork*。`fork` 系统调用确保子进程看到的内存初始内容与父进程在 fork 时看到的内存内容相同。实现此操作的一种方法是将父进程的整个内存复制到为子进程新分配的物理内存中；xv6 就是这么做的。复制过程可能很慢，如果子进程可以共享父进程的物理内存，效率会更高。然而，直接实现此操作并不可行，因为这么做会使得父进程和子进程也共用了同一份栈和堆，任何一方改写了栈和堆的内容都会干扰另一方的运行。

> Copy-on-write fork causes parent and child to safely share physical memory by appropriate use of page-table permissions and page faults.The basic plan is for the parent and child to initially share all physical pages, but for each to map them read-only (with the `PTE_W` flag clear). Parent and child can then read from the shared physical memory. If either writes a shared page, the RISC-V CPU raises a page-fault exception. A kernel supporting COW would respond by allocating a new page of physical memory and copying the shared page into that new page. Then kernel would change the relevant PTE in the faulting process’s page table to point to the copy and to allow writes as well as reads, and then resume the faulting process at the instruction that caused the fault. Because the PTE now allows writes, the re-executed store instruction will execute without a fault, and will modify a private copy of the page rather than the shared page.

COW fork 通过适当使用页表权限和缺页异常，使父进程和子进程可以安全地共享物理内存。其基本方案是父进程和子进程最初共享所有物理页面，但各自只以只读方式映射这些页面（即清除 `PTE_W` 标志）。父进程和子进程可以从共享的物理内存中读取数据。但如果其中任何一方对某个共享的页执行了写操作，RISC-V CPU 就会触发缺页异常。支持 COW 的内核的 trap 处理程序会做出响应，分配一个新的物理内存页，并将共享的物理页内容复制到该页中。内核会将出错进程页表中的相关 PTE 更改为指向新的副本，并允许对其写入和读取，然后在导致错误的指令处恢复出错进程的执行。由于 PTE 现在允许写入，因此重新执行的存储（store）指令将正确执行，此时被修改的页将是当前进程私有的副本而不再是共享的页。

>  Copy-on-write requires book-keeping to help decide when physical pages can be freed, since each page can be referenced by a varying number of page tables depending on the history of forks, page faults, execs, and exits. This book-keeping allows an important optimization: if a process incurs a store page fault and the physical page is only referred to from that process’s page table, no copy is needed.

因为 fork、缺页异常处理、exec 和 exit 等操作的发生都在不停地改变一个物理页被不同页表所引用的状态，所以引入 COW 处理后需要记录引用信息来帮助决定何时可以释放这些物理页。利用这些记录的引用信息可以实现重要的优化：譬如某个进程在写入物理内存时触发了缺页异常，但由于该物理页仅被当前此进程的页表所引用，则不需要触发 COW。

> Copy-on-write makes `fork` faster, since `fork` need not copy memory. Some of the memory will have to be copied later, when written, but it’s often the case that most of the memory never has to be copied. A common example is `fork` followed by `exec`: a few pages may be written after the `fork`, but then the child’s `exec` releases the bulk of the memory inherited from the parent. Copy-on-write `fork` eliminates the need to ever copy this memory. Furthermore, COW fork is transparent: no modifications to applications are necessary for them to benefit.

COW 使 `fork` 运行速度更快，因为 `fork` 无需复制内存。部分内存会稍后在发生写入操作时才会被复制，但通常情况下，大部分内存无需复制。一个常见的例子是 `fork` 后紧接着执行 `exec`：在 `fork` 之后可能会对少量页执行写操作，但随后子进程因为执行 `exec` 会释放从父进程继承的大部分内存。支持 COW 的 `fork` 则无需复制这些内存。此外，COW fork 对应用程序是 “透明的（transparent）”，即无需对应用程序进行任何修改即可从中受益。

## 5.4 现实世界：按需分页（Real world: Demand paging）

> Yet another widely-used feature that exploits page faults is *demand paging*. In the `exec` system call, xv6 loads all of an application’s text and data into memory before starting the application. Since applications can be large and reading from disk takes time, this startup cost can be noticeable to users. To decrease startup time, a modern kernel doesn’t initially load the executable file into memory, but just creates the user page table with all PTEs marked invalid. The kernel starts the program running; each time the program uses a page for the first time, a page fault occurs, and in response the kernel reads the content of the page from disk and maps it into the user address space. Like COW fork and lazy allocation, the kernel can implement this feature transparently to applications.

另一个广泛使用的，利用缺页异常的特性是 *按需分页（demand paging）*。在系统调用 `exec` 中，xv6 会在启动应用程序之前将应用程序的所有文本和数据加载到内存中。由于应用程序可能很大，并且从磁盘读取数据需要时间，因此用户可能会注意到这种启动成本。为了缩短启动时间，现代内核不会一开始就将可执行文件全部加载到内存中，而是仅创建用户页表，并将所有 PTE 标记为无效。内核启动程序后；每当一个物理页被程序首次访问时，都会发生缺页异常，作为响应，内核会从磁盘读取物理页内容并将其映射到用户地址空间。与 COW fork 和 lazy allocation 类似，内核可以对应用程序透明地实现此功能。

> The programs running on a computer may need more memory than the computer has RAM. To cope gracefully, the operating system may implement *paging to disk*. The idea is to store only a fraction of user pages in RAM, and to store the rest on disk in a *paging area*. The kernel marks PTEs that correspond to memory stored in the paging area (and thus not in RAM) as invalid. If an application tries to use one of the pages that has been *paged out* to disk, the application will incura page fault, and the page must be *paged in*: the kernel trap handler will allocate a page of physical RAM, read the page from disk into the RAM, and modify the relevant PTE to point to the RAM.

计算机上运行的程序所需要的内存可能比计算机实际安装的内存更多。为了妥善应对这种情况，操作系统可能需要实现 *磁盘扩展分页（paging to disk）* 的功能。其原理是只将一小部分用户程序的内容存放在内存中，其余部分则存储在磁盘中专门的 *分页区域（paging area）* 中。内核会将存储在分页区域（而非内存）中的页对应的 PTE 标记为无效。如果应用程序尝试使用已 *换出（paged out）* 到磁盘的页，则会触发缺页异常，这将导致内核将这些页 *换入（paged in）*：具体操作就是内核的 trap handler 将分配一个物理页，将已经换出到磁盘上的页从磁盘再次读入内存，并修改相关的 PTE 以指向该内存。

> What happens if a page needs to be paged in, but there is no free physical RAM? In that case, the kernel must first free a physical page by paging it out or *evicting* it to the paging area on disk, and marking the PTEs referring to that physical page as invalid. Eviction is expensive, so paging performs best if it’s infrequent: if applications use only a subset of their memory pages and the union of the subsets fits in RAM. This property is often referred to as having good locality of reference. As with many virtual memory techniques, kernels usually implement paging to disk in a way that’s transparent to applications.

如果某个页需要换入，但没有可用的物理内存，会发生什么情况呢？在这种情况下，内核必须首先释放一些物理页，方法是将某些内存中的页上的内容 “换出（page out）” 或 “驱逐（evicting）” 到磁盘上的分页区域中，并将引用该物理页的 PTE 标记为无效。驱逐操作的开销很大，因此，如果这个操作不是很频繁的话，那尚可接受，也就是说如果应用程序仅使用其内存页的一部分，并且这部分内容都在内存中。此特性通常被称为具有良好的 “引用局部性（locality of reference）”。与许多虚拟内存技术一样，内核通常以对应用程序透明的方式实现 paging to disk。

> Computers often operate with little or no *free* physical memory, regardless of how much RAM the hardware provides. For example, cloud providers multiplex many customers on a single machine to use their hardware cost-effectively. As another example, users run many applications on smart phones in a small amount of physical memory. In such settings allocating a page may require first evicting an existing page. Thus, when free physical memory is scarce, allocation is expensive.

无论我们给硬件配备多少内存，系统都不会觉得多，这导致计算机通常都只能在极少剩余甚至耗尽物理内存的情况下运行。例如，云服务提供商会在一台机器上为众多客户提供多路复用服务，以经济高效地利用硬件资源。再比如，用户在智能手机上只有少量的物理内存，但需要运行的应用程序却很多。在这种情况下，分配一个物理页往往需要先清除一个现有物理页。因此，当可用物理内存稀缺时，分配成本会非常高。

> Lazy allocation and demand paging are particularly advantageous when free memory is scarce and programs actively use only a fraction of their allocated memory. These techniques can also avoid the work wasted when a page is allocated or loaded but either never used or evicted before it can be used.

当可用内存稀缺但程序仅使用分配给它内存的一小部分时，lazy allocation 和 demand paging 的方案就有很大的优势。这些技术还可以避免因分配或加载物理页后但一直没有使用或在使用前被驱逐所造成的浪费。

## 5.5 现实世界：内存映射文件（Real world: Memory-mapped files）

> Other features that combine paging and page-fault exceptions include automatically extending stacks and *memory-mapped files*, which are files that a program maps into its address space using the `mmap` system call so that the program can read and write them using load and store instructions.

基于分页和缺页异常机制还可以实现其他功能，包括自动扩展栈和 *内存映射文件（memory-mapped files）*，所谓 memory-mapped files 是指使用 `mmap` 系统调用将文件映射到进程的地址空间，以便程序可以使用加载和存储指令对文件直接进行读取和写入。

## 5.6 练习（Exercises）

> 1. Write a user program that grows its address space by one byte by calling `sbrk(1)`. Run the program and investigate the page table for the program before the call to `sbrk` and after the call to `sbrk`. How much space has the kernel allocated? What does the PTE for the new memory contain?

1. 编写一个用户程序，通过调用 `sbrk(1)` 将其地址空间增加一个字节。运行该程序，并查看调用 `sbrk` 之前和之后程序的页表。内核分配了多少空间？新内存的 PTE 包含什么内容？

> 2. Implement COW fork.

2. 实现 COW fork。

> 3. Implement `mmap`.

3. 实现 `mmap`。
