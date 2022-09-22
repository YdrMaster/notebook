# 忒修斯：操作系统结构和状态管理的一场实验

> Theseus: an Experiment in Operating System Structure and State Management

## 摘要

> Abstract

本文介绍了一个名为忒修斯的操作系统（OS）。忒修斯是多年实验的结果，它通过减少一个组件对另一个组件持有的状态来重新设计和改进操作系统的模块化，并利用一种安全的编程语言，即 Rust，将尽可能多的操作系统责任转移给编译器。忒修斯体现了两个主要进展。首先，一个操作系统结构，其中许多微小的组件具有明确定义的、运行时持久的边界，它们之间的互动不需要彼此持有状态。第二，一种使用语言级机制实现操作系统本身的内部语言方法，这样编译器就可以执行关于操作系统语义的不变性。忒修斯的结构、内部语言设计和状态管理以超越现有作品的方式实现了核心操作系统组件的实时演进和故障恢复。

> This paper describes an operating system (OS) called Theseus. Theseus is the result of multi-year experimentation to redesign and improve OS modularity by reducing the states one component holds for another, and to leverage a safe programming language, namely Rust, to shift as many OS responsibilities as possible to the compiler. Theseus embodies two primary contributions. First, an OS structure in which many tiny components with clearly-defined, runtime-persistent bounds interact without holding states for each other. Second, an intralingual approach that realizes the OS itself using language-level mechanisms such that the compiler can enforce invariants about OS semantics. Theseus’s structure, intralingual design, and state management realize live evolution and fault recovery for core OS components in ways beyond that of existing works.

## 1 引言

> 1 Introduction

我们报告了一项关于操作系统结构设计、状态管理以及利用现代安全系统编程语言（即 Rust）的力量实现操作系统的实验。这一努力最初是由对状态溢出\[16\]的研究所激发的：一个软件组件由于处理了来自另一个组件的交互，其状态发生了变化，从而使其未来的正确性取决于上述状态。在现代系统软件中，状态溢出导致了原本模块化和隔离的组件之间的命运共享，从而阻碍了理想的计算目标的实现，如可进化性和可用性。例如，安卓系统服务中的状态溢出导致整个用户空间框架在系统服务故障时崩溃，失去所有应用程序的状态和进度，甚至那些没有使用故障服务的应用程序\[16\]。可靠的微内核进一步证明，对溢出到操作系统服务中的状态的管理是容错\[21\]和实时更新\[28\]的障碍。

> We report an experimentation of OS structural design, state management, and implementation techniques that leverage the power of modern safe systems programming languages, namely Rust. This endeavor was initially motivated by studies of state spill \[16\]: one software component harboring changed states as a result of handling an interaction from another component, such that their future correctness depends on said states. Prevalent in modern systems software, state spill leads to fate sharing between otherwise modularized and isolated components and thus hinders the realization of desirable computing goals such as evolvability and availability. For example, state spill in Android system services causes the entire userspace frameworks to crash upon a system service failure, losing the states and progress of all applications, even those not using the failed service \[16\]. Reliable microkernels further attest that management of states spilled into OS services is a barrier to fault tolerance \[21\] and live update \[28\].

系统软件的可演化性和可用性在可靠性是必要的但硬件冗余昂贵或不可能的环境中至关重要。例如，在心脏起搏器\[26\]和太空探测器\[25,62\]中，系统软件的更新必须在不停机或不丢失执行环境的情况下艰难地应用。即使在数据中心，多个网络交换机互为备份以保证可靠性，但交换机软件故障和维护更新仍然会导致网络中断\[27,48\]。

> Evolvability and availability of systems software are crucial in environments where reliability is necessary yet hardware redundancy is expensive or impossible. For example, systems software updates must be painstakingly applied without downtime or lost execution context in pacemakers \[26\] and space probes \[25, 62\]. Even in datacenters, where network switches are replicated for reliability, switch software failures and maintenance updates still lead to network outages \[27, 48\].

为了确定在操作系统代码中可以在多大程度上避免状态溢出，我们选择从头开始编写一个操作系统。我们被 Rust 所吸引，因为它的所有权模型为实现操作系统组件之间的隔离和零成本的状态转移提供了一个方便的机制。我们最初的操作系统构建经验导致了两个重要的认识。首先，缓解状态溢出或更好的状态管理一般来说需要重新思考操作系统的结构，因为状态溢出（根据定义）取决于操作系统的模块化方式。其次，像 Rust 这样的现代系统编程语言不仅可以用来编写安全的操作系统代码，还可以静态地确保操作系统行为的某些正确性不变性。

> On the quest to determine to what extent state spill can be avoided in OS code, we chose to write an OS from scratch. We were drawn to Rust because its ownership model provides a convenient mechanism for implementing isolation and zero-cost state transfer between OS components. Our initial OS-building experience led to two important realizations. First, mitigating state spill, or better state management in general, necessitates a rethinking of OS structure because state spill (by definition) depends on how the OS is modularized. Second, modern systems programming languages like Rust can be used not just to write safe OS code but also to statically ensure certain correctness invariants for OS behaviors.

我们实验的结果是忒修斯操作系统，它对系统软件的设计和实现有两个贡献。首先，忒修斯有一个新颖的操作系统结构，由许多微小的组件组成，具有明确定义的、运行时间持久的界限。该系统维护元数据，并跟踪组件之间的相互依赖关系，这有利于这些组件的实时进化和故障恢复（第 3 节）。

> The outcome of our experimentation is Theseus OS, which makes two contributions to systems software design and implementation. First, Theseus has a novel OS structure of many tiny components with clearly-defined, runtime-persistent bounds. The system maintains metadata about and tracks interdependencies between components, which facilitates live evolution and fault recovery of these components (§3).

另外，更重要的是，忒修斯贡献了内部语言操作系统的设计方法，这需要将操作系统的执行环境与其实现语言的运行时模型相匹配，并使用语言级机制实现操作系统本身。通过语言内设计，忒修斯使编译器能够将其安全检查应用于操作系统的代码，而对代码行为的理解没有任何差距，并将语义错误从运行时故障转移到编译时错误，两者都比现有操作系统的程度更高。内部语言设计超越了安全性，使编译器能够静态地检查操作系统的语义不变性，并承担资源管理的职责。这将在第 4 节中详述。

> Second, and more importantly, Theseus contributes the intralingual OS design approach, which entails matching the OS’s execution environment to the runtime model of its implementation language and implementing the OS itself using language-level mechanisms. Through intralingual design, Theseus empowers the compiler to apply its safety checks to OS code with no gaps in its understanding of code behavior, and shifts semantic errors from runtime failures into compile-time errors, both to a greater degree than existing OSes. Intralingual design goes beyond safety, enabling the compiler to statically check OS semantic invariants and assume resource bookkeeping duties. This is elaborated in §4.

忒修斯的结构和内部语言设计自然地减少了操作系统必须维护的状态，减少了其组件之间的状态溢出。我们在第 5 节中描述了忒修斯的状态管理技术，以进一步减轻状态溢出的影响。

> Theseus’s structure and intralingual design naturally reduce states the OS must maintain, reducing state spill between its components. We describe Theseus’s state management techniques to further mitigate the effects of state spill in §5.

为了证明忒修斯设计的实用性，我们在其中实现了实时演进和故障恢复（用于可用性）（§6）。有了这些，我们认为忒修斯非常适用于高端嵌入式系统和数据中心组件，在这些地方需要在没有硬件冗余的情况下或在硬件冗余之外有可用性。在这里，忒修斯作为一个新的操作系统和需要安全语言程序的限制影响较小，因为应用程序可以在一个操作者控制的环境中与操作系统共同开发。

> To demonstrate the utility of Theseus’s design, we implement live evolution and fault recovery (for availability) within it (§6). With this, we posit that Theseus is well-suited for highend embedded systems and datacenter components, where availability is needed in the absence of or in addition to hardware redundancy. Therein, Theseus’s limitations of being a new OS and needing safe-language programs have a lesser impact, as applications can be co-developed with the OS in an environment under a single operator’s control.

我们在第 7 节中评估了忒修斯实现这些目标的情况。通过一组案例研究，我们展示了忒修斯可以轻松、任意地对核心系统组件进行实时演进，其方式超出了先前的实时更新工作，例如，应用程序-内核联合演进，或微内核级组件的演进。由于忒修斯可以优雅地处理语言级故障（Rust 中的 panics），我们展示了忒修斯容忍在操作系统内核中出现的更具挑战性的瞬时硬件故障的能力。为此，我们对忒修斯中的故障表现和恢复进行了研究，并与 MINIX 3 对必然存在于微内核中的组件的故障恢复进行了比较。虽然性能不是忒修斯的主要目标，但我们发现其内部语言和无溢出设计并没有带来明显的性能损失，但其影响在不同的子系统中有所不同。

> We evaluate how well Theseus achieves these goals in §7. Through a set of case studies, we show that Theseus can easily and arbitrarily live evolve core system components in ways beyond prior live update works, e.g., joint application-kernel evolution, or evolution of microkernel-level components. As Theseus can gracefully handle language-level faults (panics in Rust), we demonstrate Theseus’s ability to tolerate more challenging transient hardware faults that manifest in the OS core. To this end, we present a study of fault manifestation and recovery in Theseus and a comparison with MINIX 3 of fault recovery for components that necessarily exist inside the microkernel. Although performance is not a primary goal of Theseus, we find that its intralingual and spill-free designs do not impose a glaring performance penalty, but that the impact varies across subsystems.

忒修斯目前在 x86_64 上实现，支持大多数硬件特性，如多核处理、抢占式多任务、SIMD 扩展、基本网络和磁盘 I/O 以及图形显示。它代表了大约 4 个人年的努力，包括大约 38000 行从头开始的 Rust 代码，900 行引导汇编代码，246 个板块，其中 176 个是第一方，以及 21 个板块的 72 个不安全代码块或语句，其中大部分是用于端口 I/O 或特殊寄存器访问。

> Theseus is currently implemented on x86_64 with support for most hardware features, such as multicore processing, preemptive multitasking, SIMD extensions, basic networking and disk I/O, and graphical displays. It represents roughly four person-years of effort and comprises ~38000 lines of from-scratch Rust code, 900 lines of bootstrap assembly code, 246 crates of which 176 are first-party, and 72 unsafe code blocks or statements across 21 crates, most of which are for port I/O or special register access.

然而，忒修斯远没有商业系统或实验性系统（如 Singularity \[33\] 和 Barrelfish \[8\]）那样完整，后者经历了大量的开发。例如，忒修斯目前缺乏 POSIX 支持和完整的标准库。因此，我们不对某些操作系统方面提出主张，例如效率或安全性；本文着重于忒修斯的结构和语言内设计，以及随之而来的对热更新和故障恢复的好处。

> However, Theseus is far less complete than commercial systems, or experimental ones such as Singularity \[33\] and Barrelfish \[8\] that have undergone substantially more development. For example, Theseus currently lacks POSIX support and a full standard library. Thus, we do not make claims about certain OS aspects, e.g., efficiency or security; this paper focuses on Theseus’s structure and intralingual design and the ensuing benefits for live evolution and fault recovery.

忒修斯的代码和文档是开源的\[61\]。

> Theseus’s code and documentation are open-source \[61\].

## 2 Rust 语言背景

> 2 Rust Language Background

Rust 编程语言\[40\]的设计是为了在编译时提供强大的类型和内存安全保证，将高级管理语言的力量和表现力与没有垃圾收集或底层运行时的 C 语言的效率结合起来。忒修斯利用许多 Rust 特性来实现内部语言、安全的操作系统设计，并采用板块（crate），即 Rust 的项目容器和翻译单元，以实现源级模块化。板块包含源代码和依赖清单。忒修斯不使用 Rust 的标准库，但使用其 core 库和 alloc 库。

> The Rust programming language [40] is designed to provide strong type and memory safety guarantees at compile time, combining the power and expressiveness of a high-level managed language with the C-like efficiency of no garbage collection or underlying runtime. Theseus leverages many Rust features to realize an intralingual, safe OS design and employs the crate, Rust’s project container and translation unit, for source-level modularity. A crate contains source code and a dependency manifest. Theseus does not use Rust’s standard library but does use its fundamental core and alloc libraries.

Rust 的所有权模型是其编译时内存安全和管理的关键。所有权是基于仿射类型的，其中一个值最多可以使用一次。在 Rust 中，每个值都有一个所有者，例如，下面 L4 中分配的字符串值 "hello!" 是属于 hello 变量的。在一个值被移动后，例如，如果 "hello!" 在 L5 中被从 hello 移动到 owned_string（L14），它的所有权将被转移，之前的所有者（hello）不能再使用它。

> Rust’s ownership model is the key to its compile-time memory safety and management. Ownership is based on affine types, in which a value can be used at most once. In Rust, every value has an owner, e.g., the string value "hello!" allocated in L4 below is owned by the hello variable. After a value is moved, e.g., if "hello!" was moved in L5 from hello to owned_string (L14), its ownership would be transferred and the previous owner (hello) could no longer use it.
