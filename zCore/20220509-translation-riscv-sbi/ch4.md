# 第四章 基础扩展（扩展号 #0x10）

> Chapter 4. Base Extension (EID #0x10)

基础扩展设计为尽可能小。因此，它只包含探测哪些 SBI 扩展可用和查询 SBI 版本的功能。所有 SBI 实现都必须支持基本扩展中的所有函数，因此没有定义错误返回。

> The base extension is designed to be as small as possible. As such, it only contains functionality for probing which SBI extensions are available and for querying the version of the SBI. All functions in the base extension must be supported by all SBI implementations, so there are no error returns defined.

## 4.1 函数：获取 SBI 标准版本（FID #0）

> 4.1. Function: Get SBI specification version (FID #0)

```c
struct sbiret sbi_get_spec_version(void);
```

返回当前的 SBI 规范版本。此功能必须始终成功。SBI 规范的次要编号在低 24 位中编码，主要编号在接下来的 7 位中编码。第 31 位必须为 0，并为将来的扩展保留。

> Returns the current SBI specification version. This function must always succeed. The minor number of the SBI specification is encoded in the low 24 bits, with the major number encoded in the next 7 bits. Bit 31 must be 0 and is reserved for future expansion.

## 4.2 函数：获取 SBI 实现号（FID #1）

> 4.2. Function: Get SBI implementation ID (FID #1)

```c
struct sbiret sbi_get_impl_id(void);
```

返回当前的 SBI 实现 ID，每个 SBI 实现都不同。此实现 ID 旨在允许软件探测 SBI 实现的怪癖。

> Returns the current SBI implementation ID, which is different for every SBI implementation. It is intended that this implementation ID allows software to probe for SBI implementation quirks.

## 4.3 函数：获取 SBI 实现版本（FID #2）

> 4.3. Function: Get SBI implementation version
(FID #2)

```c
struct sbiret sbi_get_impl_version(void);
```

返回当前的 SBI 实现版本。此版本号的编码由 SBI 实现自行决定。

> Returns the current SBI implementation version. The encoding of this version number is specific to the SBI implementation.

## 4.4 函数：探测 SBI 扩展（FID #3）

> 4.4. Function: Probe SBI extension (FID #3)

```c
struct sbiret sbi_probe_extension(long extension_id);
```

如果给定的 SBI 扩展 ID（EID）不可用，则返回 0；如果它可用，则返回 1，或实现定义的任何其他非零值。

> Returns 0 if the given SBI extension ID (EID) is not available, or 1 if it is available unless defined as any other non-zero value by the implementation.

## 4.5 函数：获取硬件供应商号（FID #4）

> 4.5. Function: Get machine vendor ID (FID #4)

```c
struct sbiret sbi_get_mvendorid(void);
```

返回对 `mvendorid` 控制状态寄存器合法的值，并且 0 始终是此寄存器的合法值。

> Return a value that is legal for the `mvendorid` CSR and 0 is always a legal value for this CSR.

## 4.6 函数：获取硬件架构号（FID #5）

> 4.6. Function: Get machine architecture ID (FID #5)

```c
struct sbiret sbi_get_marchid(void);
```

返回对 `marchid` 控制状态寄存器合法的值，并且 0 始终是此寄存器的合法值。

> Return a value that is legal for the `marchid` CSR and 0 is always a legal value for this CSR.

## 4.7 函数：获取硬件实现号（FID #6）

> 4.7. Function: Get machine implementation ID (FID #6)

```c
struct sbiret sbi_get_mimpid(void);
```

返回对 `mimpid` 控制状态寄存器合法的值，并且 0 始终是此寄存器的合法值。

> Return a value that is legal for the `mimpid` CSR and 0 is always a legal value for this CSR.

## 4.8 函数表

> 4.8. Function Listing

| 函数名 | SBI 版本 | 函数号 | 扩展号
| ------------------------ | --- | - | -
| sbi_get_sbi_spec_version | 0.2 | 0 | 0x10
| sbi_get_sbi_impl_id      | 0.2 | 1 | 0x10
| sbi_get_sbi_impl_version | 0.2 | 2 | 0x10
| sbi_probe_extension      | 0.2 | 3 | 0x10
| sbi_get_mvendorid        | 0.2 | 4 | 0x10
| sbi_get_marchid          | 0.2 | 5 | 0x10
| sbi_get_mimpid           | 0.2 | 6 | 0x10

## 4.9 SBI 实现号

> 4.9. SBI Implementation IDs

| 实现号 | 名称
| - | -
| 0 | Berkeley Boot Loader (BBL)
| 1 | OpenSBI
| 2 | Xvisor
| 3 | KVM
| 4 | RustSBI
| 5 | Diosix
| 6 | Coffer
