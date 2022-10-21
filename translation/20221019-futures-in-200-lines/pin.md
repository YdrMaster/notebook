# Pin

> **概述：**
>
> > Overview
>
> - 了解如何使用 `Pin` 以及为什么在实现自定义的 `Future` 时需要它
> - 了解如何在 Rust 中安全使用自引用类型
> - 了解如何实现跨等待点的借用
> - 获得一套实用的规则来帮助你使用 Pin
>
> > - Learn how to use Pin and why it's required when implementing your own Future
> > - Understand how to make self-referential types safe to use in Rust
> > - Learn how borrowing across await points is accomplished
> > - Get a set of practical rules to help you work with Pin
>
> `Pin` 来自 [RFC#2349](https://github.com/rust-lang/rfcs/blob/master/text/2349-pin.md) 的建议
>
> > Pin was suggested in RFC#2349

让我们直接跳到它。Pin 属于那些在开始时很难搞清楚的主题，但一旦你为它建立了一个心智模型，它就会明显变得容易推理。

> Let's jump straight to it. Pinning is one of those subjects which is hard to wrap your head around in the start, but once you unlock a mental model for it it gets significantly easier to reason about.

## 定义

> Definitions

Pin 包装了一个指针。对一个对象的引用就是一个指针。Pin 给出了一些关于*指的*（pointee，zhǐ dì，它所指向的数据）的保证，我们将在本章进一步探讨。

> Pin wraps a pointer. A reference to an object is a pointer. Pin gives some guarantees about the pointee (the data it points to) which we'll explore further in this chapter.

Pin 由 `Pin` 类型和 `Unpin` 标记组成。`Pin` 在代码中的作用是管理一组规则，以支持那些实现了 `!Unpin` 的类型。

> Pin consists of the Pin type and the Unpin marker. Pin's purpose in life is to govern the rules that need to apply for types which implement !Unpin.

嗯，你理解对了，这就是双重否定。`!Unpin` 的意思是“非-非-pin”。

> Yep, you're right, that's double negation right there. !Unpin means "not-un-pin".

这个命名方案是 Rust 的安全特性之一，它故意测试你是不是太累了，没法安全地实现带有这个标记的类型。如果你开始对 `!Unpin` 感到困惑，甚至生气，这就是一个好的迹象，说明是时候放下工作，明天以全新的思维重新开始。

> This naming scheme is one of Rust's safety features where it deliberately tests if you're too tired to safely implement a type with this marker. If you're starting to get confused, or even angry, by !Unpin it's a good sign that it's time to lay down the work and start over tomorrow with a fresh mind.

严肃一点，我觉得有必要提一下，选择这些名字是有原因的。命名并不容易，我曾考虑在本书中重新命名 `Unpin` 和 `!Unpin`，以使它们更容易理解。

> On a more serious note, I feel obliged to mention that there are valid reasons for the names that were chosen. Naming is not easy, and I considered renaming Unpin and !Unpin in this book to make them easier to reason about.

然而，Rust 社区的一位经验丰富的成员说服我，有太多的细微差别和边缘情况需要考虑，而这些细微差别和边缘情况在天真地给这些标记取不同的名字时很容易被忽略，我最终相信我们只能习惯它们，并按原样使用它们。

> However, an experienced member of the Rust community convinced me that there are just too many nuances and edge-cases to consider which are easily overlooked when naively giving these markers different names, and I'm convinced that we'll just have to get used to them and use them as is.

如果你感兴趣，可以从[内部主题](https://internals.rust-lang.org/t/naming-pin-anchor-move/6864/12)中读到一些讨论。

> If you want to you can read a bit of the discussion from the internals thread.

## Pin 和自引用结构

> Pinning and self-referential structs

让我们从上一章开始，通过编写一些比状态机更容易理解的自引用结构，来得到一个比在生成器中使用自引用的问题变得简单得多的例子：

> Let's start where we left off in the last chapter by making the problem we saw using a self-references in our generator a lot simpler by making some self-referential structs that are easier to reason about than our state machines:

现在我们的例子看起来是这样的：

> For now our example will look like this:

```rust
use std::pin::Pin;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
}

impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
        }
    }

    fn init(&mut self) {
        let self_ref: *const String = &self.a;
        self.b = self_ref;
    }

    fn a(&self) -> &str {
        &self.a
    }

    fn b(&self) -> &String {
        unsafe {&*(self.b)}
    }
}
```

让我们来看看这个例子，因为我们将在本章的其余部分使用它。

> Let's walk through this example since we'll be using it the rest of this chapter.

我们有一个自引用的结构体 `Test`。`Test` 创建时需要一个 `init` 方法，这很奇怪，但为了使这个例子尽可能的简短，我们需要这个方法。

> We have a self-referential struct Test. Test needs an init method to be created which is strange but we'll need that to keep this example as short as possible.

`Test` 提供了两个方法来获取字段 `a` 和 `b` 的引用值。由于 `b` 是 `a` 的引用，我们将其存储为指针，因为 Rust 的借用规则不允许我们定义这样的生命周期。

> Test provides two methods to get a reference to the value of the fields a and b. Since b is a reference to a we store it as a pointer since the borrowing rules of Rust doesn't allow us to define this lifetime.

现在，让我们用这个例子来详细解释我们遇到的问题。正如你所看到的，这和预期的一样：

> Now, let's use this example to explain the problem we encounter in detail. As you see, this works as expected:

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    println!("a: {}, b: {}", test2.a(), test2.b());
}
```

在我们的 `main` 函数中，我们首先实例化了两个 `Test` 的实例，并打印出 `test1` 的字段值。我们得到了我们所期望的结果：

> In our main method we first instantiate two instances of Test and print out the value of the fields on test1. We get what we'd expect:

```bash
a: test1, b: test1
a: test2, b: test2
```

让我们看看如果我们将存储在内存位置 `test1` 的数据与存储在内存位置 `test2` 的数据交换会发生什么：

> Let's see what happens if we swap the data stored at the memory location test1 with the data stored at the memory location test2 and vice a versa.

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    println!("a: {}, b: {}", test2.a(), test2.b());
}
```

直观地想，我们应该像这样得到两次 `test1` 的调试打印结果：

> Naively, we could think that what we should get a debug print of test1 two times like this

```bash
a: test1, b: test1
a: test1, b: test1
```

但我们得到的却是：

> But instead we get:

```rust
a: test1, b: test1
a: test1, b: test2
```

`test2.b` 的指针仍然指向旧的位置，现在是在 `test1` 里面。该结构不再是自指的，它持有一个指向不同对象中的字段的指针。这意味着我们不能再信任 `test2.b` 的生命周期与 `test2` 的生命周期绑定了。

> The pointer to test2.b still points to the old location which is inside test1 now. The struct is not self-referential anymore, it holds a pointer to a field in a different object. That means we can't rely on the lifetime of test2.b to be tied to the lifetime of test2 anymore.

如果你还不相信，这个应该可以说服你：

> If you're still not convinced, this should at least convince you:

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    test1.a = "I've totally changed now!".to_string();
    println!("a: {}, b: {}", test2.a(), test2.b());
}
```

不应该这样。目前还没有严重的错误，但你可以想象，使用这段代码很容易产生严重的错误。

> That shouldn't happen. There is no serious error yet, but as you can imagine it's easy to create serious bugs using this code.

我画了一幅图来直观地解释发生了什么事：

> I created a diagram to help visualize what's going on:

![图 2：交换前后](https://cfsamson.github.io/books-futures-explained/assets/swap_problem.jpg)

> Fig 2: Before and after swap swap

正如你所看到的，这导致了非预期的行为。这很容易导致段错误，产生各种惊人的未定义行为和错误。

> As you can see this results in unwanted behavior. It's easy to get this to segfault, show UB and fail in other spectacular ways as well.

## 钉在栈上

> Pinning to the stack

现在，我们可以通过使用 `Pin` 来解决这个问题。让我们来看看我们的例子会是什么样子：

> Now, we can solve this problem by using Pin instead. Let's take a look at what our example would look like then:

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}

impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned, // This makes our type `!Unpin`
        }
    }
    fn init<'a>(self: Pin<&'a mut Self>) {
        let self_ptr: *const String = &self.a;
        let this = unsafe { self.get_unchecked_mut() };
        this.b = self_ptr;
    }

    fn a<'a>(self: Pin<&'a Self>) -> &'a str {
        &self.get_ref().a
    }

    fn b<'a>(self: Pin<&'a Self>) -> &'a String {
        unsafe { &*(self.b) }
    }
}
```

现在，我们在这里所做的是把一个对象钉在栈上。如果我们的类型实现了 `!Unpin`，这一步永远是 `unsafe`。

> Now, what we've done here is pinning an object to the stack. That will always be unsafe if our type implements !Unpin.

我们在这里使用了同样的技巧，包括还是需要一个 `init`。如果我们想解决这个问题，让用户避免 `unsafe`，我们需要把我们的数据钉在堆上，这种我们一会儿就会展示。

> We use the same tricks here, including requiring an init. If we want to fix that and let users avoid unsafe we need to pin our data on the heap instead which we'll show in a second.

让我们看看如果我们现在运行我们的例子会发生什么：

> Let's see what happens if we run our example now:

```rust
pub fn main() {
    // test1 is safe to move before we initialize it
    let mut test1 = Test::new("test1");
    // Notice how we shadow `test1` to prevent it from being accessed again
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());

    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
```

现在，如果我们尝试使用上次让我们陷入麻烦的同样的操作，你会得到一个编译错误。

> Now, if we try to pull the same trick which got us in to trouble the last time you'll get a compilation error.

```rust
pub fn main() {
    let mut test1 = Test::new("test1");
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());

    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    std::mem::swap(test1.get_mut(), test2.get_mut());
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
```

正如你从运行代码得到的错误中看到的那样，类型系统阻止我们交换钉住的指针。

> As you see from the error you get by running the code the type system prevents us from swapping the pinned pointers.

> 值得注意的是，钉在栈上总是取决于我们当前所处的栈帧，所以我们不能在一个栈帧中创建一个自引用对象并返回它，因为我们对 `self` 的任何指针都是无效的。
>
> > It's important to note that stack pinning will always depend on the current stack frame we're in, so we can't create a self referential object in one stack frame and return it since any pointers we take to "self" are invalidated.
>
> 如果你把一个对象钉在栈上，它也会把很多责任委托给你。一个很容易犯的错误是，忘记了对原始变量进行遮蔽，因为你可以在初始化后像这样放弃 `Pin` 并访问旧值。
>
> > It also puts a lot of responsibility in your hands if you pin an object to the stack. A mistake that is easy to make is, forgetting to shadow the original variable since you could drop the Pin and access the old value after it's initialized like this:
>
> ```rust
> fn main() {
>    let mut test1 = Test::new("test1");
>    let mut test1_pin = unsafe { Pin::new_unchecked(&mut test1) };
>    Test::init(test1_pin.as_mut());
>    drop(test1_pin);
>
>    let mut test2 = Test::new("test2");
>    mem::swap(&mut test1, &mut test2);
>    println!("Not self referential anymore: {:?}", test1.b);
> }
> ```

## 钉在堆上

> Pinning to the heap

为了完整起见，让我们去掉一些不安全的东西和对 `init` 方法的需求，代价是进行堆分配。钉在堆上是安全的，所以用户不需要实现任何不安全的代码。

> For completeness let's remove some unsafe and the need for an init method at the cost of a heap allocation. Pinning to the heap is safe so the user doesn't need to implement any unsafe code:

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}

impl Test {
    fn new(txt: &str) -> Pin<Box<Self>> {
        let t = Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned,
        };
        let mut boxed = Box::pin(t);
        let self_ptr: *const String = &boxed.as_ref().a;
        unsafe { boxed.as_mut().get_unchecked_mut().b = self_ptr };

        boxed
    }

    fn a<'a>(self: Pin<&'a Self>) -> &'a str {
        &self.get_ref().a
    }

    fn b<'a>(self: Pin<&'a Self>) -> &'a String {
        unsafe { &*(self.b) }
    }
}

pub fn main() {
    let mut test1 = Test::new("test1");
    let mut test2 = Test::new("test2");

    println!("a: {}, b: {}", test1.as_ref().a(), test1.as_ref().b());
    println!("a: {}, b: {}", test2.as_ref().a(), test2.as_ref().b());
}
```

事实上，即使是 `!Unpin`，钉住堆分配的数据也是安全的。一旦数据被分配到堆上，它将有一个稳定的地址。

> The fact that it's safe to pin heap allocated data even if it is !Unpin makes sense. Once the data is allocated on the heap it will have a stable address.

作为 API 的用户，我们没有必要特别注意以确保自引用的指针有效。

> There is no need for us as users of the API to take special care and ensure that the self-referential pointer stays valid.

有一些方法可以安全地给钉在栈上一些保证，但现在你需要使用像 [pin_project](https://docs.rs/pin-project/) 这样的 crate 来做。

> There are ways to safely give some guarantees on stack pinning as well, but right now you need to use a crate like pin_project to do that.

Practical rules for Pinning

If T: Unpin (which is the default), then Pin<'a, T> is entirely equivalent to &'a mut T. in other words: Unpin means it's OK for this type to be moved even when pinned, so Pin will have no effect on such a type.

Getting a &mut T to a pinned T requires unsafe if T: !Unpin. In other words: requiring a pinned pointer to a type which is !Unpin prevents the user of that API from moving that value unless they choose to write unsafe code.

Pinning does nothing special with memory allocation like putting it into some "read only" memory or anything fancy. It only uses the type system to prevent certain operations on this value.

Most standard library types implement Unpin. The same goes for most "normal" types you encounter in Rust. Futures and Generators are two exceptions.

The main use case for Pin is to allow self referential types, the whole justification for stabilizing them was to allow that.

The implementation behind objects that are !Unpin is most likely unsafe. Moving such a type after it has been pinned can cause the universe to crash. As of the time of writing this book, creating and reading fields of a self referential struct still requires unsafe (the only way to do it is to create a struct containing raw pointers to itself).

You can add a !Unpin bound on a type on nightly with a feature flag, or by adding std::marker::PhantomPinned to your type on stable.

You can either pin an object to the stack or to the heap.

Pinning a !Unpin object to the stack requires unsafe

Pinning a !Unpin object to the heap does not require unsafe. There is a shortcut for doing this using Box::pin.

Unsafe code does not mean it's literally "unsafe", it only relieves the guarantees you normally get from the compiler. An unsafe implementation can be perfectly safe to do, but you have no safety net.

Projection/structural pinning
In short, projection is a programming language term. mystruct.field1 is a projection. Structural pinning is using Pin on fields. This has several caveats and is not something you'll normally see so I refer to the documentation for that.

Pin and Drop
The Pin guarantee exists from the moment the value is pinned until it's dropped. In the Drop implementation you take a mutable reference to self, which means extra care must be taken when implementing Drop for pinned types.

Putting it all together
This is exactly what we'll do when we implement our own Future, so stay tuned, we're soon finished.

Bonus section: Fixing our self-referential generator and learning more about Pin
But now, let's prevent this problem using Pin. I've commented along the way to make it easier to spot and understand the changes we need to make.

```rust
#![feature(auto_traits, negative_impls)] // needed to implement `!Unpin`
use std::pin::Pin;

pub fn main() {
    let gen1 = GeneratorA::start();
    let gen2 = GeneratorA::start();
    // Before we pin the data, this is safe to do
    // std::mem::swap(&mut gen, &mut gen2);

    // constructing a `Pin::new()` on a type which does not implement `Unpin` is
    // unsafe. An object pinned to heap can be constructed while staying in safe
    // Rust so we can use that to avoid unsafe. You can also use crates like
    // `pin_utils` to pin to the stack safely, just remember that they use
    // unsafe under the hood so it's like using an already-reviewed unsafe
    // implementation.

    let mut pinned1 = Box::pin(gen1);
    let mut pinned2 = Box::pin(gen2);

    // Uncomment these if you think it's safe to pin the values to the stack instead
    // (it is in this case). Remember to comment out the two previous lines first.
    //let mut pinned1 = unsafe { Pin::new_unchecked(&mut gen1) };
    //let mut pinned2 = unsafe { Pin::new_unchecked(&mut gen2) };

    if let GeneratorState::Yielded(n) = pinned1.as_mut().resume() {
        println!("Gen1 got value {}", n);
    }

    if let GeneratorState::Yielded(n) = pinned2.as_mut().resume() {
        println!("Gen2 got value {}", n);
    };

    // This won't work:
    // std::mem::swap(&mut gen, &mut gen2);
    // This will work but will just swap the pointers so nothing bad happens here:
    // std::mem::swap(&mut pinned1, &mut pinned2);

    let _ = pinned1.as_mut().resume();
    let _ = pinned2.as_mut().resume();
}

enum GeneratorState<Y, R> {
    Yielded(Y),
    Complete(R),
}

trait Generator {
    type Yield;
    type Return;
    fn resume(self: Pin<&mut Self>) -> GeneratorState<Self::Yield, Self::Return>;
}

enum GeneratorA {
    Enter,
    Yield1 {
        to_borrow: String,
        borrowed: *const String,
    },
    Exit,
}

impl GeneratorA {
    fn start() -> Self {
        GeneratorA::Enter
    }
}

// This tells us that this object is not safe to move after pinning.
// In this case, only we as implementors "feel" this, however, if someone is
// relying on our Pinned data this will prevent them from moving it. You need
// to enable the feature flag `#![feature(optin_builtin_traits)]` and use the
// nightly compiler to implement `!Unpin`. Normally, you would use
// `std::marker::PhantomPinned` to indicate that the struct is `!Unpin`.
impl !Unpin for GeneratorA { }

impl Generator for GeneratorA {
    type Yield = usize;
    type Return = ();
    fn resume(self: Pin<&mut Self>) -> GeneratorState<Self::Yield, Self::Return> {
        // lets us get ownership over current state
        let this = unsafe { self.get_unchecked_mut() };
            match this {
            GeneratorA::Enter => {
                let to_borrow = String::from("Hello");
                let borrowed = &to_borrow;
                let res = borrowed.len();
                *this = GeneratorA::Yield1 {to_borrow, borrowed: std::ptr::null()};

                // Trick to actually get a self reference. We can't reference
                // the `String` earlier since these references will point to the
                // location in this stack frame which will not be valid anymore
                // when this function returns.
                if let GeneratorA::Yield1 {to_borrow, borrowed} = this {
                    *borrowed = to_borrow;
                }

                GeneratorState::Yielded(res)
            }

            GeneratorA::Yield1 {borrowed, ..} => {
                let borrowed: &String = unsafe {&**borrowed};
                println!("{} world", borrowed);
                *this = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}
```

Now, as you see, the consumer of this API must either:

Box the value and thereby allocating it on the heap
Use unsafe and pin the value to the stack. The user knows that if they move the value afterwards it will violate the guarantee they promise to uphold when they did their unsafe implementation.
Hopefully, after this you'll have an idea of what happens when you use the yield or await keywords inside an async function, and why we need Pin if we want to be able to safely borrow across yield/await points.
