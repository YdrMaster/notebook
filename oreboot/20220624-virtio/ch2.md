# 2 virtio 设备的基本功能

> 2 Basic Facilities of a Virtio Device

virtio 设备通过特定于总线的方法发现和识别（参见总线专用部分：4.1 Virtio Over PCI Bus、4.2 Virtio Over MMIO 和 4.3 Virtio Over Channel I/O）。每个设备由以下部分组成：

> A virtio device is discovered and identified by a bus-specific method (see the bus specific sections: 4.1 Virtio Over PCI Bus, 4.2 Virtio Over MMIO and 4.3 Virtio Over Channel I/O). Each device consists of the following parts:

- 设备状态字段
- 功能位
- 通知
- 设备配置空间
- 一个或多个虚拟队列

> - Device status field
> - Feature bits
> - Notifications
> - Device Configuration space
> - One or more virtqueues

## 2.1 设备状态字段

> 2.1 Device Status Field

在驱动程序初始化设备期间，驱动程序遵循 3.1 中指定的步骤顺序。

> During device initialization by a driver, the driver follows the sequence of steps specified in 3.1.

设备状态字段提供此序列已完成步骤的简单底层指示。想象它连接到控制台上的信号灯，指示每个设备的状态是最有用的。定义了以下位（下面按它们通常设置的顺序列出）：

> The device status field provides a simple low-level indication of the completed steps of this sequence. It’s most useful to imagine it hooked up to traffic lights on the console indicating the status of each device. The following bits are defined (listed below in the order in which they would be typically set):

**ACKNOWLEDGE（1）** 表示客户操作系统已找到该设备并将其识别为有效的 virtio 设备。

> **ACKNOWLEDGE (1)** Indicates that the guest OS has found the device and recognized it as a valid virtio device.

**DRIVER（2）** 表示客户操作系统知道如何驱动设备。

> **DRIVER (2)** Indicates that the guest OS knows how to drive the device.

> 注意：在设置该位之前可能会有一个显着（或无限）的延迟。例如，在 Linux 下，驱动程序可以是可加载的模块。
>
> > Note: There could be a significant (or infinite) delay before setting this bit. For example, under Linux, drivers can be loadable modules.

**FAILED（128）** 表示客户机出现问题，它已放弃设备。这可能是一个内部错误，或者驱动程序由于某种原因不喜欢该设备，甚至是设备操作过程中的致命错误。

> **FAILED (128)** Indicates that something went wrong in the guest, and it has given up on the device. This could be an internal error, or the driver didn’t like the device for some reason, or even a fatal error during device operation.

**FEATURES_OK（8）** 表示驱动程序已经确认了它理解的所有特性，并且特性协商完成。

> **FEATURES_OK (8)** Indicates that the driver has acknowledged all the features it understands, and feature negotiation is complete.

**DRIVER_OK（4）** 表示驱动程序已设置并准备好驱动设备。

> **DRIVER_OK (4)** Indicates that the driver is set up and ready to drive the device.

**DEVICE_NEEDS_RESET（64） 表示设备遇到无法恢复的错误。

> **DEVICE_NEEDS_RESET (64)** Indicates that the device has experienced an error from which it can’t recover.

设备状态字段以 0 开始，并在复位期间由设备重新初始化为 0。

> The device status field starts out as 0, and is reinitialized to 0 by the device during reset.

### 2.1.1 驱动要求：设备状态字段

> 2.1.1 Driver Requirements: Device Status Field

驱动程序**必须**更新设备状态，设置位以指示 3.1 中指定的驱动程序初始化序列的已完成步骤。驱动程序**不得**清除设备状态位。如果驱动程序设置了 FAILED 位，驱动程序**必须**稍后在尝试重新初始化之前重置设备。

> The driver MUST update device status, setting bits to indicate the completed steps of the driver initialization sequence specified in 3.1. The driver MUST NOT clear a device status bit. If the driver sets the FAILED bit, the driver MUST later reset the device before attempting to re-initialize.

如果设置了 DEVICE_NEEDS_RESET，驱动程序**不应**依赖设备操作的完成。

> The driver SHOULD NOT rely on completion of operations of a device if DEVICE_NEEDS_RESET is set.

> 注意：例如，如果设置了 DEVICE_NEEDS_RESET，驱动程序不能假设请求将完成，也不能假设它们尚未完成。一个好的实现将尝试通过发出重置来恢复。
>
> Note: For example, the driver can’t assume requests in flight will be completed if DEVICE_NEEDS_RESET is set, nor can it assume that they have not been completed. A good implementation will try to recover by issuing a reset.

2.1.2 设备要求：设备状态字段

> 2.1.2 Device Requirements: Device Status Field

在 DRIVER_OK 之前，设备**不得**消耗缓冲区或向驱动程序发送任何已使用的缓冲区通知。

> The device MUST NOT consume buffers or send any used buffer notifications to the driver before DRIVER_OK.

当设备进入需要重置的错误状态时，**应该**设置 DEVICE_NEEDS_RESET。如果设置了 DRIVER_OK，则在设置 DEVICE_NEEDS_RESET 后，设备必须向驱动程序发送设备配置更改通知。

> The device SHOULD set DEVICE_NEEDS_RESET when it enters an error state that a reset is needed. If DRIVER_OK is set, after it sets DEVICE_NEEDS_RESET, the device MUST send a device configuration change notification to the driver.

## 2.2 功能位

> 2.2 Feature Bits

每个 virtio 设备都提供它所理解的所有功能。在设备初始化期间，驱动程序读取它并告诉设备它接受的子集。重新协商的唯一方法是重置设备。

> Each virtio device offers all the features it understands. During device initialization, the driver reads this and tells the device the subset that it accepts. The only way to renegotiate is to reset the device.

这允许向前和向后兼容：如果设备通过新的功能位进行了增强，旧的驱动程序将不会将该功能位写回设备。同样，如果驱动程序使用设备不支持的功能进行了增强，它会看到未提供新功能。

> This allows for forwards and backwards compatibility: if the device is enhanced with a new feature bit, older drivers will not write that feature bit back to the device. Similarly, if a driver is enhanced with a feature that the device doesn’t support, it see the new feature is not offered.

功能位分配如下：

> Feature bits are allocated as follows:

**0 到 23 和 50 到 127** 位指示特定设备类型定义的功能

> **0 to 23, and 50 to 127** Feature bits for the specific device type

保留 **24 到 40** 功能位，用于扩展队列和特性协商机制

> **24 to 40** Feature bits reserved for extensions to the queue and feature negotiation mechanisms

**41 到 49 和 128** 及以上的功能位保留用于将来的扩展。

> **41 to 49, and 128** and above Feature bits reserved for future extensions.

> 注意：例如，网络设备（即设备 ID 1）的功能位 0 表示该设备支持数据包的校验和。
>
> > Note: For example, feature bit 0 for a network device (i.e. Device ID 1) indicates that the device supports checksumming of packets.

另外，设备配置空间中的新字段也通过提供新的特征位来指示。

> In particular, new fields in the device configuration space are indicated by offering a new feature bit.

### 2.2.1 驱动要求：功能位

> 2.2.1 Driver Requirements: Feature Bits

驱动程序**不得**接受设备未提供的功能，也**不得**接受需要另一个未被接受的功能的功能。

> The driver MUST NOT accept a feature which the device did not offer, and MUST NOT accept a feature which requires another feature which was not accepted.

如果设备不提供它理解的功能，驱动程序**应该**进入向后兼容模式，否则**必须**设置 FAILED 设备状态位并停止初始化。

> The driver SHOULD go into backwards compatibility mode if the device does not offer a feature it understands, otherwise MUST set the FAILED device status bit and cease initialization.

2.2.2 设备要求：功能位

> 2.2.2 Device Requirements: Feature Bits

设备**不得**提供需要另一个未提供的功能的功能。设备**应该**接受驱动程序接受的任何有效的功能子集，否则它**必须**在设备写入 FEATURES_OK 设备状态位时失败。

> The device MUST NOT offer a feature which requires another feature which was not offered. The device SHOULD accept any valid subset of features the driver accepts, otherwise it MUST fail to set the FEATURES_OK device status bit when the driver writes it.

如果设备至少一次成功协商了一组功能（通过在设备初始化期间接受 FEATURES_OK 设备状态位），那么在设备或系统重置后它**不该**拒绝重新协商同一组功能。否则会干扰从挂起中恢复和错误恢复。

> If a device has successfully negotiated a set of features at least once (by accepting the FEATURES_OK device status bit during device initialization), then it SHOULD NOT fail re-negotiation of the same set of features after a device or system reset. Failure to do so would interfere with resuming from suspend and error recovery.

### 2.2.3 旧版接口：关于特性位的说明

> 2.2.3 Legacy Interface: A Note on Feature Bits

过渡驱动程序**必须**通过检测未提供功能位 VIRTIO_F_VERSION_1 来检测旧设备。过渡设备**必须**通过检测驱动程序未确认 VIRTIO_F_VERSION_1 来检测旧版驱动程序。

> Transitional Drivers MUST detect Legacy Devices by detecting that the feature bit VIRTIO_F_VERSION_1 is not offered. Transitional devices MUST detect Legacy drivers by detecting that VIRTIO_F_VERSION_1 has not been acknowledged by the driver.

在这种情况下，设备通过旧接口使用。

> In this case device is used through the legacy interface.

旧接口支持是可选的。因此，过渡和非过渡设备和驱动程序都符合此规范。

> Legacy interface support is OPTIONAL. Thus, both transitional and non-transitional devices and drivers are compliant with this specification.

与过渡设备和驱动程序有关的要求包含在名为“旧版接口”的部分中，如本节。

> Requirements pertaining to transitional devices and drivers is contained in sections named ’Legacy Interface’ like this one.

当设备通过旧接口使用时，过渡设备和过渡驱动程序必须根据旧接口部分中记录的要求进行操作。这些部分中的规范文本通常不适用于非过渡设备。

> When device is used through the legacy interface, transitional devices and transitional drivers MUST operate according to the requirements documented within these legacy interface sections. Specification text within these sections generally does not apply to non-transitional devices.

## 2.3 通知

> 2.3 Notifications

发送通知（驱动程序到设备或设备到驱动程序）的概念在本规范中起着重要作用。通知的作用方式与传输方式有关。

> The notion of sending a notification (driver to device or device to driver) plays an important role in this specification. The modus operandi of the notifications is transport specific.

共有三种类型的通知：

> There are three types of notifications:

- 配置更改通知
- 缓冲区可用通知
- 缓冲区占用通知。

> - configuration change notification
> - available buffer notification
> - used buffer notification.

配置更改通知和缓冲区占用通知由设备发送，接收者是驱动程序。配置改变通知表示设备配置空间发生了变化；缓冲区占用通知表示可能已经在通知指定的虚队列上使用了缓冲区。

> Configuration change notifications and used buffer notifications are sent by the device, the recipient is the driver. A configuration change notification indicates that the device configuration space has changed; a used buffer notification indicates that a buffer may have been made used on the virtqueue designated by the notification.

可用缓冲区通知由驱动程序发送，接收者是设备。这种类型的通知表明缓冲区可能已在通知指定的虚队列上可用。

> Available buffer notifications are sent by the driver, the recipient is the device. This type of notification indicates that a buffer may have been made available on the virtqueue designated by the notification.

不同通知的语义、特定于传输的实现和其他重要方面将在后续章节中详细说明。

> The semantics, the transport-specific implementations, and other important aspects of the different notifications are specified in detail in the following chapters.

大多数传输实现设备使用中断向驱动程序发送的通知。因此，在本规范的先前版本中，这些通知通常被称为中断。本规范中定义的某些名称仍保留此中断术语。有时，术语事件用于指代通知或通知的接收。

> Most transports implement notifications sent by the device to the driver using interrupts. Therefore, in previous versions of this specification, these notifications were often called interrupts. Some names defined in this specification still retain this interrupt terminology. Occasionally, the term event is used to refer to a notification or a receipt of a notification.

## 2.4 设备复位

> 2.4 Device Reset

驱动程序可能希望在不同时间启动设备重置；尤其是在设备初始化和设备清理期间需要这样做。

> The driver may want to initiate a device reset at various times; notably, it is required to do so during device initialization and device cleanup.

驱动程序用于启动重置的机制是特定于传输的。

> The mechanism used by the driver to initiate the reset is transport specific.

### 2.4.1 设备要求：设备复位

> 2.4.1 Device Requirements: Device Reset

设备**必须**在收到复位后将设备状态重新初始化为 0。

> A device MUST reinitialize device status to 0 after receiving a reset.

在通过将设备状态重新初始化为 0 来指示重置完成后，设备**不得**发送通知或与队列交互，直到驱动程序重新初始化设备。

> A device MUST NOT send notifications or interact with the queues after indicating completion of the reset by reinitializing device status to 0, until the driver re-initializes the device.

### 2.4.2 驱动要求：设备复位

> 2.4.2 Driver Requirements: Device Reset

驱动程序在读取设备状态为 0 时**应该**认为驱动程序启动的重置已完成。

> The driver SHOULD consider a driver-initiated reset complete when it reads device status as 0.

## 2.5 设备配置空间

2.5 Device Configuration Space

设备配置空间通常用于很少更改或初始化时间的参数。如果配置字段是可选的，它们的存在由功能位指示：本规范的未来版本可能会通过在尾部添加额外字段来扩展设备配置空间。

> Device configuration space is generally used for rarely-changing or initialization-time parameters. Where configuration fields are optional, their existence is indicated by feature bits: Future versions of this specification will likely extend the device configuration space by adding extra fields at the tail.

> 注意：设备配置空间对多字节字段使用小端格式。
>
> > Note: The device configuration space uses the little-endian format for multi-byte fields.

每次传输还能获得设备配置空间的版本计数，只要对设备配置空间的两次访问可能会看到该空间的不同版本，该版本计数就会发生变化。

> Each transport also provides a generation count for the device configuration space, which will change whenever there is a possibility that two accesses to the device configuration space can see different versions of that space.

### 2.5.1 驱动需求：设备配置空间

> 2.5.1 Driver Requirements: Device Configuration Space

驱动程序**不得**假设从大于 32 位宽的字段中读取是原子的，从多个字段中读取也不是：驱动程序**应该**按下面示范的方式读取设备配置空间字段：

> Drivers MUST NOT assume reads from fields greater than 32 bits wide are atomic, nor are reads from multiple fields: drivers SHOULD read device configuration space fields like so:

```c
u32 before, after;
do {
    before = get_config_generation(device);
    // read config entry/entries.
    after = get_config_generation(device);
} while (after != before);
```

对于可选的配置空间字段，驱动程序**必须**在访问该部分配置空间之前检查是否提供了相应的功能。

> For optional configuration space fields, the driver MUST check that the corresponding feature is offered before accessing that part of the configuration space.

> 注意：有关功能协商的详细信息，请参阅第 3.1 节。
>
> > Note: See section 3.1 for details on feature negotiation.

驱动程序**不得**限制结构大小和设备配置空间大小。相反，驱动程序**应该**只检查设备配置空间是否足够大以包含设备操作所需的字段。

> Drivers MUST NOT limit structure size and device configuration space size. Instead, drivers SHOULD only check that device configuration space is large enough to contain the fields necessary for device operation.

> 注意：例如，如果规范声明设备配置空间“包含单个 8 位字段”，驱动程序应该理解这意味着设备配置空间还可能包含任意数量的尾部填充，并接受任何设备配置空间大小等于或大于指定的 8 位大小。
>
> > Note: For example, if the specification states that device configuration space ’includes a single 8-bit field’ drivers should understand this to mean that the device configuration space might also include an arbitrary amount of tail padding, and accept any device configuration space size equal to or greater than the specified 8-bit size.

### 2.5.2 设备需求：设备配置空间

> 2.5.2 Device Requirements: Device Configuration Space

在驱动程序设置 FEATURES_OK 之前，设备**必须**允许读取任何设备特定的配置字段。这包括以特征位为条件的字段，只要这些特征位由设备提供。

> The device MUST allow reading of any device-specific configuration field before FEATURES_OK is set by the driver. This includes fields which are conditional on feature bits, as long as those feature bits are offered by the device.

### 2.5.3 旧版接口：关于设备配置空间字节序的说明

> 2.5.3 Legacy Interface: A Note on Device Configuration Space endian-ness

请注意，对于旧版接口，设备配置空间通常是客户机的本地字节序，而不是 PCI 的小字节序。标准包含每种设备正确的字节序。

Note that for legacy interfaces, device configuration space is generally the guest’s native endian, rather than PCI’s little-endian. The correct endian-ness is documented for each device.

### 2.5.4 旧版接口：设备配置空间

> 2.5.4 Legacy Interface: Device Configuration Space

旧设备没有配置版本字段，因此如果更新配置，很容易受到竞争条件的影响。这会影响块容量（见 5.2.4）和网络 MAC（见 5.1.4）字段；当使用旧版接口时，驱动程序**应该**多次读取这些字段，直到两次读取产生一致的结果。

> Legacy devices did not have a configuration generation field, thus are susceptible to race conditions if configuration is updated. This affects the block capacity (see 5.2.4) and network mac (see 5.1.4) fields; when using the legacy interface, drivers SHOULD read these fields multiple times until two reads generate a consistent result.

## 2.6 虚拟队列

> 2.6 Virtqueues

本节定义了在 virtio 设备上进行批量数据传输的机制，我们骄傲地称之为虚拟队列（virtqueue）。每个设备可以有零个或多个虚拟队列^1^。

> The mechanism for bulk data transport on virtio devices is pretentiously called a virtqueue. Each device can have zero or more virtqueues^1^.

驱动程序通过将可用缓冲区添加到队列中来使请求对设备可用，即，将描述请求的缓冲区添加到 virtqueue，并可选地触发驱动程序事件，即向设备发送可用缓冲区通知。

> Driver makes requests available to device by adding an available buffer to the queue, i.e., adding a buffer describing the request to a virtqueue, and optionally triggering a driver event, i.e., sending an available buffer notification to the device.

设备执行请求，并在完成时将已使用的缓冲区添加到队列中，即通过将缓冲区标记为已使用来让驱动程序知道。然后设备可以触发设备事件，即向驱动程序发送已用缓冲区通知。

> Device executes the requests and - when complete - adds a used buffer to the queue, i.e., lets the driver know by marking the buffer as used. Device can then trigger a device event, i.e., send a used buffer notification to the driver.

设备报告它为它使用的每个缓冲区写入内存的字节数。这被称为“使用长度”。

> Device reports the number of bytes it has written to memory for each buffer it uses. This is referred to as “used length”.

设备通常不需要按照驱动程序提供缓冲区的相同顺序使用缓冲区。

> Device is not generally required to use buffers in the same order in which they have been made available by the driver.

有些设备总是按照它们可用的顺序使用描述符。这些设备可以提供 VIRTIO_F_IN_ORDER 功能。如果协商，此知识可能允许优化或简化驱动程序和/或设备代码。

> Some devices always use descriptors in the same order in which they have been made available. These devices can offer the VIRTIO_F_IN_ORDER feature. If negotiated, this knowledge might allow optimizations or simplify driver and/or device code.

每个 virtqueue 最多可以包含 3 个部分：

> Each virtqueue can consist of up to 3 parts:

- 描述符区 - 用于描述缓冲区
- 驱动程序区域 - 驱动程序提供给设备的额外数据
- 设备区 - 设备提供给驱动程序的额外数据

> - Descriptor Area - used for describing buffers
> - Driver Area - extra data supplied by driver to the device
> - Device Area - extra data supplied by device to driver

> 注意：请注意，此规范的先前版本对这些部分使用了不同的名称（以下 2.7）：
>
> > Note: Note that previous versions of this spec used different names for these parts (following 2.7):
>
> - 描述符表 - 用于描述符区
> - 可用环 - 用于驾驶员区
> - 二手戒指 - 用于设备区
>
> > - Descriptor Table - for the Descriptor Area
> > - Available Ring - for the Driver Area
> > - Used Ring - for the Device Area

支持两种格式：Split Virtqueues（参见 2.7 Split Virtqueues）和 Packed Virtqueues（参见 2.8 Packed Virtqueues）。
每个驱动程序和设备都支持 Packed 或 Split Virtqueue 格式，或两者都支持。

> Two formats are supported: Split Virtqueues (see 2.7 Split Virtqueues) and Packed Virtqueues (see 2.8 Packed Virtqueues).
Every driver and device supports either the Packed or the Split Virtqueue format, or both.
