# 第 5 章 中断和设备驱动（Chapter 5 Interrupts and device drivers）

> A *driver* is the code in an operating system that manages a particular device: it configures the device hardware, tells the device to perform operations, handles the resulting interrupts, and interacts with processes that may be waiting for I/O from the device. Driver code can be tricky because a driver executes concurrently with the device that it manages. In addition, the driver must understand the device’s hardware interface, which can be complex and poorly documented.

*驱动（driver）* 是操作系统中管理特定设备的代码：它配置硬件设备、通知设备执行操作、处理产生的中断以及与可能正在等待设备输入和输出的进程交互。编写驱动程序代码可能比较棘手，因为执行驱动程序的硬件（处理器）和驱动程序管理的硬件并非同一个设备（译者注：这也是下文要提到的中断等异步处理的来源）。此外，编写驱动程序必须了解设备的硬件接口，而这可能很复杂且常常缺少良好的文档说明。

> Devices that need attention from the operating system can usually be configured to generate interrupts, which are one type of trap. The kernel trap handling code recognizes when a device has raised an interrupt and calls the driver’s interrupt handler; in xv6, this dispatch happens in `devintr` (kernel/trap.c:185).

操作系统管理的设备通常需要配置为产生 “中断（interrupt）”，这是属于 trap 的一种。当设备触发中断时内核的 trap 处理代码会被执行并调用驱动程序的中断处理程序；在 xv6 中，这个处理发生在 `devintr` (kernel/trap.c:185) 中。

> Many device drivers execute code in two contexts: a *top half* that runs in a process’s kernel thread, and a *bottom half* that executes at interrupt time. The top half is called via system calls such as `read` and `write` that want the device to perform I/O. This code may ask the hardware to start an operation (e.g., ask the disk to read a block); then the code waits for the operation to complete. Eventually the device completes the operation and raises an interrupt. The driver’s interrupt handler, acting as the bottom half, figures out what operation has completed, wakes up a waiting process if appropriate, and tells the hardware to start work on any waiting next operation.

许多设备驱动程序的代码在两种上下文环境中执行：一种称之为 *上半部分（top half）*, 即在进程的内核线程中执行，另一种称之为 *下半部分（bottom half）*，即运行在中断中。上半部分通过 “系统调用（system call）” 触发，譬如调用 `read` 和 `write` 通知设备执行输入输出操作。此代码可能会要求硬件启动某些操作（例如，要求磁盘读取一个 block）；然后代码等待操作完成。最终，设备完成操作并触发中断。此时驱动程序的中断处理程序开始运行，执行下半部分，判断已完成的操作是什么，如果需要的话还要唤醒等待的进程，以及告诉硬件启动等待下一个操作。

## 5.1 代码讲解：控制台输入（Code: Console input）

> The console driver (kernel/console.c) is a simple illustration of driver structure. The console driver accepts characters typed by a human, via the *UART* serial-port hardware attached to the RISC-V. The console driver accumulates a line of input at a time, processing special input characters such as backspace and control-u. User processes, such as the shell, use the `read` system call to fetch lines of input from the console. When you type input to xv6 in QEMU, your keystrokes are delivered to xv6 by way of QEMU’s simulated UART hardware.

“控制台（console）” 驱动程序 (kernel/console.c) 是一个很好的描述驱动程序结构的例子。控制台驱动程序通过连接到 RISC-V 处理器的 *UART* 串行端口设备接收人工输入的字符（译者注：UART 的全称是 Universal Asynchronous Receiver/Transmitter）。控制台驱动程序每次累积一行输入，并处理其中的特殊输入字符，例如 “退格键（backspace）” 和 control-u（译者注：指同时按下 Ctrl 键和字符 u 键）。用户进程（例如 shell）使用 `read` 系统调用从控制台获取输入行。当你在 QEMU 中向 xv6 输入时，你的按键操作将通过 QEMU 模拟的 UART 硬件传递给 xv6。

> The UART hardware that the driver talks to is a 16550 chip [13] emulated by QEMU. On a real computer, a 16550 would manage an RS232 serial link connecting to a terminal or other computer. When running QEMU, it’s connected to your keyboard and display.

驱动程序负责控制的 UART 硬件是 QEMU 模拟的 16550 芯片 [13]。在真实计算机上，16550 芯片会管理连接到终端或其他计算机的 RS232 串行链路。在 QEMU 上，模拟的 16550 芯片直接连接到键盘和显示器。

> The UART hardware appears to software as a set of *memory-mapped* control registers. That is, there are some physical addresses that RISC-V hardware connects to the UART device, so that loads and stores interact with the device hardware rather than RAM. The memory-mapped addresses for the UART start at 0x10000000, or `UART0` (kernel/memlayout.h:21). There are a handful of UART control registers, each the width of a byte. Their offsets from `UART0` are defined in (kernel/uart.c:22). For example, the `LSR` register contains bits that indicate whether input characters are waiting to be read by the software. These characters (if any) are available for reading from the `RHR` register. Each time one is read, the UART hardware deletes it from an internal FIFO of waiting characters, and clears the “ready” bit in `LSR` when the FIFO is empty. The UART transmit hardware is largely independent of the receive hardware; if software writes a byte to the `THR`, the UART transmits that byte.

对于 UART 硬件，从软件的角度来看，就是一组 *内存映射（memory-mapped）* 的控制寄存器。也就是说，RISC-V 处理器通过一些物理地址访问 UART 设备，同样是调用 “加载（load）” 和 “存储（store）” 指令，但此时访问的是其他设备硬件而不是内存。UART 的内存映射的起始物理地址是 0x10000000，代码中对应的是宏常量 `UART0` (kernel/memlayout.h:21)。UART 有少量控制寄存器，每个寄存器的宽度为一个字节。它们相对于 `UART0` 的偏移量值定义在 (kernel/uart.c:22) 中。例如，`LSR` 寄存器中的比特位用于指示输入字符是否正在等待软件读取。我们可以从 `RHR` 寄存器读取字符（如果有的话）。每次读取一个字符时，UART 硬件都会将其从内部缓存字符的 “队列（FIFO，First In First Out 的缩写）” 中删除，并在队列为空时清除 `LSR` 中的 “就绪（ready）” 位。UART 控制器中负责发送的硬件单元很大程度上独立于接收单元；如果软件将一个字节写入寄存器 `THR`，则 UART 发送该字节。

> Xv6’s `main` calls `consoleinit` (kernel/console.c:182) to initialize the UART hardware. This code configures the UART to generate a receive interrupt when the UART receives each byte of input, and a *transmit complete* interrupt each time the UART finishes sending a byte of output (kernel/uart.c:53).

xv6 的 `main` 函数调用 `consoleinit` (kernel/console.c:182) 来初始化 UART 硬件。这段代码配置 UART，使其在接收到每个输入字节时产生一个接收中断，并在每次发送完一个输出字节时产生一个 *传输完成（transmit complete）* 中断 (kernel/uart.c:53)。

> The xv6 shell reads from the console by way of a file descriptor opened by `init.c` (user/init.c:19). Calls to the `read` system call make their way through the kernel to `consoleread` (kernel/console.c:80). `consoleread` waits for input to arrive (via interrupts) and be buffered in `cons.buf`, copies the input to user space, and (after a whole line has arrived) returns to the user process. If the user hasn’t typed a full line yet, any reading processes will wait in the `sleep` call (kernel/console.c:96) (Chapter 7 explains the details of `sleep`).

在 `init.c` (user/init.c:19) 中, xv6 的 shell 程序通过打开一个文件描述符从控制台读取数据。调用系统调用 `read` 在内核中会执行 `consoleread` (kernel/console.c:80)。`consoleread` 等待输入到达（通过中断），并得到缓存在 `cons.buf` 中的输入字符，`consoleread` 在将累积的一整行输入复制到用户空间后结束读取并返回用户空间。如果用户尚未输入整行，则所有读取进程都会阻塞在 `sleep` 调用 (kernel/console.c:96) 中（第 7 章详细介绍了 `sleep`）。

> When the user types a character, the UART hardware asks the RISC-V to raise an interrupt, which activates xv6’s trap handler. The trap handler calls `devintr` (kernel/trap.c:185), which looks at the RISC-V `scause` register to discover that the interrupt is from an external device. Then it asks a hardware unit called the PLIC [3] to tell it which device interrupted (kernel/trap.c:193). If it was the UART, `devintr` calls `uartintr`.

当用户输入字符时，UART 硬件会向 RISC-V 处理器发出中断，从而激活 xv6 的 trap handler。trap handler 会调用 `devintr` (kernel/trap.c:185)，它会查看 RISC-V 的 `scause` 寄存器，以确定中断是否来自外部设备。如果是来自外部设备，它会请求一个名为 PLIC [3] 的硬件单元告知它哪个设备发出了中断 (kernel/trap.c:193)。如果是 UART，`devintr` 则调用 `uartintr`。

> `uartintr` (kernel/uart.c:177) reads any waiting input characters from the UART hardware and hands them to `consoleintr` (kernel/console.c:136); it doesn’t wait for characters, since future input will raise a new interrupt. The job of `consoleintr` is to accumulate input characters in `cons.buf` until a whole line arrives. `consoleintr` treats backspace and a few other characters specially. When a newline arrives, `consoleintr` wakes up a waiting `consoleread` (if there is one).

`uartintr` (kernel/uart.c:177) 从 UART 硬件读取所有等待的输入字符，并将它们传递给 `consoleintr` (kernel/console.c:136)；它不会等待字符，因为后续的输入会引发新的中断。`consoleintr` 的作用是将输入字符累积到 `cons.buf` 中，直到满足一行的条件。`consoleintr` 会对 “退格键（backspace）” 和其他一些字符进行特殊处理。当 “换行符（newline）” 到达时，`consoleintr` 会唤醒正在等待的 `consoleread`（如果有的话）。

> Once woken, `consoleread` will observe a full line in `cons.buf`, copy it to user space, and return (via the system call machinery) to user space.

一旦被唤醒，`consoleread` 将会获取到 `cons.buf` 中的一整行字符并将其复制到用户空间，然后（通过系统调用机制）返回到用户空间。

## 5.2 代码讲解：控制台输出（Code: Console output）

> A `write` system call on a file descriptor connected to the console eventually arrives at `uartputc` (kernel/uart.c:87). The device driver maintains an output buffer (`uart_tx_buf`) so that writing processes do not have to wait for the UART to finish sending; instead, `uartputc` appends each character to the buffer, calls `uartstart` to start the device transmitting (if it isn’t already), and returns. The only situation in which `uartputc` waits is if the buffer is already full.

对连接到控制台的文件描述符执行 `write` 系统调用时，程序执行路径最终会到达 `uartputc` (kernel/uart.c:87)。设备驱动程序维护一个输出缓冲区 (`uart_tx_buf`)，以便写入进程无需等待 UART 发送完成；`uartputc` 会将每个字符追加到缓冲区，调用 `uartstart` 启动设备发送（如果尚未启动），然后返回。唯一会让 `uartputc` 等待的情况是缓冲区已满。

> Each time the UART finishes sending a byte, it generates an interrupt. `uartintr` calls `uartstart`, which checks that the device really has finished sending, and hands the device the next buffered output character. Thus if a process writes multiple bytes to the console, typically the first byte will be sent by `uartputc`’s call to `uartstart`, and the remaining buffered bytes will be sent by `uartstart` calls from `uartintr` as transmit complete interrupts arrive.

UART 每次发送完一个字节后，都会产生一个中断。`uartintr`（中断处理程序）会调用 `uartstart`，后者会检查设备是否确实已完成发送，并将下一个缓冲的输出字符交给设备。因此，如果一个进程向控制台写入多个字节，通常第一个字节会由 `uartputc` 调用 `uartstart` 发送，其余缓冲的字节则会在 “发送完成” 中断到达时由 `uartintr` 调用 `uartstart` 发送。

> A general pattern to note is the decoupling of device activity from process activity via buffering and interrupts. The console driver can process input even when no process is waiting to read it; a subsequent read will see the input. Similarly, processes can send output without having to wait for the device. This decoupling can increase performance by allowing processes to execute concurrently with device I/O, and is particularly important when the device is slow (as with the UART) or needs immediate attention (as with echoing typed characters). This idea is sometimes called *I/O concurrency*.

注意我们这里用到了一个通用设计模式，即通过缓冲和中断将设备活动与进程活动解耦。即使没有进程在等待读取输入，控制台驱动程序也会处理输入（译者注：驱动会将其收到的数据先缓存起来）；后续（进程的）读取操作将直接读取（驱动程序缓存的）输入。同样，进程可以发送输出而无需等待设备完成。这种解耦允许进程与设备 I/O 并发执行，从而提高性能，尤其是在设备速度较慢（例如 UART）或需要立即处理（例如回显输入字符）时尤为重要。这种理念有时被称为 *输入/输出并发(I/O concurrency)*。

## 5.3 驱动中的并发（Concurrency in drivers）

> You may have noticed calls to `acquire` in `consoleread` and in `consoleintr`. These calls acquire a lock, which protects the console driver’s data structures from concurrent access. There are three concurrency dangers here: two processes on different CPUs might call `consoleread` at the same time; the hardware might ask a CPU to deliver a console (really UART) interrupt while that CPU is already executing inside `consoleread`; and the hardware might deliver a console interrupt on a different CPU while `consoleread` is executing. Chapter 6 explains how to use locks to ensure that these dangers don’t lead to incorrect results.

您可能已经注意到在 `consoleread` 和 `consoleintr` 中对 `acquire` 的调用。这些调用会获取一个锁，用于保护控制台驱动程序的数据结构免受并发访问的影响。这里存在三种并发风险：首先，不同 CPU 上的两个进程可能同时调用 `consoleread`；其次，当 `consoleread` 在一个 CPU 上执行时，硬件可能会触发同一个 CPU 上的控制台中断（实际上是由于收到 UART 中断）；再次，`consoleread` 在一个 CPU 上执行的同时在另一个 CPU 上触发控制台中断。第 6 章将解释如何使用锁来确保这些风险不会导致错误的结果。

> Another way in which concurrency requires care in drivers is that one process may be waiting for input from a device, but the interrupt signaling arrival of the input may arrive when a different process (or no process at all) is running. Thus interrupt handlers are not allowed to think about the process or code that they have interrupted. For example, an interrupt handler cannot safely call `copyout` with the current process’s page table. Interrupt handlers typically do relatively little work (e.g., just copy the input data to a buffer), and wake up top-half code to do the rest.

驱动程序中针对并发处理还需要注意的另一个方面是，一个进程可能正在等待来自设备的输入，但输入的中断信号到达时，可能正在运行的是另一个进程（或当前根本没有进程在运行）。因此，中断处理程序不能对被其中断的进程或代码有任何假设。例如，中断处理程序直接基于当前进程的页表来调用 `copyout` 是危险的。中断处理程序通常只执行相对较少的工作（例如，仅将输入数据复制到缓冲区），然后唤醒 top half 代码来完成剩余的工作。

## 5.4 定时器中断（Timer interrupts）

> Xv6 uses timer interrupts to maintain its idea of the current time and to switch among computebound processes. Timer interrupts come from clock hardware attached to each RISC-V CPU. Xv6 programs each CPU’s clock hardware to interrupt the CPU periodically.

xv6 使用定时器中断来保持对当前时间的感知，并在 “计算密集型（computebound）” 进程之间切换（译者注：所谓 “计算密集型” 进程是指那些不轻易主动放弃处理器的进程，譬如执行 `while (1) i++` 这样的指令序列）。定时器中断来自连接到每个 RISC-V CPU 的时钟硬件。xv6 会对每个 CPU 的时钟硬件进行配置，使其定期产生中断。

> Code in `start.c` (kernel/start.c:53) sets some control bits that allow supervisor-mode access to the timer control registers, and then asks for the first timer interrupt. The `time` control register contains a count that the hardware increments at a steady rate; this serves as a notion of the current time. The `stimecmp` register contains a time at which the the CPU will raise a timer interrupt; setting `stimecmp` to the current value of `time` plus *x* will schedule an interrupt *x* time units in the future. For `qemu`’s RISC-V emulation, 1000000 time units is roughly a tenth of second.

`start.c` (kernel/start.c:53) 中的代码（译者注：即 `timerinit` 函数）设置了一些控制位，允许我们在管理员模式下访问定时器控制寄存器，然后启动第一个定时器中断。控制寄存器 `time` 包含一个计数值，硬件会以稳定的频率递增该计数值；我们可以利用它来表示当前时间。`stimecmp` 寄存器可以用于设置 CPU 触发定时器中断的间隔；将 `stimecmp` 设置为 `time` 的当前值加上 *x* 将会在未来 *x* 个时间单位内产生一次中断。对于 `qemu` 仿真的 RISC-V 硬件平台上，1000000 个这样的时间单位大约等于十分之一秒。

> Timer interrupts arrive via `usertrap` or `kerneltrap` and `devintr`, like other device interrupts. Timer interrupts arrive with `scause`’s low bits set to five; `devintr` in `trap.c` detects this situation and calls `clockintr` (kernel/trap.c:164). The latter function increments `ticks`, allowing the kernel to track the passage of time. The increment occurs on only one CPU, to avoid time passing faster if there are multiple CPUs. `clockintr` wakes up any processes waiting in the `sleep` system call, and schedules the next timer interrupt by writing `stimecmp`.

与其他设备中断一样，定时器中断到达时会触发调用 `usertrap` 或 `kerneltrap`，并最终调用 `devintr`。定时器中断到达时， `scause` 的低位值被设置为 5；`trap.c` 中的 `devintr` 会根据这个标记调用 `clockintr` (kernel/trap.c:164)。后者会增加 `ticks`，使内核能够跟踪时间的流逝。对 `ticks` 的递增只会发生在一个 CPU 上（译者注：即 `cpuid() == 0` 的 CPU 上），以避免在多处理器情况下时间流逝过快（译者注：`ticks` 是一个全局变量，如果多个处理器都会修改它自然会加快计时的频率，而且整体频率也不固定）。此外 `clockintr` 还会唤醒在 `sleep` 系统调用中等待的进程，并通过写入 `stimecmp` 来调度下一个定时器中断。

> `devintr` returns 2 for a timer interrupt in order to indicate to `kerneltrap` or `usertrap` that they should call `yield` so that CPUs can be multiplexed among runnable processes.

`devintr` 针对定时器中断的情况其返回值是 2，如果是这种情况，`kerneltrap` 或 `usertrap` 会调用 `yield`，这样 CPU 就可以被其他 “就绪（runnable）” 的进程复用（译者注：`yield` 会迫使 CPU 上的进程主动让出处理器给其他就绪的进程）。

> The fact that kernel code can be interrupted by a timer interrupt that forces a context switch via `yield` is part of the reason why early code in `usertrap` is careful to save state such as `sepc` before enabling interrupts. These context switches also mean that kernel code must be written in the knowledge that it may move from one CPU to another without warning.

内核代码的执行可以被定时器中断打断，并通过 `yield` 强制进行上下文切换，这也是 `usertrap` 在函数开始部分在启用中断之前小心地保存 `sepc` 等状态的原因之一。这些上下文切换也意味着内核代码必须在编写时考虑到它可能会在没有任何感觉的情况下从一个 CPU 被转移到另一个 CPU。

## 5.5 现实世界（Real world）

> Xv6, like many operating systems, allows interrupts and even context switches (via `yield`) while executing in the kernel. The reason for this is to retain quick response times during complex system calls that run for a long time. However, as noted above, allowing interrupts in the kernel is the source of some complexity; as a result, a few operating systems allow interrupts only while executing user code.

与许多操作系统一样，xv6 在内核执行时允许中断，甚至允许上下文切换（通过 `yield`）。这样做的目的是为了在执行一些耗时较长的复杂系统调用时保持快速响应。然而，如上所述，在内核中允许中断会带来一些复杂性；因此，一些操作系统仅在用户态执行代码时才允许中断。

> Supporting all the devices on a typical computer in its full glory is much work, because there are many devices, the devices have many features, and the protocol between device and driver can be complex and poorly documented. In many operating systems, the drivers account for more code than the core kernel.

在典型的计算机上全面支持所有设备是一项艰巨的工作，因为设备数量众多，功能各异，而且设备和驱动程序之间的协议可能非常复杂，文档也很少。在许多操作系统中，驱动程序的代码量甚至超过了内核的核心部分的代码量。

> The UART driver retrieves data a byte at a time by reading the UART control registers; this pattern is called *programmed I/O*, since software is driving the data movement. Programmed I/O is simple, but too slow to be used at high data rates. Devices that need to move lots of data at high speed typically use *direct memory access* (DMA). DMA device hardware directly writes incoming data to RAM, and reads outgoing data from RAM. Modern disk and network devices use DMA. A driver for a DMA device would prepare data in RAM, and then use a single write to a control register to tell the device to process the prepared data.

UART 驱动程序通过读取 UART 控制寄存器，一次获取一个字节的数据；由于数据的搬运由软件驱动，这种模式称为 *程序控制输入输出（programmed I/O）*。程序控制输入输出实现简单简单，但速度太慢，无法支持高速吞吐的需求。需要高速搬运大量数据的设备通常使用 *直接内存访问（direct memory access）* (简称 DMA)。DMA 设备硬件直接将读取到的数据写入 RAM，并从 RAM 将数据取出发送出去。现代磁盘和网络设备都使用 DMA。DMA 设备的驱动程序会将（需要发送的）数据在内存中准备好，然后只要通过对（DMA 设备的）控制寄存器发起一次写操作，通知设备自己去处理准备好的数据就好了。

> Interrupts make sense when a device needs attention at unpredictable times, and not too often. But interrupts have high CPU overhead. Thus high speed devices, such as network and disk controllers, use tricks that reduce the need for interrupts. One trick is to raise a single interrupt for a whole batch of incoming or outgoing requests. Another trick is for the driver to disable interrupts entirely, and to check the device periodically to see if it needs attention. This technique is called *polling*. Polling makes sense if the device performs operations at a high rate, but it wastes CPU time if the device is mostly idle. Some drivers dynamically switch between polling and interrupts depending on the current device load.

对于设备来说，如果其等待的事件的发生是不可预期的，且频率不高时，中断是有意义的。但中断的 CPU 开销很高。因此，高速设备（例如网络和磁盘控制器）会使用一些技巧来减少对中断的需求。一个技巧是为需要批处理的输入或输出只触发一个中断。另一个技巧是驱动程序完全禁用中断，而是定期检查设备的状态。这种技术称为 *轮询（polling）*。如果设备需要高速执行某些操作，轮询是有意义的，但如果设备大部分时间处于空闲状态，轮询会浪费 CPU 时间。某些驱动程序会根据当前设备负载动态地在轮询和中断两种处理方式之间切换。

> The UART driver copies incoming data first to a buffer in the kernel, and then to user space. This makes sense at low data rates, but such a double copy can significantly reduce performance for devices that generate or consume data very quickly. Some operating systems are able to directly move data between user-space buffers and device hardware, often with DMA.

UART 驱动程序会将传入的数据先复制到内核的缓冲区中，然后再复制到用户空间去。这在低速处理下是合理的，但对于产生数据或消费数据速度都非常快的设备来说，这种两次复制会显著降低性能。某些操作系统能够直接在用户空间缓冲区和设备硬件之间搬运数据，这通常也是通过使用 DMA。

> As mentioned in Chapter 1, the console appears to applications as a regular file, and applications read input and write output using the `read` and `write` system calls. Applications may want to control aspects of a device that cannot be expressed through the standard file system calls (e.g., enabling/disabling line buffering in the console driver). Unix operating systems support the `ioctl` system call for such cases.

如第 1 章所述，控制台对应用程序而言就像一个普通文件，应用程序同样可以使用 `read` 和 `write` 这一对系统调用实现读取输入和发送输出。此外，应用程序可能需要对设备发起一些操作但这些操作无法简单地通过类似 `read` 和 `write` 这样的标准文件系统调用的方式来表达（例如，在控制台驱动程序中启用或者禁用行缓冲）。Unix 操作系统支持像 `ioctl` 这样的系统调用来处理这种情况。

> Some usages of computers require that the system must respond in a bounded time. For example, in safety-critical systems missing a deadline can lead to disasters. Xv6 is not suitable for hard real-time settings. Operating systems for hard real-time tend to be libraries that link with the application in a way that allows for an analysis to determine the worst-case response time. Xv6 is also not suitable for soft real-time applications, when missing a deadline occasionally is acceptable, because xv6’s scheduler is too simplistic and it has kernel code path where interrupts are disabled for a long time.

某些计算机在使用中要求系统必须在限定的时间内返回响应。例如，在 “安全关键型（safety-critical）” 系统中，错过截止时间可能会导致严重的灾难。xv6 不适用于这样的 “硬实时（hard real-time）” 场景。硬实时操作系统通常以库的形式与应用程序链接，以便通过分析确定最坏情况下的响应时间。xv6 也不适用于 “软实时（soft real-time）” 应用，所谓 “软实时” 是指可以接受偶尔错过截止时间，但 xv6 的调度程序过于简单，而且其内核代码执行路径中存在长时间禁用中断的地方（译者注：禁用中断，特别是禁用定时器中断会导致任务切换无法发生，考虑到被阻塞的进程如果恰好是需要限定时间内返回结果的进程那么就会达不到实时性要求）。

## 5.6 练习（Exercises）

> 1. Modify `uart.c` to not use interrupts at all. You may need to modify `console.c` as well.

1. 修改 `uart.c` 使其完全不使用中断。您可能还需要修改 `console.c`。

> 2. Add a driver for an Ethernet card.

2. 添加以太网卡的驱动程序。