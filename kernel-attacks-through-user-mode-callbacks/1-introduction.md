# 1. 简介

在 Windows NT 里，Win32 环境子系统允许应用程序与
Windows 操作系统以及像窗口管理器（USER）、图形设备接口（GDI）
这样的组件交互。该子系统提供了一系列函数，
统称为 Win32 API，并且遵循客户端 - 服务端模式，
其中，作为客户端的应用程序与具有更高特权的作为服务端的组件交互。

一直以来，Win32 子系统服务端的实现位于客户端 - 服务端运行时子系统
（CSRSS）。为了提供最优的性能，每一个客户端的线程在 Win32
服务端都有一个与之对应的线程，并在一种称为快速本地过程调用
（Fast LPC）的特殊的进程间通信机制中等待。由于在 Fast LPC
中对应线程间的切换不需要内核的调度事件，
所以服务端线程在抢先式线程调度器中获得机会之前，
可以在客户端线程的剩余时间切片里运行。另外，
为了尽可能将客户端和 Win32 服务端的切换降至最少，
共享内存用于大量数据的传输，
还用于提供客户端对服务端管理的数据结构的只读访问。

不管对传统 Win32 子系统所做的性能优化，
微软决定在 Windows NT 4.0 的发行将很大一部分的服务端组件迁移到内核模式。
这便迎来了 Win32k.sys，用于管理窗口管理器
（USER）和图形设备接口（GDI）的内核模式驱动程序。
迁移到内核模式，通过更少的线程与情境切换
（此外还使用更快的用户模式内核模式切换）和减少内存使用，
大大的降低了原有子系统设计的消耗。
不过，考虑到用户模式与内核模式的切换相比于相同特权级下直接的代码
与数据访问还是比较慢，一些原来的技巧，
像缓存管理结构到用户模式部分的客户端地址空间，
依旧维护了下来。另外，为了避免切换特权级，一些管理结构只存储在用户模式。
由于 Win32k 需要访问这些信息并且支持基本的像窗口钩子的功能，
所以需要一种传递控制给用户模式客户端的方式。这便是采用用户模式回调机制实现的。

用户模式回调函数使得 Win32k 能够调用回用户模式并执行许多任务，
像调用应用程序定义的钩子，提供事件通知和实现用户模式与内核的数据交换。
在这篇文章中，我们谈谈 Wink32k 里的用户模式回调函数所带来的挑战与安全隐患。
特别的，我们会阐明，Win32k 在实现线程安全时对全局锁的依赖
并没有很好的与用户模式回调的理念融为一体。最近，
为了处理与用户模式回调函数有关的多种漏洞类型，
MS11-034 与 MS11-054 修复了若干漏洞。
但是，由于一些问题的复杂特性与用户模式回调函数的普遍性，
Win32k 里很可能还存在更为细微的漏洞。因此，为了缓和一些更普遍的漏洞类型，
我们结论性地讨论了一些点子，微软与用户应该怎样做才能更进一步地缓和将来可能出现在
Win32k 子系统里的攻击。

本文剩余部分组织如下：在第 2 节里，我们回顾一些理解本文所必要的背景知识，
重点关注用户对象和用户模式回调函数。在第 3 节，
我们讨论 Win32k 里函数名的修饰并特定于 Win32k
展示几种漏洞类型与用户模式回调函数。在第 4 节，
我们评估用户模式回调函数触发的漏洞的可利用性，
而在第 5 节为了应对这些攻击，我们尝试给出对普遍存在的漏洞类型的缓和方案。
最后，在第 6 节我们提出对 Win32k 前景的看法和建议，
并在第 7 节中给出本文的结论。。
