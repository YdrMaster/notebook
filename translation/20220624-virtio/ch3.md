# 3 一般初始化和设备操作

> 3 General Initialization And Device Operation

我们从设备初始化的概述开始，然后详细介绍设备的细节以及如何执行每个步骤。本节最好与描述如何与特定设备通信的总线专用部分一起阅读。

> We start with an overview of device initialization, then expand on the details of the device and how each step is preformed. This section is best read along with the bus-specific section which describes how to communicate with the specific device.

## 3.1 设备初始化

> 3.1 Device Initialization

### 3.1.1 驱动需求：设备初始化

> 3.1.1 Driver Requirements: Device Initialization

驱动程序**必须**按照以下顺序初始化设备：

> The driver MUST follow this sequence to initialize a device:

1. 重置设备。
2. 设置 ACKNOWLEDGE 状态位：客户操作系统已注意到设备。
3. 设置 DRIVER 状态位：客户操作系统知道如何驱动设备。
4. 读取设备特征位，并将操作系统和驱动程序理解的特征位子集写入设备。在此步骤中，驱动程序**可以**读取（但**不得**写入）设备专用的配置字段，以在接受之前检查它是否可以支持该设备。
5. 设置 FEATURES_OK 状态位。在此步骤之后，驱动程序**不得**接受新的功能位。
6. 重新读取设备状态以确保 FEATURES_OK 位仍然设置：否则，设备不支持我们的功能子集并且设备不可用。
7. 执行设备专用的设置，包括发现设备的虚队列、可选的 per-bus 设置、读取和可能写入设备的 virtio 配置空间以及填充虚队列。
8. 设置 DRIVER_OK 状态位。此时设备处于“活动状态”。

> 1. Reset the device.
> 2. Set the ACKNOWLEDGE status bit: the guest OS has noticed the device.
> 3. Set the DRIVER status bit: the guest OS knows how to drive the device.
> 4. Read device feature bits, and write the subset of feature bits understood by the OS and driver to the device. During this step the driver MAY read (but MUST NOT write) the device-specific configuration fields to check that it can support the device before accepting it.
> 5. Set the FEATURES_OK status bit. The driver MUST NOT accept new feature bits after this step.
> 6. Re-read device status to ensure the FEATURES_OK bit is still set: otherwise, the device does not support our subset of features and the device is unusable.
> 7. Perform device-specific setup, including discovery of virtqueues for the device, optional per-bus setup, reading and possibly writing the device’s virtio configuration space, and population of virtqueues.
> 8. Set the DRIVER_OK status bit. At this point the device is “live”.

如果这些步骤中的任何一个出现不可恢复的错误，驱动程序**应该**设置 FAILED 状态位以指示它已放弃设备（如果需要，它可以稍后重置设备以重新启动）。在这种情况下，驱动程序**不得**继续初始化。

> If any of these steps go irrecoverably wrong, the driver SHOULD set the FAILED status bit to indicate that it has given up on the device (it can reset the device later to restart if desired). The driver MUST NOT continue initialization in that case.

在设置 DRIVER_OK 之前，驱动程序**不得**向设备发送任何缓冲区可用通知。

> The driver MUST NOT send any buffer available notifications to the device before setting DRIVER_OK.

### 3.1.2 旧版接口：设备初始化

> 3.1.2 Legacy Interface: Device Initialization

旧版设备不支持 FEATURES_OK 状态位，因此设备没有一种优雅的方式来指示不受支持的功能组合。他们也没有提供一个明确的机制来结束功能协商，这意味着设备在首次使用时就确定了功能，并且不能引入任何从根本上改变设备初始操作的功能。

> Legacy devices did not support the FEATURES_OK status bit, and thus did not have a graceful way for the device to indicate unsupported feature combinations. They also did not provide a clear mechanism to end feature negotiation, which meant that devices finalized features on first-use, and no features could be introduced which radically changed the initial operation of the device.

旧版驱动程序实现通常在设置 DRIVER_OK 位之前使用设备，有时甚至在将功能位写入设备之前。

> Legacy driver implementations often used the device before setting the DRIVER_OK bit, and sometimes even before writing the feature bits to the device.

结果是步骤 5 和 6 被省略，而步骤 4、7 和 8 被混为一谈。

> The result was the steps 5 and 6 were omitted, and steps 4, 7 and 8 were conflated.

因此，在使用旧版接口时：

> Therefore, when using the legacy interface:

- 过渡驱动程序**必须**执行 3.1 中描述的初始化序列，但省略步骤 5 和 6。
- 过渡设备**必须**支持驱动程序在步骤 4 之前写入设备配置字段。
- 过渡设备**必须**支持在步骤 8 之前使用设备的驱动程序。

- The transitional driver MUST execute the initialization sequence as described in 3.1 but omitting the steps 5 and 6.
- The transitional device MUST support the driver writing device configuration fields before the step 4.
- The transitional device MUST support the driver using the device before the step 8.
