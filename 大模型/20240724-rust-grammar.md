# Rust 重要语法讲解

## Rust 中的可变变量和可变引用

```rust
#![allow(unused)]

// 定义一个结构体
struct MyStruct(i32);
impl MyStruct {
    // 这个方法依赖可变引用 `&mut self`
    fn inc(&mut self) {
        self.0 += 1;
    }
}

// 创建结构体实例
let mut s = MyStruct(0); // 这是可变变量
s = MyStruct(10); // 可变变量可以重新赋值

let mut mut_ref = &mut s; // 这是可变引用
mut_ref.inc(); // 通过可变引用可以调用依赖可变引用的方法

s.inc(); // 可以从可变变量直接调用依赖可变引用的方法（因为可以从可变变量获得可变引用）
```

## 可变性传递的缺陷

现在的 Rust 中，一个切片总是被视为一个整体，借用切片的一项或一小片等于借用了整个切片。

```rust
let mut array = [1, 2, 3, 4, 5];
let slice = &mut array[..];

{
    let _0 = &mut slice[0];
    let _1 = &mut slice[1];
    // 错误！

    // *_0 = 10;
    // *_1 = 20;
}
{
    // 可以通过 safe API 实现
    // 有许多种用于断开借用关系的 API
    // split 一个张量同样会断开借用关系，slice 一个张量同样无法断开借用关系
    // READ: <https://github.com/InfiniTensor/InfiniLM/blob/main/tensor/src/split.rs>
    // READ: <https://github.com/InfiniTensor/InfiniLM/blob/main/tensor/src/slice.rs>
    let (_0, _tail) = slice.split_first_mut().unwrap();
    *_0 = 10;
    _tail[0] = 20;
}
{
    // 也可以通过切片解构实现，切片解构的具体语法后面讲
    let [_0, _1, ..] = slice else { unreachable!() };
    *_0 = 10;
    *_1 = 20;
}
```

结构体没有这个问题。可以同时可变借用结构体的两个成员。

```rust
struct MyStruct {
    a: i32,
    b: i32,
}

let mut s = MyStruct { a: 1, b: 2 };
let _a = &mut s.a;
let _b = &mut s.b;

*_a = 10;
*_b = 20;
```

但是如果通过方法来借用就不行了，因为编译器没法判断一个方法实现了借用了结构体的哪些字段，只能当作结构体整体都被借用了。
一旦整体被借，那么里面的任何成员都不能再借了。

```rust
impl MyStruct {
    fn a(&mut self) -> &mut i32 {
        &mut self.a
    }
}

let _a = s.a();
let _b = &mut s.b;
*_a = 10;
```

> **NOTICE** 本节回避了生命周期问题，没有做任何生命周期标注。显式生命周期标注留到下次讲。

## 胖指针概念

阅读 [Rust Reference: Pointer](https://doc.rust-lang.org/stable/reference/types/pointer.html#pointer-types) 了解 Rust 中的指针类型。
阅读 [Rust Reference: Trait Object](https://doc.rust-lang.org/stable/reference/types/trait-object.html) 了解 trait object 的详细信息。

运行和理解以下代码：

```rust
#![allow(unused)]

use std::mem::{size_of, size_of_val};
macro_rules! print_size {
    ($ty:ty; $info:expr) => {
        println!(
            "size of {:.<18} {:>2} - {}",
            stringify!($ty),
            size_of::<$ty>(),
            $info
        )
    };
}

print_size!(u8        ; "这是一个值");
print_size!(usize     ; "这也是一个值");
print_size!(*const u8 ; "这是一个基本类型的裸指针");
print_size!(*mut u8   ; "这也是一个基本类型的裸指针");
print_size!(&u8       ; "这是一个引用");
print_size!(&mut u8   ; "这是一个可变引用");
print_size!([u8; 2]   ; "这是 2 个 u8 组成的数组");
print_size!([usize; 2]; "这是 2 个 usize 组成的数组");
print_size!(&[u8]     ; "这是 u8 的切片");
print_size!(&mut [u8] ; "这是 u8 的可变切片");
println!();

let array: [u8; 3] = [1, 2, 3];
let slice: &[u8] = &array[..];
println!("array.address = {:p}", &array);
println!("array.size    = {}", size_of_val(&array));
println!("slice.address = {:p}", &slice);
println!("slice.size    = {}", size_of_val(&slice));
println!("slice.ptr     = {:p}", slice);
println!("slice.len     = {}", size_of_val(slice));

let ptr = &slice as *const _ as *const usize;
println!(
    "fat ptr: {ptr:p} -> ({:#x}, {})",
    unsafe { ptr.read() },
    unsafe { ptr.add(1).read() }
);
println!();

trait MyTrait {
    fn addr(&self) -> *const () {
        self as *const _ as _
    }
}

struct MyStruct1([u8; 3]);
impl MyTrait for MyStruct1 {}

struct MyStruct2([u8; 5]);
impl MyTrait for MyStruct2 {}

print_size!(MyStruct1         ; "这是一个结构体");
print_size!(MyStruct2         ; "这也是一个结构体");
print_size!(*const MyStruct1  ; "这是一个结构体的指针");
print_size!(*const MyStruct2  ; "这也是一个结构体的指针");
print_size!(*const dyn MyTrait; "这是一个“trait object”的指针");
print_size!(&MyStruct1        ; "这是一个结构体的引用");
print_size!(&MyStruct2        ; "这也是一个结构体的引用");
print_size!(&dyn MyTrait      ; "这是一个“trait object”的引用");
println!();

let s1 = MyStruct1([1, 2, 3]);
let s2 = MyStruct2([1, 2, 3, 4, 5]);
let s1_ref: &dyn MyTrait = &s1;
let s2_ref: &dyn MyTrait = &s2;
println!("s1.address     = {:p}", &s1);
println!("s2.address     = {:p}", &s2);
println!("s1_ref.address = {:p}", &s1_ref);
println!("s2_ref.address = {:p}", &s2_ref);
let ptr = &s1_ref as *const _ as *const usize;
println!(
    "s1_ref as fat ptr: {ptr:p} -> ({:#x}, {:#x})",
    unsafe { ptr.read() },
    unsafe { ptr.add(1).read() }
);
let ptr = &s2_ref as *const _ as *const usize;
println!(
    "s2_ref as fat ptr: {ptr:p} -> ({:#x}, {:#x})",
    unsafe { ptr.read() },
    unsafe { ptr.add(1).read() }
);
```

## 模式匹配

阅读 [Rust Reference: Pattern](https://doc.rust-lang.org/stable/reference/patterns.html) 了解全部模式。
