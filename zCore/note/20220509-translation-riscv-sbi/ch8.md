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
