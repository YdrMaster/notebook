# 生成器和 async/await

> Generators and async/await

> **概述：**
>
> > Overview:
>
> - 理解 async/await 语法的底层逻辑
> - 亲眼目睹我们为什么需要 `Pin`
> - 理解是什么让 Rust 的异步模式非常节省内存
>
> > - Understand how the async/await syntax works under the hood
> > - See first hand why we need Pin
> > - Understand what makes Rust's async model very memory efficient
>
> 生成器的动机可以在 [RFC#2033](https://github.com/rust-lang/rfcs/blob/master/text/2033-experimental-coroutines.md) 中找到。它写得很好，我推荐你通读它（它既谈到了 async/await，也谈到了生成器）。
>
> > The motivation for Generators can be found in RFC#2033. It's very well written and I can recommend reading through it (it talks as much about async/await as it does about generators).

为什么要学习生成器？

> Why learn about generators?

生成器/让出和 async/await 非常相似，一旦你理解了一个，就应该能够理解另一个。

> Generators/yield and async/await are so similar that once you understand one you should be able to understand the other.

对我来说，使用生成器来提供可运行的简短例子要比使用 Futures 容易得多，因为 Futures 需要我们现在引入很多概念，而这些概念我们稍后会涉及，只是为了展示一个例子。

> It's much easier for me to provide runnable and short examples using Generators instead of Futures which require us to introduce a lot of concepts now that we'll cover later just to show an example.

async/await 的工作原理与生成器类似，但它不是返回一个生成器，而是返回一个实现 Future 特质的特殊对象。

> Async/await works like generators but instead of returning a generator it returns a special object implementing the Future trait.

一个小小的收获是，在本章结束时，你会对生成器和 async/await 都得到一个相当好的介绍。

> A small bonus is that you'll have a pretty good introduction to both Generators and Async/Await by the end of this chapter.

基本上，在设计 Rust 如何处理并发时，有三种主要的选择可以讨论：

> Basically, there were three main options discussed when designing how Rust would handle concurrency:

1. 有栈协程，也就是所谓的绿色线程。
2. 使用组合子。
3. 无栈协程，也就是所谓的生成器。

> 1. Stackful coroutines, better known as green threads.
> 2. Using combinators.
> 3. Stackless coroutines, better known as generators.

我们在[背景资料](some-background-information.md#绿色线程有栈协程)中介绍了绿色线程，所以我们在此不再重复。我们将集中讨论今天 Rust 使用的无栈协程的变种。

> We covered green threads in the background information so we won't repeat that here. We'll concentrate on the variants of stackless coroutines which Rust uses today.

## 组合子

> Combinators

`Futures 0.1` 使用组合子。如果你在 JavaScript 中使用过承诺，那你已经知道组合子。在 Rust 中，它们看起来像这样：

> Futures 0.1 used combinators. If you've worked with Promises in JavaScript, you already know combinators. In Rust they look like this:

```rust
let future = Connection::connect(conn_str).and_then(|conn| {
    conn.query("somerequest").map(|row|{
        SomeStruct::from(row)
    }).collect::<Vec<SomeStruct>>()
});

let rows: Result<Vec<SomeStruct>, SomeLibraryError> = block_on(future);
```

**使用这种技术时，我关注到主要有三项弊端：**

> There are mainly three downsides I'll focus on using this technique:

1. 产生的错误信息可能极其冗长和玄妙
2. 内存使用无法达到最佳
3. 不允许跨组合子步骤进行借用。

> 1. The error messages produced could be extremely long and arcane
> 2. Not optimal memory usage
> 3. Did not allow borrowing across combinator steps.

第 3 点，实际上是 `Futures 0.1` 的一个主要缺点。

> Point #3, is actually a major drawback with Futures 0.1.

不允许跨暂停点的借用非常不符合人体工程学，为了完成一些任务，需要额外的分配或复制，这是低效的。

> Not allowing borrows across suspension points ends up being very un-ergonomic and to accomplish some tasks it requires extra allocations or copying which is inefficient.

导致内存使用量高于最佳状态的原因是，这基本上是一种基于回调的方法，每个闭包都存储了它需要的所有计算数据。这意味着，当我们把这些东西连锁起来时，每增加一步，存储所需状态的内存就会增加。

> The reason for the higher than optimal memory usage is that this is basically a callback-based approach, where each closure stores all the data it needs for computation. This means that as we chain these, the memory required to store the needed state increases with each added step.

## 无栈协程/生成器

> Stackless coroutines/generators

这是目前 Rust 中使用的模式。它有几个显著的优点：

> This is the model used in Rust today. It has a few notable advantages:

1. 使用 async/await 作为关键字，很容易将普通的 Rust 代码转换为无栈协程（甚至可以使用宏来完成）。
2. 不需要上下文切换和保存/恢复 CPU 状态
3. 不需要处理动态堆栈分配
4. 非常节省内存
5. 允许我们跨越暂停点进行借用

> 1. It's easy to convert normal Rust code to a stackless coroutine using async/await as keywords (it can even be done using a macro).
> 2. No need for context switching and saving/restoring CPU state
> 3. No need to handle dynamic stack allocation
> 4. Very memory efficient
> 5. Allows us to borrow across suspension points

最后一点是与 `Futures 0.1` 的对比。通过 async/await，我们可以做到这一点。

> The last point is in contrast to Futures 0.1. With async/await we can do this:

```rust
async fn myfn() {
    let text = String::from("Hello world");
    let borrowed = &text[0..5];
    somefuture.await;
    println!("{}", borrowed);
}
```

Rust 中的异步是通过生成器实现的。所以要理解异步的真正工作原理，我们首先要理解生成器。Rust 中的生成器是作为状态机实现的。

> Async in Rust is implemented using Generators. So to understand how async really works we need to understand generators first. Generators in Rust are implemented as state machines.

一条计算链的内存占用是由单个步骤所需的最大占用量决定的。

> The memory footprint of a chain of computations is defined by the largest footprint that a single step requires.

这意味着在计算链上增加步骤可能根本不需要增加任何内存，这也是 Rust 中 Futures 和异步开销很小的原因之一。

> That means that adding steps to a chain of computations might not require any increased memory at all and it's one of the reasons why Futures and Async in Rust has very little overhead.

## 生成器如何工作

> How generators work

在今天的 Nightly Rust 中，你可以使用 `yield` 关键字。基本上，在一个闭包中使用这个关键字，可以将其转换为一个生成器。在我们有 `Pin` 的概念之前，一个闭包可以是这样的：

> In Nightly Rust today you can use the yield keyword. Basically using this keyword in a closure, converts it to a generator. A closure could look like this before we had a concept of Pin:

```rust
#![feature(generators, generator_trait)]
use std::ops::{Generator, GeneratorState};

fn main() {
    let a: i32 = 4;
    let mut gen = move || {
        println!("Hello");
        yield a * 2;
        println!("world!");
    };

    if let GeneratorState::Yielded(n) = gen.resume() {
        println!("Got value {}", n);
    }

    if let GeneratorState::Complete(()) = gen.resume() {
        ()
    };
}
```

早期，在对 `Pin` 的设计达成共识之前，这会编译成类似于这样的东西：

> Early on, before there was a consensus about the design of Pin, this compiled to something looking similar to this:

```rust
fn main() {
    let mut gen = GeneratorA::start(4);

    if let GeneratorState::Yielded(n) = gen.resume() {
        println!("Got value {}", n);
    }

    if let GeneratorState::Complete(()) = gen.resume() {
        ()
    };
}

// If you've ever wondered why the parameters are called Y and R the naming from
// the original rfc most likely holds the answer
enum GeneratorState<Y, R> {
    Yielded(Y),  // originally called `Yield(Y)`
    Complete(R), // originally called `Return(R)`
}

trait Generator {
    type Yield;
    type Return;
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return>;
}

enum GeneratorA {
    Enter(i32),
    Yield1(i32),
    Exit,
}

impl GeneratorA {
    fn start(a1: i32) -> Self {
        GeneratorA::Enter(a1)
    }
}

impl Generator for GeneratorA {
    type Yield = i32;
    type Return = ();
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return> {
        // lets us get ownership over current state
        match std::mem::replace(self, GeneratorA::Exit) {
            GeneratorA::Enter(a1) => {

          /*----code before yield----*/
                println!("Hello");
                let a = a1 * 2;

                *self = GeneratorA::Yield1(a);
                GeneratorState::Yielded(a)
            }

            GeneratorA::Yield1(_) => {
          /*-----code after yield-----*/
                println!("world!");

                *self = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}
```

The yield keyword was discussed first in RFC#1823 and in RFC#1832.

Now that you know that the yield keyword in reality rewrites your code to become a state machine, you'll also know the basics of how await works. It's very similar.

Now, there are some limitations in our naive state machine above. What happens when you have a borrow across a yield point?

We could forbid that, but one of the major design goals for the async/await syntax has been to allow this. These kinds of borrows were not possible using Futures 0.1 so we can't let this limitation just slip and call it a day yet.

Instead of discussing it in theory, let's look at some code.

We'll use the optimized version of the state machines which is used in Rust today. For a more in depth explanation see Tyler Mandry's excellent article: How Rust optimizes async/await

```rust
let mut generator = move || {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        yield borrowed.len();
        println!("{} world!", borrowed);
    };
```

We'll be hand-coding some versions of a state-machines representing a state machine for the generator defined above.

We step through each step "manually" in every example, so it looks pretty unfamiliar. We could add some syntactic sugar like implementing the Iterator trait for our generators which would let us do this:

```rust
while let Some(val) = generator.next() {
    println!("{}", val);
}
```

It's a pretty trivial change to make, but this chapter is already getting long. Just keep this in the back of your head as we move forward.

Now what does our rewritten state machine look like with this example?

```rust
enum GeneratorA {
    Enter,
    Yield1 {
        to_borrow: String,
        borrowed: &String, // uh, what lifetime should this have?
    },
    Exit,
}

impl Generator for GeneratorA {
    type Yield = usize;
    type Return = ();
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return> {
        // lets us get ownership over current state
        match std::mem::replace(self, GeneratorA::Exit) {
            GeneratorA::Enter => {
                let to_borrow = String::from("Hello");
                let borrowed = &to_borrow; // <--- NB!
                let res = borrowed.len();

                *self = GeneratorA::Yield1 {to_borrow, borrowed};
                GeneratorState::Yielded(res)
            }

            GeneratorA::Yield1 {to_borrow, borrowed} => {
                println!("Hello {}", borrowed);
                *self = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}
```

If you try to compile this you'll get an error (just try it yourself by pressing play).

What is the lifetime of &String. It's not the same as the lifetime of Self. It's not static. Turns out that it's not possible for us in Rust's syntax to describe this lifetime, which means, that to make this work, we'll have to let the compiler know that we control this correctly ourselves.

That means turning to unsafe.

Let's try to write an implementation that will compile using unsafe. As you'll see we end up in a self-referential struct. A struct which holds references into itself.

As you'll notice, this compiles just fine!

```rust
enum GeneratorState<Y, R> {
    Yielded(Y),
    Complete(R),
}

trait Generator {
    type Yield;
    type Return;
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return>;
}

enum GeneratorA {
    Enter,
    Yield1 {
        to_borrow: String,
        borrowed: *const String, // NB! This is now a raw pointer!
    },
    Exit,
}

impl GeneratorA {
    fn start() -> Self {
        GeneratorA::Enter
    }
}
impl Generator for GeneratorA {
    type Yield = usize;
    type Return = ();
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return> {
            match self {
            GeneratorA::Enter => {
                let to_borrow = String::from("Hello");
                let borrowed = &to_borrow;
                let res = borrowed.len();
                *self = GeneratorA::Yield1 {to_borrow, borrowed: std::ptr::null()};

                // NB! And we set the pointer to reference the to_borrow string here
                if let GeneratorA::Yield1 {to_borrow, borrowed} = self {
                    *borrowed = to_borrow;
                }

                GeneratorState::Yielded(res)
            }

            GeneratorA::Yield1 {borrowed, ..} => {
                let borrowed: &String = unsafe {&**borrowed};
                println!("{} world", borrowed);
                *self = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}
Remember that our example is the generator we created which looked like this:

let mut gen = move || {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        yield borrowed.len();
        println!("{} world!", borrowed);
    };
Below is an example of how we could run this state-machine and as you see it does what we'd expect. But there is still one huge problem with this:

pub fn main() {
    let mut gen = GeneratorA::start();
    let mut gen2 = GeneratorA::start();

    if let GeneratorState::Yielded(n) = gen.resume() {
        println!("Got value {}", n);
    }

    if let GeneratorState::Yielded(n) = gen2.resume() {
        println!("Got value {}", n);
    }

    if let GeneratorState::Complete(()) = gen.resume() {
        ()
    };
}
The problem is that in safe Rust we can still do this:

Run the code and compare the results. Do you see the problem?

pub fn main() {
    let mut gen = GeneratorA::start();
    let mut gen2 = GeneratorA::start();

    if let GeneratorState::Yielded(n) = gen.resume() {
        println!("Got value {}", n);
    }

    std::mem::swap(&mut gen, &mut gen2); // <--- Big problem!

    if let GeneratorState::Yielded(n) = gen2.resume() {
        println!("Got value {}", n);
    }

    // This would now start gen2 since we swapped them.
    if let GeneratorState::Complete(()) = gen.resume() {
        ()
    };
}
```

Wait? What happened to "Hello"? And why did our code segfault?

Turns out that while the example above compiles just fine, we expose consumers of this API to both possible undefined behavior and other memory errors while using just safe Rust. This is a big problem!

I've actually forced the code above to use the nightly version of the compiler. If you run the example above on the playground, you'll see that it runs without panicking on the current stable (1.42.0) but panics on the current nightly (1.44.0). Scary!

We'll explain exactly what happened here using a slightly simpler example in the next chapter and we'll fix our generator using Pin so don't worry, you'll see exactly what goes wrong and see how Pin can help us deal with self-referential types safely in a second.

Before we go and explain the problem in detail, let's finish off this chapter by looking at how generators and the async keyword is related.

Async and generators
Futures in Rust are implemented as state machines much the same way Generators are state machines.

You might have noticed the similarities in the syntax used in async blocks and the syntax used in generators:

```rust
let mut gen = move || {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        yield borrowed.len();
        println!("{} world!", borrowed);
    };
```

Compare that with a similar example using async blocks:

```rust
let mut fut = async {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        SomeResource::some_task().await;
        println!("{} world!", borrowed);
    };
```

The difference is that Futures have different states than what a Generator would have.

An async block will return a Future instead of a Generator, however, the way a Future works and the way a Generator work internally is similar.

Instead of calling Generator::resume we call Future::poll, and instead of returning Yielded or Complete it returns Pending or Ready. Each await point in a future is like a yield point in a generator.

Do you see how they're connected now?

Thats why knowing how generators work and the challenges they pose also teaches you how futures work and the challenges we need to tackle when working with them.

The same goes for the challenges of borrowing across yield/await points.

Bonus section - self referential generators in Rust today
Thanks to PR#45337 you can actually run code like the one in our example in Rust today using the static keyword on nightly. Try it for yourself:

Beware that the API is changing rapidly. As I was writing this book, generators had an API change adding support for a "resume" argument to get passed into the generator closure.

Follow the progress on the tracking issue #4312 for RFC#033.

```rust
#![feature(generators, generator_trait)]
use std::ops::{Generator, GeneratorState};

pub fn main() {
    let gen1 = static || {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        yield borrowed.len();
        println!("{} world!", borrowed);
    };

    let gen2 = static || {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        yield borrowed.len();
        println!("{} world!", borrowed);
    };

    let mut pinned1 = Box::pin(gen1);
    let mut pinned2 = Box::pin(gen2);

    if let GeneratorState::Yielded(n) = pinned1.as_mut().resume(()) {
        println!("Gen1 got value {}", n);
    }

    if let GeneratorState::Yielded(n) = pinned2.as_mut().resume(()) {
        println!("Gen2 got value {}", n);
    };

    let _ = pinned1.as_mut().resume(());
    let _ = pinned2.as_mut().resume(());
}
```
