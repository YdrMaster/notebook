# 第六章 定时器扩展（扩展号 #0x54494D45 “TIME”)

> Chapter 6. Timer Extension (EID #0x54494D45 "TIME")

这取代了旧的定时器扩展（EID #0x00）。遵循 v0.2 中定义的新调用约定。

> This replaces legacy timer extension (EID #0x00). It follows the new calling convention defined in v0.2.

## 6.1 函数：设置定时器（函数号 #0）

> 6.1. Function: Set Timer (FID #0)

```c
struct sbiret sbi_set_timer(uint64_t stime_value)
```

设置定时器在 **stime_value** 时间后产生下一个事件。**stime_value** 是绝对时间。该函数还清除挂起的定时器中断位。

> Programs the clock for next event after stime_value time. stime_value is in absolute time. This function must clear the pending timer interrupt bit as well.

如果特权态希望清除定时器中断但不安排下一个定时器事件，它可以请求一个无限远的定时器中断（即 `(uint64_t)-1`），或者它可以通过清除 `sie.STIE` 控制状态寄存器位来屏蔽定时器中断。

> If the supervisor wishes to clear the timer interrupt without scheduling the next timer event, it can either request a timer interrupt infinitely far into the future (i.e., `(uint64_t)-1`), or it can instead mask the timer interrupt by clearing `sie.STIE` CSR bit.

## 6.2 函数表

> 6.2. Function Listing

| 函数名 | SBI 版本 | 函数号 | 扩展号
| ------------- | --- | - | -
| sbi_set_timer | 0.2 | 0 | 0x54494D45
