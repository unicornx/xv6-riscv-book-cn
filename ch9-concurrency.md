# 第 9 章 再次讨论并发问题（Chapter 9 Concurrency revisited）

> Simultaneously obtaining good parallel performance, correctness despite concurrency, and understandable code is a big challenge in kernel design. Straightforward use of locks is the best path to correctness, but is not always possible. This chapter highlights examples in which xv6 is forced to use locks in an involved way, and examples where xv6 uses lock-like techniques but not locks.

对于内核并发设计，要同时保证高超的性能、正确的逻辑以及良好的代码可读性是一个巨大的挑战。直接使用锁是实现正确性的最佳途径，但并非总是可行的。本章重点介绍了 xv6 中不得不以复杂的方式使用锁的案例，此外，也介绍了另外一些案例，在这些案例中 xv6 没有直接使用锁而是使用了其他类似锁的技术。

## 9.1 加锁的方式（Locking patterns）

> Cached items are often a challenge to lock. For example, the file system’s block cache (kernel/bio.c:26) stores copies of up to `NBUF` disk blocks. It’s vital that a given disk block have at most one copy in the cache; otherwise, different processes might make conflicting changes to different copies of what ought to be the same block. Each cached block is stored in a `struct buf` (kernel/buf.h:1). A `struct buf` has a lock field which helps ensure that only one process uses a given disk block at a time. However, that lock is not enough: what if a block is not present in the cache at all, and two processes want to use it at the same time? There is no `struct buf` (since the block isn’t yet cached), and thus there is nothing to lock. Xv6 deals with this situation by associating an additional lock (`bcache.lock`) with the set of identities of cached blocks. Code that needs to check if a block is cached (e.g., `bget` (kernel/bio.c:59)), or change the set of cached blocks, must hold `bcache.lock`; after that code has found the block and `struct buf` it needs, it can release `bcache.lock` and lock just the specific block. This is a common pattern: one lock for the set of items, plus one lock per item.

对缓存项上锁通常是一件非常具有挑战性的事。例如，文件系统的 “块缓存（block cache）” (kernel/bio.c:26) 记录了最多 `NBUF` 个磁盘 block 的副本。至关重要的是，给定的磁盘 block 在缓存中只能有一个副本；否则如果不同的进程对同一个 block 的多个副本进行修改的话，必然会导致冲突。每个缓存都存储在 `struct buf` (kernel/buf.h:1) 中。`struct buf` 有一个 `lock` 字段，用于确保同一时间只有一个进程能够访问给定的磁盘 block。然而，仅靠这把锁还不够：如果某个 block 根本不存在于缓存中，而两个进程想要同时使用它，该怎么办？由于没有 `struct buf`（因为该 block 尚未被缓存），因此也就没有可锁定的内容。为了处理这种情况，xv6 为整个缓存 block 集合引入一把额外的锁 (`bcache.lock`) 。在需要检查某个 block 是否已缓存（例如 `bget` (kernel/bio.c:59)）或更改缓存 block 集合时代码必须先持有 `bcache.lock`；在找到所需的 block 和 `struct buf` 后，代码就可以释放 `bcache.lock` 而仅仅锁定特定的 block。这是一种常见的模式：对一组对象加一把全局的锁，同时每个对象还有一把私有的锁。

> Ordinarily the same function that acquires a lock will release it. But a more precise way to view things is that a lock is acquired at the start of a sequence that must appear atomic, and released when that sequence ends. If the sequence starts and ends in different functions, or different threads, or on different CPUs, then the lock acquire and release must do the same. The function of the lock is to force other uses to wait, not to pin a piece of data to a particular agent. One example is the `acquire` in `yield` (kernel/proc.c:512), which is released in the scheduler thread rather than in the acquiring process. Another example is the `acquiresleep` in `ilock` (kernel/fs.c:293); this code often sleeps while reading the disk; it may wake up on a different CPU, which means the lock may be acquired and released on different CPUs.

通常，获取锁的函数也会负责释放锁。但更准确的理解是，锁在一个原子性的序列开始时被获取，并在该序列结束时被释放。如果这个序列分布在不同的函数、不同的线程或者在不同的 CPU 上开始和结束，则锁的获取和释放也必须遵循相同的逻辑。锁的作用是强制其他用户等待，而不是将数据绑定到特定的对象上（译者注：指前面提到的函数、线程或者 CPU）。一个例子是 `yield`（kernel/proc.c:512）中的 `acquire`，该锁在 scheduler 线程中才被释放，而不是在获取锁的函数中。另一个例子是 `ilock`（kernel/fs.c:293）中的 `acquiresleep`；这段代码在读取磁盘时经常处于休眠状态；执行这段代码的线程可能在不同的 CPU 上被唤醒，这意味着上锁和解锁可能发生在不同的 CPU 上。

> Freeing an object that is protected by a lock embedded in the object is a delicate business, since owning the lock is not enough to guarantee that freeing would be correct. The problem case arises when some other thread is waiting in `acquire` to use the object; freeing the object implicitly frees the embedded lock, which will cause the waiting thread to malfunction. One solution is to track how many references to the object exist, so that it is only freed when the last reference disappears. See `pipeclose` (kernel/pipe.c:59) for an example; `pi->readopen` and `pi->writeopen` track whether the pipe has file descriptors referring to it.

如果一个对象中含有一个用于保护该对象的锁成员（译者注：下文我们把该锁成员称之为 “内嵌锁（embedded lock）”），则在释放该对象时需要额外谨慎，因为拥有锁并不足以保证释放操作的正确性。如果在释放这样的一个对象时存在其他线程正在 `acquire` 中等待使用该对象，则会出现问题；因为释放该对象会隐式地释放内嵌锁，这将导致等待的线程出现问题（译者注：这里的释放对象可以理解为调用 `kfree`，这会释放锁所在的内存并导致内存中的值被改写，从而造成解锁的发生，但这种解锁动作并不是我们显式执行的结果）。一种解决方案是跟踪该对象的引用数量，以便仅在最后一个引用消失时才释放它。例如，请参阅 `pipeclose` (kernel/pipe.c:59)；`pi->readopen` 和 `pi->writeopen` 会跟踪管道是否有文件描述符引用它。

> Usually one sees locks around sequences of reads and writes to sets of related items; the locks ensure that other threads see only completed sequences of updates (as long as they, too, lock). What about situations where the update is a simple write to a single shared variable? For example, `setkilled` and `killed` (kernel/proc.c:619) lock around their simple uses of `p->killed`. If there were no lock, one thread could write `p->killed` at the same time that another thread reads it. This is a race, and the C language specification says that a race yields *undefined behavior*, which means the program may crash or yield incorrect results[1]. The locks prevent the race and avoid the undefined behavior.

通常，我们会看到锁用于保护一个由多个读写操作组成的对相关数据项的访问序列；锁确保其他（使用同一把锁的）其他线程在该执行序列没有完成之前必须等待。但如果操作只是对单个共享变量的单个写入，情况又会如何呢？例如，`setkilled` 和 `killed` (kernel/proc.c:619) 会在对 `p->killed` 的简单读写前后上锁和解锁。如果没有锁，可能会出现当一个线程对 `p->killed` 写入的同时另一个线程读取它。这是一种竞争，C 语言规范指出竞争会导致 *未定义行为（undefined behavior）*，这意味着程序可能崩溃或产生不正确的结果 [1]。锁可以阻止竞争，从而避免未定义行为。

> One reason races can break programs is that, if there are no locks or equivalent constructs, the compiler may generate machine code that reads and writes memory in ways quite different than the original C code. For example, the machine code of a thread calling `killed` could copy `p->killed` to a register and read only that cached value; this would mean that the thread might never see any writes to `p->killed`. The locks prevent such caching.

竞争会导致程序崩溃的一个原因是，如果没有锁或等效的机制，编译器可能会生成与原始 C 代码截然不同的读写内存的机器码。例如，调用`killed` 的线程的机器指令可能会将 `p->killed` 复制到寄存器，并只读取该寄存器中缓存的值；这意味着该线程可能永远不会看到对 `p->killed` 更新。锁可以防止这种缓存。

## 9.2 类似锁的并发保护方式（Lock-like patterns）

> In many places xv6 uses a reference count or a flag in a lock-like way to indicate that an object is allocated and should not be freed or re-used. A process’s `p->state` acts in this way, as do the reference counts in `file`, `inode`, and `buf` structures. While in each case a lock protects the flag or reference count, it is the latter that prevents the object from being prematurely freed.

xv6 在很多地方以 “类似锁（lock-like）” 的方式使用引用计数或标志来标识对象已分配且不应被释放或重用。进程的 `p->state` 以这种方式运行，`file`、`inode` 和 `buf` 结构中的引用计数也是如此。虽然在每种情况下，锁都会保护标志或引用计数，但后者会阻止对象被过早释放。

> The file system uses `struct inode` reference counts as a kind of shared lock that can be held by multiple processes, in order to avoid deadlocks that would occur if the code used ordinary locks. For example, the loop in `namex` (kernel/fs.c:652) locks the directory named by each pathname component in turn. However, `namex` must release each lock at the end of the loop, since if it held multiple locks it could deadlock with itself if the pathname included a dot (e.g., `a/./b`). It might also deadlock with a concurrent lookup involving the directory and ... As Chapter 8 explains, the solution is for the loop to carry the directory inode over to the next iteration with its reference count incremented, but not locked.

文件系统使用 `struct inode` 结构体中的引用计数（即 `ref` 成员字段）作为一种可由多个进程持有的 “共享锁”，以避免代码使用普通锁时可能发生的死锁。例如，`namex`（kernel/fs.c:652）中的循环依次锁定每个路径名中的 component 所对应的目录。但是，`namex` 必须在循环结束时释放每个锁，因为如果它持有多个锁，当路径名包含一个 “.”（例如，`a/./b`）时，它可能会与自身发生死锁。它还可能与涉及目录的并发查找发生死锁 ...... 正如第 8 章所解释的，解决方案是让循环将目录的 inode 传递到下一次迭代，其引用计数递增，但不锁定。

> Some data items are protected by different mechanisms at different times, and may at times be protected from concurrent access implicitly by the structure of the xv6 code rather than by explicit locks. For example, when a physical page is free, it is protected by `kmem.lock` (kernel/kalloc.c:24). If the page is then allocated as a pipe (kernel/pipe.c:23), it is protected by a different lock (the embedded `pi->lock`). If the page is re-allocated for a new process’s user memory, it is not protected by a lock at all. Instead, the fact that the allocator won’t give that page to any other process (until it is freed) protects it from concurrent access. The ownership of a new process’s memory is complex: first the parent allocates and manipulates it in `fork`, then the child uses it, and (after the child exits) the parent again owns the memory and passes it to `kfree`. There are two lessons here: a data object may be protected from concurrency in different ways at different points in its lifetime, and the protection may take the form of implicit structure rather than explicit locks.

某些数据项在不同时间受不同机制保护，有时可能通过 xv6 代码结构（而非显式锁）隐式地防止并发访问。例如，当一个物理页空闲时，它受 `kmem.lock` (kernel/kalloc.c:24) 保护。如果该页随后被分配给管道 (kernel/pipe.c:23)，则受另一个锁（嵌入的 `pi->lock`）保护。如果该页被重新分配给新进程的用户内存，则它完全不受锁保护。相反，分配器不会将该页分配给任何其他进程（直到它被释放），这一事实保护了它免受并发访问。新进程内存的所有权非常复杂：首先，父进程在 `fork` 中分配并操作它，然后子进程使用它，并且（子进程退出后）父进程再次拥有该内存并将其传递给 `kfree`。这里有两条教训：在数据对象的生命周期的不同阶段，可以采用不同的方式保护其免于并发，并且这种保护可以采用隐式结构而不是显式锁的形式。

> A final lock-like example is the need to disable interrupts around calls to `mycpu()` (kernel/proc.c:83). Disabling interrupts causes the calling code to be atomic with respect to timer interrupts that could force a context switch, and thus move the process to a different CPU.

最后一个类似锁的例子是，需要在 `mycpu()` 调用前后禁用中断 (kernel/proc.c:83)。禁用中断会导致这段代码不会受定时器中断干扰而具有原子性，因为定时器中断可能强制进行上下文切换，从而将进程移动到不同的 CPU 上。

[1]: "Threads and data races" in https://en.cppreference.com/w/c/language/memory_model

## 9.3 完全不上锁（No locks at all）

> There are a few places where xv6 shares mutable data with no locks at all. One is in the implementation of spinlocks, although one could view the RISC-V atomic instructions as relying on locks implemented in hardware. Another is the `started` variable in `main.c` (kernel/main.c:7), used to prevent other CPUs from running until CPU zero has finished initializing xv6; the `volatile` ensures that the compiler actually generates load and store instructions.

xv6 中有些地方在共享可变数据时并不采用锁进行保护。一个是自旋锁的实现，尽管 RISC-V 的原子指令也可以看作依赖于硬件实现的锁。另一个是 `main.c` (kernel/main.c:7) 中的 `started` 变量，用于阻止其他 CPU 在第一个 CPU（编号为 0）完成 xv6 初始化之前运行；`volatile` 这个 C 语言的类型限定符确保编译器确实生成了加载和存储指令。

> Xv6 contains cases in which one CPU or thread writes some data, and another CPU or thread reads the data, but there is no specific lock dedicated to protecting that data. For example, in `fork`, the parent writes the child’s user memory pages, and the child (a different thread, perhaps on a different CPU) reads those pages; no lock explicitly protects those pages. This is not strictly a locking problem, since the child doesn’t start executing until after the parent has finished writing. It is a potential memory ordering problem (see Chapter 6), since without a memory barrier there’s no reason to expect one CPU to see another CPU’s writes. However, since the parent releases locks, and the child acquires locks as it starts up, the memory barriers in `acquire` and `release` ensure that the child’s CPU sees the parent’s writes.

xv6 中还存在这样的情况：一个 CPU 或线程写入数据，另一个 CPU 或线程读取数据，但没有专门的锁来保护这些数据。例如，在 `fork` 中，父进程写入子进程的用户内存页，而子进程（另一个线程，可能位于不同的 CPU 上）读取这些页；没有锁明确地保护这些页。严格来说，这并非是一个需要引入锁保护的问题，因为子进程直到父进程完成写入后才开始执行。这是一个潜在的 “内存排序（memory ordering）” 问题（参见第 6 章），因为如果没有内存屏障，就无法保证一个 CPU 正确地看到另一个 CPU 的写入结果。但是，由于父进程会释放锁，而子进程会在启动时获取锁，因此 `acquire` 和 `release` 中执行的内存屏障可以确保子进程的 CPU 能够看到父进程的写入。

## 9.4 并行性（Parallelism）

> Locking is primarily about suppressing parallelism in the interests of correctness. Because performance is also important, kernel designers often have to think about how to use locks in a way that both achieves correctness and allows parallelism. While xv6 is not systematically designed for high performance, it’s still worth considering which xv6 operations can execute in parallel, and which might conflict on locks.

锁定主要是为了正确性而限制了并行性。由于性能也很重要，内核设计人员经常需要考虑如何使用锁，既能实现正确性，又能实现并行性。虽然 xv6 并非一个为追求高性能而设计的系统，但（具体实现 xv6 时）仍然值得花些时间考虑一下哪些操作可以并行执行，哪些操作可能会在上锁时发生冲突。

> Pipes in xv6 are an example of fairly good parallelism. Each pipe has its own lock, so that different processes can read and write different pipes in parallel on different CPUs. For a given pipe, however, the writer and reader must wait for each other to release the lock; they can’t read/write the same pipe at the same time. It is also the case that a read from an empty pipe (or a write to a full pipe) must block, but this is not due to the locking scheme.

xv6 中的管道是展示并行性的相当好的一个示例。每个管道都有自己的锁，因此不同的进程可以在不同的 CPU 上并行地读写不同的管道。然而，对于给定的管道，写入者和读取者必须等待对方释放锁；他们不能同时读写同一个管道。从空管道读取（或向满管道写入）也必须阻塞，但这并不是由于锁方案造成的。

> Context switching is a more complex example. Two kernel threads, each executing on its own CPU, can call `yield`, `sched`, and `swtch` at the same time, and the calls will execute in parallel. The threads each hold a lock, but they are different locks, so they don’t have to wait for each other. Once in `scheduler`, however, the two CPUs may conflict on locks while searching the table of processes for one that is `RUNNABLE`. That is, xv6 is likely to get a performance benefit from multiple CPUs during context switch, but perhaps not as much as it could.

上下文切换是一个更复杂的例子。两个内核线程，每个线程都在各自的 CPU 上执行，可以同时调用 `yield`、`sched` 和 `swtch`，并且这些调用将并行执行。每个线程都持有一个锁，但它们是不同的锁，因此它们不必互相等待。然而，一旦进入 `scheduler`，两个 CPU 在搜索进程表以查找 `RUNNABLE` 的进程时可能会发生锁冲突。也就是说，xv6 在上下文切换期间可能会从多 CPU 中获得性能优势，但可能不如预期的那样显著。

> Another example is concurrent calls to `fork` from different processes on different CPUs. The calls may have to wait for each other for `pid_lock` and `kmem.lock`, and for per-process locks needed to search the process table for an `UNUSED` process. On the other hand, the two forking processes can copy user memory pages and format page-table pages fully in parallel.

另一个例子是来自不同 CPU 上的不同进程的并发 `fork` 调用。这些调用可能需要互相等待 `pid_lock` 和 `kmem.lock`，以及在进程表中搜索 `UNUSED` 进程时需要等待每个进程的锁。另一方面，两个 fork 进程可以完全并行地复制用户内存页并格式化页表页。

> The locking scheme in each of the above examples sacrifices parallel performance in certain cases. In each case it’s possible to obtain more parallelism using a more elaborate design. Whether it’s worthwhile depends on details: how often the relevant operations are invoked, how long the code spends with a contended lock held, how many CPUs might be running conflicting operations at the same time, whether other parts of the code are more restrictive bottlenecks. It can be difficult to guess whether a given locking scheme might cause performance problems, or whether a new design is significantly better, so measurement on realistic workloads is often required.

上述每个示例中的锁定方案在某些情况下都会牺牲并行性能。在每种情况下，使用更精细的设计都可能获得更高的并行性。这是否值得取决于一些细节，譬如：相关操作的调用频率；代码在争用锁的情况下花费的时间；可能会有多少个 CPU 同时运行冲突的操作；代码的其他部分是否是更具限制性的瓶颈等。很难猜测给定的锁定方案是否会导致性能问题，或者新的设计是否显著优于现有方案，因此通常需要基于实际工作负载进行测量。

## 9.5 练习（Exercises）

> 1. Modify xv6’s pipe implementation to allow a read and a write to the same pipe to proceed in parallel on different CPUs.

1. 修改 xv6 的管道实现，以允许对同一管道的读取和写入在不同的 CPU 上并行进行。

> 2. Modify xv6’s `scheduler()` to reduce lock contention when different CPUs are looking for runnable processes at the same time.

2. 修改 xv6 的 `scheduler()`，以减少不同 CPU 同时寻找可运行进程时的锁争用。

> 3. Eliminate some of the serialization in xv6’s `fork()`.

3. 消除 xv6 的 `fork()` 中的部分串行化代码。
