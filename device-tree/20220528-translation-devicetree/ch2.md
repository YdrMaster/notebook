# 第二章 设备树

> CHAPTER TWO THE DEVICETREE

## 2.1 概述

> 2.1 Overview

DTSpec 指定了一个叫做设备树的结构来描述系统硬件。引导程序将设备树加载到客户程序的内存中，并将设备树的指针传递给客户程序。

> DTSpec specifies a construct called a devicetree to describe system hardware. A boot program loads a devicetree into a client program’s memory and passes a pointer to the devicetree to the client.

本章描述了设备树的逻辑结构，并指定了一套用于描述设备节点的基本属性。第三章规定了符合 DTSpec 的设备树所要求的某些设备节点。第四章描述了 DTSpec 定义的设备绑定--代表某些设备类型或设备类别的要求。第五章描述了设备树的内存编码。

> This chapter describes the logical structure of the devicetree and specifies a base set of properties for use in describing device nodes. Chapter 3 specifies certain device nodes required by a DTSpec-compliant devicetree. Chapter 4 describes the DTSpec-defined device bindings – the requirements for representing certain device types or classes of devices. Chapter 5 describes the in-memory encoding of the devicetree.

设备树是一种树形数据结构，其节点描述了系统中的设备。每个节点都有描述所代表设备特性的属性/值对。每个节点都有一个父节点，除了根节点，它没有父节点。

> A devicetree is a tree data structure with nodes that describe the devices in a system. Each node has property/value pairs that describe the characteristics of the device being represented. Each node has exactly one parent except for the root node, which has no parent.

符合 DTSpec 的设备树描述了系统中不一定能被客户程序动态地检测到的设备信息。例如，PCI 的结构使客户端能够探测并确认接入的设备，因此描述 PCI 设备的设备树节点可能不需要。然而，如果系统中的 PCI 主机桥设备不能被探测到，则需要一个设备节点来描述它。

> A DTSpec-compliant devicetree describes device information in a system that cannot necessarily be dynamically detected by a client program. For example, the architecture of PCI enables a client to probe and detect attached devices, and thus devicetree nodes describing PCI devices might not be required. However, a device node is required to describe a PCI host bridge device in the system if it cannot be detected by probing.

### 示例

> Example

图 2.1 显示了一个简单的设备树的例子，它几乎完整到足以启动一个简单的操作系统，描述了平台类型、CPU、内存和一个串口设备。设备节点显示了每个节点内的属性和值。

> Fig. 2.1 shows an example representation of a simple devicetree that is nearly complete enough to boot a simple operating system, with the platform type, CPU, memory and a single UART described. Device nodes are shown with properties and values inside each node.

![设备树示例](2.1.png)

## 2.2 设备树结构与约定

> 2.2 Devicetree Structure and Conventions

### 2.2.1 节点名

> 2.2.1 Node Names

#### 节点名要求

> Node Name Requirements

设备树中的每个节点都根据以下约定命名：

```text
节点名@单元地址
```

> Each node in the devicetree is named according to the following convention:
>
> ```text
> node-name@unit-address
> ```

**节点名**部分指定节点的名称。它的长度**应当**为 1 到 31 个字符，并且仅由表 2.1 中字符集中的字符组成。

> The *node-name* component specifies the name of the node. It shall be 1 to 31 characters in length and consist solely ofcharacters from the set of characters in Table 2.1.

表 2.1：合法的节点名字符

| 字符 | 描述
| :-: | -
| 0-9 | digit
| a-z | 小写字母
| A-Z | 大写字母
|  ,  | 逗号
|  .  | 句号
|  _  | 下划线
|  +  | 加号
|  -  | 连字符

**节点名**应以小写或大写字母开始，并应描述设备的一般类别。

> The *node-name* shall start with a lower or uppercase character and should describe the general class of device.

名称中的**单元地址**部分特定于节点所在的总线类型。它由表 2.1 的字符集中的一个或多个ASCII字符组成。**单元地址**必须与节点的 `reg` 属性中指定的第一个地址相匹配。如果节点没有 `reg` 属性，**@单元地址**必须被省略，仅节点名称就能将节点与树中同级别的其他节点区分开来。特定总线的绑定可能会对 `reg` 和**单元地址**的格式指定额外的、更具体的要求。

> The *unit-address* component of the name is specific to the bus type on which the node sits. It consists of one or more ASCII characters from the set of characters in Table 2.1. The *unit-address* must match the first address specified in the reg property of the node. If the node has no reg property, the *@unit-address* must be omitted and the node-name alone differentiates the node from other nodes at the same level in the tree. The binding for a particular bus may specify additional, more specific requirements for the format of reg and the *unit-address*.

不包含 **@单元地址**的**节点名**应当在树结构的同级的所有属性名中是唯一的。

> In the case of *node-name* without an *@unit-address* the *node-name* shall be unique from any property names at the same level in the tree.

根节点没有**节点名**或**单元地址**。它由一个正斜杠（/）标识。

> The root node does not have a *node-name* or *unit-address*. It is identified by a forward slash (/).

![节点名的示例](2.2.png)

图 2.2 中：

> In Fig 2.2:

- 名称为 cpu 的节点通过其单位地址值 0 和 1 来区分。
- 名字为 ethernet 的节点通过它们的单位地址值 fe002000 和 fe003000 来区分。

> - The nodes with the name cpu are distinguished by their unit-address values of 0 and 1.
> - The nodes with the name ethernet are distinguished by their unit-address values of fe002000 and fe003000.

### 2.2.2 推荐通用名称

> 2.2.2 Generic Names Recommendation

节点的名称应该相对通用，反映设备的功能，而不是其精确的编程模型。如果合适的话，该名称应该是以下选择之一：

> The name of a node should be somewhat generic, reflecting the function of the device and not its precise programming model. If appropriate, the name should be one of the following choices:

- adc
- accelerometer
- air-pollution-sensor
- atm
- audio-codec
- audio-controller
- backlight
- bluetooth
- bus
- cache-controller
- camera
- can
- charger
- clock
- clock-controller
- co2-sensor
- compact-flash
- cpu
- cpus
- crypto
- disk
- display
- dma-controller
- dsi
- dsp
- eeprom
- efuse
- endpoint
- ethernet
- ethernet-phy
- fdc
- flash
- gnss
- gpio
- gpu
- gyrometer
- hdmi
- hwlock
- i2c
- i2c-mux
- ide
- interrupt-controlle
- iommu
- isa
- keyboard
- key
- keys
- lcd-controller
- led
- leds
- led-controller
- light-sensor
- lora
- magnetometer
- mailbox
- mdio
- memory
- memory-controller
- mmc
- mmc-slot
- mouse
- nand-controller
- nvram
- oscillator
- parallel
- pc-card
- pci
- pcie
- phy
- pinctrl
- pmic
- pmu
- port
- ports
- power-monitor
- pwm
- regulator
- reset-controller
- rng
- rtc
- sata
- scsi
- serial
- sound
- spi
- sram-controller
- ssi-controller
- syscon
- temperature-sensor
- timer
- touchscreen
- tpm
- usb
- usb-hub
- usb-phy
- video-codec
- vme
- watchdog
- wifi

### 2.2.3 路径名

设备树中的一个节点可以通过指定从根节点经过所有中间节点再到所需节点的完整路径来唯一地识别。

> A node in the devicetree can be uniquely identified by specifying the full path from the root node, through all descendant nodes, to the desired node.

指定一个设备路径的约定是：

```text
/节点名-1/节点名-2/节点名-N
```

> The convention for specifying a device path is:
>
> ```text
> /node-name-1/node-name-2/node-name-N
> ```

例如，图 2.2 中 #1 号 cpu 的设备路径是：

```text
/cpus/cpu@1
```

> For example, in Fig. 2.2, the device path to cpu #1 would be:
>
> ```text
> /cpus/cpu@1
> ```

根节点的路径是 /。

> The path to the root node is /.

如果节点的完整路径是明确的，可以省略单元地址。

> A unit address may be omitted if the full path to the node is unambiguous.

遇到不明确的路径对客户程序是未定义行为。

> If a client program encounters an ambiguous path, its behavior is undefined.

### 2.2.4 属性

> 2.2.4 Properties

设备树中的每个节点都有描述该节点特性的属性。属性由一个名称和一个值组成。

> Each node in the devicetree has properties that describe the characteristics of the node. Properties consist of a name and a value.

#### 属性名

> Property Names

属性名是由表 2.2 中显示的字符组成的 1 至 31 个字符的字符串。

> Property names are strings of 1 to 31 characters from the characters show in Table 2.2.

表 2.2：合法的属性名字符

| 字符 | 描述
| :-: | -
| 0-9 | digit
| a-z | 小写字母
| A-Z | 大写字母
|  ,  | 逗号
|  .  | 句号
|  _  | 下划线
|  +  | 加号
|  ?  | 问号
|  #  | 井号
|  -  | 连字符

非标准属性名称应指定一个独特的字符串前缀，如股票代码，识别定义该属性的公司或组织的名称。例如：

> Nonstandard property names should specify a unique string prefix, such as a stock ticker symbol, identifying the name of the company or organization that defined the property. Examples:

```text
fsl,channel-fifo-len
ibm,ppc-interrupt-server#s
linux,network-index
```

#### 属性值

> Property Values

一个属性值是一个由零个或多个字节组成的数组，包含与属性相关的信息。

> A property value is an array of zero or more bytes that contain information associated with the property.

如果传达的是二值信息，属性可能会有一个空值。在这种情况下，属性的存在或不存在就具有足够的描述性。

> Properties might have an empty value if conveying true-false information. In this case, the presence or absence of the property is sufficiently descriptive.

表 2.3 描述 DTSpec 定义的基本值类型集合

> Table 2.3 describes the set of basic value types defined by the DTSpec.

- `<empty>`

  值为空。用于传达二值信息，属性本身的存在或不存在就足以说明问题。

  > Value is empty. Used for conveying true-false information, when the presence or absence of the property itself is sufficiently descriptive.

- `<u32>`

  大端格式的 32 位整数。例如：32 位值 0x11223344 在内存中会被表示为：

  > A 32-bit integer in big-endian format. Example: the 32-bit value 0x11223344 would be represented in memory as:

  ```text
  address   11
  address+1 22
  address+2 33
  address+3 44
  ```

- `<u64>`

  代表一个大端格式的 64 位整数。由两个 `<u32>` 值组成，第一个值包含整数的高 32 位，第二个值包含低 32 位。

  > Represents a 64-bit integer in big-endian format. Consists of two `<u32>` values where the first value contains the most significant bits of the integer and the second value contains the least significant bits.

  例如：64 位值 0x1122334455667788 会被表示为 2 个单元，即 `<0x11223344 0x55667788>`。

  > Example: the 64-bit value 0x1122334455667788 would be represented as two cells as: `<0x11223344 0x55667788>`.

  值将在内存中表示为：

  > The value would be represented in memory as:

  ```text
  address   11
  address+1 22
  address+2 33
  address+3 44
  ```

- `<string>`

  字符串是可打印的，且以空结尾。例如：字符串 `"hello"` 在内存中表示为：

  > Strings are printable and null-terminated. Example: the string “hello” would be represented in memory as:

  ```text
    address 68 'h'
  address+1 65 'e'
  address+2 6C 'l'
  address+3 6C 'l'
  address+4 6F 'o'
  address+5 00 '\0'
  ```

- `<prop-encoded-array>`

  格式特定于属性。参见属性定义。

  > Format is specific to the property. See the property definition.

- `<phandle>`

  一个 `<u32>` 值。`phandle` 值是引用设备树中另一个节点的一种方式。任何可以被引用的节点都定义了一个具有唯一的 `<u32>` 值的 `phandle` 属性。这个数字被用于具有 `phandle` 值类型的属性的值。

  > A `<u32>` Value. A phandle value is a way to reference another node in the devicetree. Any node that can be referenced defines a phandle property with a unique `<u32>` value. That number is used for the value of properties with a phandle value type.

- `<stringlist>`

  串联在一起的 `<string>` 值列表。

  > A list of `<string>` values concatenated together.

  例如：字符串列表 `"hello", "world"` 在内存中表示为：

  > Example: The string list “hello”, “world” would be represented in memory as:

  ```text
     address 68 'h'
   address+1 65 'e'
   address+2 6C 'l'
   address+3 6C 'l'
   address+4 6F 'o'
   address+5 00 '\0'
   address+6 77 'w'
   address+7 6f 'o'
   address+8 72 'r'
   address+9 6C 'l'
  address+10 64 'd'
  address+11 00 '\0'
  ```

## 2.3 标准属性

> 2.3 Standard Properties

DTSpec 为设备节点指定了一组标准属性。这些属性将在本节中详细描述。由 DTSpec 定义的设备节点（见第三章）可以指定关于使用标准属性的额外要求或约束。第四章描述了特定设备的表示方法，也可能指定额外的要求。

> DTSpec specifies a set of standard properties for device nodes. These properties are described in detail in this section. Device nodes defined by DTSpec (see Chapter 3) may specify additional requirements or constraints regarding the use of the standard properties. Chapter 4 describes the representation of specific devices and may also specify additional requirements.

---

**注意**：本文件中所有的设备树节点的例子都使用 DTS（设备树源码）格式来指定节点和属性。

> **Note**: All examples of devicetree nodes in this document use the DTS (Devicetree Source) format for specifying nodes and properties.

---

### 2.3.1 `compatible`

属性名：`compatible`

值类型：`<stringlist>`

描述：

**兼容性**属性值由一个或多个字符串组成，这些字符串定义了电子设备的特定编程模型。这个字符串列表应该被客户程序用来选择设备驱动程序。属性值由一个空结尾字符串的串联列表组成，从最具体到最一般。它们允许设备表达其与一系列类似设备的兼容性，有可能允许一个设备驱动程序与多个设备相匹配。

> The *compatible* property value consists of one or more strings that define the specific programming model for e device. This list of strings should be used by a client program for device driver selection. The property lue consists of a concatenated list of null terminated strings, from most specific to most general. They allow device to express its compatibility with a family of similar devices, potentially allowing a single device driver  match against several devices.

推荐的格式是 `"制造商,型号"`，其中制造商是一个描述制造商名称的字符串（如股票代码），而型号则指定型号。

> The recommended format is "manufacturer,model", where manufacturer is a string describing the name of the manufacturer (such as a stock ticker symbol), and model specifies the model number.

兼容的字符串应仅由小写字母、数字和破折号组成，并应以字母开头。通常只在供应商前缀之后使用单个逗号。不应使用下划线。

> The compatible string should consist only of lowercase letters, digits and dashes, and should start with a letter. A single comma is typically only used following a vendor prefix. Underscores should not be used.

示例：

```dts
compatible = "fsl,mpc8641", "ns16550";
```

在这个例子中，操作系统首先会尝试找到一个支持 `fsl,mpc8641` 的设备驱动程序。如果没有找到驱动程序，它就会试图找到一个支持更普遍的 `ns16550` 设备类型的驱动程序。

> In this example, an operating system would first try to locate a device driver that supported fsl,mpc8641. If a driver was not found, it would then try to locate a driver that supported the more general ns16550 device type.

### 2.3.2 `model`

属性名：`medel`

值类型：`<string>`

描述：

型号属性值是一个 `<string>`，指定设备的制造商和型号。

> The model property value is a `<string>` that specifies the manufacturer’s model number of the device.

推荐的格式是：`"制造商,型号"`，其中制造商是一个描述制造商名称的字符串（如股票代码），而型号则指定型号。

> The recommended format is: "manufacturer,model", where manufacturer is a string describing the name of the manufacturer (such as a stock ticker symbol), and model specifies the model number.

示例：

```dts
model = "fsl,MPC8349EMITX";
```

### 2.3.3 `phandle`

属性名：`phandle`

值类型：`<u32>`

描述：

`phandle` 属性为一个节点指定了一个数字标识符，该标识符在设备树内是唯一的。`phandle` 属性值由需要引用与该属性相关联的节点的其他节点使用。

> The phandle property specifies a numerical identifier for a node that is unique within the devicetree. The phandle property value is used by other nodes that need to refer to the node associated with the property.

示例：

请参阅以下设备树摘录：

> See the following devicetree excerpt:

```dts
pic@10000000 {
  phandle = <1>;
  interrupt-controller;
  reg = <0x10000000 0x100>;
};
```

一个 `phandle` 值为 1 的定义。另一个设备节点可以引用 `phandle` 值为 1 的 pic 节点。

> A phandle value of 1 is defined. Another device node could reference the pic node with a phandle value of 1:

```dts
another-device-node {
interrupt-parent = <1>;
};
```

---

**注意**：旧版本的设备树可能包含这个属性的废弃形式，称为 `linux,phandle`。为了兼容，如果没有 `phandle` 属性，客户程序可能希望支持 `linux,phandle`。这两个属性的含义和用途是相同的。

> **Note**: Older versions of devicetrees may be encountered that contain a deprecated form of this property called linux, phandle. For compatibility, a client program might want to support linux,phandle if a phandle property is not present. The meaning and use of the two properties is identical.

---

**注意**：大多数 DTS 形式的设备树（见附录 A）将不包含明确的 `phandle` 属性。当 DTS 被编译成二进制 DTB 格式时，DTC 工具会自动插入 `phandle` 属性。

**Note**: Most devicetrees in DTS (see Appendix A) will not contain explicit phandle properties. The DTC tool automatically inserts the phandle properties when the DTS is compiled into the binary DTB format.

---

### 2.3.4 `status`

属性名：`status`

值类型：`<string>`

描述：

状态属性指示了设备的运行状态。缺少状态属性应被视为该属性存在，其值为 `"okay"`。有效值在表 2.4 中列出并定义。

> The status property indicates the operational status of a device. The lack of a status property should be treated as if the property existed with the value of "okay". Valid values are listed and defined in Table 2.4.

表 2.4：状态属性的值

> Table 2.4: Values for status property

- `"okay"`

  表明设备正在运行。

  > Indicates the device is operational.

- `"disabled"`

  表示该设备目前没有运行，但将来可能会运行（例如，有些东西没有插上电源，或者关闭）。关于禁用对某一特定设备的含义，请参考设备绑定的细节。

  > Indicates that the device is not presently operational, but it might become operational in the future (for example, something is not plugged in, or switched off). Refer to the device binding for details on what disabled means for a given device.

- `"reserved"`

  表示设备可以运行，但不应该被使用。通常，这用于由另一个软件组件控制的设备，如平台固件。

  > Indicates that the device is operational, but should not be used. Typically this is used for devices that are controlled by another software component, such as platform firmware.

- `"fail"`

  表示设备不能运行。在设备中检测到一个严重的错误，如果不进行修理，它就不可能投入使用。

  > Indicates that the device is not operational. A serious error was detected in the device, and it is unlikely to become operational without repair.

- `"fail-sss"`

  表示设备无法运行。在设备中检测到一个严重的错误，如果不进行修理，它就不可能投入使用。该值的 `sss` 部分是设备指定的，表示检测到的错误情况。

  > Indicates that the device is not operational. A serious error was detected in the device and it is unlikely to become operational without repair. The sss portion of the value is specific to the device and indicates the error condition detected.

### 2.3.5 `#address-cells` 和 `#size-cells`

属性名：`#address-cells`，`#size-cells`

值类型：`<u32>`

描述：

`#address-cells` 和 `#size-cells` 属性可用于任何在设备树层次结构中拥有子节点的设备节点，并描述应如何寻址子设备节点。`#address-cells` 属性定义了用于编码子节点的 `reg` 属性中的地址字段的 `<u32>` 单元的数量。`#size-cells` 属性定义了用于编码子节点的 `reg` 属性中的大小字段的 `<u32>` 单元的数量。

> The #address-cells and #size-cells properties may be used in any device node that has children in the devicetree hierarchy and describes how child device nodes should be addressed. The #address-cells property defines the number of `<u32>` cells used to encode the address field in a child node’s reg property. The #size-cells property defines the number of `<u32>` cells used to encode the size field in a child node’s reg property.

`#address-cells` 和 `#size-cells` 属性不从设备树中的祖先继承。它们应当明确定义。

> The #address-cells and #size-cells properties are not inherited from ancestors in the devicetree. They shall be explicitly defined.

符合 DTSpec 的引导程序应当在所有非叶节点上提供 `#address-cell` 和 `#size-cells`。

> A DTSpec-compliant boot program shall supply #address-cells and #size-cells on all nodes that have children.

如果缺少，客户程序应该假定 `#address-cells` 的默认值为 2，`#size-cells` 的默认值为 1。

> If missing, a client program should assume a default value of 2 for #address-cells, and a value of 1 for #size-cells

示例：

请参阅以下设备树摘录：

> See the following devicetree excerpt:

```dts
soc {
  #address-cells = <1>;
  #size-cells = <1>;

  serial@4600 {
    compatible = "ns16550";
    reg = <0x4600 0x100>;
    clock-frequency= <0>;
    interrupts = <0xA 0x8>;
    interrupt-parent = <&ipic>;
  };
};
```

此示例中，`soc` 节点的 `#address-cells` 和 `#size-cells` 属性都被设置为 1。这个设置指定了需要一个单元来表示地址，需要一个单元来表示作为这个节点的子节点的大小。

> In this example, the #address-cells and #size-cells properties of the soc node are both set to 1. This setting specifies that one cell is required to represent an address and one cell is required to represent the size of nodes that are children of this node.

串行设备寄存器属性必须遵循父节点（soc）中设置的这个规范——地址由一个单元（0x4600）表示，大小由一个单元（0x100）表示。

> The serial device reg property necessarily follows this specification set in the parent (soc) node—the address is represented by a single cell (0x4600), and the size is represented by a single cell (0x100).

### 2.3.6 `reg`

属性名：`reg`

属性值：`<prop-encoded-array>`编码为任意数量的（地址，长度）对。

> Property value: `<prop-encoded-array>` encoded as an arbitrary number of (address, length) pairs.

描述：

寄存器属性描述了设备的资源在其父总线所定义的地址空间内的地址。最常见的是这意味着内存映射的 IO 寄存器块的偏移量和长度，但在某些总线类型上可能有不同的含义。根节点定义的地址空间中的地址是 CPU 物理地址。

> The reg property describes the address of the device’s resources within the address space defined by its parent bus. Most commonly this means the offsets and lengths of memory-mapped IO register blocks, but may have a different meaning on some bus types. Addresses in the address space defined by the root node are CPU real addresses.

该值是一个 `<prop-encoded-array>`，由任意数量的地址和长度对 `<address length>` 组成。指定地址和长度所需的 `<u32>` 单元的数量是针对总线的，由设备节点的父节点的 `#address-cell` 和 `#size-cells` 属性指定。如果父节点为 `#size-cells` 指定的值为 0，那么 `reg` 值中的长度字段应被省略。

> The value is a `<prop-encoded-array>`, composed of an arbitrary number of pairs of address and length, `<address length>`. The number of `<u32>` cells required to specify the address and length are bus-specific and are specified by the #address-cells and #size-cells properties in the parent of the device node. If the parent node specifies a value of 0 for #size-cells, the length field in the value of reg shall be omitted.

示例：

假设一个片上系统的设备有两个寄存器块，一块是位于 SOC 中偏移量 0x3000 的 32 字节，一块是位于偏移量 0xFE00 的 256 字节。寄存器属性将被编码如下（假设 `#address-cell` 和 `#size-cells` 值为 1）。

> Suppose a device within a system-on-a-chip had two blocks of registers, a 32-byte block at offset 0x3000 in the SOC and a 256-byte block at offset 0xFE00. The reg property would be encoded as follows (assuming #address-cells and #size-cells values of 1):

```dts
reg = <0x3000 0x20 0xFE00 0x100>;
```

### 2.3.7 `virtual-reg`

属性名：`virtual-reg`

值类型：`<u32>`

描述：

虚寄存器属性指定了一个有效地址，该地址映射到设备节点的 `reg` 属性中指定的第一个物理地址。这个属性使引导程序能够向客户程序提供已经设置好的虚拟到物理的映射关系。

> The virtual-reg property specifies an effective address that maps to the first physical address specified in the reg property of the device node. This property enables boot programs to provide client programs with virtual-to-physical mappings that have been set up.

### 2.3.8 `ranges`

属性名：`virtual-reg`

值类型：`<empty>` 或 `<prop-encoded-array>` 编码为任意数量的（子总线地址，父总线地址，长度）三元组。

> Value type: `<empty>` or `<prop-encoded-array>` encoded as an arbitrary number of (child-bus-address, parent-busaddress, length) triplets.

描述：

范围属性提供了一种定义总线地址空间（子地址空间）和总线节点父节点地址空间（父地址空间）之间的映射或转换的方法。

> The ranges property provides a means of defining a mapping or translation between the address space of the bus (the child address space) and the address space of the bus node’s parent (the parent address space).

范围属性的值的格式是任意数量的（子总线地址，父总线地址，长度）三元组。

> The format of the value of the ranges property is an arbitrary number of triplets of (child-bus-address, parentbus-address, length).

- 子总线地址是子总线地址空间内的物理地址。表示地址的单元数取决于总线，可以从该节点的 `#address-cells`（`ranges` 属性出现的节点）确定。
- 父总线地址是父总线地址空间内的物理地址。表示父地址的单元数取决于总线，可以从定义父地址空间的节点的 `#address-cells` 属性确定。
- 长度指定子地址空间中范围的大小。表示大小的单元格的数量可以从该节点的 `#size-cells`（`ranges` 属性出现的节点）确定。

> - The child-bus-address is a physical address within the child bus’ address space. The number of cells to represent the address is bus dependent and can be determined from the #address-cells of this node (the node in which the ranges property appears).
> - The parent-bus-address is a physical address within the parent bus’ address space. The number of cells to represent the parent address is bus dependent and can be determined from the #address-cells property of the node that defines the parent’s address space.
> - The length specifies the size of the range in the child’s address space. The number of cells to represent the size can be determined from the #size-cells of this node (the node in which the ranges property appears).

如果该属性定义了一个 `<empty>` 值，它指定父地址空间和子地址空间是相同的，并且不需要地址转换。

> If the property is defined with an `<empty>` value, it specifies that the parent and child address space is identical, and no address translation is required.

如果该属性不存在于总线节点中，则假定该节点的子节点与父地址空间之间不存在映射。

> If the property is not present in a bus node, it is assumed that no mapping exists between children of the node and the parent address space.

地址翻译示例：

> Address Translation Example:

```dts
  <0x0 0xe0000000 0x00100000>;
```

```dts
soc {
  compatible = "simple-bus";
  #address-cells = <1>;
  # size-cells = <1>;
  ranges = <0x0 0xe0000000 0x00100000>;

  serial@4600 {
    device_type = "serial";
    compatible = "ns16550";
    reg = <0x4600 0x100>;
    clock-frequency = <0>;
    interrupts = <0xA 0x8>;
    interrupt-parent = <&ipic>;
  };
};
```

soc 节点指定范围属性

> The soc node specifies a ranges property of

```dts
  <0x0 0xe0000000 0x00100000>;
```

此属性值指定对于 1024 KB 范围的地址空间，地址为物理 0x0 的子节点映射到物理地址 0xe0000000 的父地址。使用此映射，串行设备节点可以通过地址 0xe0004600 的加载或存储来寻址，偏移量为 0x4600（在 reg 中指定）加上范围中指定的 0xe0000000 映射。

> This property value specifies that for a 1024 KB range of address space, a child node addressed at physical 0x0 maps to a parent address of physical 0xe0000000. With this mapping, the serial device node can be addressed by a load or store at address 0xe0004600, an offset of 0x4600 (specified in reg) plus the 0xe0000000 mapping specified in ranges.

### 2.3.9 `dma-ranges`

属性名：`dma-ranges`

值类型：`<empty>` 或 `<prop-encoded-array>` 编码为任意数量的（子总线地址，父总线地址，长度）三元组。

> Value type: `<empty>` or `<prop-encoded-array>` encoded as an arbitrary number of (child-bus-address, parent-busaddress, length) triplets.

描述：

DMA 范围属性用于描述内存映射总线的直接内存访问（DMA）结构，其设备树父级可以通过总线 DMA 操作访问。它提供了一种在总线的物理地址空间和总线的父级物理地址空间之间定义映射或转换的方法。

> The dma-ranges property is used to describe the direct memory access (DMA) structure of a memory-mapped bus whose devicetree parent can be accessed from DMA operations originating from the bus. It provides a means of defining a mapping or translation between the physical address space of the bus and the physical address space of the parent of the bus.

DMA 范围属性值的格式是（子总线地址、父总线地址、长度）的任意数量的三元组。每个指定的三元组描述一个连续的 DMA 地址范围。

> The format of the value of the dma-ranges property is an arbitrary number of triplets of (child-bus-address, parent-bus-address, length). Each triplet specified describes a contiguous DMA address range.

- 子总线地址是子总线地址空间内的物理地址。表示地址的单元数取决于总线，并且可以从该节点的 `#address-cells`（`dma-ranges` 属性出现的节点）确定。
- 父总线地址是父总线地址空间内的物理地址。表示父地址的单元数取决于总线，可以从定义父地址空间的节点的 `#address-cells` 属性确定。
- 长度指定子地址空间中范围的大小。表示大小的单元格的数量可以从该节点的 `#size-cells`（`dma-ranges` 属性出现的节点）确定。

> - The child-bus-address is a physical address within the child bus’ address space. The number of cells to represent the address depends on the bus and can be determined from the #address-cells of this node (the node in which the dma-ranges property appears).
> - The parent-bus-address is a physical address within the parent bus’ address space. The number of cells to represent the parent address is bus dependent and can be determined from the #address-cells property of the node that defines the parent’s address space.
> - The length specifies the size of the range in the child’s address space. The number of cells to represent the size can be determined from the #size-cells of this node (the node in which the dma-ranges property appears).

### 2.3.10 `dma-coherent`

属性名：`dma-coherent`

值类型：`<empty>`

描述：

对于默认情况下 I/O 不连贯的架构，DMA 连贯性属性用于指示设备能够进行连贯的 DMA 操作。某些架构默认具有连贯 DMA，此属性不适用。

> For architectures which are by default non-coherent for I/O, the dma-coherent property is used to indicate a device is capable of coherent DMA operations. Some architectures have coherent DMA by default and this property is not applicable.

### 2.3.11 `name`（废弃）

属性名：`name`

值类型：`<string>`

描述：

名称属性是指定节点名称的字符串。此属性已弃用，不推荐使用。但是，它可能用于较旧的不符合 DTSpec 的设备树。操作系统应该根据树的节点名称来确定节点的名称（参见第 2.2.1 节）。

> The name property is a string specifying the name of the node. This property is deprecated, and its use is not recommended. However, it might be used in older non-DTSpec-compliant devicetrees. Operating system should determine a node’s name based on the node-name component of the node name (see Section 2.2.1).

### 2.3.12 `device_type`（废弃）

属性名：`device_type`

值类型：`<string>`

描述：

设备类型属性在 IEEE 1275 中用于描述设备的 FCode 编程模型。由于 DTSpec 没有 FCode，因此不推荐使用该属性的新用途，并且它应该仅包含在 cpu 和内存节点上，以便与 IEEE 1275 派生的设备树兼容。

> The device_type property was used in IEEE 1275 to describe the device’s FCode programming model. Because DTSpec does not have FCode, new use of the property is deprecated, and it should be included only on cpu and memory nodes for compatibility with IEEE 1275–derived devicetrees.

## 2.4 中断和中断映射

> 2.4 Interrupts and Interrupt Mapping

DTSpec 采用中断树模型来表示开放固件推荐实践中指定的中断：中断映射，版本 0.9 \[b7\]。在设备树中存在一个逻辑中断树，它表示平台硬件中中断的层次结构和路由。虽然通常称为中断树，但从技术上讲，它是有向无环图。

> DTSpec adopts the interrupt tree model of representing interrupts specified in Open Firmware Recommended Practice: Interrupt Mapping, Version 0.9 \[b7\]. Within the devicetree a logical interrupt tree exists that represents the hierarchy and routing of interrupts in the platform hardware. While generically referred to as an interrupt tree it is more technically a directed acyclic graph.

中断源到中断控制器的物理连接在设备树中以 `interrupt-parent` 属性表示。代表中断生成设备的节点包含一个 `interrupt-parent` 属性，该属性具有一个 `phandle` 值，该值指向设备的中断被路由到的设备，通常是中断控制器。如果产生中断的设备没有`interrupt-parent` 属性，则假定其中断父级为其设备树父级。

> The physical wiring of an interrupt source to an interrupt controller is represented in the devicetree with the interrupt-parent property. Nodes that represent interrupt-generating devices contain an interrupt-parent property which has a phandle value that points to the device to which the device’s interrupts are routed, typically an interrupt controller. If an interrupt-generating device does not have an interrupt-parent property, its interrupt parent is assumed to be its devicetree parent.

每个中断生成设备都包含一个中断属性，其值描述该设备的一个或多个中断源。每个源都用称为中断说明符的信息表示。中断说明符的格式和含义是特定于中断域的，即，它取决于其中断域根节点上的属性。中断域的根使用 `#interrupt-cells` 属性来定义编码中断说明符所需的 `<u32>` 值的数量。例如，对于 Open PIC 中断控制器，中断说明符采用两个 32 位值，由中断号和中断电平/检测信息组成。

> Each interrupt generating device contains an interrupts property with a value describing one or more interrupt sources for that device. Each source is represented with information called an interrupt specifier. The format and meaning of an interrupt specifier is interrupt domain specific, i.e., it is dependent on properties on the node at the root of its interrupt domain. The #interrupt-cells property is used by the root of an interrupt domain to define the number of `<u32>` values needed to encode an interrupt specifier. For example, for an Open PIC interrupt controller, an interrupt-specifer takes two 32-bit values and consists of an interrupt number and level/sense information for the interrupt.

中断域是解释中断说明符的上下文。域的根是中断控制器（1）或中断连接（2）。

> An interrupt domain is the context in which an interrupt specifier is interpreted. The root of the domain is either (1) an interrupt controller or (2) an interrupt nexus.

1. 中断控制器是一个物理设备，需要一个驱动程序来处理通过它路由的中断。它也可能级联到另一个中断域。中断控制器由设备树中该节点上存在的中断控制器属性指定。
2. 中断连接定义了一个中断域和另一个中断域之间的转换。翻译基于特定领域和特定总线的信息。域之间的这种转换是使用中断映射属性执行的。例如，PCI 控制器设备节点可以是定义从 PCI 中断名称空间（INTA、INTB 等）到具有中断请求（IRQ）编号的中断控制器的转换的中断连接。

> 1. An interrupt controller is a physical device and will need a driver to handle interrupts routed through it. It may also cascade into another interrupt domain. An interrupt controller is specified by the presence of an interrupt-controller property on that node in the devicetree.
> 2. An interrupt nexus defines a translation between one interrupt domain and another. The translation is based on both domain-specific and bus-specific information. This translation between domains is performed with the interrupt-map property. For example, a PCI controller device node could be an interrupt nexus that defines a translation from the PCI interrupt namespace (INTA, INTB, etc.) to an interrupt controller with Interrupt Request (IRQ) numbers.

当中断树的遍历到达没有中断属性且因此没有显式中断父节点的中断控制器节点时，确定中断树的根。

> The root of the interrupt tree is determined when traversal of the interrupt tree reaches an interrupt controller node without an interrupts property and thus no explicit interrupt parent.

请参见图 2.3 中的设备树图形表示示例，其中显示了中断父关系。它显示了设备树的自然结构以及每个节点在逻辑中断树中的位置。

> See Fig. 2.3 for an example of a graphical representation of a devicetree with interrupt parent relationships shown. It shows both the natural structure of the devicetree as well as where each node sits in the logical interrupt tree.

![2.3](2.3.png)

> **NOTICE** 这个图来自 0.3 版，因为在 0.4-rc1 中缺失

在图 2.3 所示的示例中：

> In the example shown in Fig. 2.3:

- open-pic 中断控制器是中断树的根。
- 中断树根有三个子节点——将它们的中断直接路由到 open-pic 的设备
  - 设备 1
  - PCI 主机控制器
  - GPIO 控制器
- 存在三个中断域；一个植根于 open-pic 节点，一个植根于 PCI 主机桥节点，一个植根于 GPIO 控制器节点。
- 有两个连接节点；一个在 PCI 主机桥上，一个在 GPIO 控制器上。

> - The open-pic interrupt controller is the root of the interrupt tree.
> - The interrupt tree root has three children—devices that route their interrupts directly to the open-pic
>   - device1
>   - PCI host controller
>   - GPIO Controller
> - Three interrupt domains exist; one rooted at the open-pic node, one at the PCI host bridge node, and one at the GPIO Controller node.
> - There are two nexus nodes; one at the PCI host bridge and one at the GPIO controller.

### 2.4.1 产生中断的设备的属性

> 2.4.1 Properties for Interrupt Generating Devices

#### `interrupts`

属性：`interrupts`

值类型：`<prop-encoded-array>` 编码任意数量的中断说明符

描述：

设备节点的 `interrupts` 属性定义了设备生成的一个或多个中断。`interrupts` 属性的值由任意数量的中断说明符组成。中断说明符的格式由中断域根的绑定定义。

> The `interrupts` property of a device node defines the interrupt or interrupts that are generated by the device. The value of the `interrupts` property consists of an arbitrary number of interrupt specifiers. The format of an interrupt specifier is defined by the binding of the interrupt domain root.

`interrupts` 被 `interrupts-extended` 属性覆盖，通常只能使用其中一个。

> `interrupts` is overridden by the `interrupts-extended` property and normally only one or the other should be used.

示例:

开放 PIC 兼容中断域中中断说明符的常见定义由两个单元组成；中断号和电平/感应信息。请参见以下示例，该示例定义了一个中断说明符，中断号为 0xA，电平/感应编码为 8。

> A common definition of an interrupt specifier in an open PIC–compatible interrupt domain consists of two cells; an interrupt number and level/sense information. See the following example, which defines a single interrupt specifier, with an interrupt number of 0xA and level/sense encoding of 8.

```dts
interrupts = <0xA 8>;
```

#### `interrupt-parent`

属性：`interrupt-parent`

值类型：`<phandle>`

描述：

由于中断树中节点的层次结构可能与设备树不匹配，因此可以使用 `interrupt-parent` 属性来明确定义中断父级。该值是中断父节点的 `phandle`。如果设备中缺少此属性，则假定其中断父级为其设备树父级。

> Because the hierarchy of the nodes in the interrupt tree might not match the devicetree, the `interrupt-parent` property is available to make the definition of an interrupt parent explicit. The value is the `phandle` to the interrupt parent. If this property is missing from a device, its interrupt parent is assumed to be its devicetree parent.

#### `interrupts-extended`

属性：`interrupts-extended`

值类型：`<phandle>` `<prop-encoded-array>`

描述：

`interrupts-extended` 属性列出了设备产生的中断。当设备连接到多个中断控制器时，应该使用 `interrupts-extended` 代替 `interrupts`，因为它为每个中断说明符都编码一个父 `phandle`。

> The `interrupts-extended` property lists the interrupt(s) generated by a device. `interrupts-extended` should be used instead of `interrupts` when a device is connected to multiple interrupt controllers as it encodes a parent phandle with each interrupt specifier.

示例：

此示例显示了具有连接到两个独立中断控制器的两个中断输出的设备如何使用中断扩展属性来描述连接。*pic* 是 `#interrupt-cells` 说明符为 2 的中断控制器，而 *gic* 是 `#interrupts-cells` 说明符为 1 的中断控制器。

> This example shows how a device with two interrupt outputs connected to two separate interrupt controllers would describe the connection using an interrupts-extended property. pic is an interrupt controller with an `#interrupt-cells` specifier of 2, while gic is an interrupt controller with an `#interrupts-cells` specifier of 1.

```dts
interrupts-extended = <&pic 0xA 8>, <&gic 0xda>;
```

`interrupts` 和 `interrupts-extended` 属性是互斥的。设备节点应该使用其中之一，但不能同时使用两者。只有在需要与 `interrupts-extended` 中断扩展的软件兼容时才允许使用两者。如果 `interrupts-extended` 和 `interrupts` 都存在，则 `interrupts-extended` 优先。

> The `interrupts` and `interrupts-extended` properties are mutually exclusive. A device node should use one or the other, but not both. Using both is only permissible when required for compatibility with software that does not understand interrupts-extended. If both interrupts-extended and interrupts are present then interrupts-extended takes precedence.

### 2.4.2 中断控制器的属性

> 2.4.2 Properties for Interrupt Controllers

#### `#interrupt-cells`

属性：`#interrupt-cells`

值类型：`<u32>`

描述：

`#interrupt-cells` 属性定义了为中断域编码中断说明符所需的单元数。

> The `#interrupt-cells` property defines the number of cells required to encode an interrupt specifier for an interrupt domain.

#### `interrupt-controller`

属性：`interrupt-controller`

值类型：`<empty>`

描述：

中断控制器属性的存在将节点定义为中断控制器节点。

> The presence of an interrupt-controller property defines a node as an interrupt controller node.

### 2.4.3 中断连接属性

> 2.4.3 Interrupt Nexus Properties

中断连接节点应具有 `#interrupt-cells` 属性。

> An interrupt nexus node shall have an #interrupt-cells property.

#### `interrupt-map`

属性：`interrupt-map`

值类型：`<prop-encoded-array>` 编码任意数量的中断映射条目。

描述：

`interrupt-map` 是连接节点上的一个属性，它将一个中断域与一组父中断域桥接，并指定子域中的中断说明符如何映射到它们各自的父域。

> An `interrupt-map` is a property on a nexus node that bridges one interrupt domain with a set of parent interrupt domains and specifies how interrupt specifiers in the child domain are mapped to their respective parent domains.
