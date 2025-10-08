xv6：一个简单的，类 Unix 的教学操作系统（xv6: a simple, Unix-like teaching operating system）
----

Russ Cox Frans Kaashoek Robert Morris

September 2, 2025

- [前言](./foreword.md)
- [第 1 章 操作系统接口](./ch1-os-interfaces.md)
- [第 2 章 操作系统组织](./ch2-os-organization.md)
- [第 3 章 页表](./ch3-pagetables.md)
- [第 4 章 陷阱和系统调用](./ch4-traps.md)
- [第 5 章 缺页异常](./ch5-pagefaults.md)
- [第 6 章 中断和设备驱动](./ch6-interrupts.md)
- [第 7 章 锁](./ch7-locking.md)
- [第 8 章 调度](./ch8-scheduling.md)
- [第 9 章 睡眠与唤醒](./ch9-sleep.md)
- [第 10 章 文件系统](./ch10-filesystem.md)
- [第 11 章 再次讨论并发问题](./ch11-concurrency.md)
- [第 12 章 总结](./ch12-summary.md)
- [参考书目](./bibliography.md)

本仓库是基于 [book-riscv-rev5.pdf][1] 的中文翻译。对应 [xv6-riscv-book 书源码仓库][3] bea3a08 "Issue #57"。本仓库也下载了一份 [pdf](./pdfs/book-riscv-rev5.pdf)。

正文中注明的参考代码行对应的代码文件请参阅 [xv6-src-booklet-rev5.pdf][4]，对应 [xv6-riscv 代码仓库][2] 2ce32bd "PR 383"。本仓库也下载了一份 [pdf](./pdfs/xv6-src-booklet-rev5.pdf)。

本仓库所有内容遵循 [xv6-riscv-book 书源码仓库][3] 相同的 [版权许可](./LICENSE)。同时为方便中国大陆读者访问，本仓库有一个 Gitee 的 mirror 在：<https://gitee.com/unicornx/xv6-riscv-book-cn>。

本仓库的翻译风格追求以意译为主，直译为辅，即更多追求内容的准确和阅读的流畅，一些觉得原文没有说清楚或者感觉有问题的，以译者注的方式补充在原文中。同时为方便大家核对翻译，在译文之外保留了原文英文内容。欢迎大家一起学习，如果发现有问题欢迎给本仓库提 issue 或者联系我：Chen Wang <unicorn_wang@outlook.com>。

最后衷心感谢 MIT 6.1810 课程的所有编写者和教学团队！这门课程精心设计的内容让我们对操作系统有了更深入的理解。同时也感谢他们的无私奉献和开源精神，将这门精彩的课程完整公开给全球的学习者。

[1]:https://pdos.csail.mit.edu/6.828/2025/xv6/book-riscv-rev5.pdf
[2]:https://github.com/mit-pdos/xv6-riscv
[3]:https://github.com/mit-pdos/xv6-riscv-book
[4]:https://pdos.csail.mit.edu/6.828/2025/xv6/xv6-src-booklet-rev5.pdf