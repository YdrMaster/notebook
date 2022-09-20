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

忒修斯的结构和内部语言设计自然地减少了操作系统必须维护的状态，减少了其组件之间的状态溢出。我们在第 5 节中描述了忒修斯的状态管理技术，以进一步减轻状态溢出的影响。为了证明忒修斯设计的实用性，我们在其中实现了实时演进和故障恢复（为了可用性）（§6）。有了这些，我们认为忒修斯非常适用于高端嵌入式系统和数据中心组件，在没有硬件冗余的情况下，或者在硬件冗余之外，需要可用性。在这里，忒修斯作为一个新的操作系统和需要安全语言程序的限制影响较小，因为应用程序可以在一个操作者控制的环境中与操作系统共同开发。

> Theseus’s structure and intralingual design naturally reduce states the OS must maintain, reducing state spill between its components. We describe Theseus’s state management techniques to further mitigate the effects of state spill in §5. To demonstrate the utility of Theseus’s design, we implement live evolution and fault recovery (for availability) within it (§6). With this, we posit that Theseus is well-suited for highend embedded systems and datacenter components, where availability is needed in the absence of or in addition to hardware redundancy. Therein, Theseus’s limitations of being a new OS and needing safe-language programs have a lesser impact, as applications can be co-developed with the OS in an environment under a single operator’s control.
