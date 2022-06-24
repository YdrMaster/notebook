# 1 介绍

> 1 Introduction

本文档描述了“virtio”系列设备的标准。这些设备存在于虚拟环境中，但在设计上，它们对虚拟机中的软件来说就像物理设备一样——所以本文档将它们视为物理设备。这种相似性允许软件使用标准驱动程序和发现机制。

> This document describes the specifications of the “virtio” family of devices. These devices are found in virtual environments, yet by design they look like physical devices to the guest within the virtual machine - and this document treats them as such. This similarity allows the guest to use standard drivers and discovery mechanisms.

virtio 和本规范的目的是虚拟环境和其中的软件应该有一个**简洁**、**高效**、**标准**且**可扩展**的虚拟设备机制，而不是特定于每个环境或每个操作系统的机制。

> The purpose of virtio and this specification is that virtual environments and guests should have a straightforward, efficient, standard and extensible mechanism for virtual devices, rather than boutique per-environment or per-OS mechanisms.

**简洁**：virtio 设备使用正常的中断和 DMA 总线机制，任何设备驱动程序作者都应该熟悉这些机制。没有奇异的翻页或写时复制机制：它只是一个普通的设备。^1^

**高效**：virtio 设备由用于输入和输出的描述符环组成，它们布局整齐，以避免驱动程序和设备写入相同的缓存行造成缓存影响。

**标准**：virtio 对其运行环境不做任何假设，只支持设备所连接的总线。在本规范中，virtio 设备通过 MMIO、Channel I/O 和 PCI 总线传输 ^2^ 实现，早期的草案也在此处未包括的其他总线上实现。

**可扩展**：virtio 设备包含在设备设置期间由客户操作系统确认的功能位。这允许向前和向后兼容：设备提供它知道的所有功能，并且驱动程序承认它理解并希望使用的那些。

> **Straightforward**: Virtio devices use normal bus mechanisms of interrupts and DMA which should be familiar to any device driver author. There is no exotic page-flipping or COW mechanism: it’s just a normal device.^1^
>
> **Efficient**: Virtio devices consist of rings of descriptors for both input and output, which are neatly laid out to avoid cache effects from both driver and device writing to the same cache lines.
>
> **Standard**: Virtio makes no assumptions about the environment in which it operates, beyond supporting the bus to which device is attached. In this specification, virtio devices are implemented over MMIO, Channel I/O and PCI bus transports ^2^, earlier drafts have been implemented on other buses not included here.
>
> **Extensible**: Virtio devices contain feature bits which are acknowledged by the guest operating system during device setup. This allows forwards and backwards compatibility: the device offers all the features it knows about, and the driver acknowledges those it understands and wishes to use.

## 1.1 规范类参考

> 1.1 Normative References

## 1.2 非规范参考

> 1.2 Non-Normative References

## 1.3 术语

> 1.3 Terminology

本文档中的关键词**必须**、**不得**、**要求**、**应**、**不应**、**应该**、**不该**、**推荐**、**可以**和**可选**按照 \[RFC2119\] 中的描述进行解释。

> The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in \[RFC2119\].

### 1.3.1 旧版接口：术语

> 1.3.1 Legacy Interface: Terminology

本规范 1.0 版之前的规范草案（例如，参见 \[Virtio PCI Draft\]）在驱动程序和设备之间定义了类似但不同的接口。由于这些已经广泛使用，本规范包含**可选**功能以简化从这些早期草案接口的转换。

> Specification drafts preceding version 1.0 of this specification (e.g. see \[Virtio PCI Draft\]) defined a similar, but different interface between the driver and the device. Since these are widely deployed, this specification accommodates OPTIONAL features to simplify transition from these earlier draft interfaces.

具体来说，设备和驱动程序**可能**支持：

> Specifically devices and drivers MAY support:

**旧版接口**是本规范的早期草案（1.0 之前）指定的接口

> **Legacy Interface** is an interface specified by an earlier draft of this specification (before 1.0)

**旧版设备**是在本规范发布之前实现的设备，在主机端实现了旧版接口

> **Legacy Device** is a device implemented before this specification was released, and implementing a legacy interface on the host side

**旧版驱动**是在此规范发布之前实现的驱动程序，在虚拟机端实现了旧版驱动

> **Legacy Driver** is a driver implemented before this specification was released, and implementing a legacy interface on the guest side

旧设备和旧驱动程序不符合本规范。

> Legacy devices and legacy drivers are not compliant with this specification.

为了简化从这些早期草案接口的转换，设备**可以**实现：

> To simplify transition from these earlier draft interfaces, a device MAY implement:

**过渡设备**是支持符合本规范的驱动程序并允许旧版驱动程序的设备。

> **Transitional Device** a device supporting both drivers conforming to this specification, and allowing legacy drivers.

类似地，驱动程序**可以**实现：

> Similarly, a driver MAY implement:

**过渡驱动程序**是支持符合此规范的设备和旧设备的驱动程序。

> **Transitional Driver** a driver supporting both devices conforming to this specification, and legacy devices.

> **注意**：旧接口不属于**需要**级；即，除非您需要向后兼容，否则不要实现它们！

> > **Note**: Legacy interfaces are not required; ie. don’t implement them unless you have a need for backwards compatibility!

没有旧版兼容性的设备或驱动程序分别称为非过渡设备和非过渡驱动程序。

> Devices or drivers with no legacy compatibility are referred to as non-transitional devices and drivers, respectively.

### 1.3.2 从早期的规范草案过渡

> 1.3.2 Transition from earlier specification drafts

对于已经实现旧接口的设备和驱动程序，必须进行一些更改以支持此规范。

> For devices and drivers already implementing the legacy interface, some changes will have to be made to support this specification.

在这种情况下，读者将注意力集中在章节标题中标记为“旧接口”的章节可能是有益的。这些突出了自早期草案以来所做的更改。

> In this case, it might be beneficial for the reader to focus on sections tagged ”Legacy Interface” in the section title. These highlight the changes made since the earlier drafts.

## 1.4 结构规范

> 1.4 Structure Specifications

许多设备和驱动程序的内存结构布局都使用 C 结构语法记录。假定所有结构都没有额外的填充。为了强调这一点，已知常见的 C 编译器（需要）在结构中插入额外填充的情况使用 `GNU C __attribute__((packed))` 语法进行标记。

> Many device and driver in-memory structure layouts are documented using the C struct syntax. All structures are assumed to be without additional padding. To stress this, cases where common C compilers are known to insert extra padding within structures are tagged using the `GNU C __attribute__((packed))` syntax.

对于结构定义中使用的整数数据类型，使用以下约定：

> For the integer data types used in the structure definitions, the following conventions are used:

**u8**, **u16**, **u32**, **u64** 表示对应长度的无符号整数，以位为单位。

> **u8**, **u16**, **u32**, **u64** An unsigned integer of the specified length in bits.

**le16**, **le32**, **le64** 表示对应长度的无符号整数，以位为单位，采用小端字节序。

> **le16**, **le32**, **le64** An unsigned integer of the specified length in bits, in little-endian byte order.

**be16**, **be32**, **be64** 表示对应长度的无符号整数，以位为单位，采用大端字节序。

> **be16**, **be32**, **be64** An unsigned integer of the specified length in bits, in big-endian byte order.

本规范中要定义的一些字段不在字节边界上开始或结束。这样的字段称为位域。位域定义不会跨整数类型字段。

> Some of the fields to be defined in this specification don’t start or don’t end on a byte boundary. Such fields are called bit-fields. A set of bit-fields is always a sub-division of an integer typed field.

整数字段中的位字段总是按顺序列出，从最低有效位到最高有效位。

> Bit-fields within integer fields are always listed in order, from the least significant to the most significant bit.

位域被认为是指定宽度的无符号整数，保留了位的下一个重要关系。

> The bit-fields are considered unsigned integers of the specified width with the next in significance relationship of the bits preserved.

例如：

> For example:

```c
struct S {
    be16 {
        A : 15;
        B : 1;
    } x;
    be16 y;
};
```

记录存储在 x 的低 15 位中的值 A 和存储在 x 最高位中的值 B，16 位整数 x 依次使用结构 S 开头的大端字节顺序存储，并且紧随其后的是一个以大端字节顺序存储的无符号整数 y，位于结构开头的 2 个字节（16 位）的偏移量处。

> documents the value A stored in the low 15 bit of x and the value B stored in the high bit of x, the 16-bit integer x in turn stored using the big-endian byte order at the beginning of the structure S, and being followed immediately by an unsigned integer y stored in big-endian byte order at an offset of 2 bytes (16 bits) from the beginning of the structure.

请注意，此表示法有点类似于 C 位域语法，但不应该简单地转换为可移植代码的位域表示法：它匹配 C 编译器在小端架构上表示位域的方式，但不适于大端架构上的 C 编译器。

> Note that this notation somewhat resembles the C bitfield syntax but should not be naively converted to a bitfield notation for portable code: it matches the way bitfields are packed by C compilers on little-endian architectures but not the way bitfields are packed by C compilers on big-endian architectures.

假设 CPU_TO_BE16 将 16 位整数从本机 CPU 转换为大端字节序，以下是生成要存储到 x 中的值的等效可移植 C 代码：

> Assuming that CPU_TO_BE16 converts a 16-bit integer from a native CPU to the big-endian byte order, the following is the equivalent portable C code to generate a value to be stored into x:

```c
CPU_TO_BE16(B << 15 | A)
```

## 常量规范

> 1.5 Constant Specifications

在许多情况下，设备和驱动程序之间的接口中使用的数值是使用 C `#define` 和 `/**/` 注释语法记录的。多个相关值以通用名称为前缀组合在一起，使用 `_` 作为分隔符。使用 `_XXX` 作为后缀表示一个组中的所有值。例如：

> In many cases, numeric values used in the interface between the device and the driver are documented using the C #define and /**/ comment syntax. Multiple related values are grouped together with a common name as a prefix, using _as a separator. Using_XXX as a suffix refers to all values in a group. For example:

```c
/* Fld 字段的值 A */
/* Field Fld value A description */
#define VIRTIO_FLD_A (1 << 0)
/* Fld 字段的值 B */
/* Field Fld value B description */
#define VIRTIO_FLD_B (1 << 1)
```

记录字段 Fld 的两个数值，其中 Fld 的值 1 指代 A，Fld 的值 2 指代 B。注意 `<<` 指的是左移操作。

> documents two numeric values for a field Fld, with Fld having value 1 referring to A and Fld having value 2 referring to B. Note that << refers to the shift-left operation.

此外，在这种情况下，VIRTIO_FLD_A 和 VIRTIO_FLD_B 分别指 Fld 的值 1 和 2。此外，VIRTIO_FLD_XXX 指的是 VIRTIO_FLD_A 或 VIRTIO_FLD_B。

> Further, in this case VIRTIO_FLD_A and VIRTIO_FLD_B refer to values 1 and 2 of Fld respectively. Further, VIRTIO_FLD_XXX refers to either VIRTIO_FLD_A or VIRTIO_FLD_B.
