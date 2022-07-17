# 设备类型

> 5 Device Types

基于 virtio 内置的队列、配置空间和功能协商机制，几种设备被定义出来。

> On top of the queues, config space and feature negotiation facilities built into virtio, several devices are defined.

以下设备 ID 用于标识不同类型的 virtio 设备。 一些设备 ID 是为本标准中当前未定义的设备保留的。

> The following device IDs are used to identify different types of virtio devices. Some device IDs are reserved for devices which are not currently defined in this standard.

如何发现可用的设备及其类型取决于总线。

> Discovering what devices are available and their type is bus-dependent.

> **DEVICE TYPE ID TABLE**

本文档未指定上述某些设备，因为它们被视为不成熟或特别小众。请注意，有些仅由唯一的现有实现指定；它们可能成为未来规范的一部分，被完全放弃，或存在于该标准之外。我们将不再谈论它们。

> Some of the devices above are unspecified by this document, because they are seen as immature or especially niche. Be warned that some are only specified by the sole existing implementation; they could become part of a future specification, be abandoned entirely, or live on outside this standard. We shall speak of them no further.

## 5.1 网络设备

> 5.1 Network Device

virtio 网络设备是一个虚拟以太网卡，是目前 virtio 支持的最复杂的设备。它已迅速增强，并清楚地展示了如何使现有设备支持新的功能。空缓冲区被放置在一个虚拟队列中用于接收数据包，而传出的数据包被排入另一个队列以按此顺序传输。第三个命令队列用于控制高级过滤功能。

> The virtio network device is a virtual ethernet card, and is the most complex of the devices supported so far by virtio. It has enhanced rapidly and demonstrates clearly how support for new features are added to an existing device. Empty buffers are placed in one virtqueue for receiving packets, and outgoing packets are enqueued into another for transmission in that order. A third command queue is used to control advanced filtering features.

### 5.1.1 设备 ID

> 5.1.1 Device ID

1

### 5.1.2 虚拟队列

> 5.1.2 Virtqueues

**0** 接收队列 1
**1** 传输队列 1
...
**2(N-1)** 接收队列 N
**2(N-1)+1** 传输队列 N
**2N** 控制队列

> **0** receiveq1
> **1** transmitq1
> ...
> **2(N-1)** receiveqN
> **2(N-1)+1** transmitqN
> **2N** controlq

如果 VIRTIO_NET_F_MQ 和 VIRTIO_NET_F_RSS 均未协商，则 N=1，否则 N 通过 max_virtqueue_pairs 设置。

> N=1 if neither VIRTIO_NET_F_MQ nor VIRTIO_NET_F_RSS are negotiated, otherwise N is set by max_virtqueue_pairs.

控制队列仅在 VIRTIO_NET_F_CTRL_VQ 设置时存在。

> controlq only exists if VIRTIO_NET_F_CTRL_VQ set.

### 5.1.3 功能位

> 5.1.3 Feature bits

**VIRTIO_NET_F_CSUM (0)** 设备处理带有部分校验和的数据包。这种“校验和卸载”是现代网卡的常见功能。

> **VIRTIO_NET_F_CSUM (0)** Device handles packets with partial checksum. This “checksum offload” is a common feature on modern network cards.

**VIRTIO_NET_F_GUEST_CSUM (1)** 驱动程序处理带有部分校验和的数据包。

> **VIRTIO_NET_F_GUEST_CSUM (1)** Driver handles packets with partial checksum.

**VIRTIO_NET_F_CTRL_GUEST_OFFLOADS (2)** 控制通道卸载重配置支持。

> **VIRTIO_NET_F_CTRL_GUEST_OFFLOADS (2)** Control channel offloads reconfiguration support.

**VIRTIO_NET_F_MTU(3)** 支持设备最大 MTU 报告。如果设备提供，设备会告知驱动程序其最大 MTU 的值。如果协商，驱动程序使用 mtu 作为最大 MTU 值。

> **VIRTIO_NET_F_MTU(3)** Device maximum MTU reporting is supported. If offered by the device, device advises driver about the value of its maximum MTU. If negotiated, the driver uses mtu as the maximum MTU value.

**VIRTIO_NET_F_MAC (5)** 设备已给出 MAC 地址。

> **VIRTIO_NET_F_MAC (5)** Device has given MAC address.

**VIRTIO_NET_F_GUEST_TSO4 (7)** 驱动可以接收 TSOv4。

> **VIRTIO_NET_F_GUEST_TSO4 (7)** Driver can receive TSOv4.

**VIRTIO_NET_F_GUEST_TSO6 (8)** 驱动可以接收 TSOv6。

> **VIRTIO_NET_F_GUEST_TSO6 (8)** Driver can receive TSOv6.

**VIRTIO_NET_F_GUEST_ECN (9)** 驱动可以接收带有 ECN 的 TSO。

> **VIRTIO_NET_F_GUEST_ECN (9)** Driver can receive TSO with ECN.

**VIRTIO_NET_F_GUEST_UFO (10)** 驱动可以接收UFO。

> **VIRTIO_NET_F_GUEST_UFO (10)** Driver can receive UFO.

**VIRTIO_NET_F_HOST_TSO4 (11)** 设备可以接收 TSOv4。

> **VIRTIO_NET_F_HOST_TSO4 (11)** Device can receive TSOv4.

**VIRTIO_NET_F_HOST_TSO6 (12)** 设备可以接收 TSOv6。

> **VIRTIO_NET_F_HOST_TSO6 (12)** Device can receive TSOv6.

**VIRTIO_NET_F_HOST_ECN (13)** 设备可以接收带有 ECN 的 TSO。

> **VIRTIO_NET_F_HOST_ECN (13)** Device can receive TSO with ECN.

**VIRTIO_NET_F_HOST_UFO (14)** 设备可以接收到 UFO。

> **VIRTIO_NET_F_HOST_UFO (14)** Device can receive UFO.

**VIRTIO_NET_F_MRG_RXBUF (15)** 驱动可以合并接收缓冲区。

> **VIRTIO_NET_F_MRG_RXBUF (15)** Driver can merge receive buffers.

**VIRTIO_NET_F_STATUS (16)** 配置状态字段可用。

> **VIRTIO_NET_F_STATUS (16)** Configuration status field is available.

**VIRTIO_NET_F_CTRL_VQ (17)** 控制通道可用。

> **VIRTIO_NET_F_CTRL_VQ (17)** Control channel is available.

**VIRTIO_NET_F_CTRL_RX (18)** 控制通道 RX 模式支持。

> **VIRTIO_NET_F_CTRL_RX (18)** Control channel RX mode support.

**VIRTIO_NET_F_CTRL_VLAN (19)** 控制通道 VLAN 过滤。

> **VIRTIO_NET_F_CTRL_VLAN (19)** Control channel VLAN filtering.

**VIRTIO_NET_F_GUEST_ANNOUNCE(21)** 驱动程序可以发送免费 ARP。

> **VIRTIO_NET_F_GUEST_ANNOUNCE(21)** Driver can send gratuitous packets.

**VIRTIO_NET_F_MQ(22)** 设备支持具有自动接收转向的多队列。

> **VIRTIO_NET_F_MQ(22)** Device supports multiqueue with automatic receive steering.

**VIRTIO_NET_F_CTRL_MAC_ADDR(23)** 通过控制通道设置 MAC 地址。

> **VIRTIO_NET_F_CTRL_MAC_ADDR(23)** Set MAC address through control channel.

**VIRTIO_NET_F_HOST_USO (56)** 设备可以接收 USO 数据包。与 UFO（对数据包进行分段）不同，当每个较小的数据包都具有 UDP 标头时，USO 将大型 UDP 数据包拆分为几个段。

> **VIRTIO_NET_F_HOST_USO (56)** Device can receive USO packets. Unlike UFO (fragmenting the packet) the USO splits large UDP packet to several segments when each of these smaller packets has UDP header.

**VIRTIO_NET_F_HASH_REPORT(57)** 设备可以报告每个数据包的哈希值和计算的哈希类型。

> **VIRTIO_NET_F_HASH_REPORT(57)** Device can report per-packet hash value and a type of calculated hash.

**VIRTIO_NET_F_GUEST_HDRLEN(59)** 驱动程序可以提供准确的 hdr_len 值。设备受益于知道确切的标头长度。

> **VIRTIO_NET_F_GUEST_HDRLEN(59)** Driver can provide the exact hdr_len value. Device benefits from knowing the exact header length.

**VIRTIO_NET_F_RSS(60)** 设备支持带有 Toeplitz 散列计算和可配置散列参数的 RSS（接收端缩放），用于接收控制。

> **VIRTIO_NET_F_RSS(60)** Device supports RSS (receive-side scaling) with Toeplitz hash calculation and configurable hash parameters for receive steering.

**VIRTIO_NET_F_RSC_EXT(61)** 设备可以处理重复的 ACK 并报告合并段的数量和重复的 ACK。

> **VIRTIO_NET_F_RSC_EXT(61)** Device can process duplicated ACKs and report number of coalesced segments and duplicated ACKs.

**VIRTIO_NET_F_STANDBY(62)** 设备可以作为具有相同 MAC 地址的主设备的备用设备。

> **VIRTIO_NET_F_STANDBY(62)** Device may act as a standby for a primary device with the same MAC address.

**VIRTIO_NET_F_SPEED_DUPLEX(63)** 设备报告速度和双工。

> **VIRTIO_NET_F_SPEED_DUPLEX(63)** Device reports speed and duplex.

#### 5.1.3.1 功能位依赖关系

> 5.1.3.1 Feature bit requirements

一些网络功能位需要其他网络功能位（参见 2.2.1）：

> Some networking feature bits require other networking feature bits (see 2.2.1):

**VIRTIO_NET_F_GUEST_TSO4** Requires **VIRTIO_NET_F_GUEST_CSUM**.
**VIRTIO_NET_F_GUEST_TSO6** Requires **VIRTIO_NET_F_GUEST_CSUM**.
**VIRTIO_NET_F_GUEST_ECN** Requires **VIRTIO_NET_F_GUEST_TSO4** or **VIRTIO_NET_F_GUEST_TSO6**.
**VIRTIO_NET_F_GUEST_UFO** Requires **VIRTIO_NET_F_GUEST_CSUM**.
**VIRTIO_NET_F_HOST_TSO4** Requires **VIRTIO_NET_F_CSUM**.
**VIRTIO_NET_F_HOST_TSO6** Requires **VIRTIO_NET_F_CSUM**.
**VIRTIO_NET_F_HOST_ECN** Requires **VIRTIO_NET_F_HOST_TSO4** or **VIRTIO_NET_F_HOST_TSO6**.
**VIRTIO_NET_F_HOST_UFO** Requires **VIRTIO_NET_F_CSUM**.
**VIRTIO_NET_F_HOST_USO** Requires **VIRTIO_NET_F_CSUM**.
**VIRTIO_NET_F_CTRL_RX** Requires **VIRTIO_NET_F_CTRL_VQ**.
**VIRTIO_NET_F_CTRL_VLAN** Requires **VIRTIO_NET_F_CTRL_VQ**.
**VIRTIO_NET_F_GUEST_ANNOUNCE** Requires **VIRTIO_NET_F_CTRL_VQ**.
**VIRTIO_NET_F_MQ** Requires **VIRTIO_NET_F_CTRL_VQ**.
**VIRTIO_NET_F_CTRL_MAC_ADDR** Requires **VIRTIO_NET_F_CTRL_VQ**.
**VIRTIO_NET_F_RSC_EXT** Requires **VIRTIO_NET_F_HOST_TSO4** or **VIRTIO_NET_F_HOST_TSO6**.
**VIRTIO_NET_F_RSS** Requires **VIRTIO_NET_F_CTRL_VQ**.

**VIRTIO_NET_F_GSO (6)** 设备处理任何 GSO 类型的数据包。这应该表示支持分段卸载，但经过进一步调查，很明显需要多个位。

> **VIRTIO_NET_F_GSO (6)** Device handles packets with any GSO type. This was supposed to indicate segmentation offload support, but upon further investigation it became clear that multiple bits were needed.

**VIRTIO_NET_F_GUEST_RSC4 (41)** 设备合并 TCPIP v4 数据包。这是由管理程序补丁实现的，用于认证目的，当前的 Windows 驱动程序依赖于它。如果 virtio 网络设备报告此功能，它将不起作用。

> **VIRTIO_NET_F_GUEST_RSC4 (41)** Device coalesces TCPIP v4 packets. This was implemented by hypervisor patch for certification purposes and current Windows driver depends on it. It will not function if virtio-net device reports this feature.

**VIRTIO_NET_F_GUEST_RSC6 (42)** 设备合并 TCPIP v6 数据包。类似于 **VIRTIO_NET_F_GUEST_RSC4**。

> **VIRTIO_NET_F_GUEST_RSC6 (42)** Device coalesces TCPIP v6 packets. Similar to VIRTIO_NET_F_GUEST_RSC4.
