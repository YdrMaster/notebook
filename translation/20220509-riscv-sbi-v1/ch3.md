# 第三章 二进制编码

> Chapter 3. Binary Encoding

所有 SBI 函数共享一组二进制编码，这有助于 SBI 扩展的混合。SBI 规范遵循以下调用约定。

> All SBI functions share a single binary encoding, which facilitates the mixing of SBI extensions. The SBI specification follows the below calling convention.

- 特权态与特权执行环境之间通过 `ECALL` 传递指令。
- `a7` 传递 SBI 扩展号（EID），
- 对于所有基于 v0.2 之后的标准定义的 SBI 扩展，`a6` 传递 `a7` 指定的 SBI 扩展的 SBI 函数号（FID）。
- 除 `a0` 和 `a1` 之外的所有寄存器由被调用者保护。
- SBI 函数必须通过 `a0` 和 `a1` 返回一对值，其中 `a0` 返回错误码。这类似于返回 C 结构：

  ```c
  struct sbiret {
    long error;
    long value;
  };
  ```

> - An `ECALL` is used as the control transfer instruction between the supervisor and the SEE.
> - `a7` encodes the SBI extension ID (EID),
> - `a6` encodes the SBI function ID (FID) for a given extension ID encoded in `a7` for any SBI extension defined in or after SBI v0.2.
> - All registers except `a0` & `a1` must be preserved across an SBI call by the callee.
> - SBI functions must return a pair of values in `a0` and `a1`, with `a0` returning an error code. This is analogous to returning the C structure

出于兼容性的考虑，SBI 扩展 ID（EID）和 SBI 函数 ID（FID）被编码为带符号的 32 位整数。当传入寄存器时，这些遵循上述调用约定规则的标准。

> In the name of compatibility, SBI extension IDs (EIDs) and SBI function IDs (FIDs) are encoded as signed 32-bit integers. When passed in registers these follow the standard above calling convention rules.

下面的表 1 列出了 SBI 标准错误代码。

> The Table 1 below provides a list of Standard SBI error codes.

| 错误类型 | 译文 | 值
| --------------------------- | ---------- | -
| `SBI_SUCCESS`               | 操作成功    | 0
| `SBI_ERR_FAILED`            | 操作失败    | -1
| `SBI_ERR_NOT_SUPPORTED`     | 不支持的操作 | -2
| `SBI_ERR_INVALID_PARAM`     | 参数错误    | -3
| `SBI_ERR_DENIED`            | 操作被拒绝  | -4
| `SBI_ERR_INVALID_ADDRESS`   | 地址错误    | -5
| `SBI_ERR_ALREADY_AVAILABLE` | 已经启用    | -6
| `SBI_ERR_ALREADY_STARTED`   | 已经开始    | -7
| `SBI_ERR_ALREADY_STOPPED`   | 已经停止    | -8

具有不受支持的 SBI 扩展 ID（EID）或不受支持的 SBI 函数 ID（FID）的 `ECALL` 必须返回错误代码 `SBI_ERR_NOT_SUPPORTED`。

> An `ECALL` with an unsupported SBI extension ID (EID) or an unsupported SBI function ID (FID) must return the error code `SBI_ERR_NOT_SUPPORTED`.

每个 SBI 函数都应该首选 `unsigned long` 作为数据类型。它使规范保持简单且易于适应所有 RISC-V ISA 类型。如果数据定义为 32 位宽，更高权限的软件必须确保它只使用 32 位数据。

> Every SBI function should prefer `unsigned long` as the data type. It keeps the specification simple and easily adaptable for all RISC-V ISA types. In case the data is defined as 32bit wide, higher privilege software must ensure that it only uses 32 bit data only.

如果 SBI 函数需要将硬件线程列表传递给更高权限模式，它必须使用如下定义的硬件线程掩码。这适用于 v0.2 及之后定义的任何扩展。

> If an SBI function needs to pass a list of harts to the higher privilege mode, it must use a hart mask as defined below. This is applicable to any extensions defined in or after v0.2.

任何需要硬件线程掩码的函数都需要传递以下两个参数。

> Any function, requiring a hart mask, need to pass following two arguments.

- `unsigned long hart_mask` 是一个包含线程序号的标量位向量
- `unsigned long hart_mask_base` 是向量表示的硬件线程的起始序号。

> - `unsigned long hart_mask` is a scalar bit-vector containing hartids
> - `unsigned long hart_mask_base` is the starting hartid from which bit-vector must be computed.

在单个 SBI 函数调用中，可以设置的最大硬件线程数始终为 XLEN。如果低权限模式需要传递的信息不止 XLEN 个线程，它应该多次调用 SBI 函数。`hart_mask_base` 可以设置为 -1 表示可以忽略 `hart_mask` 并且必须考虑所有可用的硬件线程。

> In a single SBI function call, maximum number harts that can be set is always XLEN. If a lower privilege mode needs to pass information about more than XLEN harts, it should invoke multiple instances of the SBI function call. hart_mask_base can be set to -1 to indicate that hart_mask can be ignored and all available harts must be considered.

任何使用硬件线程掩码的函数都可能返回下表 2 中列出的错误值，这些错误值是函数标准错误值的补充。

> Any function using hart mask may return error values listed in the Table 2 below which are in addition to function specific error values.

| 错误类型 |  描述  | 译文
| ----------------------- | - | -
| `SBI_ERR_INVALID_PARAM` | Either `hart_mask_base` or any of the hartid from `hart_mask` is not valid i.e. either the hartid is not enabled by the platform or is not available to the supervisor. | `hart_mask_base` 无效或 `hart_mask` 指定的硬件线程存在无效值，即平台未启用硬件线程或特权态无法使用该硬件线程。
