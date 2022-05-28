# 修正及重构 rustsbi-qemu

[rustsbi-qemu 0.1.1 发布版本](https://github.com/rustsbi/rustsbi-qemu/releases/tag/v0.1.1) 有 bug。
用于获取 pmp 寄存器的内联汇编可能碰到了不该碰的东西，造成各种奇怪的现象。
改用 riscv crate 封装的安全函数，已解决。
