# Rust 中的 Futures

> Futures in Rust

> **概述：**
>
> > Overview:
>
> - 对 Rust 中的并发概括性的介绍
> - 了解 Rust 在处理异步代码时提供了什么，没有提供什么
> - 了解为什么我们在 Rust 中需要一个运行时库
> - 了解“叶子 future”和“非叶子 future”之间的区别
> - 深入了解如何处理 CPU 密集型任务
>
> > - Get a high level introduction to concurrency in Rust
> > - Know what Rust provides and not when working with async code
> > - Get to know why we need a runtime-library in Rust
> > - Understand the difference between "leaf-future" and a "non-leaf-future"
> > - Get insight on how to handle CPU intensive tasks

## Futures

> Futures

那么 future 是什么？

> So what is a future?

future 是对一些将在未来完成的操作的表示。

> A future is a representation of some operation which will complete in the future.

Rust 中 的 async 使用了一种基于 `Poll` 的方法，其中一个异步任务将分为三个阶段。

> Async in Rust uses a Poll based approach, in which an asynchronous task will have three phases.

1. **轮询阶段。**一个 Future 被轮询，这导致任务的推进，直到它无法继续推进的那一点。我们通常把运行时中轮询 Future 的部分称为执行器。
2. **等待阶段。**一个事件源，最常被称为反应器，注册一个 Future 正在等待一个事件的发生，并确保它将在该事件准备好时唤醒 Future。
3. **唤醒阶段。**事件发生了，Future 被唤醒了。现在由在步骤 1 中对 Future 进行轮询的执行器来安排 Future再次被轮询，并进一步推进，直到它完成或达到一个新的无法继续推进的点，循环重复。

> 1. The Poll phase. A Future is polled which results in the task progressing until a point where it can no longer make progress. We often refer to the part of the runtime which polls a Future as an executor.
> 2. The Wait phase. An event source, most often referred to as a reactor, registers that a Future is waiting for an event to happen and makes sure that it will wake the Future when that event is ready.
> 3. The Wake phase. The event happens and the Future is woken up. It's now up to the executor which polled the Future in step 1 to schedule the future to be polled again and make further progress until it completes or reaches a new point where it can't make further progress and the cycle repeats.

现在，当我们谈论 future 时，我发现及早区分**非叶子** future 和**叶子** future 是很有用的，因为在实践中它们有很大的不同。

> Now, when we talk about futures I find it useful to make a distinction between non-leaf futures and leaf futures early on because in practice they're pretty different from one another.

## 叶子 future

> Leaf futures

运行时创建*叶子 future* 来表示资源，比如一个套接字。

> Runtimes create leaf futures which represent a resource like a socket.

```rust
// stream 是一个**叶子 future**
//
// > stream is a **leaf-future**
let mut stream = tokio::net::TcpStream::connect("127.0.0.1:3000");
```

对这些资源的操作，如对套接字的 `Read`，将是非阻塞的，并返回一个 future，我们称之为叶子 future，因为它是我们真正在等待的 future。

> Operations on these resources, like a Read on a socket, will be non-blocking and return a future which we call a leaf future since it's the future which we're actually waiting on.

除非你在写一个运行时，否则你不太可能自己实现一个叶子未来，但我们也会在本书中介绍它们是如何构造的。

> It's unlikely that you'll implement a leaf future yourself unless you're writing a runtime, but we'll go through how they're constructed in this book as well.

你也不太可能把一个叶子未来传递给运行时并单独运行到完成，你读完下一段就会明白。

> It's also unlikely that you'll pass a leaf-future to a runtime and run it to completion alone as you'll understand by reading the next paragraph.

## 非叶子 future

> Non-leaf-futures

非叶子 future 是我们作为运行时的*用户*使用 `async` 关键字自己编写的那种 future，以创建一个可以在执行器上运行的**任务**。

> Non-leaf-futures are the kind of futures we as users of a runtime write ourselves using the async keyword to create a task which can be run on the executor.

一个异步程序的大部分将由非叶子 future 组成，它是一种可暂停的计算。这是一个重要的区别，因为这些 future 代表了一组操作。通常，这样的任务会 `await` 一个叶子 future，作为完成任务的众多操作之一。

> The bulk of an async program will consist of non-leaf-futures, which are a kind of pause-able computation. This is an important distinction since these futures represents a set of operations. Often, such a task will await a leaf future as one of many operations to complete the task.

```rust
// 非叶子 future
//
// > Non-leaf-future
let non_leaf = async {
    let mut stream = TcpStream::connect("127.0.0.1:3000").await.unwrap();// <- yield
    println!("connected!");
    let result = stream.write(b"hello world\n").await; // <- yield
    println!("message sent!");
    ...
};
```

这些任务的关键在于，它们能够将控制权交给运行时的调度器，然后在稍后的时间点上重新恢复执行。

> The key to these tasks is that they're able to yield control to the runtime's scheduler and then resume execution again where it left off at a later point.

与叶子 future 相比，这类 future 本身并不代表I/O资源。当我们对它们进行轮询时，它们会一直运行到一个返回 `Pending` 的叶子 future，然后将控制权交给调度器（这是我们所说的运行时的一部分）。

> In contrast to leaf futures, these kind of futures do not themselves represent an I/O resource. When we poll them they will run until they get to a leaf-future which returns Pending and then yield control to the scheduler (which is a part of what we call the runtime).
