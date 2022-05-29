# 第一章 简介

> CHAPTER ONE INTRODUCTION

## 1.1 目的和范围

> 1.1 Purpose and Scope

为了初始化和引导计算机系统，各种软件组件相互作用。固件可能在将控制权移交给操作系统、引导程序或管理程序等软件之前，对系统硬件进行低级别的初始化。bootloader 和 hypervisor 可以反过来加载并转移控制权给操作系统。标准、一致的接口和约定促进了这些软件组件之间的交互。在本文中，引导程序这个词被用来泛指一个软件组件，它初始化系统状态并执行另一个被称为客户程序的软件组件。引导程序例如：固件、bootloader 和 hypervisor。客户程序例如：bootloader、hypervisor、操作系统和专用程序。一个软件可以既是客户程序又是引导程序（例如 hypervisor）。

> To initialize and boot a computer system, various software components interact. Firmware might perform low-level initialization of the system hardware before passing control to software such as an operating system, bootloader, or hypervisor. Bootloaders and hypervisors can, in turn, load and transfer control to operating systems. Standard, consistent interfaces and conventions facilitate the interactions between these software components. In this document the term boot program is used to generically refer to a software component that initializes the system state and executes another software component referred to as a client program. Examples of a boot program include: firmware, bootloaders, and hypervisors. Examples of a client program include: bootloaders, hypervisors, operating systems, and special purpose programs. A piece of software may be both a client program and a boot program (e.g. a hypervisor).

本规范，即设备树规范（DTSpec），提供了一个完整的引导程序到客户程序接口的定义，结合最低系统要求，促进了各种系统的开发。

> This specification, the Devicetree Specification (DTSpec), provides a complete boot program to client program interface definition, combined with minimum system requirements that facilitate the development of a wide variety of systems.

本规范针对的是嵌入式系统的要求。一个嵌入式系统通常由系统硬件、操作系统和应用软件组成，它们是为执行一套固定的、特定的任务而定制的。这与通用计算机不同，后者旨在由用户使用各种软件和 I/O 设备进行定制。嵌入式系统的其他特征可能包括：

> This specification is targeted towards the requirements of embedded systems. An embedded system typically consists of system hardware, an operating system, and application software that are custom designed to perform a fixed, specific set of tasks. This is unlike general purpose computers, which are designed to be customized by a user with a variety of software and I/O devices. Other characteristics of embedded systems may include:

- 一套固定的I/O设备，可能为应用高度定制
- 针对尺寸和成本进行优化的系统板
- 有限的用户接口
- 资源限制，如有限的内存和有限的非易失性存储
- 实时性要求
- 使用各种各样的操作系统，包括 Linux、实时操作系统和定制或专有操作系统

> - a fixed set of I/O devices, possibly highly customized for the application
> - a system board optimized for size and cost
> - limited user interface
> - resource constraints like limited memory and limited nonvolatile storage
> - real-time constraints
> - use of a wide variety of operating systems, including Linux, real-time operating systems, and custom or proprietary operating systems

### 本文的结构

> Organization of this Document

> - 第一章介绍了 DTSpec 所指定的架构。
> - 第二章介绍了设备树的概念，并描述了其逻辑结构和标准属性。
> - 第三章规定了符合 DTSpec 的设备树所需的基本设备节点集的定义。
> - 第四章描述了某些类别的设备和特定设备类型的设备绑定。
> - 第五章规定了设备树的 DTB 编码。
> - 第六章规定了 DTS 源码语言。

> - Chapter 1 introduces the architecture being specified by DTSpec.
> - Chapter 2 introduces the devicetree concept and describes its logical structure and standard properties.
> - Chapter 3 specifies the definition of a base set of device nodes required by DTSpec-compliant devicetrees.
> - Chapter 4 describes device bindings for certain classes of devices and specific device types.
> - Chapter 5 specifies the DTB encoding of devicetrees.
> - Chapter 6 specifies the DTS source language.

### 本文档中使用的约定

> Conventions Used in this Document

**应当**用于表示为符合标准而必须严格遵守的强制性要求，不允许有任何偏差（同义词是**必须**）。

> The word *shall* is used to indicate mandatory requirements strictly to be followed in order to conform to the standard and from which no deviation is permitted (*shall* equals *is required to*).

**最好**用来表示在几种可能性中，有一种被推荐为特别合适，而不提及或排除其他可能性；或者表示某种行动方案是首选，但不一定是必须的；或者表示（在否定形式中）某种方案是不提倡的，但不禁止（同义词是**建议**）。

> The word *should* is used to indicate that among several possibilities one is recommended as particularly suitable, without mentioning or excluding others; or that a certain course of action is preferred but not necessarily required; or that (in the negative form) a certain course of action is deprecated but not prohibited (*should* equals *is recommended that*).

**可能**用于表示在标准范围内允许的操作（同义词是**允许**）。

> The word *may* is used to indicate a course of action permissible within the limits of the standard (*may* equals *is permitted*).

设备树结构的示例经常以设备树语法形式出现。有关此语法的概述参见第六章。

> Examples of devicetree constructs are frequently shown in Devicetree Syntax form. See Section 6 for an overview of this syntax.

## 与 IEEE™ 1275 和 ePAPR 的关系

> Relationship to IEEE™ 1275 and ePAPR

DTSpec 与 `IEEE 1275` 开放固件标准——IEEE 引导（初始化配置）固件标准：核心需求与实现——有松散的联系。

> DTSpec is loosely related to the IEEE 1275 Open Firmware standard—*IEEE Standard for Boot (Initialization Configuration) Firmware: Core Requirements and Practices* `IEEE1275`.

> **IEEE1275**：**引导（初始化配置）固件：核心需求与实现**，1994，这是定义 DTSpec 和 ePAPR 所采用的设备树概念的核心标准（也被称为IEEE 1275）。它可以从 [Global Engineering](http://global.ihs.com) 获得。

> > - `IEEE1275`: *Boot (Initialization Configuration) Firmware: Core Requirements and Practices*, 1994, This is the core standard (also known as IEEE 1275) that defines the devicetree concept adopted by the DTSpec and ePAPR. It is available from Global Engineering (<http://global.ihs.com/>).

最初的 IEEE 1275 规范及其衍生产品，如 `CHRP` 和 `PAPR`，解决了通用计算机的问题，例如一个单一版本的操作系统如何在同一系列的几个不同的计算机上工作，以及从用户安装的 I/O 设备中加载操作系统的问题。

> The original IEEE 1275 specification and its derivatives such as `CHRP` and `PAPR` address problems of general purpose computers, such as how a single version of an operating system can work on several different computers within the same family and the problem of loading an operating system from user-installed I/O devices.

> - `CHRP`: **`PowerPC` 微处理器通用硬件参考平台绑定**，1.8 版，开放固件工作组，1998（<http://devicetree.org/open-firmware/bindings/chrp/chrp1_8a.ps>）。这个文档规定了 Open PIC 兼容中断控制器的属性。
> - `PAPR`: **描述 *Power* 架构平台要求的 *Power.org* 标准**, power.org。

> > - `CHRP`: *PowerPC Microprocessor Common Hardware Reference Platform (CHRP) Binding*, Version 1.8, Open Firmware Working Group, 1998 (<http://devicetree.org/open-firmware/bindings/chrp/chrp1_8a.ps>). This document specifies the properties for Open PIC-compatible interrupt controllers.
> > - `PAPR`: *Power.org Standard for Power Architecture Platform Requirements*, power.org。

由于嵌入式系统的性质，开放的通用计算机所面临的一些问题并不适用。DTSpec 省略了 IEEE 1275 规范中值得注意的特征，包括：

> Because of the nature of embedded systems, some of these problems faced by open, general purpose computers do not apply. Notable features of the IEEE 1275 specification that are omitted from the DTSpec include:

- 即插即用设备驱动
- FCode
- 基于 Forth 的可编程开放固件用户界面
- FCode 调试
- 操作系统调试

> - Plug-in device drivers
> - FCode
> - The programmable Open Firmware user interface based on Forth
> - FCode debugging
> - Operating system debugging

来自 IEEE 1275 的设备树架构的概念被保留下来，引导程序可以通过该架构描述系统硬件信息并将其传达给客户程序，使得客户程序不必硬编码系统硬件的描述。

> What is retained from IEEE 1275 are concepts from the devicetree architecture by which a boot program can describe and communicate system hardware information to a client program, thus eliminating the need for the client program to have hard-coded descriptions of system hardware.

本规范部分取代了 `ePAPR` 规范。 ePAPR 记录了 Power ISA 如何使用设备树，涵盖了一般概念以及 Power ISA 特定的绑定。本文档的文本源自 ePAPR，但要么删除了特定于体系结构的绑定，要么将它们移动到附录中。

> This specification partially supersedes the `ePAPR` specification. ePAPR documents how devicetree is used by the Power ISA, and covers both general concepts, as well as Power ISA specific bindings. The text of this document was derived from ePAPR, but either removes architecture specific bindings, or moves them into an appendix.

> - `ePAPR`：**描述嵌入式 *Power* 架构平台要求的 *Power.org* 标准**, power.org, 2011, <https://www.power.org/documentation/power-org-standard-for-embedded-power-architecture-platform-requirements-epapr-v1-1-2/>

> > - `ePAPR`: *Power.org Standard for Embedded Power Architecture Platform Requirements*, power.org, 2011, <https://www.power.org/documentation/power-org-standard-for-embedded-power-architecture-platform-requirements-epapr-v1-1-2/>

## 1.3 32 位和 64 位支持

> 1.3 32-bit and 64-bit Support

DTSpec 支持具有 32 位和 64 位寻址能力的 CPU。如果需要，DTSpec 分别描述 32 位和 64 位寻址的要求或注意事项。

> The DTSpec supports CPUs with both 32-bit and 64-bit addressing capabilities. Where applicable, sections of the DTSpec describe any requirements or considerations for 32-bit and 64-bit addressing.

## 1.4 术语定义

> 1.4 Definition of Terms

**AMP** 非对称多处理器。计算机可用的 CPU 被分成若干组，每组运行一个不同的操作系统镜像。这些 CPU 可能相同，也可能不相同。

> **AMP** Asymmetric Multiprocessing. Computer available CPUs are partitioned into groups, each running a distinct operating system image. The CPUs may or may not be identical.

**引导 CPU** 引导程序引导到客户程序入口的第一个 CPU。

> **boot CPU** The first CPU which a boot program directs to a client program’s entry point.

**Book III-E** 嵌入式环境。Power ISA 的一部分，定义了嵌入式 Power 处理器实现中使用的特权指令和相关设施。

> **Book III-E** Embedded Environment. Section of the Power ISA defining supervisor instructions and related facilities used in embedded Power processor implementations.

**引导程序** 用来泛指一个软件组件，它初始化系统状态并执行另一个被称为客户程序的软件组件。引导程序例如：固件、bootloader 和 hypervisor。

> **boot program** Used to generically refer to a software component that initializes the system state and executes another software component referred to as a client program. Examples of a boot program include: firmware, bootloaders, and hypervisors.

**客户程序** 某些程序，通常包含应用程序或操作系统软件。客户程序的示例包括：bootloader、hypervisor、操作系统和专用程序。

> **client program** Program that typically contains application or operating system software. Examples of a client program include: bootloaders, hypervisors, operating systems, and special purpose programs.

**单元** 由 32 位组成的信息单元。

> **cell** A unit of information consisting of 32 bits.

**DMA** 直接内存访问

> **DMA** Direct memory access

**DTB** 二进制设备树对象。设备树的紧凑二进制表示。

> **DTB** Devicetree blob. Compact binary representation of the devicetree.

**DTC** 设备树编译器。一个用于从 DTS 文件生成 DTB 文件的开源工具。

> **DTC** Devicetree compiler. An open source tool used to create DTB files from DTS files.

**DTS** 设备树语法。提供给 DTC 的设备树的文本表示。请参阅附录 A 设备树源码格式（版本 1）。

> **DTS** Devicetree syntax. A textual representation of a devicetree consumed by the DTC. See Appendix A Devicetree Source Format (version 1).

**有效地址** 由处理器访存或分支指令计算的内存地址。

> **effective address** Memory address as computed by processor storage access or branch instruction.

**物理地址** 处理器用于访问外部设备的地址，通常是内存控制器。

> **physical address** Address used by the processor to access external device, typically a memory controller.

**Power ISA** Power 指令集架构。

> **Power ISA** Power Instruction Set Architecture.

**中断描述符** 描述中断的属性值。通常包括用于指定中断号、灵敏度和触发机制的信息。

> **interrupt specifier** A property value that describes an interrupt. Typically information that specifies an interrupt number and sensitivity and triggering mechanism is included.

**次要CPU** 对于客户程序，引导 CPU 以外的 CPU 被视为次要 CPU。

> **secondary CPU** CPUs other than the boot CPU that belong to the client program are considered secondary CPUs.

**SMP** 对称多处理器。一种计算机架构，其中两个或多个相同的 CPU 可以共享内存和 IO，并在单个操作系统下运行。

> **SMP** Symmetric multiprocessing. A computer architecture where two or more identical CPUs can share memory and IO and operate under a single operating system.

**SoC** 片上系统。集成了一个或多个 CPU 内核以及许多其他外围设备的单个计算机芯片。

> **SoC** System on a chip. A single computer chip integrating one or more CPU core as well as number of other peripherals.

**单元地址** 节点名称的一部分，指定节点在父节点地址空间中的地址。

> **unit address** The part of a node name specifying the node’s address in the address space of the parent node.

**静止 CPU** 静止的 CPU 处于一种状态，它不能干扰其他 CPU 的正常运行，其状态也不能受到其他正常运行中的 CPU 影响，除非通过明确的方法启用或重启静止的CPU。

> **quiescent CPU** A quiescent CPU is in a state where it cannot interfere with the normal operation of other CPUs, nor can its state be affected by the normal operation of other running CPUs, except by an explicit method for enabling or re-enabling the quiescent CPU.
