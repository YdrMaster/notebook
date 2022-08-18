# 第八章 远程栅栏扩展（扩展号 #0x52464E43 “RFNC”）

> Chapter 8. RFENCE Extension (EID #0x52464E43 "RFNC")

此扩展定义了所有与远程栅栏相关的函数并替换了旧扩展（扩展号 #0x05 - #0x07）。所有函数都遵循二进制编码部分中定义的 `hart_mask`。任何希望使用地址范围（即 **start_addr** 和 **size**）的函数都必须遵守以下对范围参数的约束。

> This extension defines all remote fence related functions and replaces the legacy extensions (EIDs #0x05 - #0x07). All the functions follow `the hart_mask` as defined in binary encoding section. Any function wishes to use range of addresses (i.e. **start_addr** and **size**), have to abide by the below constraints on range parameters.

如果满足以下条件，则远程栅栏函数刷新整个 TLB

> The remote fence function acts as a full TLB flush if

- **start_addr** 和 **size** 都是 0
- **size** 等于 2^XLEN-1

> - **start_addr** and **size** are both 0
> - **size** is equal to 2^XLEN-1

## 8.1 函数：远程指令栅栏（函数号 #0）

> 8.1. Function: Remote FENCE.I (FID #0)

```c
struct sbiret sbi_send_ipi(unsigned long hart_mask,
                           unsigned long hart_mask_base)
```

指示其他硬件线程执行 `FENCE.I` 指令。

> Instructs remote harts to execute `FENCE.I` instruction.

`sbiret.error` 中可能返回的错误码如下表 9 所示。

> The possible error codes returned in sbiret.error are shown in the Table 9 below.

| 错误码 | 描述 | 译文
| ----------- | ---------------------------------------------------- | -
| SBI_SUCCESS | IPI was sent to all the targeted harts successfully. | 处理器间中断已成功发送到所有目标硬件线程。

## 8.2 函数：远程虚地址栅栏（函数号 #1）

> 8.2. Function: Remote SFENCE.VMA (FID #1)

```c
struct sbiret sbi_remote_sfence_vma(unsigned long hart_mask,
                                    unsigned long hart_mask_base,
                                    unsigned long start_addr,
                                    unsigned long size)
```

指示其他硬件线程执行一个或多个 `SFENCE.VMA` 指令，覆盖 **start_addr** 和 **size** 表示的虚地址范围。

> Instructs the remote harts to execute one or more `SFENCE.VMA` instructions, covering the range of virtual addresses between **start** and **size**.

`sbiret.error` 中可能返回的错误码如下表 10 所示。

> The possible error codes returned in sbiret.error are shown in the Table 10 below.

| 错误码 | 描述 | 译文
| ----------- | ---------------------------------------------------- | -
| SBI_SUCCESS | IPI was sent to all the targeted harts successfully. | 处理器间中断已成功发送到所有目标硬件线程。
| SBI_ERR_INVALID_ADDRESS | **start_addr** or **size** is not valid. | **start_address** 或 **size** 不合理。

## 8.3 函数：带地址空间号的远程虚地址栅栏（函数号 #2）

> 8.3. Function: Remote SFENCE.VMA with ASID (FID #2)

```c
struct sbiret sbi_remote_sfence_vma_asid(unsigned long hart_mask,
                                         unsigned long hart_mask_base,
                                         unsigned long start_addr,
                                         unsigned long size,
                                         unsigned long asid)
```

指示其他硬件线程执行一个或多个 `SFENCE.VMA` 指令，覆盖 **start_addr** 和 **size** 表示的虚地址范围。仅涵盖指定的 `ASID`。

> Instruct the remote harts to execute one or more `SFENCE.VMA` instructions, covering the range of virtual addresses between **start** and **size**. This covers only the given `ASID`.

`sbiret.error` 中可能返回的错误码如下表 11 所示。

> The possible error codes returned in sbiret.error are shown in the Table 11 below.

| 错误码 | 描述 | 译文
| ----------- | ---------------------------------------------------- | -
| SBI_SUCCESS | IPI was sent to all the targeted harts successfully. | 处理器间中断已成功发送到所有目标硬件线程。
| SBI_ERR_INVALID_ADDRESS | **start_addr** or **size** is not valid. | **start_address** 或 **size** 不合理。

## 8.4 函数：带虚拟机号的远程 HFENCE.GVMA（函数号 #3）

> 8.4. Function: Remote HFENCE.GVMA with VMID (FID #3)

```c
struct sbiret sbi_remote_hfence_gvma_vmid(unsigned long hart_mask,
                                          unsigned long hart_mask_base,
                                          unsigned long start_addr,
                                          unsigned long size,
                                          unsigned long vmid)
```

> TODO

## 8.5 函数：远程 HFENCE.GVMA（函数号 #4）

> 8.5. Function: Remote HFENCE.GVMA (FID #4)

```c
struct sbiret sbi_remote_hfence_gvma(unsigned long hart_mask,
                                     unsigned long hart_mask_base,
                                     unsigned long start_addr,
                                     unsigned long size)
```

> TODO

## 8.6 函数：带地址空间号的远程 HFENCE.VVMA（函数号 #5）

> 8.6. Function: Remote HFENCE.VVMA with ASID (FID #5)

```c
struct sbiret sbi_remote_hfence_vvma_asid(unsigned long hart_mask,
                                          unsigned long hart_mask_base,
                                          unsigned long start_addr,
                                          unsigned long size,
                                          unsigned long asid)
```

> TODO

## 8.7 函数：远程 HFENCE.VVMA（函数号 #5）

> 8.7. Function: Remote HFENCE.VVMA (FID #6)

```c
struct sbiret sbi_remote_hfence_gvma_vmid(unsigned long hart_mask,
                                          unsigned long hart_mask_base,
                                          unsigned long start_addr,
                                          unsigned long size)
```

> TODO

## 8.8 函数表

> 8.8. Function Listing

| 函数名 | SBI 版本 | 函数号 | 扩展号
| --------------------------- | --- | - | -
| sbi_remote_fence_i          | 0.2 | 0 | 0x52464E43
| sbi_remote_sfence_vma       | 0.2 | 1 | 0x52464E43
| sbi_remote_sfence_vma_asid  | 0.2 | 2 | 0x52464E43
| sbi_remote_hfence_gvma_vmid | 0.2 | 3 | 0x52464E43
| sbi_remote_hfence_gvma      | 0.2 | 4 | 0x52464E43
| sbi_remote_hfence_vvma_asid | 0.2 | 5 | 0x52464E43
| sbi_remote_hfence_vvma      | 0.2 | 6 | 0x52464E43
