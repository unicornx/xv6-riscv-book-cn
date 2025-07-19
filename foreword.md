# 前言和致谢（Foreword and acknowledgments）

> This is a draft text intended for a class on operating systems. It explains the main concepts of operating systems by studying an example kernel, named xv6. Xv6 is modeled on Dennis Ritchie’s and Ken Thompson’s Unix Version 6 (v6) [17]. Xv6 loosely follows the structure and style of v6, but is implemented in ANSI C [7] for a multi-core RISC-V [15].

这是一篇用于操作系统课程的教材草案。它通过研究一个名为 xv6 的示例内核来解释操作系统的主要概念。xv6 以 Dennis Ritchie 和 Ken Thompson 的 Unix 版本 6 (v6) [17] 为蓝本。xv6 大致沿用了 v6 的结构和风格，但采用 ANSI C [7] 语言实现，适用于 “多核（multi-core）” 的 RISC-V [15]。

> This text should be read along with the source code for xv6, an approach inspired by John Lions’ Commentary on UNIX 6th Edition [11]; the text has hyperlinks to the source code at https://github.com/mit-pdos/xv6-riscv. See https://pdos.csail.mit.edu/6.1810 for additional pointers to on-line resources for v6 and xv6, including several lab assignments using xv6.

本文应结合 xv6 的源代码一起阅读，其灵感源自 John Li ons 的《UNIX 第六版评注》[11]；本文提供了源代码的超链接，网址为 https://github.com/mit-pdos/xv6-riscv。https://pdos.csail.mit.edu/6.1810 上提供了更多 v6 和 xv6 的在线资源，包括一些 xv6 的实验作业。

> We have used this text in 6.828 and 6.1810, the operating system classes at MIT. We thank the faculty, teaching assistants, and students of those classes who have all directly or indirectly contributed to xv6. In particular, we would like to thank Adam Belay, Austin Clements, and Nickolai Zeldovich. Finally, we would like to thank people who emailed us bugs in the text or suggestions for improvements: Abutalib Aghayev, Sebastian Boehm, brandb97, Anton Burtsev, Raphael Carvalho, Tej Chajed,Brendan Davidson, Rasit Eskicioglu, Color Fuzzy, Wojciech Gac, Giuseppe, Tao Guo, Haibo Hao, Naoki Hayama, Chris Henderson, Robert Hilderman, Eden Hochbaum, Wolfgang Keller, Paweł Kraszewski, Henry Laih, Jin Li, Austin Liew, lyazj@github.com, Pavan Maddamsetti, Jacek Masiulaniec, Michael McConville, m3hm00d, miguelgvieira, Mark Morrissey, Muhammed Mourad, Harry Pan, Harry Porter, Siyuan Qian, Zhefeng Qiao, Askar Safin, Salman Shah, Huang Sha, Vikram Shenoy, Adeodato Simó, Ruslan Savchenko, Pawel Szczurko, Warren Toomey, tyfkda, tzerbib, Vanush Vaswani, Xi Wang, and Zou Chang Wei, Sam Whitlock, Qiongsi Wu, LucyShawYang, ykf1114@gmail.com, and Meng Zhou.

我们在麻省理工学院 (MIT) 的操作系统课程 6.828 和 6.1810 中使用了本书。我们感谢这些课程的教师、助教和学生，他们为 xv6 做出了直接或间接的贡献。特别感谢 Adam Belay、Austin Clements 和 Nickolai Zeldovich。最后，我们要感谢通过电子邮件向我们报告文本中的错误以及提出改进建议的人们：Abutalib Aghayev、Sebastian Boehm、brandb97、Anton Burtsev、Raphael Carvalho、Tej Chajed、Brendan Davidson、Rasit Eskicioglu、Color Fuzzy、Wojciech Gac、Giuseppe、Tao Guo、Haibo Hao、Naoki Hayama、Chris Henderson、Robert Hilderman、Eden Hochbaum、Wolfgang Keller、Paweł Kraszewski、Henry Laih、Jin Li、Austin Liew、lyazj@github.com、Pavan Maddamsetti、Jacek Masiulaniec、Michael McConville、m3hm00d、miguelgvieira、Mark Morrissey、Muhammed Mourad、Harry Pan、Harry Porter、Siyuan Qian、Zhefeng Qiao、Askar Safin、Salman Shah、Huang Sha、Vikram Shenoy、 Adeodato Simó、Ruslan Savchenko、Pawel Szczurko、Warren Toomey、tyfkda、tzerbib、Vanush Vaswani、Xi Wang 和 Zou Chang Wei、Sam Whitlock、Qiongsi Wu、LucyShawYang、ykf1114@gmail.com 和 Meng Zhou。

> If you spot errors or have suggestions for improvement, please send email to Frans Kaashoek and Robert Morris (kaashoek,rtm@csail.mit.edu).

如果您发现错误或有改进建议，请给 Frans Kaashoek 和 Robert Morris 发送电子邮件(kaashoek,rtm@csail.mit.edu)。
