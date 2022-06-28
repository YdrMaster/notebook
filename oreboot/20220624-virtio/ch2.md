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
- 缓冲区已用通知。

> - configuration change notification
> - available buffer notification
> - used buffer notification.

配置更改通知和缓冲区已用通知由设备发送，接收者是驱动程序。配置改变通知表示设备配置空间发生了变化；缓冲区已用通知表示可能已经在通知指定的虚拟队列上使用了缓冲区。

> Configuration change notifications and used buffer notifications are sent by the device, the recipient is the driver. A configuration change notification indicates that the device configuration space has changed; a used buffer notification indicates that a buffer may have been made used on the virtqueue designated by the notification.

可用缓冲区通知由驱动程序发送，接收者是设备。这种类型的通知表明缓冲区可能已在通知指定的虚拟队列上可用。

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

有些设备总是按照虚拟队列变为可用的顺序使用描述符。这些设备可以提供 VIRTIO_F_IN_ORDER 功能。如果协商，这种特点可能允许优化或简化驱动程序和/或设备代码。

> Some devices always use descriptors in the same order in which they have been made available. These devices can offer the VIRTIO_F_IN_ORDER feature. If negotiated, this knowledge might allow optimizations or simplify driver and/or device code.

每个虚拟队列最多可以包含 3 个部分：

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
> - 可用环 - 用于驱动区
> - 已用环 - 用于设备区
>
> > - Descriptor Table - for the Descriptor Area
> > - Available Ring - for the Driver Area
> > - Used Ring - for the Device Area

支持两种格式：Split Virtqueues（参见 2.7 Split Virtqueues）和 Packed Virtqueues（参见 2.8 Packed Virtqueues）。

> Two formats are supported: Split Virtqueues (see 2.7 Split Virtqueues) and Packed Virtqueues (see 2.8 Packed Virtqueues).

每个驱动程序和设备都支持至少一种格式。

> Every driver and device supports either the Packed or the Split Virtqueue format, or both.

### 2.6.1 虚拟队列重置

> 2.6.1 Virtqueue Reset

当协商 VIRTIO_F_RING_RESET 时，驱动程序可以单独重置一个虚拟队列。重置虚拟队列的方法是特定于传输方式的。

> When VIRTIO_F_RING_RESET is negotiated, the driver can reset a virtqueue individually. The way to reset the virtqueue is transport specific.

虚拟队列的重置分为两部分。驱动程序首先重置队列，然后可以选择重新启用它。

> Virtqueue reset is divided into two parts. The driver first resets a queue and can afterwards optionally reenable it.

#### 2.6.1.1 虚拟队列重置

> 2.6.1.1 Virtqueue Reset

##### 2.6.1.1.1 设备要求：虚拟队列重置

> 2.6.1.1.1 Device Requirements: Virtqueue Reset

驱动程序重置队列后，设备**不得**执行来自该虚拟队列的任何请求，或通知驱动程序。

> After a queue has been reset by the driver, the device MUST NOT execute any requests from that virtqueue, or notify the driver for it.

设备**必须**将虚拟队列的任何状态重置为默认状态，包括可用状态和已使用状态。

> The device MUST reset any state of a virtqueue to the default state, including the available state and the used state.

##### 2.6.1.1.2 驱动要求：虚拟队列重置

> 2.6.1.1.2 Driver Requirements: Virtqueue Reset

驱动程序告诉设备重置队列后，驱动程序**必须**验证队列是否已实际重置。

> After the driver tells the device to reset a queue, the driver MUST verify that the queue has actually been reset.

队列成功重置后，驱动程序**可以**释放与该虚拟队列关联的任何资源。

> After the queue has been successfully reset, the driver MAY release any resource associated with that virtqueue.

#### 2.6.1.2 虚拟队列重启

> 2.6.1.2 Virtqueue Re-enable

这个过程与整个设备初始化过程中单个队列的初始化过程相同。

> This process is the same as the initialization process of a single queue during the initialization of the entire device.

##### 2.6.1.2.1 设备要求：虚拟队列重启

> 2.6.1.2.1 Device Requirements: Virtqueue Re-enable

设备**必须**观察任何可能已被驱动程序更改的队列配置，例如最大队列大小。

> The device MUST observe any queue configuration that may have been changed by the driver, like the maximum queue size.

##### 2.6.1.2.2 驱动要求：虚拟队列重启

> 2.6.1.2.2 Driver Requirements: Virtqueue Re-enable

当重新启用队列时，驱动程序**必须**像在初始虚拟队列发现期间一样配置队列资源，但可以选择使用不同的参数。

> When re-enabling a queue, the driver MUST configure the queue resources as during initial virtqueue discovery, but optionally with different parameters.

## 2.7 拆分虚拟队列

> 2.7 Split Virtqueues

拆分虚拟队列格式是该标准 1.0 版（及更早版本）支持的唯一格式。

> The split virtqueue format was the only format supported by the version 1.0 (and earlier) of this standard.

split virtqueue 格式将虚拟队列分成几个部分，其中每个部分可由驱动程序或设备写入，但不能同时由两者写入。在使缓冲区可用以及将其标记为已使用时，需要更新零件内的多个零件和/或位置。

> The split virtqueue format separates the virtqueue into several parts, where each part is write-able by either the driver or the device, but not both. Multiple parts and/or locations within a part need to be updated when making a buffer available and when marking it as used.

每个队列都有一个 16 位的队列大小参数，它设置条目数并暗示队列的总大小。

> Each queue has a 16-bit queue size parameter, which sets the number of entries and implies the total size of the queue.

每个虚拟队列由三部分组成：

> Each virtqueue consists of three parts:

- 描述符表 - 占据描述符区
- 可用环 - 占据驱动区域
- 已用环 - 占据设备区域

> - Descriptor Table - occupies the Descriptor Area
> - Available Ring - occupies the Driver Area
> - Used Ring - occupies the Device Area

其中每个部分在客户内存中物理上是连续的，并且具有不同的对齐要求。

> where each part is physically-contiguous in guest memory, and has different alignment requirements.

下表总结了虚拟队列各部分的内存对齐和大小要求（以字节为单位）：

> The memory alignment and size requirements, in bytes, of each part of the virtqueue are summarized in the following table:

| 虚拟队列部件 | 对齐 | 尺寸
| ---------- | ---- | -
| 描述符表    | 16   | 16*队列长度
| 可用环      | 2    | 6+2*队列长度
| 已用环      | 4    | 6+8*队列长度

> | Virtqueue Part   | Alignment | Size
> | ---------------- | --------- | -
> | Descriptor Table | 16        | 16*(Queue Size)
> | Available Ring   | 2         | 6+2*(Queue Size)
> | Used Ring        | 4         | 6+8*(Queue Size)

对齐列给出了虚拟队列每个部分的最小对齐。

> The Alignment column gives the minimum alignment for each part of the virtqueue.

尺寸列给出了虚拟队列每个部分的总字节数。

> The Size column gives the total number of bytes for each part of the virtqueue.

队列尺寸对应于虚拟队列^2^中的最大缓冲区数。队列尺寸值始终是 2 的幂。最大队列尺寸值为 32768。指定此值的方式取决于总线。

> Queue Size corresponds to the maximum number of buffers in the virtqueue2. Queue Size value is always a power of 2. The maximum Queue Size value is 32768. This value is specified in a bus-specific way.

当驱动程序想要向设备发送缓冲区时，它会填充描述符表中的一个槽（或将几个链接在一起），并将描述符索引写入可用的环中。然后它通知设备。当设备完成一个缓冲区时，它会将描述符索引写入已使用的环，并发送已使用缓冲区的通知。

> When the driver wants to send a buffer to the device, it fills in a slot in the descriptor table (or chains several together), and writes the descriptor index into the available ring. It then notifies the device. When the device has finished a buffer, it writes the descriptor index into the used ring, and sends a used buffer notification.

### 2.7.1 驱动要求：虚拟队列

> 2.7.1 Driver Requirements: Virtqueues

驱动程序必须确保每个虚拟队列部分的第一个字节的物理地址是上表中指定对齐值的倍数。

> The driver MUST ensure that the physical address of the first byte of each virtqueue part is a multiple of the specified alignment value in the above table.

### 2.7.2 旧版接口：关于虚拟队列布局的说明

> 2.7.2 Legacy Interfaces: A Note on Virtqueue Layout

对于旧版接口，虚拟队列布局有几个额外的限制：

> For Legacy Interfaces, several additional restrictions are placed on the virtqueue layout:

每个虚拟队列占用两个或更多物理上连续的页面（通常定义为 4096 字节，但取决于传输方式；以下称为队列对齐）并由三部分组成：

> Each virtqueue occupies two or more physically-contiguous pages (usually defined as 4096 bytes, but depending on the transport; henceforth referred to as Queue Align) and consists of three parts:

| 描述符表 | 可用环（填充……）| 已用环
| ------- | ------------- | -

> | Descriptor Table | Available Ring (...padding...) | Used Ring
> | ---------------- | ------------------------------ | -

特定于总线的队列大小字段控制虚拟队列的总字节数。当使用旧版接口时，过渡驱动程序必须从设备中检索队列大小字段，并且必须根据以下公式为虚拟队列分配总字节数（队列对齐在 qalign 中给出，队列大小在 qsz 中给出）：

> The bus-specific Queue Size field controls the total number of bytes for the virtqueue. When using the legacy interface, the transitional driver MUST retrieve the Queue Size field from the device and MUST allocate the total number of bytes for the virtqueue according to the following formula (Queue Align given in qalign and Queue Size given in qsz):

```c
#define ALIGN(x) (((x) + qalign) & ~qalign)
static inline unsigned virtq_size(unsigned int qsz)
{
    return ALIGN(sizeof(struct virtq_desc)*qsz + sizeof(u16)*(3 + qsz))
         + ALIGN(sizeof(u16)*3 + sizeof(struct virtq_used_elem)*qsz);
}
```

这会浪费一些填充空间。使用旧接口时，过渡设备和驱动程序都必须使用以下虚拟队列布局结构来定位虚拟队列的元素：

> This wastes some space with padding. When using the legacy interface, both transitional devices and drivers MUST use the following virtqueue layout structure to locate elements of the virtqueue:

```c
struct virtq {
    // The actual descriptors (16 bytes each)
    struct virtq_desc desc[ Queue Size ];

    // A ring of available descriptor heads with free-running index.
    struct virtq_avail avail;

    // Padding to the next Queue Align boundary.
    u8 pad[ Padding ];

    // A ring of used descriptor heads with free-running index.
    struct virtq_used used;
};
```

### 2.7.3 旧版接口：关于虚拟队列字节序的注释

> 2.7.3 Legacy Interfaces: A Note on Virtqueue Endianness

请注意，当使用旧版接口时，过渡设备和驱动程序**必须**使用虚拟机的本机字节序作为字段和虚拟队列中的字节序。这与本标准规定的非传统接口的小端相反。假设主机已经知道虚拟机字节序。

> Note that when using the legacy interface, transitional devices and drivers MUST use the native endian of the guest as the endian of fields and in the virtqueue. This is opposed to little-endian for non-legacy interface as specified by this standard. It is assumed that the host is already aware of the guest endian.

### 2.7.4 消息成帧

> 2.7.4 Message Framing

带有描述符的消息的成帧与缓冲区的内容无关。例如，网络传输缓冲区由一个 12 字节的标头组成，后跟网络数据包。这可以最简单地放在描述符表中，作为 12 字节输出描述符，后跟 1514 字节输出描述符，但在报头和数据包相邻的情况下，它也可以由单个 1526 字节输出描述符组成，甚至三个或更多描述符（在这种情况下可能会降低效率）。

> The framing of messages with descriptors is independent of the contents of the buffers. For example, a network transmit buffer consists of a 12 byte header followed by the network packet. This could be most simply placed in the descriptor table as a 12 byte output descriptor followed by a 1514 byte output descriptor, but it could also consist of a single 1526 byte output descriptor in the case where the header and packet are adjacent, or even three or more descriptors (possibly with loss of efficiency in that case).

请注意，某些设备实现对总描述符大小有很大但合理的限制（例如基于主机操作系统中的 IOV_MAX）。这在实践中不是问题：对于创建不合理大小的描述符（例如将网络数据包分成 1500 个单字节描述符）的驱动程序，我们不会给予同情！

> Note that, some device implementations have large-but-reasonable restrictions on total descriptor size (such as based on IOV_MAX in the host OS). This has not been a problem in practice: little sympathy will be given to drivers which create unreasonably-sized descriptors such as by dividing a network packet into 1500 singlebyte descriptors!

#### 2.7.4.1 设备要求：消息帧

> 2.7.4.1 Device Requirements: Message Framing

设备**不得**对描述符的特定排列做出假设。设备可能有一个链中允许的描述符的合理限制。

> The device MUST NOT make assumptions about the particular arrangement of descriptors. The device MAY have a reasonable limit of descriptors it will allow in a chain.

#### 2.7.4.2 驱动程序要求：消息帧

> 2.7.4.2 Driver Requirements: Message Framing

驱动程序**必须**将任何设备可写描述符元素放在任何设备可读描述符元素之后。

> The driver MUST place any device-writable descriptor elements after any device-readable descriptor elements.

驱动程序**不应该**使用过多的描述符来描述缓冲区。

> The driver SHOULD NOT use an excessive number of descriptors to describe a buffer.

#### 2.7.4.3 旧版接口：消息帧

> 2.7.4.3 Legacy Interface: Message Framing

遗憾的是，最初的驱动程序实现使用了简单的布局，并且设备开始依赖它，尽管有这个规范的措辞。此外，virtio_blk SCSI 命令的规范要求来自帧边界的直观字段长度（参见 5.2.6.3 旧接口：设备操作）。

> Regrettably, initial driver implementations used simple layouts, and devices came to rely on it, despite this specification wording. In addition, the specification for virtio_blk SCSI commands required intuiting field lengths from frame boundaries (see 5.2.6.3 Legacy Interface: Device Operation).

因此，当使用传统接口时，VIRTIO_F_ANY_LAYOUT 功能向设备和驱动程序指示没有对帧进行任何假设。未协商时对过渡驱动程序的要求包含在每个设备部分中。

> Thus when using the legacy interface, the VIRTIO_F_ANY_LAYOUT feature indicates to both the device and the driver that no assumptions were made about framing. Requirements for transitional drivers when this is not negotiated are included in each device section.

### 2.7.5 虚拟队列描述符表

> 2.7.5 The Virtqueue Descriptor Table

描述符表是指驱动程序用于设备的缓冲区。addr 是物理地址，缓冲区可以通过 next 链接。每个描述符都描述了一个缓冲区，该缓冲区对于设备是只读的（“设备可读”）或对于设备是只写的（“设备可写”），但描述符链可以同时包含设备可读和设备可写缓冲区。

> The descriptor table refers to the buffers the driver is using for the device. addr is a physical address, and the buffers can be chained via next. Each descriptor describes a buffer which is read-only for the device (“device-readable”) or write-only for the device (“device-writable”), but a chain of descriptors can contain both device-readable and device-writable buffers.

提供给设备的内存的实际内容取决于设备类型。最常见的做法是在数据开始时带有一个标头（包含小端字段）以供设备读取，并在其后添加一个状态尾标以供设备写入。

> The actual contents of the memory offered to the device depends on the device type. Most common is to begin the data with a header (containing little-endian fields) for the device to read, and postfix it with a status tailer for the device to write.

```c
struct virtq_desc {
    /*Address (guest-physical).*/
    le64 addr;
    /*Length.*/
    le32 len;
    /*This marks a buffer as continuing via the next field.*/
    # define VIRTQ_DESC_F_NEXT 1
    /* This marks a buffer as device write-only (otherwise device read-only). */
    # define VIRTQ_DESC_F_WRITE 2
    /*This means the buffer contains a list of buffer descriptors.*/
    # define VIRTQ_DESC_F_INDIRECT 4
    /* The flags as indicated above. */
    le16 flags;
    /* Next field if flags & NEXT */
    le16 next;
};
```

表中描述符的数量由该虚拟队列的队列大小定义：这是最大可能的描述符链长度。

> The number of descriptors in the table is defined by the queue size for this virtqueue: this is the maximum possible descriptor chain length.

如果 VIRTIO_F_IN_ORDER 已经协商好，驱动程序使用环形顺序的描述符：从表中的偏移量 0 开始，并在表的末尾环绕。

> If VIRTIO_F_IN_ORDER has been negotiated, driver uses descriptors in ring order: starting from offset 0 in the table, and wrapping around at the end of the table.

> 注意：旧的【virtio PCI 草案】将此结构称为 vring_desc，常量称为 VRING_DESC_F_NEXT 等，但布局和值相同。
>
> > Note: The legacy [Virtio PCI Draft] referred to this structure as vring_desc, and the constants as VRING_DESC_F_NEXT, etc, but the layout and values were identical.

#### 2.7.5.1 设备要求：虚拟队列描述符表

> 2.7.5.1 Device Requirements: The Virtqueue Descriptor Table

设备**不得**写入设备可读缓冲区，设备**不应该**读取设备可写缓冲区（它可能出于调试或诊断目的这样做）。设备**不得**写入任何描述符表条目。
、
> A device MUST NOT write to a device-readable buffer, and a device SHOULD NOT read a device-writable buffer (it MAY do so for debugging or diagnostic purposes). A device MUST NOT write to any descriptor table entry.

#### 2.7.5.2 驱动要求：虚拟队列描述符表

> 2.7.5.2 Driver Requirements: The Virtqueue Descriptor Table

驱动程序**不得**添加总长度超过 2^32^ 字节的描述符链；这意味着描述符链中的循环是被禁止的！

> Drivers MUST NOT add a descriptor chain longer than 2^32^ bytes in total; this implies that loops in the descriptor chain are forbidden!

如果 VIRTIO_F_IN_ORDER 已经协商好，并且当在设备可用的表中偏移 x 处设置 VRING_DESC_F_NEXT 标志的描述符时，驱动程序必须将表中的最后一个描述符（即 x = queue_size - 1）的 next 设置为 0，其他描述符的 next 设置为 x + 1。

> If VIRTIO_F_IN_ORDER has been negotiated, and when making a descriptor with VRING_DESC_F_NEXT set in flags at offset x in the table available to the device, driver MUST set next to 0 for the last descriptor in the table (where x = queue_size − 1) and to x + 1 for the rest of the descriptors.

#### 2.7.5.3 间接描述符

> 2.7.5.3 Indirect Descriptors

一些设备通过同时分派大量大型请求而受益。VIRTIO_F_INDIRECT_DESC 特性允许这样做（参见 A virtio_queue.h）。为了增加环容量，驱动程序可以在内存中的任何位置存储一个间接描述符表，并在主虚拟队列（带有 VIRTQ_DESC_F_INDIRECT 标志）中插入一个描述符，该描述符指的是包含该间接描述符表的内存缓冲区；addr 和 len 指的是间接表地址和分别以字节为单位的长度。

> Some devices benefit by concurrently dispatching a large number of large requests. The VIRTIO_F_INDIRECT_DESC feature allows this (see A virtio_queue.h). To increase ring capacity the driver can store a table of indirect descriptors anywhere in memory, and insert a descriptor in main virtqueue (with flags&VIRTQ_DESC_F_INDIRECT on) that refers to memory buffer containing this indirect descriptor table; addr and len refer to the indirect table address and length in bytes, respectively.

间接表布局结构如下所示（`len` 是引用该表的描述符的长度，它是一个变量，因此这段代码无法编译）：

> The indirect table layout structure looks like this (len is the length of the descriptor that refers to this table, which is a variable, so this code won’t compile):

```c
struct indirect_descriptor_table {
    /* 真正的描述符（每个 16 字节） */
    /*The actual descriptors (16 bytes each)*/
    struct virtq_desc desc[len / 16];
};
```

第一个间接描述符位于间接描述符表的开头（索引 0），其他间接描述符由 next 链接。一个没有有效 next 的间接描述符（没有 VIRTQ_DESC_F_NEXT 标志）表示描述符的结束。单个间接描述符表可以包括设备可读和设备可写描述符。

> The first indirect descriptor is located at start of the indirect descriptor table (index 0), additional indirect descriptors are chained by next. An indirect descriptor without a valid next (with flags&VIRTQ_DESC_F_NEXT off) signals the end of the descriptor. A single indirect descriptor table can include both devicereadable and device-writable descriptors.

如果已协商 VIRTIO_F_IN_ORDER，则间接描述符使用顺序索引，按顺序：索引 0 后跟索引 1 后跟索引 2，依此类推。

> If VIRTIO_F_IN_ORDER has been negotiated, indirect descriptors use sequential indices, in-order: index 0 followed by index 1 followed by index 2, etc.

##### 2.7.5.3.1 驱动程序要求：间接描述符

> 2.7.5.3.1 Driver Requirements: Indirect Descriptors

除非协商了 VIRTIO_F_INDIRECT_DESC 功能，否则驱动程序**不得**设置 VIRTQ_DESC_F_INDIRECT 标志。驱动程序**不得**在间接描述符中设置 VIRTQ_DESC_F_INDIRECT 标志（即每个描述符只有一个表）。

> The driver MUST NOT set the VIRTQ_DESC_F_INDIRECT flag unless the VIRTIO_F_INDIRECT_DESC feature was negotiated. The driver MUST NOT set the VIRTQ_DESC_F_INDIRECT flag within an indirect descriptor (ie. only one table per descriptor).

驱动程序**不得**创建长于设备队列大小的描述符链。

> A driver MUST NOT create a descriptor chain longer than the Queue Size of the device.

驱动程序**不得**在标志中同时设置 VIRTQ_DESC_F_INDIRECT 和 VIRTQ_DESC_F_NEXT。

> A driver MUST NOT set both VIRTQ_DESC_F_INDIRECT and VIRTQ_DESC_F_NEXT in flags.

如果已协商 VIRTIO_F_IN_ORDER，则间接描述符**必须**按顺序出现，next 取值为 1 表示第一个描述符，2 表示第二个描述符，以此类推。

> If VIRTIO_F_IN_ORDER has been negotiated, indirect descriptors MUST appear sequentially, with next taking the value of 1 for the 1st descriptor, 2 for the 2nd one, etc.

##### 2.7.5.3.2 设备要求：间接描述符

> 2.7.5.3.2 Device Requirements: Indirect Descriptors

设备**必须**忽略描述符中指向间接表的只写（VIRTQ_DESC_F_WRITE）标志。

> The device MUST ignore the write-only flag (flags&VIRTQ_DESC_F_WRITE) in the descriptor that refers to an indirect table.

设备必须处理零个或多个正常链式描述符后跟带有 VIRTQ_DESC_F_INDIRECT 标志的单个描述符的情况。

> The device MUST handle the case of zero or more normal chained descriptors followed by a single descriptor with flags&VIRTQ_DESC_F_INDIRECT.

> 注意：虽然不寻常（大多数实现要么仅使用非间接描述符创建链，要么使用单个间接元素），但这种布局是有效的。
>
> > Note: While unusual (most implementations either create a chain solely using non-indirect descriptors, or use a single indirect element), such a layout is valid.

### 2.7.6 Virtqueue 可用环

> 2.7.6 The Virtqueue Available Ring

可用环具有以下布局结构：

> The available ring has the following layout structure:

```c
struct virtq_avail {
    # define VIRTQ_AVAIL_F_NO_INTERRUPT 1
    le16 flags;
    le16 idx;
    le16 ring[ /* 队列尺寸 Queue Size */ ];
    le16 used_event; /* 除非 VIRTIO_F_EVENT_IDX Only if VIRTIO_F_EVENT_IDX */
};
```

驱动程序使用可用环为设备提供缓冲区：每个环条目都指向描述符链的头部。它仅由驱动程序写入并由设备读取。

> The driver uses the available ring to offer buffers to the device: each ring entry refers to the head of a descriptor chain. It is only written by the driver and read by the device.

idx 字段指示驱动程序将在环中放置下一个描述符条目的位置（以队列长度为模）。

> idx field indicates where the driver would put the next descriptor entry in the ring (modulo the queue size).

idx 从 0 开始递增。

> This starts at 0, and increases.

> 注意：旧的【Virtio PCI 草案】将此结构称为 vring_avail，常量称为 VRING_AVAIL_F_NO_INTERRUPT，但布局和值相同。
>
> > Note: The legacy [Virtio PCI Draft] referred to this structure as vring_avail, and the constant as VRING_AVAIL_F_NO_INTERRUPT, but the layout and value were identical.

#### 2.7.6.1 驱动要求：虚拟队列可用环

> 2.7.6.1 Driver Requirements: The Virtqueue Available Ring

驱动程序不得减少虚拟队列上的可用 idx（即，无法“取消暴露”缓冲区）。

> A driver MUST NOT decrement the available idx on a virtqueue (ie. there is no way to “unexpose” buffers).

### 2.7.7 抑制已用缓冲区通知

> 2.7.7 Used Buffer Notification Suppression

如果未协商 VIRTIO_F_EVENT_IDX 功能位，则可用环中的标志字段为驱动程序提供了一种简化的机制，告诉设备在使用缓冲区时不要发布通知。否则 used_event 是一种性能更高的替代方案，驱动程序能够指定设备在需要通知之前可以前进多远。

> If the VIRTIO_F_EVENT_IDX feature bit is not negotiated, the flags field in the available ring offers a crude mechanism for the driver to inform the device that it doesn’t want notifications when buffers are used. Otherwise used_event is a more performant alternative where the driver specifies how far the device can progress before a notification is required.

这些通知抑制方法都不可靠，因为它们不与设备同步，但它们可以作为有用的优化。

> Neither of these notification suppression methods are reliable, as they are not synchronized with the device, but they serve as useful optimizations.

#### 2.7.7.1 驱动程序要求：抑制已用缓冲区通知

> 2.7.7.1 Driver Requirements: Used Buffer Notification Suppression

如果未协商 VIRTIO_F_EVENT_IDX 功能位：

> If the VIRTIO_F_EVENT_IDX feature bit is not negotiated:

- 驱动程序**必须**将标志设置为 0 或 1。
- 驱动程序**可以**将标志设置为 1 以通知设备不需要通知。

> - The driver MUST set flags to 0 or 1.
> - The driver MAY set flags to 1 to advise the device that notifications are not needed.

否则，如果协商了 VIRTIO_F_EVENT_IDX 功能位：

> Otherwise, if the VIRTIO_F_EVENT_IDX feature bit is negotiated:

- 驱动程序**必须**将标志设置为 0。
- 驱动程序**可以**使用 used_event 告诉设备通知是不必要的，直到设备将带有 used_event 指定索引的条目写入已使用环（换句话说，直到已使用环中的 idx 将达到值 used_event + 1）。

> - The driver MUST set flags to 0.
> - The driver MAY use used_event to advise the device that notifications are unnecessary until the device writes an entry with an index specified by used_event into the used ring (equivalently, until idx in the used ring will reach the value used_event + 1).

驱动程序**必须**处理来自设备的虚假通知。

> The driver MUST handle spurious notifications from the device.

#### 2.7.7.2 设备要求：抑制已用缓冲区通知

> 2.7.7.2 Device Requirements: Used Buffer Notification Suppression

如果未协商 VIRTIO_F_EVENT_IDX 功能位：

> If the VIRTIO_F_EVENT_IDX feature bit is not negotiated:

- 设备必须忽略 used_event 值。
- 设备将描述符索引写入已用环后：
  - 如果 flags 为 1，则设备**不应**发送通知。
  - 如果 flags 为 0，设备**必须**发送通知。

> - The device MUST ignore the used_event value.
> - After the device writes a descriptor index into the used ring:
>   - If flags is 1, the device SHOULD NOT send a notification.
>   - If flags is 0, the device MUST send a notification.

否则，如果协商了 VIRTIO_F_EVENT_IDX 功能位：

> Otherwise, if the VIRTIO_F_EVENT_IDX feature bit is negotiated:

- 设备**必须**忽略标志的低位。
- 设备将描述符索引写入使用的环后：
  - 如果已用环中的 idx 字段（确定描述符索引的放置位置）等于 used_event，则设备**必须**发送通知。
  - 否则设备**不应**发送通知。

> - The device MUST ignore the lower bit of flags.
> - After the device writes a descriptor index into the used ring:
>   - If the idx field in the used ring (which determined where that descriptor index was placed) was equal to used_event, the device MUST send a notification.
>   - Otherwise the device SHOULD NOT send a notification.

> 注意：例如，如果 used_event 为 0，则使用 VIRTIO_F_EVENT_IDX 的设备将在使用第一个缓冲区后（以及在第 65536 个缓冲区等之后）向驱动程序发送已用缓冲区通知。
>
> > Note: For example, if used_event is 0, then a device using VIRTIO_F_EVENT_IDX would send a used buffer notification to the driver after the first buffer is used (and again after the 65536th buffer, etc).

### 2.7.8 虚拟队列已用环

> 2.7.8 The Virtqueue Used Ring

已用环具有以下布局结构：

> The used ring has the following layout structure:

```c
struct virtq_used {
    #define VIRTQ_USED_F_NO_NOTIFY 1
    le16 flags;
    le16 idx;
    struct virtq_used_elem ring[ /* 队列长度 Queue Size */];
    le16 avail_event; /* 除非 VIRTIO_F_EVENT_IDX Only if VIRTIO_F_EVENT_IDX */
};
/* 出于对齐考虑，此处使用 le32 */
/* le32 is used here for ids for padding reasons. */
struct virtq_used_elem {
    /* 已用描述符链的起始序号 */
    /* Index of start of used descriptor chain. */
    le32 id;
    /*
     * 写入描述符链描述的缓冲区的设备可写部分的字节数。
     *
     * The number of bytes written into the device writable portion of the buffer described by the descriptor chain.
     */
    le32 len;
};
```

已用环是设备在使用完缓冲区后返回缓冲区的地方：它仅由设备写入，并由驱动程序读取。

> The used ring is where the device returns buffers once it is done with them: it is only written to by the device, and read by the driver.

环中的每个条目都是二元组：id 表示描述缓冲区的描述符链的头条目（这与软件先前放置在可用环中的条目匹配），len 表示写入缓冲区的总字节数。

> Each entry in the ring is a pair: id indicates the head entry of the descriptor chain describing the buffer (this matches an entry placed in the available ring by the guest earlier), and len the total of bytes written into the buffer.

> 注意： len 对于使用不受信任的缓冲区的驱动程序特别有用：如果驱动程序不知道设备已写入的确切数量，则驱动程序必须提前将缓冲区归零以确保不会发生数据泄漏。
>
> > Note: len is particularly useful for drivers using untrusted buffers: if a driver does not know exactly how much has been written by the device, the driver would have to zero the buffer in advance to ensure no data leakage occurs.
>
> 例如，网络驱动程序可以将接收到的缓冲区直接交给非特权用户空间应用程序。如果网络设备没有覆盖该缓冲区中的字节，这可能会将已释放内存的内容从其他进程泄漏到应用程序。
>
> > For example, a network driver may hand a received buffer directly to an unprivileged userspace application. If the network device has not overwritten the bytes which were in that buffer, this could leak the contents of freed memory from other processes to the application.

idx 字段指示设备将在环中放置下一个描述符条目的位置（以队列长度为模）。

> idx field indicates where the device would put the next descriptor entry in the ring (modulo the queue size).

idx 从 0 开始递增。

> This starts at 0, and increases.

> 注意：旧的【Virtio PCI 草案】将这些结构称为 vring_used 和 vring_used_elem，常量称为 VRING_USED_F_NO_NOTIFY，但布局和值相同。
>
> > Note: The legacy [Virtio PCI Draft] referred to these structures as vring_used and vring_used_elem, and the constant as VRING_USED_F_NO_NOTIFY, but the layout and value were identical.

#### 2.7.8.1 旧版接口：虚拟队列已用环

> 2.7.8.1 Legacy Interface: The Virtqueue Used Ring

从历史上看，许多驱动程序忽略了 len 值，因此，许多设备错误地设置了 len。因此，在使用旧版接口时，如果可能，最好忽略已用环对象中的 len 值。

> Historically, many drivers ignored the len value, as a result, many devices set len incorrectly. Thus, when using the legacy interface, it is generally a good idea to ignore the len value in used ring entries if possible.

每个设备类型列出了具体的已知问题。

> Specific known issues are listed per device type.

#### 2.7.8.2 设备要求：虚拟队列已用环

> 2.7.8.2 Device Requirements: The Virtqueue Used Ring

设备**必须**在更新使用的 idx 之前设置 len。

> The device MUST set len prior to updating the used idx.

在更新使用的 idx 之前，设备**必须**至少从第一个设备可写缓冲区开始将 len 个字节写入描述符。

> The device MUST write at least len bytes to descriptor, beginning at the first device-writable buffer, prior to updating the used idx.

设备**可以**向描述符写入超过 len 个字节。

> The device MAY write more than len bytes to descriptor.

> 注意：存在潜在的错误情况，设备可能不知道缓冲区的哪些部分已被写入。这就是为什么 len 被允许被低估的原因：这比驱动程序认为未初始化的内存已被覆盖更可取。
>
> > Note: There are potential error cases where a device might not know what parts of the buffers have been written. This is why len is permitted to be an underestimate: that’s preferable to the driver believing that uninitialized memory has been overwritten when it has not.

#### 2.7.8.3 驱动要求：虚拟队列已用环

> 2.7.8.3 Driver Requirements: The Virtqueue Used Ring

驱动程序**不得**对设备可写缓冲区中超出第一个 len 字节的数据做出假设，并且应该忽略此数据。

> The driver MUST NOT make assumptions about data in device-writable buffers beyond the first len bytes, and SHOULD ignore this data.

### 2.7.9 描述符的有序使用

> 2.7.9 In-order use of descriptors

一些设备总是按照描述符变得可用的顺序使用它们。这些设备可以提供 VIRTIO_F_IN_ORDER 功能。如果经过协商，该特性允许设备通过仅写出单个已用环条目（其 ID 对应于描述批次中最后一个缓冲区的描述符链的头条目）来通知驱动程序使用了一批缓冲区。

> Some devices always use descriptors in the same order in which they have been made available. These devices can offer the VIRTIO_F_IN_ORDER feature. If negotiated, this knowledge allows devices to notify the use of a batch of buffers to the driver by only writing out a single used ring entry with the id corresponding to the head entry of the descriptor chain describing the last buffer in the batch.

然后设备根据批次的大小在环中向前跳。因此，它将使用的 idx 增加批次的大小。

> The device then skips forward in the ring according to the size of the batch. Accordingly, it increments the used idx by the size of the batch.

驱动程序需要查找使用的 id 并计算批量大小，以便能够前进到设备将写入下一个使用的环条目的位置。

> The driver needs to look up the used id and calculate the batch size to be able to advance to where the next used ring entry will be written by the device.

这将导致在与批次中的第一个可用环条目匹配的偏移处的已用环条目，在与下一批中的第一个可用环条目匹配的偏移处用于下一批的已用环条目，等等。

> This will result in the used ring entry at an offset matching the first available ring entry in the batch, the used ring entry for the next batch at an offset matching the first available ring entry in the next batch, etc.

假定设备已完全使用（读取或写入）跳过的缓冲区（没有写入使用的环条目）。

> The skipped buffers (for which no used ring entry was written) are assumed to have been used (read or written) by the device completely.

### 2.7.10 抑制可用缓冲区通知

> 2.7.10 Available Buffer Notification Suppression

设备可以抑制可用缓冲区通知的方式类似于驱动程序可以抑制已使用缓冲区通知的方式，详见第 2.7.7 节。设备在已用环中操作标志或 avail_event 的方式与驱动程序在可用环中操作标志或 used_event 的方式相同。

> The device can suppress available buffer notifications in a manner analogous to the way drivers can suppress used buffer notifications as detailed in section 2.7.7. The device manipulates flags or avail_event in the used ring the same way the driver manipulates flags or used_event in the available ring.

#### 2.7.10.1 驱动程序要求：抑制可用缓冲区通知

> 2.7.10.1 Driver Requirements: Available Buffer Notification Suppression

驱动程序**必须**在分配已用环时将已用环中的标志初始化为 0。

> The driver MUST initialize flags in the used ring to 0 when allocating the used ring.

如果未协商 VIRTIO_F_EVENT_IDX 功能位：

> If the VIRTIO_F_EVENT_IDX feature bit is not negotiated:

- 驱动程序必须忽略 avail_event 值。
- 驱动程序将描述符索引写入可用环后：
  - 如果 flags 为 1，驱动程序不应发送通知。
  - 如果 flags 为 0，驱动程序必须发送通知。

> - The driver MUST ignore the avail_event value.
> - After the driver writes a descriptor index into the available ring:
>   - If flags is 1, the driver SHOULD NOT send a notification.
>   - If flags is 0, the driver MUST send a notification.

否则，如果协商了 VIRTIO_F_EVENT_IDX 功能位：

> Otherwise, if the VIRTIO_F_EVENT_IDX feature bit is negotiated:

- 驱动程序必须忽略标志的低位。
- 驱动程序将描述符索引写入可用环后：
  - 如果可用环中的 idx 字段（确定描述符索引放置的位置）等于avail_event，驱动程序必须发送通知。
  - 否则驱动程序不应发送通知。

> - The driver MUST ignore the lower bit of flags.
> - After the driver writes a descriptor index into the available ring:
>   - If the idx field in the available ring (which determined where that descriptor index was placed) was equal to avail_event, the driver MUST send a notification.
>   - Otherwise the driver SHOULD NOT send a notification.

#### 2.7.10.2 设备要求：抑制可用缓冲区通知

> 2.7.10.2 Device Requirements: Available Buffer Notification Suppression

如果未协商 VIRTIO_F_EVENT_IDX 功能位：

> If the VIRTIO_F_EVENT_IDX feature bit is not negotiated:

- 设备必须将标志设置为 0 或 1。
- 设备可以将标志设置为 1，以告知驱动程序不需要通知。

> - The device MUST set flags to 0 or 1.
> - The device MAY set flags to 1 to advise the driver that notifications are not needed.

否则，如果协商了 VIRTIO_F_EVENT_IDX 功能位：

> Otherwise, if the VIRTIO_F_EVENT_IDX feature bit is negotiated:

- 设备必须将标志设置为 0。
- 设备可以使用 avail_event 告诉驱动程序不必发送通知，直到驱动程序将具有由 avail_event 指定的索引的条目写入可用环（也就是说，直到可用环中的 idx 达到值 avail_event + 1）。

> - The device MUST set flags to 0.
> - The device MAY use avail_event to advise the driver that notifications are unnecessary until the driver writes entry with an index specified by avail_event into the available ring (equivalently, until idx in the available ring will reach the value avail_event + 1).

设备必须处理来自驱动程序的虚假通知。

> The device MUST handle spurious notifications from the driver.

### 2.7.11 操作虚拟队列的参考

> 2.7.11 Helpers for Operating Virtqueues

Linux 内核源代码在 include/uapi/linux/virtio_ring.h 中包含上述定义和更有用的辅助例程。IBM 和 Red Hat 根据（3 条款）BSD 许可明确许可它可以被所有其他项目免费使用，`and is reproduced (with slight variation) in A virtio_queue.h`。

> The Linux Kernel Source code contains the definitions above and helper routines in a more usable form, in include/uapi/linux/virtio_ring.h. This was explicitly licensed by IBM and Red Hat under the (3-clause) BSD license so that it can be freely used by all other projects, and is reproduced (with slight variation) in A virtio_queue.h.

### 2.7.12 操作虚拟队列

> 2.7.12 Virtqueue Operation

虚拟队列操作有两个部分：为设备提供新的可用缓冲区，以及处理设备中已使用的缓冲区。

> There are two parts to virtqueue operation: supplying new available buffers to the device, and processing used buffers from the device.

> 注意：例如，最简单的 virtio 网络设备有两个虚拟队列：发送虚拟队列和接收虚拟对垒。驱动程序将传出（设备可读）数据包添加到传输虚拟队列，然后在使用后释放它们。类似地，传入（设备可写）缓冲区被添加到接收虚拟队列，并在使用后进行处理。
>
> > Note: As an example, the simplest virtio network device has two virtqueues: the transmit virtqueue and the receive virtqueue. The driver adds outgoing (device-readable) packets to the transmit virtqueue, and then frees them after they are used. Similarly, incoming (device-writable) buffers are added to the receive virtqueue, and processed after they are used.

以下是更详细地使用 split 虚拟队列格式时对这两个部分的要求。

> What follows is the requirements of each of these two parts when using the split virtqueue format in more detail.

### 2.7.13 为设备提供缓冲区

> 2.7.13 Supplying Buffers to The Device

驱动程序按如下的步骤为设备的某个虚拟队列提供缓冲区：

> The driver offers buffers to one of the device’s virtqueues as follows:

1. 驱动程序将缓冲区放入描述符表中的空闲描述符中，必要时进行链接（参见 2.7.5 虚拟队列描述符表）。
2. 驱动程序将描述符链头的索引放入可用环的下一个环条目中。
3. 如果可以进行批处理，则可以重复执行第 1 步和第 2 步。
4. 驱动程序执行适当的内存屏障，以确保设备在下一步之前看到更新的描述符表和可用环。
5. 可用的 idx 增加了添加到可用环的描述符链头的数量。
6. 驱动程序执行适当的内存屏障，以确保在检查通知抑制之前更新 idx 字段。
7. 如果此类通知未被抑制，驱动程序将向设备发送可用缓冲区通知。

> 1. The driver places the buffer into free descriptor(s) in the descriptor table, chaining as necessary (see 2.7.5 The Virtqueue Descriptor Table).
> 2. The driver places the index of the head of the descriptor chain into the next ring entry of the available ring.
> 3. Steps 1 and 2 MAY be performed repeatedly if batching is possible.
> 4. The driver performs a suitable memory barrier to ensure the device sees the updated descriptor table and available ring before the next step.
> 5. The available idx is increased by the number of descriptor chain heads added to the available ring.
> 6. The driver performs a suitable memory barrier to ensure that it updates the idx field before checking for notification suppression.
> 7. The driver sends an available buffer notification to the device if such notifications are not suppressed.

> 注意，上面的代码没有对可用的环形缓冲区回绕采取预防措施：这是不可能的，因为环形缓冲区与描述符表的大小相同，因此步骤 (1) 将防止这种情况发生。
>
> Note that the above code does not take precautions against the available ring buffer wrapping around: this is not possible since the ring buffer is the same size as the descriptor table, so step (1) will prevent such a condition.

此外，最大队列大小为 32768（适合 16 位的 2 的最高幂），因此 16 位 idx 值始终可以区分满缓冲区和空缓冲区。

> In addition, the maximum queue size is 32768 (the highest power of 2 which fits in 16 bits), so the 16-bit idx value can always distinguish between a full and empty buffer.

以下是更详细的每个阶段的要求。

> What follows is the requirements of each stage in more detail.

#### 2.7.13.1 将缓冲区放入描述符表中

> 2.7.13.1 Placing Buffers Into The Descriptor Table

缓冲区由零个或多个物理连续的设备可读元素组成，然后是零个或多个物理连续的设备可写元素（每个元素至少有一个元素）。该算法将其映射到描述符表中，形成描述符链：

> A buffer consists of zero or more device-readable physically-contiguous elements followed by zero or more physically-contiguous device-writable elements (each has at least one element). This algorithm maps it into the descriptor table to form a descriptor chain:

对于每个缓冲区元素，b：

> for each buffer element, b:

1. 获取下一个空闲描述符表项，d。
2. 设置 d.addr 为 b 开头的物理地址。
3. 将 d.len 设置为 b 的长度。
4. 如果 b 是设备可写的，则将 d.flags 设置为 VIRTQ_DESC_F_WRITE，否则设置为 0。
5. 如果这之后有缓冲元素：
   (a) 将 d.next 设置为下一个空闲描述符元素的索引。
   (b) 设置 d.flags 中的 VIRTQ_DESC_F_NEXT 位。

> 1. Get the next free descriptor table entry, d
> 2. Set d.addr to the physical address of the start of b
> 3. Set d.len to the length of b.
> 4. If b is device-writable, set d.flags to VIRTQ_DESC_F_WRITE, otherwise 0.
> 5. If there is a buffer element after this:
>    (a) Set d.next to the index of the next free descriptor element.
>    (b) Set the VIRTQ_DESC_F_NEXT bit in d.flags.

在实践中，d.next 通常用于链接空闲描述符，并在开始映射之前保留一个单独的计数以检查是否有足够的空闲描述符。

> In practice, d.next is usually used to chain free descriptors, and a separate count kept to check there are enough free descriptors before beginning the mappings.

#### 2.7.13.2 更新可用环

> 2.7.13.2 Updating The Available Ring

描述符链头是上述算法中的第一个 d，即引用缓冲区第一部分的描述符表条目的索引。一个简单的驱动程序实现可以执行以下操作（假设适当的小端转换）：

> The descriptor chain head is the first d in the algorithm above, ie. the index of the descriptor table entry referring to the first part of the buffer. A naive driver implementation MAY do the following (with the appropriate conversion to-and-from little-endian assumed):

```c
avail->ring[avail->idx % qsz] = head;
```

但是，通常驱动程序可以在更新 idx 之前添加许多描述符链（此时它们对设备可见），因此通常会记录驱动程序添加的数量：

> However, in general the driver MAY add many descriptor chains before it updates idx (at which point they become visible to the device), so it is common to keep a counter of how many the driver has added:

```c
avail->ring[(avail->idx + added++) % qsz] = head;
```

#### 2.7.13.3 更新 idx

> 2.7.13.3 Updating idx

idx 总是递增，并在 65536 处自然换行：

> idx always increments, and wraps naturally at 65536:

```c
avail->idx += added;
```

一旦驱动程序更新了可​​用的 idx，就会公开描述符及其内容。设备**可以**访问驱动程序创建的描述符链和它们立即引用的内存。

> Once available idx is updated by the driver, this exposes the descriptor and its contents. The device MAY access the descriptor chains the driver created and the memory they refer to immediately.

##### 2.7.13.3.1 驱动要求：更新 idx

> 2.7.13.3.1 Driver Requirements: Updating idx

驱动程序**必须**在 idx 更新之前执行适当的内存屏障，以确保设备看到最新的副本。

> The driver MUST perform a suitable memory barrier before the idx update, to ensure the device sees the most up-to-date copy.

#### 2.7.13.4 通知设备

> 2.7.13.4 Notifying The Device

设备通知的实际方法是特定于总线的，但通常它可能很昂贵。因此，如果设备不需要这些通知，它可能会抑制这些通知，如第 2.7.10 节所述。

> The actual method of device notification is bus-specific, but generally it can be expensive. So the device MAY suppress such notifications if it doesn’t need them, as detailed in section 2.7.10.

在检查通知是否被抑制之前，驱动程序必须小心公开新的 idx 值。

> The driver has to be careful to expose the new idx value before checking if notifications are suppressed.

##### 2.7.13.4.1 驱动要求：通知设备

> 2.7.13.4.1 Driver Requirements: Notifying The Device

驱动程序必须在读取标志或 avail_event 之前执行适当的内存屏障，以避免错过通知。

> The driver MUST perform a suitable memory barrier before reading flags or avail_event, to avoid missing a notification.

### 2.7.14 从设备接收使用的缓冲区

> 2.7.14 Receiving Used Buffers From The Device

一旦设备使用了描述符引用的缓冲区（读取或写入它们，或两者的一部分，取决于虚拟队列和设备的性质），它会向驱动程序发送已使用的缓冲区通知，如第 2.7.7 节所述。

> Once the device has used buffers referred to by a descriptor (read from or written to them, or parts of both, depending on the nature of the virtqueue and the device), it sends a used buffer notification to the driver as detailed in section 2.7.7.

> 注意：为了获得最佳性能，驱动程序**可以**在处理使用的环时禁用使用的缓冲区通知，但要注意在清空环和重新启用通知之间丢失通知的问题。这通常通过在重新启用通知后重新检查更多使用的缓冲区来处理：
>
> Note: For optimal performance, a driver MAY disable used buffer notifications while processing the used ring, but beware the problem of missing notifications between emptying the ring and reenabling notifications. This is usually handled by re-checking for more used buffers after notifications are reenabled:

```c
virtq_disable_used_buffer_notifications(vq);

for (;;) {
    if (vq->last_seen_used != le16_to_cpu(virtq->used.idx)) {
        virtq_enable_used_buffer_notifications(vq);
        mb();

        if (vq->last_seen_used != le16_to_cpu(virtq->used.idx))
            break;

        virtq_disable_used_buffer_notifications(vq);
    }

    struct virtq_used_elem *e = virtq.used->ring[vq->last_seen_used%vsz];
    process_buffer(e);
    vq->last_seen_used++;
}
```
