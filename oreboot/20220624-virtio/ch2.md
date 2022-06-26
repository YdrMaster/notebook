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
