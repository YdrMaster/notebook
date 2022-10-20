# Waker 和上下文

> Waker and Context

> **概述：**
>
> > Overview:
>
> - 理解 Waker 对象是如何构建的
> - 了解运行时如何知道叶子 future 何时可以恢复
> - 了解动态派发和特质对象的基础知识
>
> > - Understand how the Waker object is constructed
> > - Learn how the runtime knows when a leaf-future can resume
> > - Learn the basics of dynamic dispatch and trait objects
>
> [RFC#2592](https://github.com/rust-lang/rfcs/blob/master/text/2592-futures.md#waking-up) 描述了 Waker 类型。
>
> > The Waker type is described as part of RFC#2592.

## Waker

> The Waker

Waker 类型允许运行时的反应器部分和执行器部分之间的松耦合。

> The Waker type allows for a loose coupling between the reactor-part and the executor-part of a runtime.

通过拥有一个与执行 future 的对象不相连的唤醒机制，运行时实现者可以想出有趣的新唤醒机制。这方面的一个例子可以是启动一个线程来做一些工作，最终通知 future，完全独立于当前的运行时。

> By having a wake up mechanism that is not tied to the thing that executes the future, runtime-implementors can come up with interesting new wake-up mechanisms. An example of this can be spawning a thread to do some work that eventually notifies the future, completely independent of the current runtime.

如果没有 Waker，执行器将是通知正在运行的任务的唯一途径，而有了 Waker，我们解开了耦合，从而很容易用新的叶子任务扩展生态系统。

> Without a waker, the executor would be the only way to notify a running task, whereas with the waker, we get a loose coupling where it's easy to extend the ecosystem with new leaf-level tasks.

> 如果你想阅读更多关于 `Waker` 类型背后的原因，我会推荐 [Withoutboats 关于它们的系列文章](https://boats.gitlab.io/blog/post/wakers-i/)。
>
> > If you want to read more about the reasoning behind the Waker type I can recommend Withoutboats articles series about them.

## 上下文类型

> The Context type

正如文档中所说的那样，目前这种类型只包裹了一个 `Waker`，但它为 Rust 中 API 的未来发展提供了一些灵活性。例如，上下文可以保存任务局部存储，并在以后的迭代中为调试钩子提供空间。

> As the docs state as of now this type only wraps a Waker, but it gives some flexibility for future evolutions of the API in Rust. The context can for example hold task-local storage and provide space for debugging hooks in later iterations.

## 理解 Waker

> Understanding the Waker

在实现我们自己的 `Future` 时，我们遇到的最困惑的事情之一就是如何实现 `Waker`。创建一个 `Waker` 涉及到创建一个虚表，它允许我们使用动态派发来调用我们自己构建的类型擦除的特质对象上的方法。

> One of the most confusing things we encounter when implementing our own Futures is how we implement a Waker. Creating a Waker involves creating a vtable which allows us to use dynamic dispatch to call methods on a type erased trait object we construct ourselves.

> 如果你想了解更多关于 Rust 中的动态派发，我会推荐 Adam Schwalm 写的一篇文章，叫做[探索 Rust 中的动态派发](https://alschwalm.com/blog/static/2017/03/07/exploring-dynamic-dispatch-in-rust/)。
>
> > If you want to know more about dynamic dispatch in Rust I can recommend an article written by Adam Schwalm called Exploring Dynamic Dispatch in Rust.

让我们更详细地解释一下。

> Let's explain this a bit more in detail.

## Rust 中的胖指针

> Fat pointers in Rust

为了更好地理解我们如何在 Rust 中实现 Waker，我们需要退一步，谈谈一些基本原理。让我们先来看看 Rust 中一些不同指针类型的大小。

> To get a better understanding of how we implement the Waker in Rust, we need to take a step back and talk about some fundamentals. Let's start by taking a look at the size of some different pointer types in Rust.

运行下面的代码（你必须按“▶”键才能看到输出）。

> Run the following code *(You'll have to press "play" to see the output)*:

```rust
trait SomeTrait { }

fn main() {
    println!("======== The size of different pointers in Rust: ========");
    println!("&dyn Trait:------{}", size_of::<&dyn SomeTrait>());
    println!("&[&dyn Trait]:---{}", size_of::<&[&dyn SomeTrait]>());
    println!("Box<Trait>:------{}", size_of::<Box<SomeTrait>>());
    println!("Box<Box<Trait>>:-{}", size_of::<Box<Box<SomeTrait>>>());
    println!("&i32:------------{}", size_of::<&i32>());
    println!("&[i32]:----------{}", size_of::<&[i32]>());
    println!("Box<i32>:--------{}", size_of::<Box<i32>>());
    println!("&Box<i32>:-------{}", size_of::<&Box<i32>>());
    println!("[&dyn Trait;4]:--{}", size_of::<[&dyn SomeTrait; 4]>());
    println!("[i32;4]:---------{}", size_of::<[i32; 4]>());
}
```

正如你从运行后的输出中看到的那样，引用的大小是不同的。许多是 8 字节（这是 64 位系统上的指针大小），但有些是 16 字节。

> As you see from the output after running this, the sizes of the references varies. Many are 8 bytes (which is a pointer size on 64 bit systems), but some are 16 bytes.

16 字节大小的指针被称为“胖指针”，因为它们带有额外的信息。

> The 16 byte sized pointers are called "fat pointers" since they carry extra information.

**例如 `&[i32]`：**

> Example `&[i32]`:

- 前 8 个字节是指向数组中第一个元素的实际指针（或者切片所指的数组的一部分）。
- 第二个 8 字节是分片的长度。

> - The first 8 bytes is the actual pointer to the first element in the array (or part of an array the slice refers to)
> - The second 8 bytes is the length of the slice.

**例如 `&dyn SomeTrait`：**

> Example `&dyn SomeTrait`:

`&dyn SomeTrait` 是对一个特质的引用，或者 Rust 所说的特质对象。

> This is the type of fat pointer we'll concern ourselves about going forward. &dyn SomeTrait is a reference to a trait, or what Rust calls a trait object.

一个指向特征对象的指针的布局是这样的：

> The layout for a pointer to a trait object looks like this:

- 前 8 个字节指向特质对象的数据
- 后面的 8 个字节指向该特征对象的虚表。

> - The first 8 bytes points to the data for the trait object
> - The second 8 bytes points to the vtable for the trait object

这样做的原因是允许我们引用一个我们一无所知的对象，只知道它实现了我们 trait 定义的方法。为了达到这个目的，我们使用动态派发。

> The reason for this is to allow us to refer to an object we know nothing about except that it implements the methods defined by our trait. To accomplish this we use dynamic dispatch.

让我们用代码而不是文字来解释这个问题，从这些部分实现我们自己的特质对象：

> Let's explain this in code instead of words by implementing our own trait object from these parts:

```rust
// A reference to a trait object is a fat pointer: (data_ptr, vtable_ptr)
trait Test {
    fn add(&self) -> i32;
    fn sub(&self) -> i32;
    fn mul(&self) -> i32;
}

// This will represent our home-brewed fat pointer to a trait object
# [repr(C)]
struct FatPointer<'a> {
    /// A reference is a pointer to an instantiated `Data` instance
    data: &'a mut Data,
    /// Since we need to pass in literal values like length and alignment it's
    /// easiest for us to convert pointers to usize-integers instead of the other way around.
    vtable: *const usize,
}

// This is the data in our trait object. It's just two numbers we want to operate on.
struct Data {
    a: i32,
    b: i32,
}

// ====== function definitions ======
fn add(s: &Data) -> i32 {
    s.a + s.b
}
fn sub(s: &Data) -> i32 {
    s.a - s.b
}
fn mul(s: &Data) -> i32 {
    s.a * s.b
}

fn main() {
    let mut data = Data {a: 3, b: 2};
    // vtable is like special purpose array of pointer-length types with a fixed
    // format where the three first values contains some general information like
    // a pointer to drop and the length and data alignment of `data`.
    let vtable = vec![
        0,                  // pointer to `Drop` (which we're not implementing here)
        size_of::<Data>(),  // length of data
        align_of::<Data>(), // alignment of data

        // we need to make sure we add these in the same order as defined in the Trait.
        add as usize, // function pointer - try changing the order of `add`
        sub as usize, // function pointer - and `sub` to see what happens
        mul as usize, // function pointer
    ];

    let fat_pointer = FatPointer { data: &mut data, vtable: vtable.as_ptr()};
    let test = unsafe { std::mem::transmute::<FatPointer, &dyn Test>(fat_pointer) };

    // And voalá, it's now a trait object we can call methods on
    println!("Add: 3 + 2 = {}", test.add());
    println!("Sub: 3 - 2 = {}", test.sub());
    println!("Mul: 3 * 2 = {}", test.mul());
}
```

稍后，当我们实现我们自己的 Waker 时，我们实际上会像这里一样设置一个虚表。我们创建它的方式略有不同，但现在你已经知道了通常的特质对象是如何工作的，你可能会认识到我们在做什么，这使得它不再神秘。

> Later on, when we implement our own Waker we'll actually set up a vtable like we do here. The way we create it is slightly different, but now that you know how regular trait objects work you will probably recognize what we're doing which makes it much less mysterious.

## 补充章节

> Bonus section

你可能想知道为什么 Waker 要这样实现，而不只是作为一个普通的特质？

> You might wonder why the Waker was implemented like this and not just as a normal trait?

原因是灵活性。以我们的方式实现 Waker，在选择使用何种内存管理方案上有很大的灵活性。

> The reason is flexibility. Implementing the Waker the way we do here gives a lot of flexibility of choosing what memory management scheme to use.

“正常”的方式是通过使用一个 `Arc` 引用计数来记录何时可以放弃一个 Waker 对象。然而，这并不是唯一的方法，你也可以使用纯粹的全局函数和状态，或者其他任何你希望的方式。

> The "normal" way is by using an Arc to use reference count keep track of when a Waker object can be dropped. However, this is not the only way, you could also use purely global functions and state, or any other way you wish.

这就为运行时的实现者留下了很多选择。

> This leaves a lot of options on the table for runtime implementors.

---

[下一章——生成器和 async/await](generators-and-async-await.md)
