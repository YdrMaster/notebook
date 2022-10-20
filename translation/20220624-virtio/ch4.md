# 4 虚拟 IO 设备传输选项

> 4 Virtio Transport Options

virtio 可以使用各种不同的总线，因此该标准分为 virtio 通用部分和各总线专用部分。

> Virtio can use various different buses, thus the standard is split into virtio general and bus-specific sections.

## 4.1 PCI 总线上的 virtio

> 4.1 Virtio Over PCI Bus

todo

## 4.2 内存映射 IO 上的 virtio

> 4.2 Virtio Over MMIO

不支持 PCI 的虚拟环境（虚拟嵌入式设备中的常见情况）可能会使用简单的内存映射设备（“virtio-mmio”）而不是 PCI 设备。

> Virtual environments without PCI support (a common situation in embedded devices models) might use simple memory mapped device (“virtio-mmio”) instead of the PCI device.

内存映射的 virtio 设备行为基于 PCI 设备规范。因此，包括设备初始化、队列配置和缓冲区传输在内的大多数操作几乎相同。以下部分描述了现有差异。

> The memory mapped virtio device behaviour is based on the PCI device specification. Therefore most operations including device initialization, queues configuration and buffer transfers are nearly identical. Existing differences are described in the following sections.

### 4.2.1 内存映射 IO 设备发现

> 4.2.1 MMIO Device Discovery

与 PCI 不同，MMIO 不提供通用设备发现机制。对于每个设备，客户操作系统需要知道所使用的寄存器和中断的位置。此示例显示了使用扁平设备树的系统的推荐定义：

> Unlike PCI, MMIO provides no generic device discovery mechanism. For each device, the guest OS will need to know the location of the registers and interrupt(s) used. The suggested binding for systems using flattened device trees is shown in this example:

```dts
// 示例：virtio_block 设备占用 0x1e000 开始的 512 字节和中断号 42。
// EXAMPLE: virtio_block device taking 512 bytes at 0x1e000, interrupt 42.
virtio_block@1e000 {
    compatible = "virtio,mmio";
    reg = <0x1e000 0x200>;
    interrupts = <42>;
}
```

### 4.2.2 内存映射 IO 设备寄存器布局

> 4.2.2 MMIO Device Register Layout

内存映射 virtio 设备提供一组内存映射控制寄存器，后跟一个设备特定的配置空间，如表 4.1 中所述。

> MMIO virtio devices provide a set of memory mapped control registers followed by a device-specific configuration space, described in the table 4.1.

所有寄存器值都按小端序组织。

> All register values are organized as Little Endian.

todo

#### 4.2.2.1 设备要求：内存映射 IO 设备寄存器布局

> 4.2.2.1 Device Requirements: MMIO Device Register Layout

设备**必须**在 MagicValue 中返回 0x74726976。

> The device MUST return 0x74726976 in MagicValue.

设备**必须**在版本中返回值 0x2。

> The device MUST return value 0x2 in Version.

从发生的那一刻起，设备**必须**通过设置 InterruptStatus 中的相应位来呈现每个事件，直到驱动程序通过将相应的位掩码写入 InterruptACK 寄存器来确认中断。不表示事件的位**必须**为零。

> The device MUST present each event by setting the corresponding bit in InterruptStatus from the moment it takes place, until the driver acknowledges the interrupt by writing a corresponding bit mask to the InterruptACK register. Bits which do not represent events which took place MUST be zero.

复位后，设备**必须**清除设备中所有队列的 InterruptStatus 中的所有位和 QueueReady 寄存器中的就绪位。

> Upon reset, the device MUST clear all bits in InterruptStatus and ready bits in the QueueReady register for all queues in the device.

如果存在驱动程序看到不一致配置状态的任何风险，则设备**必须**更改 ConfigGeneration 中返回的值。

> The device MUST change value returned in ConfigGeneration if there is any risk of a driver seeing an inconsistent configuration state.

当 QueueReady 为零（0x0）时，设备**不得**访问虚拟队列内容。

> The device MUST NOT access virtual queue contents when QueueReady is zero (0x0).

如果已协商 VIRTIO_F_RING_RESET，则设备**必须**在重置时在 QueueReset 中显示 0。

> If VIRTIO_F_RING_RESET has been negotiated, the device MUST present a 0 in QueueReset on reset.

如果已协商 VIRTIO_F_RING_RESET，则在使用 QueueReady 启用 virtqueue 后，设备**必须**在 QueueReset 中显示 0。

> If VIRTIO_F_RING_RESET has been negotiated, The device MUST present a 0 in QueueReset after the virtqueue is enabled with QueueReady.

一旦向 QueueReset 写 1，设备必须重置队列。只要队列重置正在进行，设备**必须**继续在 QueueReset 中显示 1。当队列重置完成时，设备**必须**在 QueueReset 和 QueueReady 中都显示 0。（见 2.6.1）。

> The device MUST reset the queue when 1 is written to QueueReset. The device MUST continue to present 1 in QueueReset as long as the queue reset is ongoing. The device MUST present 0 in both QueueReset and QueueReady when queue reset has completed. (see 2.6.1).

#### 4.2.2.2 驱动要求：内存映射设备寄存器布局

> 4.2.2.2 Driver Requirements: MMIO Device Register Layout

驱动程序**不得**访问未在表 4.1 中描述的内存位置（或者，在配置空间的情况下，在设备规范中描述），不得写入只读寄存器（方向 R）并且不得从只写寄存器（方向 W）中读取。

> The driver MUST NOT access memory locations not described in the table 4.1 (or, in case of the configuration space, described in the device specification), MUST NOT write to the read-only registers (direction R) and MUST NOT read from the write-only registers (direction W).

驱动程序**必须**仅使用 32 位宽和对齐的读取和写入来访问表 4.1 中描述的控制寄存器。对于特定于设备的配置空间，驱动程序必须对 8 位宽字段使用 8 位宽访问，对于 16 位宽字段使用 16 位宽和对齐访问，对于 32 和 64 位宽字段使用 32 位宽和对齐访问。

> The driver MUST only use 32 bit wide and aligned reads and writes to access the control registers described in table 4.1. For the device-specific configuration space, the driver MUST use 8 bit wide accesses for 8 bit wide fields, 16 bit wide and aligned accesses for 16 bit wide fields and 32 bit wide and aligned accesses for 32 and 64 bit wide fields.

驱动程序**必须**忽略具有不是 0x74726976 的 MagicValue 的设备，即使它**可能**会报告错误。

> The driver MUST ignore a device with MagicValue which is not 0x74726976, although it MAY report an error.

驱动程序**必须**忽略版本不是 0x2 的设备，即使它**可能**会报告错误。

> The driver MUST ignore a device with Version which is not 0x2, although it MAY report an error.

驱动程序**必须**忽略 DeviceID 为 0x0 的设备，但**不得**报告任何错误。

> The driver MUST ignore a device with DeviceID 0x0, but MUST NOT report any error.

在从 DeviceFeatures 读取之前，驱动程序**必须**向 DeviceFeaturesSel 写入一个值。

> Before reading from DeviceFeatures, the driver MUST write a value to DeviceFeaturesSel.

在写入 DriverFeatures 寄存器之前，驱动程序**必须**向 DriverFeaturesSel 寄存器写入一个值。

> Before writing to the DriverFeatures register, the driver MUST write a value to the DriverFeaturesSel register.

驱动程序**必须**向 QueueNum 写入一个值，该值小于或等于设备在 QueueNumMax 中提供的值。

> The driver MUST write a value to QueueNum which is less than or equal to the value presented by the device in QueueNumMax.

当 QueueReady 不为零时，驱动程序**不得**访问 QueueNum、QueueDescLow、QueueDescHigh、QueueDriverLow、QueueDriverHigh、QueueDeviceLow、QueueDeviceHigh。

> When QueueReady is not zero, the driver MUST NOT access QueueNum, QueueDescLow, QueueDescHigh, QueueDriverLow, QueueDriverHigh, QueueDeviceLow, QueueDeviceHigh.

要停止使用队列，驱动程序必须将零（0x0）写入此 QueueReady 并且**必须**读回该值以确保同步。

> To stop using the queue the driver MUST write zero (0x0) to this QueueReady and MUST read the value back to ensure synchronization.

驱动程序**必须**忽略 InterruptStatus 中的未定义位。

> The driver MUST ignore undefined bits in InterruptStatus.

驱动程序**必须**在完成处理中断时将一个带有位掩码的值写入 InterruptACK 中，该位掩码描述它处理的事件，并且**不得**设置该值中的任何未定义位。

> The driver MUST write a value with a bit mask describing events it handled into InterruptACK when it finishes handling an interrupt and MUST NOT set any of the undefined bits in the value.

如果 VIRTIO_F_RING_RESET 已经协商好，在驱动程序向 QueueReset 写入 1 以重置队列后，驱动程序**不得**认为队列重置完成，直到它在 QueueReset 中读回 0。在确保其他 virtqueue 字段已正确设置后，驱动程序可以通过向 QueueReady 写入 1 来重新启用队列。驱动程序**可以**将驱动程序可写队列配置值设置为与队列重置之前使用的值不同的值。（见 2.6.1）。

> If VIRTIO_F_RING_RESET has been negotiated, after the driver writes 1 to QueueReset to reset the queue, the driver MUST NOT consider queue reset to be complete until it reads back 0 in QueueReset. The driver MAY re-enable the queue by writing 1 to QueueReady after ensuring that other virtqueue fields have been set up correctly. The driver MAY set driver-writeable queue configuration values to different values than those that were used before the queue reset. (see 2.6.1).

### 4.2.3 内存映射 IO 专用的初始化和设备操作

> 4.2.3 MMIO-specific Initialization And Device Operation

#### 4.2.3.1 设备初始化

> 4.2.3.1 Device Initialization

##### 4.2.3.1.1 驱动要求：设备初始化

> 4.2.3.1.1 Driver Requirements: Device Initialization

驱动程序**必须**通过从 MagicValue 和 Version 读取和检查值来启动设备初始化。

> The driver MUST start the device initialization by reading and checking values from MagicValue and Version.

如果两个值都有效，它**必须**读取 DeviceID，如果它的值为零（0x0）必须中止初始化并且不得访问任何其他寄存器。

> If both values are valid, it MUST read DeviceID and if its value is zero (0x0) MUST abort initialization and MUST NOT access any other register.

不期望共享内存的驱动程序**不得**使用共享内存寄存器。

> Drivers not expecting shared memory MUST NOT use the shared memory registers.

进一步的初始化**必须**遵循 3.1 设备初始化中描述的程序。

> Further initialization MUST follow the procedure described in 3.1 Device Initialization.

#### 4.2.3.2 虚拟队列配置

> 4.2.3.2 Virtqueue Configuration

驱动程序通常会通过以下方式初始化虚拟队列：

> The driver will typically initialize the virtual queue in the following way:

1. 通过将其索引（第一个队列为 0）写入 QueueSel 选择队列。
2. 检查队列是否尚未使用：读取 QueueReady，并期望返回值为零 (0x0)。
3. 从 QueueNumMax 中读取最大队列大小（元素数）。如果返回值为零 (0x0)，则队列不可用。
4. 分配队列内存并将其归零，确保内存在物理上是连续的。
5. 通过将大小写入 QueueNum 来通知设备队列大小。
6. 将队列的 Descriptor Area、Driver Area 和 Device Area 的物理地址分别写入 QueueDescLow/QueueDescHigh、QueueDriverLow/QueueDriverHigh 和 QueueDeviceLow/QueueDeviceHigh 寄存器对。
7. 将 0x1 写入 QueueReady。

> 1. Select the queue writing its index (first queue is 0) to QueueSel.
> 2. Check if the queue is not already in use: read QueueReady, and expect a returned value of zero (0x0).
> 3. Read maximum queue size (number of elements) from QueueNumMax. If the returned value is zero (0x0) the queue is not available.
> 4. Allocate and zero the queue memory, making sure the memory is physically contiguous.
> 5. Notify the device about the queue size by writing the size to QueueNum.
> 6. Write physical addresses of the queue’s Descriptor Area, Driver Area and Device Area to (respectively) the QueueDescLow/QueueDescHigh, QueueDriverLow/QueueDriverHigh and QueueDeviceLow/QueueDeviceHigh register pairs.
> 7. Write 0x1 to QueueReady.

#### 4.2.3.3 可用缓冲区通知

> 4.2.3.3 Available Buffer Notifications

当 VIRTIO_F_NOTIFICATION_DATA 尚未协商时，驱动程序通过将待通知队列的 16 位 virtqueue 索引写入 QueueNotify 来向设备发送可用缓冲区通知。

> When VIRTIO_F_NOTIFICATION_DATA has not been negotiated, the driver sends an available buffer notification to the device by writing the 16-bit virtqueue index of the queue to be notified to QueueNotify.

当 VIRTIO_F_NOTIFICATION_DATA 协商完成后，驱动程序通过将以下 32 位值写入 QueueNotify 来向设备发送可用缓冲区通知：

> When VIRTIO_F_NOTIFICATION_DATA has been negotiated, the driver sends an available buffer notification to the device by writing the following 32-bit value to QueueNotify:

```c
le32 {
    vqn : 16;
    next_off : 15;
    next_wrap : 1;
};
```

有关组件的定义，请参见 2.9 驱动程序通知。

> See 2.9 Driver Notifications for the definition of the components.

#### 4.2.3.4 来自设备的通知

> 4.2.3.4 Notifications From The Device

内存映射的 virtio 设备使用单个专用中断信号，当设置了 InterruptStatus 描述中描述的至少一个位时，该信号被设置。这是设备向设备发送已用缓冲区通知或配置更改通知的方式。

> The memory mapped virtio device is using a single, dedicated interrupt signal, which is asserted when at least one of the bits described in the description of InterruptStatus is set. This is how the device sends a used buffer notification or a configuration change notification to the device.

##### 4.2.3.4.1 驱动程序要求：来自设备的通知

> 4.2.3.4.1 Driver Requirements: Notifications From The Device

收到中断后，驱动程序必须读取 InterruptStatus 以检查导致中断的原因（参见寄存器说明）。设置的已用缓冲区通知位应该被解释为每个活动虚拟队列的已用缓冲区通知。处理完中断后，驱动程序必须通过将对应于已处理事件的位掩码写入 InterruptACK 寄存器来确认它。

> After receiving an interrupt, the driver MUST read InterruptStatus to check what caused the interrupt (see the register description). The used buffer notification bit being set SHOULD be interpreted as a used buffer notification for each active virtqueue. After the interrupt is handled, the driver MUST acknowledge it by writing a bit mask corresponding to the handled events to the InterruptACK register.

todo
