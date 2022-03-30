# A `fork()` in the road

[原文](https://www.microsoft.com/en-us/research/uploads/prod/2019/04/fork-hotos19.pdf)

## 作者

- **Andrew Baumann** 微软研究院
- **Jonathan Appavoo** 波士顿大学
- **Orran Krieger** 波士顿大学
- **Timothy Roscoe** 苏黎世联邦理工学院

## 版权声明

Permission to make digital or hard copies of all or part of this work for personal or classroom use is granted without fee provided that copies are not made or distributed for profit or commercial advantage and that copies bear this notice and the full citation on the first page. Copyrights for components of this work owned by others than the author(s) must be honored. Abstracting with credit is permitted. To copy otherwise, or republish, to post on servers or to redistribute to lists, requires prior specific permission and/or a fee. Request permissions from permissions@acm.org. HotOS ’19, May 13–15, 2019, Bertinoro, Italy © 2019 Copyright held by the owner/author(s). Publication rights licensed to ACM.

允许为个人或教学使用本作品的全部或部分内容制作电子版或硬拷贝，但不得以营利或商业利益为目的制作或分发拷贝，且拷贝首页须注明本通知和完整的引文。除作者外，本作品中其他部分的版权必须得到尊重。允许摘录并注明出处。以其他方式复制，或重新发表，在服务器上发布或重新分发到列表，需要事先获得具体许可和/或支付费用。请向 permissions@acm.org 申请许可。HotOS '19, May 13-15, 2019, Bertinoro, Italy © 2019 版权由所有者/作者持有。出版权授权给ACM。

ACM ISBN 978-1-4503-6727-1/19/05. . . $15.00

<https://doi.org/10.1145/3317550.3321435>

## 目录

- [摘要](#摘要)
- [引言](#1-引言)
- [历史： `fork` 始于一个技巧](#2-历史：fork-始于一个技巧)
- [`fork` *API* 的优点](#3-fork-api-的优点)
- [在新时代的 `fork`](#4-在新时代的-fork)
- [实现 `fork`](#5-实现-fork)
- [去除 `fork`](#6-去除-fork)
- [`fork` 滚出我的操作系统！](#7-fork-滚出我的操作系统)

## 摘要

The received wisdom suggests that Unix’s unusual combination of fork() and exec() for process creation was an inspired design. In this paper, we argue that fork was a clever hack for machines and programs of the 1970s that has long outlived its usefulness and is now a liability. We catalog the ways in which fork is a terrible abstraction for the modern programmer to use, describe how it compromises OS implementations, and propose alternatives.

普遍接受的观点表明，*Unix* 将 `fork()` 和 `exec()` 不寻常地结合起来创建进程是一个巧妙的设计。在本文中，我们认为 `fork` 对于 20 世纪 70 年代的机器和程序是一种聪明的技巧，但它早已不再有用，现在成了一种负担。我们列举了现代程序员使用 `fork` 的方式来证明 `fork` 已经成为了一种糟糕的抽象，描述了它是如何损害操作系统的实现，并提出了替代方案。

As the designers and implementers of operating systems, we should acknowledge that fork’s continued existence as a first-class OS primitive holds back systems research, and deprecate it. As educators, we should teach fork as a historical artifact, and not the first process creation mechanism students encounter.

作为操作系统的设计者和实现者，我们应该承认 `fork` 作为一流的操作系统原语，其持续存在阻碍了系统研究，而且应该将其废弃。作为教育者，我们应该把 `fork` 作为一个历史文物来教授，而不是作为学生遇到的第一个进程创建机制。

**ACM Reference Format**:

Andrew Baumann, Jonathan Appavoo, Orran Krieger, and Timothy Roscoe. 2019. A fork() in the road. In Workshop on Hot Topics in Operating Systems (HotOS ’19), May 13–15, 2019, Bertinoro, Italy. ACM, New York, NY, USA, 9 pages. <https://doi.org/10.1145/3317550.3321435>

## 1 引言

When the designers of Unix needed a mechanism to create processes, they added a peculiar new system call: fork(). As every undergraduate now learns, fork creates a new process identical to its parent (the caller of fork), with the exception of the system call’s return value. The Unix idiom of fork() followed by exec() to execute a different program in the child is now well understood, but still stands in stark contrast to process creation in systems developed independently of Unix \[e.g., 1, 30, 33, 54\].

当 *Unix* 的设计者需要一个创建进程的机制时，他们增加了一个奇特的新系统调用：`fork()`。现在每个本科生都知道，`fork` 会创建一个与其父进程（`fork` 的调用者）相同的新进程，但系统调用的返回值除外。`fork()` 后跟 `exec()` 以在子进程中执行其他程序的 *Unix* 习语现在已经被很好地理解了，但仍然与独立于 *Unix* 开发的系统中的进程创建形成鲜明对比（例如，[1](#1)、[30](#30)、[33](#33)、[54](#54)）。

50 years later, fork remains the default process creation API on POSIX: Atlidakis et al. \[8\] found 1304 Ubuntu packages (7.2% of the total) calling fork, compared to only 41 uses of the more modern posix_spawn(). Fork is used by almost every Unix shell, major web and database servers (e.g., Apache, PostgreSQL, and Oracle), Google Chrome, the Redis key-value store, and even Node.js. The received wisdom appears to hold that fork is a good design. Every OS textbook we reviewed \[4, 7, 9, 35, 75, 78\] covered fork in uncritical or positive terms, often noting its “simplicity” compared to alternatives. Students today are taught that “the fork system call is one of Unix’s great ideas” \[46\] and “there are lots of ways to design APIs for process creation; however, the combination of fork() and exec() are simple and immensely powerful...the Unix designers simply got it right” [7].

50 年过去了，`fork` 仍然是 *POSIX* 上默认的进程创建 *API*：*Atlidakis* 等人（[8](#8)）发现有 1304 个 *Ubuntu* 软件包（占总数的 7.2%）调用 `fork` ，而只有 41 个使用更现代的 `posix_spawn()`。几乎所有的 *Unix shell*、主要的网络和数据库服务器（如*Apache*、*PostgreSQL* 和 *Oracle*）、谷歌浏览器、*Redis* 键值存储，甚至是 *Node.js* 都在使用 *fork*。普遍接受的观点似乎认为 `fork` 是一个很好的设计。我们浏览的每一本操作系统教科书（[4](#4)、[7](#7)、[9](#9)、[35](#35)、[75](#75)、[78](#78)）都以不加批判或积极的措辞介绍了 `fork` ，经常提到它与其他替代方案相比的“简单性”。今天，学生们被教导说：“ `fork` 系统调用是 *Unix* 的伟大灵感之一”，“创建进程的 *API* 有多种设计；然而，`fork()` 和 `exec()` 的组合是简单而强大的……*Unix* 的设计者们做对了”（[7](#7)）。

Our goal is to set the record straight. Fork is an anachronism: a relic from another era that is out of place in modern systems where it has a pernicious and detrimental impact. As a community, our familiarity with fork can blind us to its faults (§4). Generally acknowledged problems with fork include that it is not thread-safe, it is inefficient and unscalable, and it introduces security concerns. Beyond these limitations, fork has lost its classic simplicity; it today impacts all the other operating system abstractions with which it was once orthogonal. Moreover, a fundamental challenge with fork is that, since it conflates the process and the address space in which it runs, fork is hostile to user-mode implementation of OS functionality, breaking everything from buffered IO to kernel-bypass networking. Perhaps most problematically, fork doesn’t compose—every layer of a system from the kernel to the smallest user-mode library must support it.

我们的目标是要澄清事实。`fork` 是一个不合时宜的东西：一个来自另一个时代、不适应现代系统的遗物，有害无益，影响恶劣。作为一个社区，我们对 `fork` 的熟悉程度会使我们对它的缺点视而不见（[§4](#4-在新时代的分岔)）。`fork` 公认的问题包括：不是线程安全的、低效且不可扩展，而且它引入了安全问题。除了这些限制之外，`fork` 已经失去了其经典的简单性；今天，它影响了曾经与它正交的所有其他操作系统抽象。此外，`fork` 面临的一个重大挑战是，由于混淆了进程和进程所运行的地址空间，它对操作系统功能的用户模式实现是不利的，破坏了从缓冲 IO 到内核旁路网络的一切。也许最麻烦的是，`fork` 不能组合——从内核到最小的用户态库，系统的每一层都必须支持它。

We illustrate the havoc fork wreaks on OS implementations using our experiences with prior research systems (§5). Fork limits the ability of OS researchers and developers to innovate because any new abstraction must be special-cased for it. Systems that support fork and exec efficiently are forced to duplicate per-process state lazily. This encourages the centralisation of state, a major problem for systems not structured using monolithic kernels. On the other hand, research systems that avoid implementing fork are unable to run the enormous body of software that uses it.

我们使用我们在先前研究操作系统获得的经验（[§5](#5-实现-fork)）来说明 `fork` 对操作系统实现的破坏。`fork` 限制了操作系统研究者和开发者的创新能力，因为任何新的抽象都必须为它进行特殊处理。有效支持 `fork` 和 `exec` 的系统被迫懒洋洋地复制每个进程的状态。这鼓励了状态的集中化，对于不使用宏内核结构的系统来说是个大问题。另一方面，避免实现 `fork` 的研发中系统无法运行使用它的大量软件。

We end with a discussion of alternatives (§6) and a call to action (§7): fork should be removed as a first-class primitive of our systems, and replaced with good-enough emulation for legacy applications. It is not enough to add new primitives to the OS—fork must be removed from the kernel.

我们以对替代方案的讨论（[§6](6-去除-fork)）和行动号召（[§7](7-fork-滚出我的操作系统)）结束：应该将 `fork` 从我们系统的一级原语中去除，并替换为对旧式应用程序足够好的兼容性包装。向操作系统添加新的原语是不够的——必须从内核中删除 `fork`。

## 2 历史：`fork` 始于一个技巧

Although the term originates with Conway, the first implementation of a fork operation is widely credited to the Project Genie time-sharing system \[61\]. Ritchie and Thompson \[70\] themselves claimed that Unix fork was present “essentially as we implemented it” in Genie. However, the Genie monitor’s fork call was more flexible than that of Unix: it permitted the parent process to specify the address space and machine context for the new child process \[49, 71\]. By default, the child shared the address space of its parent (somewhat like a modern thread); optionally, the child could be given an entirely different address space of memory blocks to which the user had access; presumably, in order to run a different program. Crucially, however, there was no facility to copy the address space, as was done unconditionally by Unix.

尽管这个术语起源于 *Conway*，但 `fork` 操作的第一个实现被公认地归功于 *Project Genie* 分时系统（[61](#61)）。*Ritchie* 和 *Thompson*（[70](#70)）二人声称，*Unix* 的 *fork* “基本上是按照我们（在 *Genie* 中）实现的方式（实现的）”。然而，*Genie* 监视器的 `fork` 调用比 *Unix* 的更灵活：它允许父进程为新的子进程指定地址空间和机器上下文（[49](#49)、[71](#71)）。默认情况下，子进程共享其父进程的地址空间（有点像现代的线程）；也可以选择给子进程一个完全不同的内存块地址空间，用户可以访问这些内存块；大概是为了运行一个不同的程序。然而，至关重要的是，没有复制地址空间的操作，不像 *Unix* 无条件地那样做。

Ritchie \[69\] later noted that “it seems reasonable to suppose that it exists in Unix mainly because of the ease with which fork could be implemented without changing much else.” He goes on to describe how the first fork was implemented in 27 lines of PDP-7 assembly, and consisted of copying the current process out to swap and keeping the child resident in memory.1 Ritchie also noted that a combined Unix fork-exec “would have been considerably more complicated, if only because exec as such did not exist; its function was already performed, using explicit IO, by the shell.

*Ritchie*（[69](#69)）后来指出，"似乎有理由认为，它之所以存在于 *Unix* 中，主要是因为 `fork` 可以很容易地实现，而不需要改变其他很多东西"。他接着描述了第一个 `fork` 是如何在 27 行 *PDP-7* 汇编中实现的，包括将当前进程复制到交换区，并将子进程留在内存中\[**注**\]。*Ritchie* 还指出，*Unix* `fork`-`exec` 的组合“会复杂得多，要不是因为 `exec` 本身还不存在的话；其功能已经由*shell* 通过显式 IO 完成了。”

> Sharing memory between parent and child (as in Genie) was impractical, because the PDP-7 lacked virtual memory hardware; instead, Unix implemented multiprocessing by swapping full processes to disk.
>
> （实际上）在父子进程之间共享内存（像 *Genie* 那样）是不切实际的，因为 *PDP-7* 缺乏虚拟内存硬件；相反，*Unix* 通过将完整进程交换到磁盘来实现多进程。

The TENEX operating system \[18\] yields a notable counter-example to the Unix approach. It was also influenced by Project Genie, but evolved independently of Unix. Its designers also implemented a fork call for process creation, however, more similarly to Genie, the TENEX fork either shared the address space between parent and child, or else created the child with an empty address space \[19\]. There was no Unix-style copying of the address space, likely because virtual memory hardware was available.

*TENEX* 操作系统（[18](#18)）是 *Unix* 方法的一个鲜明的反例。它也受到了 *Project Genie* 的影响，但独立于 *Unix* 发展。它的设计者也实现了一个用于创建进程的 `fork` 调用，然而，与 *Genie* 更相似的是，*TENEX* 的 `fork` 要么在父子进程之间共享地址空间，要么用一个空的地址空间创建子进程。没有 *Unix* 式的地址空间拷贝，可能是因为硬件虚拟内存已经可用。

> TENEX also supported copy-on-write memory, but this does not appear to have been used by fork \[20\].
>
> *TENEX* 还支持内存的写时复制，但似乎没有用在 `fork`。

Unix fork was not a necessary “inevitability” \[61\]. It was an expedient PDP-7 implementation shortcut that, for 50 years, has pervaded modern OSes and applications.

*Unix* `fork` 并不是一种必要的“必然性”（[61](#61)）。它是一种权宜的 *PDP-7* 实现捷径，50 年来，它已经遍及现代操作系统和应用程序。

## 3 `fork` *API* 的优点

When Unix was rewritten for the PDP-11 (with memory translation hardware permitting multiple processes to remain resident), copying the process’s entire memory only to immediately discard it in exec was already, arguably, inefficient. We suspect that copying fork survived the early years of Unix mainly because programs and memory were small (eight 8 KiB pages on the PDP-11), memory access was fast relative to instruction execution, and it provided a compelling abstraction. There are two main aspects to this:

当为 *PDP-11* 重写 *Unix* 时（内存转换硬件允许多个进程保持驻留），复制进程的整个内存然后在 `exec` 时立即将其丢弃，已经可以说是低效的了。我们怀疑复制的 `fork` 在 *Unix* 的早期幸存下来，主要是因为程序和内存都很小（在 *PDP-11* 上有 8 个 8KB 的页），内存访问相对于指令执行来说是很快的，而且它提供了一个引人瞩目的抽象。这主要体现在两个方面：

Fork was simple. As well as being easy to implement, fork simplified the Unix API. Most obviously, fork needs no arguments, because it provides a simple default for all the state of a new process: inherit it from the parent. In stark contrast, the Windows CreateProcess() API takes explicit parameters specifying every aspect of the child’s kernel state—10 parameters and many optional flags.

**`fork` 够简单**。除了容易实现之外，`fork` 还简化了 *Unix* 的 *API*。最明显的是，`fork` 不需要参数，因为它为新进程的所有状态提供了一个简单的默认值：从父进程继承。与此形成鲜明对比的是，*Windows* 的 `CreateProcess()` *API* 需要明确的参数来指定子进程内核状态的各个方面——10 个参数和许多可选标志。

More significantly, creating a process with fork is orthogonal to starting a new program, and the space between fork and exec serves a useful purpose. Since fork duplicates the parent, the same system calls that permit a process to modify its kernel state can be reused in the child prior to exec: the shell opens, closes, and remaps file descriptors prior to command execution, and programs can reduce permissions or alter the namespace of a child to run it in restricted context.

更重要的是，用 `fork` 创建一个进程与启动一个新的程序是正交的，而且 `fork` 和 `exec` 之间的空隙有一个有用的目的。由于 `fork` 复制了父进程，允许一个进程修改其内核状态的同一个系统调用可以在 `exec` 之前在子进程中反复使用：`shell` 在命令执行之前打开、关闭和重新映射文件描述符，程序可以减少权限或改变子进程的名字空间，以便在受限的上下文中运行它。

Fork eased concurrency. In the days before threads or asynchronous IO, fork without exec provided an effective form of concurrency. In the days before shared libraries, it enabled a simple form of code reuse. A program could initialise, parse its configuration files, and then fork multiple copies of itself that ran either different functions from the same binary or processed different inputs. This design lives on in pre-forking servers; we return to it in §6.

**`fork` 简化了并发**。在没有线程或异步 IO 的时代，`fork` 但不 `exec` 实现了一种有效的并发形式。在动态库出现之前，它提供了一种简单形式的代码重用。一个程序可以初始化，解析其配置文件，然后派生多个自身的副本，这些副本运行来自同一二进制文件的不同功能或处理不同的输入。这种设计存在于预分叉服务器中；我们将在 [§6](#6-去除-fork) 中回到这个问题。

## 4 在新时代的 `fork`

At first glance, fork still seems simple. We argue that this is a deceptive myth, and that fork’s effects cause modern applications more harm than good.

乍一看，`fork` 似乎仍然很简单。我们认为这是一个具有欺骗性的神话，`fork` 的影响对现代应用带来的弊大于利。

Fork is no longer simple. Fork’s semantics have infected the design of each new API that creates process state. The POSIX specification now lists 25 special cases in how the parent’s state is copied to the child \[63\]: file locks, timers, asynchronous IO operations, tracing, etc. In addition, numerous system call flags control fork’s behaviour with respect to memory mappings (Linux madvise() flags MADV_DONTFORK/DOFORK/WIPEONFORK, etc.), file descriptors (O_CLOEXEC, FD_CLOEXEC) and threads (pthread_atfork()). Any non-trivial OS facility must document its behaviour across a fork, and user-mode libraries must be prepared for their state to be forked at any time. The simplicity and orthogonality of fork is now a myth.

**`fork` 不再简单**。`fork` 的语义已经感染了每个创建进程状态的新 *API* 的设计。*POSIX* 规范现在列出了 25 种关于如何将父进程状态复制到子进程的特殊情况（[63](#63)）：文件锁、定时器、异步 IO 操作、跟踪等等。许多系统调用标志控制 `fork` 与内存映射相关的行为（*Linux* `madvise()` 标志 `MADV_DONTFORK`/`DOFORK`/`WIPEONFORK` 等）、文件描述符（`O_CLOEXEC`、`FD_CLOEXEC`）和线程（`pthread_atfork()`）。任何有意义的操作系统设施都必须记录其在跨 `fork` 时的行为，用户态库必须准备好它们的状态在任何时候被 `fork`。`fork` 的简单性和正交性现在就是个神话。

Fork doesn’t compose. Because fork duplicates an entire address space, it is a poor fit for OS abstractions implemented in user-mode. Buffered IO is a classic example: a user must explicitly flush IO prior to fork, lest output be duplicated \[73\].

**`fork` 不可组合**。因为 `fork` 复制了整个地址空间，所以它不适合在用户态实现的操作系统抽象。缓冲 IO 是一个典型的例子：用户必须在 `fork` 之前显式刷新 IO，以免重复输出（[73](#73)）。

Fork isn’t thread-safe. Unix processes today support threads, but a child created by fork has only a single thread (a copy of the calling thread). Unless the parent serialises fork with respect to its other threads, the child address space may end up as an inconsistent snapshot of the parent. A simple but common case is one thread doing memory allocation and holding a heap lock, while another thread forks. Any attempt to allocate memory in the child (and thus acquire the same lock) will immediately deadlock waiting for an unlock operation that will never happen.

**`fork` 不是线程安全的**。如今 *Unix* 进程支持线程，但由 `fork` 创建的子进程只有一个线程（调用线程的副本）。除非父进程对其其他线程进行 `fork` 序列化，否则子进程的地址空间作为父进程的快照，可能与父进程不一致。一个简单但常见的情况是，一个线程在做内存分配并锁定一个堆锁，而另一个线程则 `fork` 了。任何试图在子进程中分配内存的行为（从而请求相同的锁）都会立即死锁，等待永远不会发生的解锁操作。

## 5 实现 `fork`

## 6 去除 `fork`

## 7 `fork` 滚出我的操作系统！

## 参考文献

### 1

### 2

### 3

### 4

### 5

### 6

### 7

### 8

### 9

### 10

### 11

### 12

### 13

### 14

### 15

### 16

### 17

### 18

### 19

### 20

### 21

### 22

### 23

### 24

### 25

### 26

### 27

### 28

### 29

### 30

### 31

### 32

### 33

### 34

### 35

### 36

### 37

### 38

### 39

### 40

### 41

### 42

### 43

### 44

### 45

### 46

### 47

### 48

### 49

### 50

### 51

### 52

### 53

### 54

### 55

### 56

### 57

### 58

### 59

### 60

### 61

### 62

### 63

### 64

### 65

### 66

### 67

### 68

### 69

### 70

### 71

### 72

### 73

### 74

### 75

### 76

### 77

### 78

### 79

### 80

### 81

### 82
