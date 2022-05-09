# 第五章 旧扩展（扩展号 #0x00 - #0x0F）

> Chapter 5. Legacy Extensions (EIDs #0x00 - #0x0F)

与 SBI v0.2（或更高版本）规范相比，旧版 SBI 扩展遵循稍微不同的调用约定，其中：

> The legacy SBI extensions follow a slightly different calling convention as compared to the SBI v0.2 (or higher) specification where:

- `a6` 寄存器中的 SBI 函数号字段被忽略，因为它们被编码为多个 SBI 扩展号。
- 不通过 `a1` 寄存器返回值。
- 除 `a0` 外的所有寄存器在调用期间都由被调用者保护。
- `a0` 寄存器返回的值由 SBI 旧扩展决定。

> - The SBI function ID field in `a6` register is ignored because these are encoded as multiple SBI extension IDs.
> - Nothing is returned in `a1` register.
> - All registers except `a0` must be preserved across an SBI call by the callee.
> - The value returned in `a0` register is SBI legacy extension specific.

SBI 实现在代表特权态访问内存时发生的页面和访问错误被重定向回特权态，其中 `sepc` CSR 指向错误的 `ECALL` 指令。

> The page and access faults taken by the SBI implementation while accessing memory on behalf of the supervisor are redirected back to the supervisor with sepc CSR pointing to the faulting ECALL instruction.

旧的 SBI 扩展已弃用，取而代之的是下面列出的其他扩展。旧版控制台 SBI 函数（`sbi_console_getchar()` 和 `sbi_console_putchar()`）预计将被弃用； 他们没有替代品。

> The legacy SBI extensions is deprecated in favor of the other extensions listed below. The legacy console SBI functions (`sbi_console_getchar()` and `sbi_console_putchar()`) are expected to be deprecated; they have no replacement.

## 5.1 扩展：设置定时器（扩展号 #0x00）

> 5.1. Extension: Set Timer (EID #0x00)

```c
long sbi_set_timer(uint64_t stime_value)
```

设置定时器在 **stime_value** 时间后产生下一个事件。该函数还清除挂起的定时器中断位。

> Programs the clock for next event after **stime_value** time. This function also clears the pending timer interrupt bit.

如果特权态希望清除定时器中断但不安排下一个定时器事件，它可以请求一个无限远的定时器中断（即 `(uint64_t)-1`），或者它可以通过清除 `sie.STIE` 控制状态寄存器位来屏蔽定时器中断。

> If the supervisor wishes to clear the timer interrupt without scheduling the next timer event, it can either request a timer interrupt infinitely far into the future (i.e., (uint64_t)-1), or it can instead mask the timer interrupt by clearing `sie.STIE` CSR bit.

若成功，此 SBI 调用返回 0，否则返回一个实现决定的负错误码。

> This SBI call returns 0 upon success or an implementation specific negative error code.

## 5.2 扩展：控制台写字符（扩展号 #0x01）

> 5.2. Extension: Console Putchar (EID #0x01)

```c
long sbi_console_putchar(int ch)
```

向调试控制台写 **ch** 表示的字符。

> Write data present in **ch** to debug console.

不同于 `sbi_console_getchar()`，如果仍有任何待传输的字符或接收终端尚未准备好接收字节，则此 SBI 调用将阻塞。但是如果控制台根本不存在，字符会被丢弃。

> Unlike sbi_console_getchar(), this SBI call will block if there remain any pending characters to be transmitted or if the receiving terminal is not yet ready to receive the byte. However, if the console doesn’t exist at all, then the character is thrown away.

若成功，此 SBI 调用返回 0，否则返回一个实现决定的负错误码。

> This SBI call returns 0 upon success or an implementation specific negative error code.

## 5.3 扩展：控制台读字符（扩展号 #0x02）

> 5.3. Extension: Console Getchar (EID #0x02)

```c
long sbi_console_getchar(void)
```

从调试控制台读一个字节。

> Read a byte from debug console.

若成功，此 SBI 调用返回读到的字节，否则返回 -1。

> The SBI call returns the byte on success, or -1 for failure.

## 5.4 扩展：清除处理期间中断（扩展号 #0x03）

> 5.4. Extension: Clear IPI (EID #0x03)

```c
long sbi_clear_ipi(void)
```

清除挂起的处理器间中断（如果有的话）。处理期间中断仅在调用此 SBI 调用的硬件线程中被清除。`sbi_clear_ipi()` 已弃用，因为 S 态代码可以直接清除 `sip.SSIP` 控制状态寄存器位。

> Clears the pending IPIs if any. The IPI is cleared only in the hart for which this SBI call is invoked. sbi_clear_ipi() is deprecated because S-mode code can clear sip.SSIP CSR bit directly.

如果没有处理期间中断处于未决状态，则此 SBI 调用返回 0，如果有处理器间中断处于未决状态，则返回实现决定的正值。

> This SBI call returns 0 if no IPI had been pending, or an implementation specific positive value if an IPI had been pending.

## 5.5 扩展：发送处理期间中断（扩展号 #0x04）

> 5.5. Extension: Send IPI (EID #0x04)

```c
long sbi_send_ipi(const unsigned long *hart_mask)
```

向 **hart_mask** 中定义的所有硬件线程发送处理器间中断。处理器间中断在接收的硬件线程上表现为特权级软件中断。

> Send an inter-processor interrupt to all the harts defined in **hart_mask**. Interprocessor interrupts manifest at the receiving harts as Supervisor Software Interrupts.

**hart_mask** 是一个指向硬件线程位向量的虚地址。位向量表示为一系列无符号长整数，其长度等于系统中的硬件线程数除以无符号长整数中的位数，向上取整到下一个整数。

> hart_mask is a virtual address that points to a bit-vector of harts. The bit vector is represented as a sequence of unsigned longs whose length equals the number of harts in the system divided by the number of bits in an unsigned long, rounded up to the next integer.

若成功，此 SBI 调用返回 0，否则返回一个实现决定的负错误码。

> This SBI call returns 0 upon success or an implementation specific negative error code.

## 5.6 扩展：远程指令栅栏（扩展号 #0x05）

> 5.6. Extension: Remote FENCE.I (EID #0x05)

```c
long sbi_remote_fence_i(const unsigned long *hart_mask)
```

指示其他硬件线程执行 `FENCE.I` 指令。**hart_mask** 与 `sbi_send_ipi()` 中描述的相同。

> Instructs remote harts to execute `FENCE.I` instruction. The **hart_mask** is same as described in `sbi_send_ipi()`.

若成功，此 SBI 调用返回 0，否则返回一个实现决定的负错误码。

> This SBI call returns 0 upon success or an implementation specific negative error code.

## 5.7 扩展：远程虚地址栅栏（扩展号 #0x06）

> 5.7. Extension: Remote FENCE.VMA (EID #0x06)

```c
long sbi_remote_sfence_vma(const unsigned long *hart_mask,
                           unsigned long start,
                           unsigned long size)
```

指示其他硬件线程执行一个或多个 `SFENCE.VMA` 指令，覆盖 **start** 和 **size** 表示的虚地址范围。

> Instructs the remote harts to execute one or more `SFENCE.VMA` instructions, covering the range of virtual addresses between **start** and **size**.

若成功，此 SBI 调用返回 0，否则返回一个实现决定的负错误码。

> This SBI call returns 0 upon success or an implementation specific negative error code.

## 5.8 扩展：带地址空间号的远程虚地址栅栏（扩展号 #0x07）

> 5.8. Extension: Remote SFENCE.VMA with ASID (EID #0x07)

```c
long sbi_remote_sfence_vma_asid(const unsigned long *hart_mask,
                                unsigned long start,
                                unsigned long size,
                                unsigned long asid)

```

指示其他硬件线程执行一个或多个 `SFENCE.VMA` 指令，覆盖 **start** 和 **size** 表示的虚地址范围。仅涵盖指定的 `ASID`。

> Instruct the remote harts to execute one or more `SFENCE.VMA` instructions, covering the range of virtual addresses between **start** and **size**. This covers only the given `ASID`.

若成功，此 SBI 调用返回 0，否则返回一个实现决定的负错误码。

> This SBI call returns 0 upon success or an implementation specific negative error code.

## 5.9 扩展：系统停机（扩展号 #0x08）

> 5.9. Extension: System Shutdown (EID #0x08)

```c
void sbi_shutdown(void)
```

在特权级的视角中，将所有硬件线程置于关闭状态。

> Puts all the harts to shutdown state from supervisor point of view.

无论成功还是失败，此 SBI 调用都不会返回。

> This SBI call doesn’t return irrespective whether it succeeds or fails.

## 5.10 函数表

> 5.10. Function Listing

| 函数名 | SBI 版本 | 函数号 | 扩展号 | 取代的扩展号
| -------------------------- | --- | - | ---- | -
| sbi_set_timer              | 0.1 | 0 | 0x00      | 0x54494D45
| sbi_console_putchar        | 0.1 | 0 | 0x01      | N/A
| sbi_console_getchar        | 0.1 | 0 | 0x02      | N/A
| sbi_clear_ipi              | 0.1 | 0 | 0x03      | N/A
| sbi_send_ipi               | 0.1 | 0 | 0x04      | 0x735049
| sbi_remote_fence_i         | 0.1 | 0 | 0x05      | 0x52464E43
| sbi_remote_sfence_vma      | 0.1 | 0 | 0x06      | 0x52464E43
| sbi_remote_sfence_vma_asid | 0.1 | 0 | 0x07      | 0x52464E43
| sbi_shutdown               | 0.1 | 0 | 0x08      | 0x53525354
| **保留**                   |     |   | 0x09-0x0f |
