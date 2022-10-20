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

## 运行时

> Runtimes

像 C#、JavaScript、Java、GO 和其他许多语言都带有处理并发性的运行时。因此，如果你来自这些语言之一，你可能会对这点感到奇怪。

> Languages like C#, JavaScript, Java, GO, and many others comes with a runtime for handling concurrency. So if you come from one of those languages this will seem a bit strange to you.

Rust 与这些语言的不同之处在于 Rust 没有自带处理并发的运行时，所以你需要使用一个库来为你提供这个。

> Rust is different from these languages in the sense that Rust doesn't come with a runtime for handling concurrency, so you need to use a library which provides this for you.

Futures 的相当多的复杂性实际上是源于运行时的复杂性；创建一个高效的运行时是很难的。

> Quite a bit of complexity attributed to Futures is actually complexity rooted in runtimes; creating an efficient runtime is hard.

学习如何正确使用运行时也需要相当大的努力，但你会发现这些运行时之间有一些相似之处，所以学习一个运行时会使学习下一个运行时更加容易。

> Learning how to use one correctly requires quite a bit of effort as well, but you'll see that there are several similarities between these kind of runtimes, so learning one makes learning the next much easier.

Rust 和其他语言的区别在于，你必须主动地选择运行时。在其他语言中，大多数情况下，你只是使用为你提供的那个。

> The difference between Rust and other languages is that you have to make an active choice when it comes to picking a runtime. Most often in other languages, you'll just use the one provided for you.

### 一个有用的异步运行时心智模型

> A useful mental model of an async runtime

我发现通过建立一个我们可以使用的概括性的心智模型，推理 Futures 的工作原理将变得容易。要做到这一点，我必须引入一个运行时的概念，它将驱动我们的 Futures 完成。

> I find it easier to reason about how Futures work by creating a high level mental model we can use. To do that I have to introduce the concept of a runtime which will drive our Futures to completion.

> 请注意，我在这里创建的心智模型并不是推动 Futures 完成的唯一方法，Rust 的 Futures 并没有对你实际完成这项任务的方式做出任何限制。
>
> > Please note that the mental model I create here is not the only way to drive Futures to completion and that Rust’s Futures does not impose any restrictions on how you actually accomplish this task.

**Rust 中一个完整的异步系统可以分为三个部分：**

> A fully working async system in Rust can be divided into three parts:

1. 反应器
2. 执行器
3. Future

> 1. Reactor
> 2. Executor
> 3. Future

那么，这三个部分是如何协同工作的呢？他们通过一个叫做 `Waker` 的对象来实现。Waker 是反应器告诉执行器一个特定的 Future 已经准备好运行的方式。一旦你理解了 Waker 的生命周期和所有权，你就会能用户的角度理解 future 如何工作了。这是生命周期：

> So, how does these three parts work together? They do that through an object called the Waker. The Waker is how the reactor tells the executor that a specific Future is ready to run. Once you understand the life cycle and ownership of a Waker, you'll understand how futures work from a user's perspective. Here is the life cycle:

- Waker 是由**执行器**创建的。一个常见的，但不是必须的方法是为每个在执行器那里注册的 future 创建一个新的 Waker。
- 当一个 future 在执行器处注册时，它被赋予一个由执行器创建的 Waker 对象的克隆。由于这是一个共享对象（例如 `Arc<T>`），所有的克隆实际上都指向同一个基础对象。因此，任何调用原始 Waker 的克隆的东西都会唤醒注册在它身上的特定Future。
- future 克隆了 Waker 并将其传递给反应器，反应器将其存储起来以便日后使用。

> - A Waker is created by the executor. A common, but not required, method is to create a new Waker for each Future that is registered with the executor.
> - When a future is registered with an executor, it’s given a clone of the Waker object created by the executor. Since this is a shared object (e.g. an `Arc<T>`), all clones actually point to the same underlying object. Thus, anything that calls any clone of the original Waker will wake the particular Future that was registered to it.
> - The future clones the Waker and passes it to the reactor, which stores it to use later.

在未来的某个时刻，反应器将决定 future 可以运行了。它将通过它所存储的 Waker 唤醒 future。这个动作将做一些必要的事情，以使执行器处于可以轮询 future 的位置。

> At some point in the future, the reactor will decide that the future is ready to run. It will wake the future via the Waker that it stored. This action will do what is necessary to get the executor in a position to poll the future.

Waker 对象实现了我们与任务相关的一切。该对象是由正在使用的执行器的类型决定的，但是所有的 Waker 共享一个相同的接口（它不是一个 trait，因为嵌入式系统不能处理 trait object，但是一个有用的抽象是把它看成一个 trait object）。

> The Waker object implements everything that we associate with task. The object is specific to the type of executor in use, but all Wakers share a similar interface (it's not a trait because embedded systems can't handle trait objects, but a useful abstraction is to think of it as a trait object).

由于接口在所有的执行器中都是相同的，理论上反应器可以完全无视执行器的类型，反之亦然。执行器和反应器永远不需要彼此直接沟通。

> Since the interface is the same across all executors, reactors can in theory be completely oblivious to the type of the executor, and vice-versa. Executors and reactors never need to communicate with one another directly.

这种设计赋予了 futures 框架强大的功能和灵活性，并允许 Rust 标准库为我们提供一个符合人体工程学的、零成本的抽象概念。

> This design is what gives the futures framework it's power and flexibility and allows the Rust standard library to provide an ergonomic, zero-cost abstraction for us to use.

为了尝试直观地了解这些部分是如何一起工作的，我在下一章中放了一组幻灯片，希望能有所帮助。

> In an effort to try to visualize how these parts work together I put together a set of slides in the next chapter that I hope will help.

截至本文写作时，Futures最流行的两个运行时是：

> The two most popular runtimes for Futures as of writing this is:

- [async-std](https://crates.io/crates/async-std)
- [Tokio](https://crates.io/crates/tokio)

### Rust 的标准库负责处理的事

> What Rust's standard library takes care of

1. 一个通用的接口，通过 `Future` trait 表示一个将在未来完成的操作。
2. 通过 `async` 和 `await` 关键字创建可以暂停和恢复的任务，这是一种符合人体工程学的方法。
3. 通过 Waker 类型定义的接口来唤醒一个暂停的任务。

> 1. A common interface representing an operation which will be completed in the future through the Future trait.
> 2. An ergonomic way of creating tasks which can be suspended and resumed through the async and await keywords.
> 3. A defined interface to wake up a suspended task through the Waker type.

这就是 Rust 的标准库的真正作用。正如你所看到的，没有对非阻塞 I/O 的定义，也没有对这些任务如何创建以及如何运行的定义。

> That's really what Rust's standard library does. As you see there is no definition of non-blocking I/O, how these tasks are created, or how they're run.

## I/O密集型 vs CPU密集型任务

> I/O vs CPU intensive tasks

正如你们现在知道的，你通常写的东西被称为非叶子 future。让我们来看看这个用伪 rust 描述的异步块示例：

> As you know now, what you normally write are called non-leaf futures. Let's take a look at this async block using pseudo-rust as example:

```rust
let non_leaf = async {
    let mut stream = TcpStream::connect("127.0.0.1:3000").await.unwrap(); // <-- yield

    // request a large dataset
    let result = stream.write(get_dataset_request).await.unwrap(); // <-- yield

    // wait for the dataset
    let mut response = vec![];
    stream.read(&mut response).await.unwrap(); // <-- yield

    // do some CPU-intensive analysis on the dataset
    let report = analyzer::analyze_data(response).unwrap();

    // send the results back
    stream.write(report).await.unwrap(); // <-- yield
};
```

现在，正如你在了解 Futures 的工作原理时看到的那样，我们在让出点之间编写的代码与我们的执行器在同一个线程上运行。

> Now, as you'll see when we go through how Futures work, the code we write between the yield points are run on the same thread as our executor.

这意味着，当我们的分析器在数据集上工作时，执行器正忙于进行计算而不是处理新的请求。

> That means that while our analyzer is working on the dataset, the executor is busy doing calculations instead of handling new requests.

幸运的是，有几种方法可以处理这个问题，这并不困难，但这是你必须注意的：

> Fortunately there are a few ways to handle this, and it's not difficult, but it's something you must be aware of:

1. 我们可以创建一个新的叶子 future，将我们的任务发送到另一个线程，并在任务完成后进行接收。我们可以像 `await` 其他的 future 一样等待这个叶子 future。
2. 运行时可以有某种管理器来监控不同的任务需要多少时间，并将执行器本身移到不同的线程，这样它就可以继续运行，即使我们的分析器任务阻塞了执行器原本的线程。
3. 你可以自己创建一个与运行时兼容的反应器，以你认为合适的方式进行分析，并返回一个可以被 `await` 的 future。

> 1. We could create a new leaf future which sends our task to another thread and resolves when the task is finished. We could await this leaf-future like any other future.
> 2. The runtime could have some kind of supervisor that monitors how much time different tasks take, and move the executor itself to a different thread so it can continue to run even though our analyzer task is blocking the original executor thread.
> 3. You can create a reactor yourself which is compatible with the runtime which does the analysis any way you see fit, and returns a Future which can be awaited.

现在，#1 是处理这个问题的通常方式，但有些执行器也实现了 #2。＃2 的问题是，如果你切换运行时，你需要确保它也支持这种监督，否则你最终将阻塞执行器。

> Now, #1 is the usual way of handling this, but some executors implement #2 as well. The problem with #2 is that if you switch runtime you need to make sure that it supports this kind of supervision as well or else you will end up blocking the executor.

而 #3 更多的是理论上的重要性，通常情况下，你只要把任务发送到大多数运行时提供的线程池中就可以了。

> And #3 is more of theoretical importance, normally you'd be happy by sending the task to the thread-pool most runtimes provide.

大多数执行器都可以使用诸如 `spawn_blocking` 之类的方法来完成 #1。

> Most executors have a way to accomplish #1 using methods like spawn_blocking.

这些方法将任务发送到由运行时创建的线程池中，在那里你可以执行 CPU 密集型任务或运行时不支持的“阻塞”任务。

> These methods send the task to a thread-pool created by the runtime where you can either perform CPU-intensive tasks or "blocking" tasks which are not supported by the runtime.

现在，有了这些知识，你已经在理解 Futures 的道路上走得很好了，但我们还不会停止，还有很多细节要讲。

> Now, armed with this knowledge you are already on a good way for understanding Futures, but we're not gonna stop yet, there are lots of details to cover.

休息一下或喝杯咖啡，准备好在接下来的章节中进行深入研究。

> Take a break or a cup of coffee and get ready as we go for a deep dive in the next chapters.

## 想了解更多关于并发和异步的知识吗？

> Want to learn more about concurrency and async?

如果你觉得并发和异步编程的概念总体上令人困惑，我知道你的想法，我写了一些资源，试图给出一个概括性的概述，这将使你以后更容易学习 Rust 的 Futures：

> If you find the concepts of concurrency and async programming confusing in general, I know where you're coming from and I have written some resources to try to give a high-level overview that will make it easier to learn Rust's Futures afterwards:

- [异步基础知识 - 并发和并行的区别](https://cfsamson.github.io/book-exploring-async-basics/1_concurrent_vs_parallel.html)
- [异步基础知识 - 异步的历史](https://cfsamson.github.io/book-exploring-async-basics/2_async_history.html)
- [异步基础知识 - 处理I/O的策略](https://cfsamson.github.io/book-exploring-async-basics/5_strategies_for_handling_io.html)
- [异步基础知识 - Epoll、Kqueue 和 IOCP](https://cfsamson.github.io/book-exploring-async-basics/6_epoll_kqueue_iocp.html)

> - Async Basics - The difference between concurrency and parallelism
> - Async Basics - Async history
> - Async Basics - Strategies for handling I/O
> - Async Basics - Epoll, Kqueue and IOCP

没必要通过学习 futures 来学习这些概念，所以如果你觉得有点不确定，就继续阅读这些章节吧。

> Learning these concepts by studying futures is making it much harder than it needs to be, so go on and read these chapters if you feel a bit unsure.

我会等着你回到这里来。

> I'll be right here when you're back.

不过，如果你觉得你已经掌握了基本知识，那么我们就开始行动吧！

> However, if you feel that you have the basics covered, then let's get moving!

## 补充章节--关于 Futures 和 Wakers 的补充说明

> Bonus section - additional notes on Futures and Wakers

在这一节中，我们将深入研究异步运行时的执行器部分和反应器部分之间松耦合的一些优势。

> In this section we take a deeper look at some advantages of having a loose coupling between the Executor-part and Reactor-part of an async runtime.

本章的前面，我提到执行器通常为每个在执行器上注册的 Future 创建一个新的 Waker，但 Waker 是一个类似于 `Arc<T>` 的共享对象。这种设计的原因之一是，它允许不同的反应器都能唤醒 Future。

> Earlier in this chapter, I mentioned that it is common for the executor to create a new Waker for each Future that is registered with the executor, but that the Waker is a shared object similar to a `Arc<T>`. One of the reasons for this design is that it allows different Reactors the ability to Wake a Future.

用一个例子来介绍用法，考虑一下如何创建一个新类型的 Future，它能被取消：

> As an example of how this can be used, consider how you could create a new type of Future that has the ability to be canceled:

实现这一点的一个方法是在 Future 的实例中添加一个 [AtomicBool](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicBool.html)，以及一个额外的名为 `cancel()` 的方法。`cancel()` 方法将首先设置 [AtomicBool](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicBool.html)，以示 Future 已被取消，然后立即调用实例自己的 Waker 副本。

> One way to achieve this would be to add an AtomicBool to the instance of the future, and an extra method called cancel(). The cancel() method will first set the AtomicBool to signal that the future is now canceled, and then immediately call instance's own copy of the Waker.

一旦执行器开始执行 Future，Future 就会知道它被取消了，并且会做适当的清理动作来终止自己。

> Once the executor starts executing the Future, the Future will know that it was canceled, and will do the appropriate cleanup actions to terminate itself.

以这种方式设计 Future 的主要原因是我们不需要修改执行器或其他反应器；这种变化对它们来说不可见。

> The main reason for designing the Future in this manner is because we don't have to modify either the Executor or the other Reactors; they are all oblivious to the change.

唯一可能的问题是关于 Future 本身的设计；一个被取消的 Future 仍然需要根据 [Future](https://doc.rust-lang.org/std/future/trait.Future.html) 文档中列出的规则正确地终止。这意味着，它不能只是删除它的资源，然后坐在那里；它需要返回一个值。你可以决定一个被取消的 Future 是否会永远返回 [Pending](https://doc.rust-lang.org/std/task/enum.Poll.html#variant.Pending)，或者是否会在 [Ready](https://doc.rust-lang.org/std/task/enum.Poll.html#variant.Ready) 中返回一个值。请注意，如果其他 Future 正在等待它，它们将无法启动，直到返回 [Ready](https://doc.rust-lang.org/std/task/enum.Poll.html#variant.Ready)。

> The only possible issue is with the design of the Future itself; a Future that is canceled still needs to terminate correctly according to the rules outlined in the docs for Future. That means that it can't just delete it's resources and then sit there; it needs to return a value. It is up to you to decide if a canceled future will return Pending forever, or if it will return a value in Ready. Just be aware that if other Futures are awaiting it, they won't be able to start until Ready is returned.

对于可取消的 Future，一个常见的技巧是让它们返回一个带有错误的结果，表明该 Future 被取消了；这将允许任何正在等待被取消的 Future 的人有机会推进，因为他们知道他们所依赖的 Future 被取消了。还有其他的问题，但超出了本书的范围。阅读 [futures](https://crates.io/crates/futures) 的文档和代码，可以更好地了解这些问题是什么。

> A common technique for cancelable Futures is to have them return a Result with an error that signals the Future was canceled; that will permit any Futures that are awaiting the canceled Future a chance to progress, with the knowledge that the Future they depended on was canceled. There are additional concerns as well, but beyond the scope of this book. Read the documentation and code for the futures crate for a better understanding of what the concerns are.

> 感谢 @ckaran 贡献的这段额外内容。
>
> > Thanks to @ckaran for contributing this bonus segment.

---

[下一节：Futures 和运行时工作原理的心智模型](a-mental-model-of-how-futures-and-runtimes-work.md)
