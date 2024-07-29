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

---

## The Rust Reference

今天要介绍的所有重点语法都直接来自 The Rust Reference，所以首先要介绍的是 Rust 参考手册本身。

首先，这本书不是规范。Rust 并不是像 C++ 那样规范驱动的语言，而是直接在社区中通过提案讨论和实现语法，再根据已发布的语法实现写出参考指南的。所以参考应该是和语言真正的实现高度一致的，但是只包含现状，不具有前瞻性，并且最新发布的变化可能不包括。

然后是本书的内容。Rust 语言参考是 Rust 语言核心语法的文档，所以不包括标准库文档、编译器使用指南、构建工具指南等，只包括语法。而 C++ 的 cppreference 是同时包括语法和标准库文档的。

最后是这个书要怎么读。首先这个书不假定顺序阅读，因此它也会像教材那样保证知识会顺序地呈现给读者。这个书是按语法结构来组织的，在概念上依赖大量的内部链接来构建一张知识的网络。所以这个书有两种读法，第一种是直接查询相关主题；第二种是顺着读，但是随时通过链接跳转到你感兴趣的话题。

额外要说的是这个书不要看中文版。中文版更新的延迟是按年计算的，而 Rust 真正的更新频率是约 6 周一个大版本，差不多所有内容都过时了。如果是英文看起来不顺利的，推荐借助网页翻译或者直接大模型量子速读。

## 表达式、代码块，以及函数

[Rust 是一种主要基于表达式的语言](https://doc.rust-lang.org/reference/statements-and-expressions.html)，源码中几乎所有东西都可以解释为表达式或变形/省略的表达式。

表达式可以求值。对表达式求值会产生一个值，同时产生一些副作用，这些副作用就是程序的效果。

其中有一种表达式叫做[代码块表达式](https://doc.rust-lang.org/reference/expressions/block-expr.html)，形式是一对大括号包围的一串语句（但不是所有大括号都表示代码块）。代码块中的语句是有顺序的，这个顺序就是效果发生的拓扑序。

语句分为两种，声明语句和表达式语句。

[声明语句](https://doc.rust-lang.org/reference/statements.html#declaration-statements)可以声明类型、函数、宏等语法结构，这些声明和模块中的声明一致，包括行为也一致。所以这种声明只关心作用域，作用域满足的条件下可以先使用后声明。声明语句还可以以 let 开始触发一次模式匹配。如果触发的模式匹配是绝对的，就以 `;` 结尾，如果是可反驳的（*refutable*），就以 `else` + *代码块表达式* + `;` 结尾。

表达式语句就是一个表达式。又有 3 种情况：

1. 表达式是控制流表达式，且值是 `()`，则不需要以分号结尾；
   控制流表达式就是最经典的、高考要靠的计算流程结构，包括顺序、循环、判断。

   - 顺序就是代码块表达式；
   - [循环](https://doc.rust-lang.org/reference/expressions/loop-expr.html)包括 `for`、`while` 和 `loop`；
   - 判断包括 [`if`](https://doc.rust-lang.org/reference/expressions/if-expr.html) 和 [`match`](https://doc.rust-lang.org/reference/expressions/match-expr.html)；

2. 表达式语句是代码块的最后一个语句，则不需要以分号结尾，以其值作为代码块表达式的值；
3. 以上两种情况都不是，则表达式必须以 `;` 结尾，此时其值被忽略。这种语句的目标是其求值过程产生的效果；

然后再看控制流表达式里的 if 表达式。if 表达式的结构分为 4 部分：

1. 以 if 开头；
2. 一个值为 bool 类型的表达式，或者 let 触发一次可反驳的模式匹配；
3. 一个代码块表达式；
4. else 和另一个代码块表达式，且其值的类型与第 3 部分的代码块表达式一致；

另外，如果：

- 第 4 部分的代码块表达式中没有语句，则可以省略第 4 部分；
- 第 4 部分的代码块表达式中有且只有另一个 if 表达式，则可省略代码块表达式的包围大括号，形成 else if；

if 表达式的功能是判断。如果：

- bool 类型表达式求值为 true，或模式匹配成果，则对第 3 部分的代码块表达式求值，忽略第 4 部分的代码块表达式；
- bool 类型表达式求值为 false，或模式匹配失败，则忽略第 3 部分的代码块表达式，并对第 4 部分的代码块表达式求值；

最后来看函数。函数本身是一个声明，它的结构是（暂时省略了泛型部分）：

1. 以 fn 开头；
2. 一个标识符，作为函数的名字；
3. 参数列表，以小括号包围、逗号分隔，每一项由一个绝对模式匹配、`:`，以及一个类型约束组成；
4. -> 返回值类型，如果类型为 `()` 可以省略；
5. 一个代码块表达式作为函数体；

综上所述，由于[函数体就是一个代码块表达式](https://doc.rust-lang.org/reference/items/functions.html#function-body)，所以完全遵循代码块表达式的形式。即包含一串语句，并以最后一个语句的值作为返回值。另外在代码块表达式做函数体时额外允许 [return 表达式](https://doc.rust-lang.org/reference/expressions/return-expr.html)，表示以这个表达式的值为返回值并忽略后续语句。

上述各种表达式在 Reference 中都有自己的专题，共有 19 种大类。考虑到这 19 种是 Rust 这样一种复杂的高级语言表达式种类的总量，还是相当少的。推荐大家都去把 Reference 的这个部分过一遍。

## 切片 \[T\]

群里一个同学问了关于“切片”的命名问题。

“切片”这个汉语词语是英语“slice”的翻译。如果在词典检索 [slice](https://www.bing.com/dict/search?q=slice)，可以发现这个词既可以做动词也可以做名词，做动词表示“切”，做名词表示“片”，所以中文翻译成“切片”。由于切片在 Rust 里说的是一种变量类型，所以应该主要关注其名词性含义，就是“片”。可以看它的英文同义词、近义词，来理解什么是“片”。

要进一步解释这个“片”就再看英英释义，很容易关注到第二项含义指出在信息领域，slice 表示“a part or share of something”。到这里对应到 Rust 就很顺畅了。Rust 中，得到一个切片最常见的情况是从一个数组或者动态数组 `Vec` 里用一个 Range 划定一个范围得到，所以它是 part；得到之后，切片共享了源数组的存储空间、所有权、可变性，所以它是一个 share。

关于[切片类型](https://doc.rust-lang.org/reference/types/slice.html) 额外多说一点。我们日常说的切片都是带有引用的切片，但是切片类型在定义上实际是不带引用的 `[T]`。上节课我们已经讲了引用类型本质就是指针，而切片的引用 `&[T]` 本质是占用两个指针宽度的胖指针。而切片类型本身在定义上是胖指针指向的那一段存储区域，所以它是一个*不定长类型*。不定长类型是无法在栈上表示的类型，只能以胖指针形式存在于引用、裸指针或智能指针中，所以日常不是很常见。关于不定长类型的具体信息我们今天讲到特质再涉及，

## as

关于使用 as 转换类型，有同学在问卷里问到为什么 as 转型不是一步到位的，经常出现一串 as as as。这是因为 as 转型的原类型和目标类型是有限的种类，[Reference](https://doc.rust-lang.org/reference/expressions/operator-expr.html#type-cast-expressions) 中可以找到能转型的类型对。

其实最常见的一串 as 是因为引用到指针，再由指针到另一个类型的指针。可以看到使用 as 做引用到指针的转换是不能同时转型的，要转型就需要再 as 一次。或者我想把这个引用转换成一个表示地址的整数，也需要二次 as。大家可以把这个转型关系画成图或者有限状态机，这样可以通过子图结构来分析一串 as 是否可以简化，或者发现使用 as 的最简最长路径有多长。

## 常量

群里有同学问，常量和静态变量为什么必须显式标注类型。首先，作为语言的设计问题，社区是有[讨论](https://github.com/rust-lang/rfcs/issues/1349)的。关于讨论中正反双方的意见，各位同学可以自己去读一读，也可以发表自己的意见。讨论仍然是开放的。

这里我要说我的意见，我支持常量和静态变量的类型必须写出。换句话说，禁止编译器推断常量和静态变量的类型。

我的理由是：Rust 类型推断的基本逻辑是基于用法的。

比如最常见的声明时直接绑定，`let a = T::new();` 这种，应该把它理解为因为 `a` 可以绑定到 `T::new()` 这个值，所以它的类型必须是 `T`。

对于字面量的情况，比如 `let a = 1;` 由于 1 可以对应多种类型，所以推断就可以结合其他用法共同决定，比如 `a` 传递给了一个接受 `u64` 的函数，那么 `a` 就是 `u64`。

所以类型可靠的前提是声明标识符的作者可以完全控制所有使用标识符的地方。所以一个局部变量一定是可推断的。然而静态变量和常量是可以直接 pub 到 crate 外部的，那么声明它的人就无法掌控它的用法，所以就是不可推断的。用户应该不期望同样的声明代码，自己写的和依赖别人的类型不同，这太反直觉了。

所以按我的思路，非 pub 的常量和静态变量应该可推断。但是根据可见性决定是否可推断似乎也很奇怪，不如干脆明确要求全部写出比较一致。

再多讲一点 Rust 中常量的行为。Rust 中的 const 对应 C++ 的 constexpr，修饰函数表示函数求值可以发生在编译期，修饰变量表示变量求值必须发生在编译期。那么如果一个变量在编译期保证已经求值，编译器就可以利用这个值在编译期做更多事。对应的常量的行为表现就会有所不同。

有三种可能的行为：

1. 常量传播；
2. 内联；
3. 静态化；

常量传播指的是利用这个常量作为参数或操作数对其他表达式求值。仅用于常量传播的常量由于在运行期不需要直接使用，是完全不存在于编译后的程序中的。

内联表示常量的值直接转换为指令的一部分。例如：

```rust
const ONE: i32 = 1;
a += ONE;
```

这样的常量无法传播，因为它不用于可编译期求值的表达式中。但编译器可以将常量编码到单条立即数加指令中，因此最终常量在编译后的程序中位于程序段指令的内部，不存在一个地址。

静态化表示常量在最终的程序中位于常量区，不可修改，但仍然具有一个可访问的地址。这种行为通常是因为常量体积较大，不是一个立即数，就不应该留在代码段。或者是运行时对常量寻址的操作直接出现，那么常量仍然需要正确提供一个地址。

## trait

下一项我们讲 trait。

trait，翻译为“特质”。是 Rust 最核心的抽象，没有之一。可能大家在写 Rust 的时候都是语句写的多，类型少一些，特质更少，所以没有感觉特质这个东西的重要性。这个实际上是 Rust 通过很好的语法设计隐藏了特质的复杂性，使得入手 Rust 可以更丝滑一些。

为什么说特质是 Rust 的核心抽象呢？因为 Rust 很多看起来非常核心的能力，都是通过特质实现的。就好像 C++ 中的很多核心能力是通过重载决议完成的一样。类比一下，C++ 里的类的深拷贝，本质上是通过构造函数的重载实现的，但是编译器在看到初始化变量的时候会自动进行一次重载决议找到对应的实现；而 C++11 的最重要概念移动语义，也是通过重载移动构造函数、重载移动赋值运算符，再用强制类型转换来触发重载决议实现的。而 Rust 的特质，就好像 C++ 的重载那么重要。

特质的核心功能，是提供约束。Rust 编译器能提供一系列高级的语法检查，最终实现了类型安全和无惧并发。实际上这些检查都是基于三大约束：单所有权约束、生命周期约束和特质约束。然而，所有权机制的实现依赖 Drop 特质，生命周期也可以视作一种特殊的特质，因此特质约束又在这三大约束中处于核心地位。

那么特质是什么呢？为了保持简单性，我们先从不带泛型的特质开始。要定义一个特质，首先从关键词 `trait` 开始，然后是一个标识符作为特质的名字，然后是一对大括号：

```rust
trait T {}
```

大括号里可以有 3 种内容，但是我们先不管里面的内容，直接来实现特质约束。

特质约束是基于类型系统的。具体类型可以实现特质，例如：

```rust
struct S;
impl T for S {}
```

这两行定义了类型 `S`，并为它实现了刚才定义的特质。这样，S 就成为了一个拥有 T 特质的类型，从而可以用于要求 T 约束的场景。

特质约束要求，还有个名字叫做多态性的静态分发。它本身是依赖类型系统的，是泛型的扩展。但是为了保持简单性，我们依然从一个简写形式开始，先了解约束的效果。下面的代码定义了一个函数，它要求一个满足特质 T 的参数：

```rust
fn f(t: impl T) {}
```

可以看到，这个代码非常简洁。和一般的函数定义一样，除了参数 t 的类型不是一个具体类型，而是一个满足 T 约束的类型。如果给它传递不满足 T 约束的类型，比如基本类型，就会产生编译错误。

那这个约束有什么作用呢？首先是用于做标记。例如，Rust 的无惧并发能力就是通过 Send 和 Sync 两个空白特质来标记类型，从而确定类型是否并发安全，以及是否支持跨线程移动。还有其他的一些空白特质和标记性特质，例如：

- Sized: 表示类型的大小是静态确定的；
- Copy: 表示类型可以逐字节拷贝，并且把移动行为替换为逐字节拷贝行为；
- Unpin: 表示类型可以移动到不同的地址；
- Eq: 表示类型的任意两个实例可以判定是否相等；

  > 不满足 Eq 的类型并不罕见。IEEE754 浮点数就是不可判等类型，因为其合法值 NaN 有多种表示，且定义上 NaN 与任何其他值（包括底层表示完全一致的 NaN）都不相等。

- Send: 表示类型可以安全地跨线程移动；
- Sync: 表示类型可以跨线程共享，也就是线程安全；

> **NOTICE** 关于这些特质的具体描述一般位于标准库文档中。但有时由于涉及语言级行为，The Rust Reference 会引用或描述其内容。

其次就要讲到特质的三类成员：

1. 函数；
2. 常量；
3. 关联类型；

首先是函数。由于特质的功能是提供约束，因此特质的函数成员的功能是要求实现特质的类型必须提供满足指定签名的函数定义。例如：

```rust
trait T {
    fn f();
    fn m(&self);
}
```

这就要求类型必须提供函数 f。

> **NOTICE** C++ 用户可能希望将特质理解为抽象类，但这个例子展示了实现特质和派生抽象类的区别。C++ 是无法将静态方法标记为虚的，也就无法重写不关联对象的方法，但 Rust trait 可以要求任何函数，无论是不带有 self 的函数还是带有 self 的方法。

实现 T 的类型必须提供 f 函数：

```rust
struct S;
impl T for S {
    fn f() {
        println!("This is S");
    }
    fn m(&self) {
        Self::f();
    }
}
```

带有约束 T 的函数就可以使用 m 方法了（为了简单性，这里使用了不为 t 显式泛型的写法，因此没有表示 t 的类型的标识符，无法调用 f）：

```rust
fn f(t: impl T) {
    t.m();
}
```

可以调用这个函数：

```rust
fn main() {
    f(S)
}
```

了解了特质约束的语法现象，接下来介绍一系列标准库中重要的特质：

### Drop

Drop 对应的是 C++ 中的析构，是所有权机制的重要组成部分。它是少数在 Reference 中也有词条的特质，但是详细信息还要看标准库文档的相关章节。标准库有一个模块叫做 [`std::ops`](https://doc.rust-lang.org/std/ops/index.html#traits)，这个模块是和语法联系最紧密的章节，因为其中的所有特质都用于运算符重载或者其他语法行为扩展能力。[Drop](https://doc.rust-lang.org/std/ops/trait.Drop.html) 特质也定义在这个模块中。

看文档我们可以发现，Rust 把这个特质就定义为析构。当值离开作用域或者其他原因不再需要的时候会自动调用。这里的“不再需要”是非常自然语言的说法，其实用 Rust 语言的说法就是不再有任何其他东西对值拥有所有权。这里还提到，不能主动调用 Drop::drop，以及 Copy 和 Drop 特质是互斥的。

我这里稍微修改了它的例子来展示一下离开代码块作用域时对象的析构顺序。

> 实际上如果没有执行环境也可以直接用 Playground 跑，也可以改代码，但是不如本地看起来漂亮。

```rust
struct HasDrop(i32);

struct HasTwoDrops {
    one: HasDrop,
    two: HasDrop,
}

impl Drop for HasDrop {
    fn drop(&mut self) {
        println!("Dropping a {}", self.0);
    }
}

impl Drop for HasTwoDrops {
    fn drop(&mut self) {
        println!("Dropping ({}, {})", self.one.0, self.two.0);
    }
}

fn main() {
    let _01 = HasTwoDrops {
        one: HasDrop(0),
        two: HasDrop(1),
    };
    let _23 = HasTwoDrops {
        one: HasDrop(2),
        two: HasDrop(3),
    };
    println!("Running!");
}
```

可以看到：

```plaintext
Running!
Dropping (2, 3)
Dropping a 2
Dropping a 3
Dropping (0, 1)
Dropping a 0
Dropping a 1
```

调用析构的时机是离开代码块作用域的最后一个语句求值之后。调用顺序是从外到内的。其中栈上对象是后声明先释放，这个也很合理，因为栈嘛，就是后进先出。然后结构体成员递归释放的顺序是先声明先释放。这个顺序是明确说的，所以想不到怎么会出现依赖释放顺序的行为，但是依赖释放顺序的行为不是未定义行为。

这个递归调用的释放还提醒一点：Drop 实现的行为是默认行为之外的额外释放行为，实现特质的结构执行了 drop 之后，其中的每个成员还是要执行自己的释放流程的，从而保证所有东西一定都被释放了。这个就是为什么 drop 的接口是 `&mut self`，而不是 `self`。如果 `self` 被移走，成员就无法再释放了。

最后 Drop 特质还可以用于了解对象的声明周期。比如我修改 `main`：

```rust
fn main() {
    let _01 = HasTwoDrops {
        one: HasDrop(0),
        two: HasDrop(1),
    };
    let _23 = HasTwoDrops {
        one: HasDrop(2),
        two: HasDrop(3),
    };
    let _01 = HasTwoDrops {
        one: HasDrop(1),
        two: HasDrop(0),
    };
    println!("Running!");
}
```

用另一个 01 来遮蔽上面的 01，并且改成 (1,0) 来区分：

```plaintext
Running!
Dropping (1, 0)
Dropping a 1
Dropping a 0
Dropping (2, 3)
Dropping a 2
Dropping a 3
Dropping (0, 1)
Dropping a 0
Dropping a 1
```

可以看到被遮蔽的对象生命周期并没有缩短，还是活到最后才释放，而且释放顺序是最后遮蔽的变量最先释放。如果把 01 改成 mut 然后重新绑定呢：

```rust
fn main() {
    let mut _01 = HasTwoDrops {
        one: HasDrop(0),
        two: HasDrop(1),
    };
    let _23 = HasTwoDrops {
        one: HasDrop(2),
        two: HasDrop(3),
    };
    _01 = HasTwoDrops {
        one: HasDrop(1),
        two: HasDrop(0),
    };
    println!("Running!");
}
```

```plaintext
Dropping (0, 1)
Dropping a 0
Dropping a 1
Running!
Dropping (2, 3)
Dropping a 2
Dropping a 3
Dropping (1, 0)
Dropping a 1
Dropping a 0
```

可以看到绑定的一刻原来的值被立即释放了，新的值直接占据原来值的位置，同时释放顺序也按原来标识符的声明顺序，不会因为这个值重新绑定修改顺序。

### Clone/Copy

[Clone](https://doc.rust-lang.org/reference/special-types-and-traits.html?highlight=clone#clone) 和 [Copy](https://doc.rust-lang.org/reference/special-types-and-traits.html?highlight=clone#copy) 也都是 Reference 中有词条的特质。

其中 Copy 特质要求 Clone 作为前置条件。前置条件在参考里叫做超特质（*supertrait*），但是这个名太抽象了，其实就是前置条件的意思，必须先实现 Clone 才能实现 Copy。

但是实际上 Rust 并不要求 Clone 和 Copy 行为一致，比如这个例子：

```rust
struct S {}
impl Clone for S {
    fn clone(&self) -> Self {
        println!("Cloning S");
        Self {}
    }
}
impl Copy for S {}

fn main() {
    let _s = S {};
    let _ss = _s.clone();
    let _sss = _ss;
}
```

可以看到 Clone 会打印，Copy 并不会。

Reference 中明确提到，Copy 的作用方式是修改赋值时的语义。也就是说这个例子里，绑定 _ss 和_sss 都没有消耗调源对象，所以最后 3 个对象都可以继续使用。这也是为什么这些特质需要在 Reference 中有词条。

Clone 和 Copy 更一般的用法是 derive 出来。Rust 可以利用过程宏自动为类型实现 Clone 和 Copy 的实现代码。

### Debug/Display

[Debug](https://doc.rust-lang.org/std/fmt/trait.Debug.html) 和 [Display](https://doc.rust-lang.org/std/fmt/trait.Display.html) 也是非常常用的特质，它们属于标准库的 `std::fmt` 模块，是纯粹的库功能，在参考文档中不存在。

这两个是用来格式化显示类型的。其中 Debug 可以 derive 自动生成，一般也推荐使用自动生成的版本。这种会直接模式化地显示结构体每个字段的细节。Display 不能自动生成，一般用于更美观、更利于人类阅读的格式。

> **NOTICE** 文档明确提到，实现 Display 会自动实现 ToString，所以一般都推荐实现 Display 而不是 ToString。以及从这个设定可以看出 Display 是干什么用的。

格式串中，`{}` 调用 Display，`{:?}` 调用 Debug。以及 [std::fmt 模块](https://doc.rust-lang.org/std/fmt/index.html#traits) 还有其他 trait 用于不同类型的格式化。

### 迭代器

最后一个要讲的重要特质是迭代器。迭代器系统包括 3 个特质：

1. 迭代器本体 [Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html)；
2. 转换迭代器 [IntoIterator](https://doc.rust-lang.org/std/iter/trait.IntoIterator.html)；
3. 收集器 [FromIterator](https://doc.rust-lang.org/std/iter/trait.FromIterator.html)；

> **NOTICE** 还有其他更复杂的迭代器特质，一般都是依赖迭代器特质的高级特质，不一一介绍。可以查阅标准库 [std::iter 模块](https://doc.rust-lang.org/std/iter/index.html#traits)。

> **NOTICE** 关于 Rust 中的类型转换，有 3 个常用词，as、to 和 into。其中 as 并表示转换表示，意味着成本极低的转换，比如直接转换指针类型；to 表示成本较高的转换，比如需要分配、需要拷贝大块内存或者需要其他计算，例如 ToString；Into 表示转换时会消费掉原对象的转换，一般意味着这个转换可以移动原有的资源来避免拷贝。

先讲迭代器特质。为一个类型实现迭代器特质，表示这个类型本身是迭代器。实现迭代器特质需要填写两个东西，一个是之前跳过的特质关联类型。Item 表示迭代器会迭出来的对象类型；一个是 next 方法，返回一个可空的 Item 类型的值。文档上有[实现迭代器的示例](https://doc.rust-lang.org/std/iter/index.html#implementing-iterator)。

注意到，迭代器的 next 方法需要一个可变的 self。这是因为迭代器的进度是由迭代器自己保存的，每次调用 next 迭代器要修改自己的状态表示进度推进，这意味着迭代器是否完结是本身控制的，很容易实现无限长的迭代器。迭代器一般用 `Option::Some` 来表示正常迭代，用 `Option::None` 表示迭代完成，所以一般调用 next 的情况是一开始都是 Some，一旦出现 None，则意味着以后都是 None，但这个性质本质上是实现决定的。如果实现认为可以在 None 之后重新迭出 Some，不属于错误。

IntoIterator 特质是 for 循环的核心特质，for 循环本质上几乎就是一个宏，除了在 for 的代码块表达式中支持 break 和 continue 表达式。Reference 上有 for 展开到 IntoIterator 的代码。

FromIterator 特质是 collect 方法的基础结构。

不仅是一般意义上的容器类型实现了 FromIterator 和 IntoIterator，Option 和 Result 也实现了。这个设定和函数式编程的一些概念有关。由于这样的实现，Option 和 Result 如同容器类型一样支持 map filter 变换，以及 collect 生成。
