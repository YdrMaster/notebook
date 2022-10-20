# 第三章 设备节点要求

> CHAPTER THREE DEVICE NODE REQUIREMENTS

## 3.1 基本设备节点类型

> 3.1 Base Device Node Types

后续章节指定了符合 DTSpec 的设备树中所需的基本设备节点集的要求。

> The sections that follow specify the requirements for the base set of device nodes required in a DTSpec-compliant device tree.

所有设备树都应有一个根节点，并且以下节点应出现在所有设备树的根部：

> All devicetrees shall have a root node and the following nodes shall be present at the root of all devicetrees:

- 一个 /cpus 节点
- 至少一个 /memory 节点

> - One /cpus node
> - At least one /memory node

3.2 根节点

>3.2 Root node

设备树有一个根节点，所有其他设备节点都是其后代。根节点的完整路径是 /。

> The devicetree has a single root node of which all other device nodes are descendants. The full path to the root node is /.

表 3.1：根节点属性

> Table 3.1: Root Node Properties

- `#address-cells`/`<u32>`（必要）

  指定根节点的子节点中 reg 属性里地址需要的 `<u32>` 单元的数量。

- `#size-cells`/`<u32>`（必要）

  指定根节点的子节点中 reg 属性里长度需要的 `<u32>` 单元的数量。

- `model`/`<string>`（必要）

  指定唯一确定系统板型号的字符串。推荐的格式是“制造商，型号数字”。

- `compatible`/`<stringlist>`（必要）

  指定与此平台兼容的平台架构列表。操作系统可识别此属性来选择特定于平台的代码。这个属性值的推荐格式是：

  ```dts
  “制造商，型号”
  ```

  例如

  ```dts
  compatible = "fsl,mpc8572ds"
  ```

- `serial-number`/`<string>`（可选）

  指定表示设备序列号的字符串。

- `chassis-type`/`<string>`（可选但推荐）

  指定标识系统外形尺寸的字符串。属性值可以是以下之一：

  - `desktop`
  - `laptop`
  - `convertible`
  - `server`
  - `tablet`
  - `handset`
  - `watch`
  - `embedded`

> | Property Name       | Usage | Value Type     | Definition
> | ------------------- | ----- | -------------- | -
> | `#address-cells`    | R     | `<u32>`        | Specifies the number of `<u32>` cells to represent the address in the reg property in children of root.
> | `#size-cells`       | R     | `<u32>`        | Specifies the number of `<u32>` cells to represent the size in the reg property in children of root.
> | `model`             | R     | `<string>`     | Specifies a string that uniquely identifies the model of the system board. The recommended format is “manufacturer,model-number”.
> | `compatible`        | R     | `<stringlist>` | Specifies a list of platform architectures with which this platform is compatible. This property can be used by operating systems in selecting platform specific code. The recommended form of the property value is: "manufacturer,model". For example: compatible = "fsl,mpc8572ds"
> | `serial-number`     | O     | `<string>`     | Specifies a string representing the device’s serial number.
> | `chassis-type`      | OR    | `<string>`     | Specifies a string that identifies the form-factor of the system. The property value can be one of: "desktop"/"laptop"/"convertible"/"server"/"tablet"/"handset"/"watch"/"embedded"
>
> Usage legend: R=Required, O=Optional, OR=Optional but Recommended, SD=See Definition

> **注意**：其他所有标准属性（2.3 节）都是允许但可选的。
>
> > **Note**: All other standard properties (Section 2.3) are allowed but are optional.

## 3.3 /aliases 节点

> 3.3 /aliases node

设备树可能有一个别名节点（/aliases），它定义了一个或多个别名属性。别名节点应位于设备树的根目录并具有节点名称 /aliases。

> A devicetree may have an aliases node (/aliases) that defines one or more alias properties. The alias node shall be at the root of the devicetree and have the node name /aliases.

/aliases 节点的每个属性都定义了一个别名。属性名称指定别名。属性值指定设备树中节点的完整路径。例如，属性 `serial0 = "/simple-bus@fe000000/serial@llc500"` 定义别名 serial0。

> Each property of the /aliases node defines an alias. The property name specifies the alias name. The property value specifies the full path to a node in the devicetree. For example, the property serial0 = "/simple-bus@fe000000/ serial@llc500" defines the alias serial0.

别名应为以下字符集中的 1 到 31 个字符的小写文本字符串。

> Alias names shall be a lowercase text strings of 1 to 31 characters from the following set of characters.

表 3.2：别名中的有效字符

| 字符（Character） | 描述（Description）
| --- | -
| 0-9 | 数字（digit）
| a-z | 小写字母（lowercase letter）
|  -  | 划线（dash）

别名值是设备路径并被编码为字符串。该值表示节点的完整路径，但路径不需要引用叶节点。

> An alias value is a device path and is encoded as a string. The value represents the full path to a node, but the path does not need to refer to a leaf node.

客户程序可以使用别名属性名称来引用完整的设备路径作为其字符串值的全部或部分。客户程序在将字符串视为设备路径时，应检测并使用别名。

> A client program may use an alias property name to refer to a full device path as all or part of its string value. A client program, when considering a string as a device path, shall detect and use the alias.

### 示例

```dts
aliases {
    serial0 = "/simple-bus@fe000000/serial@llc500";
    ethernet0 = "/simple-bus@fe000000/ethernet@31c000";
};
```

给定别名 serial0，客户端程序可以查看 /aliases 节点并确定别名指的是设备路径 /simple-bus@fe000000/serial@llc500。

> Given the alias serial0, a client program can look at the /aliases node and determine the alias refers to the device path /simple-bus@fe000000/serial@llc500.

## 3.4 /memory 节点

> 3.4 /memory node

所有设备树都需要一个内存设备节点，它描述了系统的物理内存布局。如果系统有多个内存范围，可以创建多个内存节点，或者可以在单个内存节点的 reg 属性中指定范围。

> A memory device node is required for all devicetrees and describes the physical memory layout for the system. If a system has multiple ranges of memory, multiple memory nodes can be created, or the ranges can be specified in the reg property of a single memory node.

节点名称的**节点名**部分（参见[第 2.2.1 节](ch2.md/#221-节点名)）**应当**为 *memory*。

> The *unit-name* component of the node name (see Section 2.2.1) shall be memory.

客户程序可以使用它选择的任何存储属性访问任何内存保留（参见[第 5.3 节](ch5/#53-保留内存块)）未覆盖的内存。但是，在更改用于访问真实页面的存储属性之前，客户程序需要自行执行架构和实现所需的操作，可能包括从缓存中刷新真实页面。引导程序负责确保在不采取与存储属性更改相关联的任何操作的情况下，客户程序可以安全地访问所有内存（包括内存预留覆盖的内存），如同 WIMG = 0b001x。即：

> The client program may access memory not covered by any memory reservations (see Section 5.3) using any storage attributes it chooses. However, before changing the storage attributes used to access a real page, the client program is responsible for performing actions required by the architecture and implementation, possibly including flushing the real page from the caches. The boot program is responsible for ensuring that, without taking any action associated with a change in storage attributes, the client program can safely access all memory (including memory covered by memory reservations) as WIMG = 0b001x. That is:

- 不需要直写
- 不禁止缓存
- 存储连贯性
- 是否保护都行

> - not Write Through Required
> - not Caching Inhibited
> - Memory Coherence
> - Required either not Guarded or Guarded

如果支持 VLE 存储属性，则 VLE=0。

> If the VLE storage attribute is supported, with VLE=0.

表 3.3：/memory 节点属性

> Table 3.3: /memory Node Properties

- `device_type`/`<string>`（必要）

  值必须为 `memory`。

- `reg`/`<prop-encoded-array>`（必要）

  由任意数量的地址和大小对组成，它们指定内存范围的物理地址和大小。

- `initial-mapped-area`/`<prop-encoded-array>`（可选）

  指定初始映射区域的地址和大小。

  是一个prop-encoded-array，由（有效地址，物理地址，大小）的三元组组成。

  有效地址和物理地址均应为 64 位（`<u64>` 值），大小应为 32 位（`<u32>` 值）。

- `hotpluggable`/`<empty>`（可选）

  向操作系统指定一个明确的提示，以后可能会删除此内存。

> | Property Name         | Usage | Value Type             | Definition
> | --------------------- | ----- | ---------------------- | -
> | `device_type`         | R     | `<string>`             | Value shall be “memory”
> | `reg`                 | R     | `<prop-encoded-array>` | Consists of an arbitrary number of address and size pairs that specify the physical address and size of the memory ranges.
> | `initial-mapped-area` | O     | `<prop-encoded-array>` | Specifies the address and size of the Initial Mapped Area. Is a prop-encoded-array consisting of a triplet of (effective address, physical address, size). The effective and physical address shall each be 64-bit (`<u64>` value), and the size shall be 32-bits (`<u32>` value).
> | `hotpluggable`        | O     | `<empty>`              | Specifies an explicit hint to the operating system that this memory may potentially be removed later.
>
> Usage legend: R=Required, O=Optional, OR=Optional but Recommended, SD=See Definition

### 3.4.1 /memory 节点和 UEFI

> 3.4.1 /memory node and UEFI

当通过 \[UEFI\] 启动时，系统内存映射是通过 `GetMemoryMap()` UEFI 启动时服务获得的，如 \[UEFI\] § 7.2 中定义，如果存在，操作系统必须忽略任何 /memory 节点。

> When booting via \[UEFI\], the system memory map is obtained via the GetMemoryMap() UEFI boot time service as defined in \[UEFI\] § 7.2, and if present, the OS must ignore any /memory nodes.

### 3.4.2 /memory 示例

> 3.4.2 /memory Examples

给定具有以下物理内存布局的 64 位 Power 系统：

> Given a 64-bit Power system with the following physical memory layout:

- RAM：从 0x0 开始，长度 0x80000000（2GB）
- RAM：从 0x100000000 开始，长度 0x100000000（4 GB）

> - RAM: starting address 0x0, length 0x80000000 (2 GB)
> - RAM: starting address 0x100000000, length 0x100000000 (4 GB)

假设 `#address-cells = <2>` 和 `#size-cells = <2>`，内存节点可以定义如下。

> Memory nodes could be defined as follows, assuming #address-cells = <2> and #size-cells = <2>.

#### 示例 #1

```dts
memory@0 {
    device_type = "memory";
    reg = <0x000000000 0x00000000 0x00000000 0x80000000
           0x000000001 0x00000000 0x00000001 0x00000000>;
};
```

#### 示例 #2

```dts
memory@0 {
    device_type = "memory";
    reg = <0x000000000 0x00000000 0x00000000 0x80000000>;
};
memory@100000000 {
    device_type = "memory";
    reg = <0x000000001 0x00000000 0x00000001 0x00000000>;
};
```

reg 属性用于定义两个内存范围的地址和大小。2 GB I/O 区域被跳过。请注意，根节点的 `#address-cells` 和 `#size-cells` 属性指定的值为 2，这意味着需要两个 32 位单元来定义内存节点的 reg 属性的地址和长度。

> The reg property is used to define the address and size of the two memory ranges. The 2 GB I/O region is skipped. Note that the #address-cells and #size-cells properties of the root node specify a value of 2, which means that two 32-bit cells are required to define the address and length for the reg property of the memory node.
