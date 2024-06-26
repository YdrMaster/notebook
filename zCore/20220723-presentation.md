﻿# 【报告】零开销页表库和设备树库在 zCore RISC-V 启动流程中的使用

## 介绍

1. 使用零开销的页表库，在启动过程中根据内核物理地址（要求 2 MiB 对齐）构造二级页表，使得内核在任何硬件平台上都在相同的虚存地址
2. 使用零开销的设备树库，从设备树解析硬件线程信息，实现所有硬件平台上多核启动流程一致

## 大纲

### 构造启动页表

1. 回顾 Sv39 虚地址空间

   | 术语            | 容量                  | 页内偏移位数
   | --------------- | --------------------- | -
   | 0 级页          |   4 KiB               | 12
   | 0 级页表/1 级页 |   2 MiB = 512 × 4 KiB | 21 = 12 + 9
   | 1 级页表/2 级页 |   1 GiB = 512 × 2 MiB | 30 = 21 + 9
   | 2 级页表/根页表 | 512 GiB = 512 × 1 GiB | 39 = 30 + 9

2. 回顾 zCore 内核内存布局
   1. 内核态用户态共享地址空间
   2. 内核位置（Sv39 虚地址空间的最后一个 2 级页）

3. 为什么要建 1 级页表

   - 为什么根页表不够？

     根页表要求 1 GiB 内页内偏移对齐，不够灵活。

   - 为什么不建 0 级页表？

     0 级页表本身需要的空间太大。

4. 浏览页表库

5. 看代码

### 从设备树解析硬件线程信息

1. 硬件线程和 SBI HSM 扩展
2. zCore 的 RISC-V 多核启动
   1. 现状
   2. 问题
3. 设备树简介
4. 浏览设备树库
5. 看代码
