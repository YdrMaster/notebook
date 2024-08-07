﻿# 裸机库开发小结——同栈陷入和换栈陷入

裸机程序要处理的最重要的硬件机制就是陷入。高效的裸机程序总是陷入驱动的，换句话说，陷入是控制流切换的主要推动力（主动让出可以替换为协程，避免线程的栈开销）。本文记录最近开发陷入处理库 [fast-trap](https://github.com/YdrMaster/fast-trap) 中的一些结论。

## 陷入产生新线程

一个陷入到来，直接结果就是控制流的转移。对于 riscv 来说，pc 将立即设置为 `xtvec` csr 指示的地址。这个过程对于原来的控制流来说，无论在这个新控制流上做了什么，陷入发生的那一刻实际上已经完成了切换线程（切换上下文和切换控制流）的一半工作。因此，将发生陷入就视为切换线程，按照切换线程来操作是十分自然的。既然切换控制流已经完成，只需要考虑如何切换上下文。被陷入抢占的控制流，其上下文是一定要保存的，问题只在于从哪里恢复上下文。但是陷入处理真的需要恢复上下文吗？其实根本不需要，因为能指导如何处理陷入的全部信息都需要由硬件提供，在陷入处理线程的栈上留下一些东西是没有意义的。所以得到一个自然的结论：不要在已有的线程上处理陷入，要在新线程上处理陷入；或者不要说开启新线程处理陷入，而是陷入引起了新线程的创建。实际上，对于一个支持协程的裸机程序，陷入将是唯一的导致线程创建的机制。

## 同栈陷入与换栈陷入

陷入产生新线程，必须要解释的要素就是线程的控制流和栈。陷入处理的控制流是比较死板的，就是一段手写汇编，保存上下文然后调用高级语言处理函数，基本不太影响设计。而陷入线程的栈的来源值得讨论。

栈，就是一块连续的内存，将提供给高级语言的编译器自动使用。作为内存块，其特殊之处在于必须一次性分配\[1\]，并且具有很长的生命周期——必须比控制流更长。线程的栈开销是比较大的，这也正是无栈协程存在的意义\[2\]。

陷入的处理使用的栈有 2 种设计：同栈陷入和换栈陷入。所谓同栈陷入，就是陷入时完全不移动栈指针，直接使用被抢占的控制流的栈。相应的，换栈陷入就是陷入时将栈指针指到另一个内存块上，避开被抢占控制流的栈。

这两种设计都是实际存在的。linux 及类似的内核通常都是换栈陷入的，甚至有时会直接为每个线程都准备一个陷入栈，一旦陷入直接进去，出来时要么回到被抢占控制流，要么切换到另一个陷入处理线程，线程和陷入处理线程是直观的一一对应的关系。而各种 RTOS 通常是同栈陷入的，发生陷入栈指针不动，直接将上下文压在栈上，然后跳转陷入处理——这个流程和一个正常的函数调用一模一样，自然开销也差不多，所以 RTOS 处理陷入是非常快速的，这也是“实时性”的重要基础。

那么一般的内核能不能采用同栈陷入以获得实时性呢？很遗憾，不行，因为同栈陷入有效有 2 个隐藏的条件：

1. 被抢占线程必须有有效的栈；
2. 硬件必须提供控制流优先级支持，以保证陷入处理比被抢占线程更重要；

RTOS 通常是自动满足这两个条件的。首先 RTOS 都是 LibOS，对于应用程序是库，而不像一般内核那样，与应用程序是跨特权级通过系统调用协作的独立应用程序。既然是一个软件，栈的有效性自然是能保证的，直接操作栈指针时关中断避免陷入就行了。而一般内核无法假设不可信任的应用程序会正确使用栈，甚至恶意软件可以故意挪动栈指针引发内核泄露甚至崩溃。其次，成熟的 RTOS 都是和优先级控制硬件紧耦合的，这是软硬件协同进化的结果，硬件优先级就是 RTOS 调度的一部分，这是无法软件模拟的，因为软件无法在常数复杂度\[3\]下完成优先级决策，没有硬件必定破坏硬实时性。

条件 #1 可以通过内核使用一个栈，每个加载的应用程序线程使用自己的栈\[4\]来规避。但条件 #2 会直接杀死内核使用同栈陷入的可行性。因为同栈陷入意味着陷入创建的线程不仅抢占了原控制流的计算资源，也抢占了原控制流的栈。还用应用程序的线程做类比，换栈陷入相当于在原控制流上启动线程并立即 `detach`，而同栈陷入相当于立即 `join`，于是原控制流必须等待陷入处理完成，并完全释放占用的栈，栈指针归位后才能继续执行。然而，陷入和原控制流基本上是完全无关的，如果没有硬件优先级的支持则必定产生优先级倒挂，重要任务不得不等待次要任务先执行。

根据以上分析，除非配合完整的优先级控制机制，同栈陷入是不可用的。反过来说，一旦有了硬件优先级机制，就必须使用同栈陷入，因为换栈陷入在每次陷入发生前必须准备空栈，导致嵌套陷入延迟增大；在 NVIC（Arm/CORTEX）/CLIC（RISC-V）下陷入-压栈-开中断的标准陷入流程是常数周期的，不需要高级语言甚至几乎不可能缓存不命中，比换栈陷入不知高到哪里去了。

## 结论

1. RTOS 和一般内核不能共享一个陷入处理库；
2. 一般内核的陷入处理库应该默认换栈陷入，并且陷入栈必须具有 `'static` 生命周期，如同一个 *detached* 线程；

---

> 1. 虚地址上的可扩容栈另当别论，即使是可扩容栈，扩容处理流程本身也是陷入处理，需要在真正的连续内存上执行。
> 2. 裸机程序根本就没有应用程序的所谓线程，线程就是应用程序的有栈协程，所以在裸机开发的讨论中，“协程”这个词是无歧义的，只有一种形式——无栈协程。
> 3. 优先级决策算法对于优先级总数 N 不可能常数复杂度，但优先级总数 N 本身其实是常数，所以即使真的没有硬件优先级的硬件上也可以实现 RTOS，比使用硬件优先级的延迟大，但总的来说还是硬实时的。
> 4. 应用程序的线程一定要使用栈吗？不排除一个手写汇编或经过特殊设计的的应用程序根本不使用栈，但根据惯例内核还是要为它准备一个。
