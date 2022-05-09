# 第七章 处理器间中断扩展（扩展号 #0x735049 “sPI: s 态处理器间中断”）

> Chapter 7. IPI Extension (EID #0x735049 "sPI: s-mode IPI")

此扩展替换旧扩展（EID #0x04）。其他与处理器间中断相关的旧扩展（0x3）现在已弃用。此扩展中的所有函数都遵循二进制编码部分中定义的 `hart_mask`。

> This extension replaces the legacy extension (EID #0x04). The other IPI related legacy extension(0x3) is deprecated now. All the functions in this extension follow the hart_mask as defined in the binary encoding section.

## 7.1 函数：发送处理期间中断（函数号 #0）

> 5.5. Extension: Send IPI (EID #0x04)

```c
struct sbiret sbi_send_ipi(unsigned long hart_mask,
                           unsigned long hart_mask_base)
```

向 **hart_mask** 中定义的所有硬件线程发送处理器间中断。处理器间中断在接收的硬件线程上表现为特权级软件中断。

> Send an inter-processor interrupt to all the harts defined in **hart_mask**. Interprocessor interrupts manifest at the receiving harts as Supervisor Software Interrupts.

`sbiret.error` 中可能返回的错误码如下表 7 所示。

> The possible error codes returned in sbiret.error are shown in the Table 7 below.

| 错误码 | 描述 | 译文
| ----------- | ---------------------------------------------------- | -
| SBI_SUCCESS | IPI was sent to all the targeted harts successfully. | 处理器间中断已成功发送到所有目标硬件线程。

## 7.2 函数表

> 7.2. Function Listing

| 函数名 | SBI 版本 | 函数号 | 扩展号
| ------------ | --- | - | -
| sbi_send_ipi | 0.2 | 0 | 0x735049
