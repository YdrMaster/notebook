# 实现 Futures——主要示例

> Implementing Futures - main example

我们将用一个假的反应器和一个简单的执行器来创建我们自己的 Futures，它允许你在浏览器中编辑、运行和修改这些代码。

> We'll create our own Futures together with a fake reactor and a simple executor which allows you to edit, run an play around with the code right here in your browser.

我将引导你完成这个例子，但如果你想深入探索，你可以随时[克隆仓库](https://github.com/cfsamson/examples-futures)并自己修改代码，或者直接从下一章中复制它。

> I'll walk you through the example, but if you want to check it out closer, you can always clone the repository and play around with the code yourself or just copy it from the next chapter.

自述文件中解释了它的几个分支，但有两个与本章有关。`main` 分支是我们在这里讨论的例子，而 `basic_example_commented` 分支是向这个例子增加大量注释的版本。

> There are several branches explained in the readme, but two are relevant for this chapter. The main branch is the example we go through here, and the basic_example_commented branch is this example with extensive comments.

>如果你想（在本机）同步完成我们的进度，请创建一个新的文件夹，并在其中运行 `cargo init`，以此来初始化一个新的 cargo 项目。我们在这里写的所有内容都可以放在 `main.rs`。
>
> > If you want to follow along as we go through, initialize a new cargo project by creating a new folder and run cargo init inside it. Everything we write here will be in main.rs

## 实现我们自己的 Futures

> Implementing our own Futures

让我们先把所有的导入弄好，方便你跟上

> Let's start off by getting all our imports right away so you can follow along

```rust
use std::{
    future::Future, pin::Pin, sync::{ mpsc::{channel, Sender}, Arc, Mutex,},
    task::{Context, Poll, RawWaker, RawWakerVTable, Waker}, mem,
    thread::{self, JoinHandle}, time::{Duration, Instant}, collections::HashMap
};
```

## 执行器

> The Executor

执行器的职责是持有一个或多个 future 并执行它们直到完成。

> The executors responsibility is to take one or more futures and run them to completion.

当执行器拿到一个 `Future`，首先要做的就是轮询它。

> The first thing an executor does when it gets a Future is polling it.

轮询时，这三种情况之一将会发生：

> When polled one of three things can happen:

- future 返回 `Ready`，我们可以调度并执行任何连锁的操作
- future 之前从未被轮询过，所以我们传给它一个 `Waker` 并暂停它
- future 已经被轮询过，但还没准备好，返回 `Pending`

> - The future returns Ready and we schedule whatever chained operations to run
> - The future hasn't been polled before so we pass it a Waker and suspend it
> - The futures has been polled before but is not ready and returns Pending

Rust 为反应器和执行器提供了一种通过 `Waker` 进行通信的方法。反应器存储这个 `Waker`，一旦一个 Future 解决了，应该再次被轮询，就对它调用`Waker::wake()`。

> Rust provides a way for the Reactor and Executor to communicate through the Waker. The reactor stores this Waker and calls Waker::wake() on it once a Future has resolved and should be polled again.

> 请注意，本章有一个补充节，叫做[park 线程的正确方法](#补充章节暂停线程的正确方法)，它展示了如何避免 `thread::park`。
>
> Notice that this chapter has a bonus section called A Proper Way to Park our Thread which shows how to avoid thread::park.

**我们的执行器将看起来像这样：**

> Our Executor will look like this:

```rust
// 我们的执行器接受任何实现了 `Future` 特质的对象
//
// > Our executor takes any object which implements the `Future` trait
fn block_on<F: Future>(mut future: F) -> F::Output {

    // 我们做的第一件事是构造一个 `Waker`，它会被传给 `reactor`，并在事件就绪时唤醒我们。
    //
    // > the first thing we do is to construct a `Waker` which we'll pass on to
    // > the `reactor` so it can wake us up when an event is ready.
    let mywaker = Arc::new(MyWaker{ thread: thread::current() });
    // > **补充说明** `Arc<MyWaker>` 转化成裸指针存到了 `waker` 里，这个 `Arc` 只提供引用计数的结构，失去了 RAII。
    let waker = mywaker_into_waker(Arc::into_raw(mywaker));

    // 上下文结构体只是 `Waker` 对象的一个包装。可能将来会添加更多东西，但现在它就是一个包装。
    //
    // > The context struct is just a wrapper for a `Waker` object. Maybe in the
    // > future this will do more, but right now it's just a wrapper.
    let mut cx = Context::from_waker(&waker);

    // 因为我们这个执行器在单线程上运行一个 future 到完成，我们可以把 `Future` 钉在栈上。
    // 这是不安全的，但节省一次分配。如果需要，我们也可以 `Box::pin`。
    // 但这其实是安全的，因为我们遮蔽了 `future`，所以它不可能再次访问，直到析构都不会移动。
    //
    // > So, since we run this on one thread and run one future to completion
    // > we can pin the `Future` to the stack. This is unsafe, but saves an
    // > allocation. We could `Box::pin` it too if we wanted. This is however
    // > safe since we shadow `future` so it can't be accessed again and will
    // > not move until it's dropped.
    //
    // > **补充说明** 这意味着我们告诉编译器，`future` 已经钉住了，不会移动。它是 unsafe，因为从此以后不移动 future 的职责落到我们肩上了。
    let mut future = unsafe { Pin::new_unchecked(&mut future) };

    // 我们在一个循环里轮询，但它不是忙等。当这两种情况之一发生时它才会运行：一个事件发生，或者一次“假唤醒”
    // （无理由发生的意料之外的唤醒）。
    //
    // > We poll in a loop, but it's not a busy loop. It will only run when
    // > an event occurs, or a thread has a "spurious wakeup" (an unexpected wakeup
    // > that can happen for no good reason).
    let val = loop {
        match Future::poll(future.as_mut(), &mut cx) {

            // 如果 Future 完成，我们就可以结束了
            //
            // > when the Future is ready we're finished
            Poll::Ready(val) => break val,

            // 如果我们收到一个 `Pending` future 就去睡个回笼觉
            //
            // > If we get a `pending` future we just go to sleep...
            Poll::Pending => thread::park(),
        };
    };
    val
}
```

在本章中你能看到的所有例子中，我选择对代码进行大量的注释。我发现这样做更容易理解，所以我不会在文档中重复注释的内容，只关注一些可能需要进一步解释的重点。

> In all the examples you'll see in this chapter I've chosen to comment the code extensively. I find it easier to follow along that way so I'll not repeat myself here and focus only on some important aspects that might need further explanation.

值得注意的是，像我们在这里做的那样简单地调用 `thread::park` 会导致死锁和错误。如果你一直读到本章末尾的[补充章节](#补充章节暂停线程的正确方法)，我们会在那里进一步解释并解决这个问题。

> It's worth noting that simply calling thread::park as we do here can lead to both deadlocks and errors. We'll explain a bit more later and fix this if you read all the way to the Bonus Section at the end of this chapter.

现在，我们尽量保持简单易懂，睡就完事了。

> For now, we keep it as simple and easy to understand as we can by just going to sleep.

既然你已经读了这么多关于生成器和 `Pin` 的文章，这应该是相当容易理解的。Future 是一个状态机，每个等待点都是一个让出点。我们可以在等待点之间借用数据，我们遇到的挑战与我们在让出点之间借用时完全一样。

> Now that you've read so much about Generators and Pin already this should be rather easy to understand. Future is a state machine, every await point is a yield point. We could borrow data across await points and we meet the exact same challenges as we do when borrowing across yield points.

> `Context` 只是对 `Waker` 的一个包装。在写这本书的时候，它不过如此。在未来，`Context` 对象可能会做更多的事情，而不仅仅是包装一个 `Waker`，所以有了这个额外的抽象来提供一些灵活性。
>
> > Context is just a wrapper around the Waker. At the time of writing this book it's nothing more. In the future it might be possible that the Context object will do more than just wrapping a Waker so having this extra abstraction gives some flexibility.

正如在[关于 Pin 的章节](pin.md)中所解释的，我们使用 `Pin` 和它所提供的保证来允许 Future 有自引用。

> As explained in the chapter about Pin, we use Pin and the guarantees that give us to allow Futures to have self references.

## 实现 Future

> The Future implementation

Futures 有一个定义良好的接口，这意味着它们可以在整个生态系统中使用。

> Futures has a well defined interface, which means they can be used across the entire ecosystem.

我们可以将这些 Futures 连锁起来，一旦一个叶子 future 准备好了，我们就会执行一系列的操作，直到任务完成或者我们到达另一个叶子 future，我们会等待并将控制权交给调度器。

> We can chain these Futures so that once a leaf-future is ready we'll perform a set of operations until either the task is finished or we reach yet another leaf-future which we'll wait for and yield control to the scheduler.

**我们的 Future 实现看起来像这样：**

> Our Future implementation looks like this:

```rust
// 这是我们的 `Waker` 的定义。我们在这里放了一个普通的线程句柄。
// 这个能用，但不是一个优秀方案。不过这很容易优化，我会在这段代码之后解释。
//
// > This is the definition of our `Waker`. We use a regular thread-handle here.
// > It works but it's not a good solution. It's easy to fix though, I'll explain
// > after this code snippet.
#[derive(Clone)]
struct MyWaker {
    thread: thread::Thread,
}

// 这是我们的 `Future` 的定义。`Future` 会持有我们需要的一切信息。
// 这一个持有一个 `reactor` 的引用，这只是为了让这个示例尽量简单。
// 它并不需要持有整个反应器的引用，但确实需要能向反应器注册自己。
//
// > This is the definition of our `Future`. It keeps all the information we
// > need. This one holds a reference to our `reactor`, that's just to make
// > this example as easy as possible. It doesn't need to hold a reference to
// > the whole reactor, but it needs to be able to register itself with the
// > reactor.
#[derive(Clone)]
pub struct Task {
    id: usize,
    reactor: Arc<Mutex<Box<Reactor>>>,
    data: u64,
}

// 这是我们会在我们的 waker 里用到的函数定义。回忆前面的“特质对象”章节。
//
// > These are function definitions we'll use for our waker. Remember the
// > "Trait Objects" chapter earlier.
//
// > **补充说明** 这个操作会使 `MyWaker` 的引用计数 -1。
fn mywaker_wake(s: &MyWaker) {
    let waker_ptr: *const MyWaker = s;
    let waker_arc = unsafe {Arc::from_raw(waker_ptr)};
    waker_arc.thread.unpark();
}

// 因为我们使用了 `Arc`，克隆就是让智能指针的引用计数增加。
//
// > Since we use an `Arc` cloning is just increasing the refcount on the smart
// > pointer.
fn mywaker_clone(s: &MyWaker) -> RawWaker {
    let arc = unsafe { Arc::from_raw(s) };
    std::mem::forget(arc.clone()); // 增加引用计数 increase ref count
    RawWaker::new(Arc::into_raw(arc) as *const (), &VTABLE)
}

// 这实际上是一个“辅助函数”，用于创建一个 `Waker` 虚表。
// 相比于从头开始创建一个特质对象，我们不需要关心虚表的实际布局，只需要提供一组固定的函数
//
// This is actually a "helper funtcion" to create a `Waker` vtable. In contrast
// to when we created a `Trait Object` from scratch we don't need to concern
// ourselves with the actual layout of the `vtable` and only provide a fixed
// set of functions
//
// > **补充说明** ↑ 里的 `This` 指的是 `RawWakerVTable::new`。
const VTABLE: RawWakerVTable = unsafe {
    RawWakerVTable::new(
        |s| mywaker_clone(&*(s as *const MyWaker)),   // 克隆 clone
        |s| mywaker_wake(&*(s as *const MyWaker)),    // 唤醒 wake
        |s| (*(s as *const MyWaker)).thread.unpark(), // 通过引用唤醒（不减少引用计数）wake by ref (don't decrease refcount)
        |s| drop(Arc::from_raw(s as *const MyWaker)), // 减少引用计数 decrease refcount
    )
};

// 我们就不在 `MyWaker` 对象的 `impl Mywaker...`里实现了，而是使用这个函数，因为它可以节省一些代码行。
//
// > Instead of implementing this on the `MyWaker` object in `impl Mywaker...` we
// > just use this pattern instead since it saves us some lines of code.
fn mywaker_into_waker(s: *const MyWaker) -> Waker {
    let raw_waker = RawWaker::new(s as *const (), &VTABLE);
    unsafe { Waker::from_raw(raw_waker) }
}

impl Task {
    fn new(reactor: Arc<Mutex<Box<Reactor>>>, data: u64, id: usize) -> Self {
        Task { id, reactor, data }
    }
}

// 这是我们的 `Future` 实现
//
// > This is our `Future` implementation
impl Future for Task {
    type Output = usize;

    // 轮询驱动状态机推进，并且这也是我们驱动 future 完成的过程中要调用的唯一的方法。
    //
    // > Poll is the what drives the state machine forward and it's the only
    // > method we'll need to call to drive futures to completion.
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {

        // 我们在我们的 `poll` 方法里要访问反应器，所以我们先获取它的锁。
        //
        // > We need to get access the reactor in our `poll` method so we acquire
        // > a lock on that.
        let mut r = self.reactor.lock().unwrap();

        // 首先我们检查这个任务是不是已经标记为完成
        //
        // > First we check if the task is marked as ready
        if r.is_ready(self.id) {

            // 如果已经完成，把它的状态设置成 `Finished`
            //
            // > If it's ready we set its state to `Finished`
            *r.tasks.get_mut(&self.id).unwrap() = TaskState::Finished;
            Poll::Ready(self.id)

        // 如果它尚未完成，检查它的 id 是不是在我们保存在反应器里的映射表中
        //
        // > If it isn't finished we check the map we have stored in our Reactor
        // > over id's we have registered and see if it's there
        } else if r.tasks.contains_key(&self.id) {

            // 这很重要。文档中说，在多次调用轮询时，只有向最近一次调用传递的上下文里的 Waker 唤醒时才应该调度。
            // 这就是为什么我们在返回 `Pending` 之前将这个 Waker 插入到映射表（将返回旧的 Waker，后者将被丢弃）。
            //
            // > This is important. The docs says that on multiple calls to poll,
            // > only the Waker from the Context passed to the most recent call
            // > should be scheduled to receive a wakeup. That's why we insert
            // > this waker into the map (which will return the old one which will
            // > get dropped) before we return `Pending`.
            r.tasks.insert(self.id, TaskState::NotReady(cx.waker().clone()));
            Poll::Pending
        } else {

            // 如果它尚未完成也不在映射表中，它就是一个新任务，我们把它注册到反应器并返回 `Pending`。
            //
            // > If it's not ready, and not in the map it's a new task so we
            // > register that with the Reactor and return `Pending`
            r.register(self.data, cx.waker().clone(), self.id);
            Poll::Pending
        }

        // 请注意，我们在持有一个 `Mutex` 的锁，它一直保护着反应器，直到这个作用域结束。
        // 这意味着，即使我们的任务立即完成，当我们在 `Poll` 方法中时，它也无法调用 `wake`。
        //
        // > Note that we're holding a lock on the `Mutex` which protects the
        // > Reactor all the way until the end of this scope. This means that
        // > even if our task were to complete immidiately, it will not be
        // > able to call `wake` while we're in our `Poll` method.

        // 既然我们可以做出这样的保证，那么现在处理这种可能的竞态就是执行器的职责了，
        // 即 `Wake` 在 `poll` 之后但在我们的线程进入睡眠之前被调用。
        //
        // Since we can make this guarantee, it's now the Executors job to
        // handle this possible race condition where `Wake` is called after
        // `poll` but before our thread goes to sleep.
    }
}
```

这大部分是很直接的。令人困惑的部分是我们需要用奇怪的方式来构建 `Waker`，但由于我们已经用原始部件创建过自己的特质对象，这看起来很熟悉。事实上，这甚至有点简单。

> This is mostly pretty straight forward. The confusing part is the strange way we need to construct the Waker, but since we've already created our own trait objects from raw parts, this looks pretty familiar. Actually, it's even a bit easier.

我们在这里使用一个 `Arc` 来传递出我们的 `MyWaker` 的引用计数借用。这是很正常的，并且使其简单而安全地工作。在这种情况下，克隆一个 `Waker` 只是增加引用计数。

> We use an Arc here to pass out a ref-counted borrow of our MyWaker. This is pretty normal, and makes this easy and safe to work with. Cloning a Waker is just increasing the refcount in this case.

丢弃一个 Waker 就是简单地减少引用计数。现在，在特殊情况下我们可以选择不使用 `Arc`。所以这个底层方法是为了允许这种情况的发生。

> Dropping a Waker is as easy as decreasing the refcount. Now, in special cases we could choose to not use an Arc. So this low-level method is there to allow such cases.

事实上，如果我们只使用 `Arc`，那么我们就没有理由用这么麻烦的方法创建我们自己的虚表和 `RawWaker`。我们可以直接实现一个普通的特质。

> Indeed, if we only used Arc there is no reason for us to go through all the trouble of creating our own vtable and a RawWaker. We could just implement a normal trait.

幸运的是，以后这可能也会在标准库中实现。目前，这个特质尚在萌芽，但我猜测在经过一段时间的成熟后，它将成为标准库的一部分。

> Fortunately, in the future this will probably be possible in the standard library as well. For now, this trait lives in the nursery, but my guess is that this will be a part of the standard library after some maturing.

我们选择在这里传递对整个反应器的引用。这并不正常。反应器通常是一个全局资源，它可以让我们在不传递引用的情况下注册关注点。

> We choose to pass in a reference to the whole Reactor here. This isn't normal. The reactor will often be a global resource which let's us register interests without passing around a reference.

> ## 为什么使用线程暂停/继续对一个库来说是个坏主意？
>
> > Why using thread park/unpark is a bad idea for a library
>
> 因为任何人都可以得到执行器线程的句柄，并在我们的线程上调用暂停/继续，这很容易造成死锁。我[在 playground 上做了一个带注释的例子](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=b2343661fe3d271c91c6977ab8e681d0)，展示了这种错误是如何发生的。你也可以 future crate 的 [issue 2010](https://github.com/rust-lang/futures-rs/pull/2010) 中读到更多关于这方面的内容。
>
> > It could deadlock easily since anyone could get a handle to the executor thread and call park/unpark on our thread. I've made an example with comments on the playground that showcases how such an error could occur. You can also read a bit more about this in issue 2010 in the futures crate.

## 反应器

> The Reactor

这是最后冲刺，严格来说与 Future 无关，但我们需要一个例子来运行。

> This is the home stretch, and not strictly Future related, but we need one to have an example to run.

由于在与外界（或至少是一些外设）进行交互时，并发大多是有意义的，所以我们需要一些东西来实际地以异步方式抽象出这种交互。

> Since concurrency mostly makes sense when interacting with the outside world (or at least some peripheral), we need something to actually abstract over this interaction in an asynchronous way.

这就是反应器的工作。大多数情况下，你会看到 Rust 中的反应器使用一个叫做 [Mio](https://github.com/tokio-rs/mio) 的库，它为几个平台提供非阻塞 API 和事件通知。

> This is the Reactors job. Most often you'll see reactors in Rust use a library called Mio, which provides non blocking APIs and event notification for several platforms.

反应器通常会给你一些东西，比如 `TcpStream`（或任何其他资源），你会用它来创建一个 I/O 请求。你得到一个 Future 的返回。

> The reactor will typically give you something like a TcpStream (or any other resource) which you'll use to create an I/O request. What you get in return is a Future.

如果我们的反应器做一些真正的 I/O 工作，我们的任务就会代表一个非阻塞的 `TcpStream`，它在全局反应器中注册了关注点。传递对反应器本身的引用很不常见，但我发现它使理解正在发生的事情更容易。

> If our reactor did some real I/O work our Task in would instead be represent a non-blocking TcpStream which registers interest with the global Reactor. Passing around a reference to the Reactor itself is pretty uncommon but I find it makes reasoning about what's happening easier.

我们的示例任务是一个定时器，它只生成一个线程，并让它在我们指定的秒数内睡眠。我们在这里创建的反应器将创建一个代表每个定时器的叶子 future。作为回报，反应器会收到一个 waker，一旦任务完成，它将调用这个 waker。

> Our example task is a timer that only spawns a thread and puts it to sleep for the number of seconds we specify. The reactor we create here will create a leaf-future representing each timer. In return the Reactor receives a waker which it will call once the task is finished.

为了能够在浏览器中运行这段代码，我们可以做的真正的 I/O 并不多，所以为了这个例子，就假装这实际上是代表了一些有用的 I/O 操作。

> To be able to run the code here in the browser there is not much real I/O we can do so just pretend that this is actually represents some useful I/O operation for the sake of this example.

**我们的反应器将看起来像这样：**

> Our Reactor will look like this:

```rust
// 任务在反应器中可能具有的不同状态
//
// > The different states a task can have in this Reactor
enum TaskState {
    Ready,
    NotReady(Waker),
    Finished,
}

// 这是一个“假的”反应器。它不是真的 I/O，但这使我们的代码能在书里和 playground 里执行
//
// > This is a "fake" reactor. It does no real I/O, but that also makes our
// > code possible to run in the book and in the playground
struct Reactor {

    // 我们需要一些把任务注册到反应器的方法。通常这会是对 I/O 事件的“关注”
    //
    // > we need some way of registering a Task with the reactor. Normally this
    // > would be an "interest" in an I/O event
    dispatcher: Sender<Event>,
    handle: Option<JoinHandle<()>>,

    // 这是任务列表
    //
    // > This is a list of tasks
    tasks: HashMap<usize, TaskState>,
}

// 这表示能向反应器线程发送的事件。在这个示例里只有一个超时和一个关闭事件。
//
// > This represents the Events we can send to our reactor thread. In this
// > example it's only a Timeout or a Close event.
#[derive(Debug)]
enum Event {
    Close,
    Timeout(u64, usize),
}

impl Reactor {

    // 我们选择返回一个原子引用计数、互斥锁保护、堆分配的 `Reactor`。只是为了让它好解释……
    // 不是，我们这么做的原因是：
    //
    // 1. 我们知道只有线程安全的反应器才会被创建出来。
    // 2. 通过堆分配，我们可以获得对一个稳定地址的引用，不依赖 `new` 函数的栈帧
    //
    // > We choose to return an atomic reference counted, mutex protected, heap
    // > allocated `Reactor`. Just to make it easy to explain... No, the reason
    // > we do this is:
    // >
    // > 1. We know that only thread-safe reactors will be created.
    // > 2. By heap allocating it we can obtain a reference to a stable address
    // > that's not dependent on the stack frame of the function that called `new`
    //
    // > **补充说明** 这个 `Box` 究竟是干啥用的……`Arc` 就会堆分配并获得稳定地址啊？
    fn new() -> Arc<Mutex<Box<Self>>> {
        let (tx, rx) = channel::<Event>();
        let reactor = Arc::new(Mutex::new(Box::new(Reactor {
            dispatcher: tx,
            handle: None,
            tasks: HashMap::new(),
        })));

        // 注意，此处我们需要使用一个弱引用。否则，我们的 `Reactor` 不会在主线程结束时析构，
        // 因为我们持有一个对它的内部引用。
        //
        // > Notice that we'll need to use `weak` reference here. If we don't,
        // > our `Reactor` will not get `dropped` when our main thread is finished
        // > since we're holding internal references to it.

        // 因为我们收集了我们启动的所有线程的 `JoinHandle`，并且确保会合并它们，
        // 我们知道 `Reactor` 会比我们在这启动的所有持有引用的线程活得都长。
        //
        // > Since we're collecting all `JoinHandles` from the threads we spawn
        // > and make sure to join them we know that `Reactor` will be alive
        // > longer than any reference held by the threads we spawn here.
        let reactor_clone = Arc::downgrade(&reactor);

        // 这会是我们的反应器线程。在我们的示例中，反应器线程只会启动新线程，这些线程将作为我们的定时器。
        //
        // > This will be our Reactor-thread. The Reactor-thread will in our case
        // > just spawn new threads which will serve as timers for us.
        let handle = thread::spawn(move || {
            let mut handles = vec![];

            // 这模仿了一些 I/O 资源
            //
            // > This simulates some I/O resource
            for event in rx {
                println!("REACTOR: {:?}", event);
                let reactor = reactor_clone.clone();
                match event {
                    Event::Close => break,
                    Event::Timeout(duration, id) => {

                        // 我们启动一个作为定时器的新线程，当它结束时会在正确的 `Waker` 上调用 `wake`。
                        //
                        // > We spawn a new thread that will serve as a timer
                        // > and will call `wake` on the correct `Waker` once
                        // > it's done.
                        let event_handle = thread::spawn(move || {
                            thread::sleep(Duration::from_secs(duration));
                            let reactor = reactor.upgrade().unwrap();
                            reactor.lock().map(|mut r| r.wake(id)).unwrap();
                        });
                        handles.push(event_handle);
                    }
                }
            }

            // 这对我们很重要，因为我们需要知道这些线程不会比我们的反应器线程活得长。
            // `Reactor` 析构时，我们的反应器线程也会被合并。
            //
            // > This is important for us since we need to know that these
            // > threads don't live longer than our Reactor-thread. Our
            // > Reactor-thread will be joined when `Reactor` gets dropped.
            handles.into_iter().for_each(|handle| handle.join().unwrap());
        });
        reactor.lock().map(|mut r| r.handle = Some(handle)).unwrap();
        reactor
    }

    // 唤醒函数会调用与 id 关联的任务的 waker 的 wake。
    //
    // > The wake function will call wake on the waker for the task with the
    // > corresponding id.
    fn wake(&mut self, id: usize) {
        self.tasks.get_mut(&id).map(|state| {

            // 此时，无论任务在什么状态，我们都能安全地将它设置为完成。
            // 这让我们获得了对我们替换之前的数据的所有权。
            //
            // > No matter what state the task was in we can safely set it
            // > to ready at this point. This lets us get ownership over the
            // > the data that was there before we replaced it.
            match mem::replace(state, TaskState::Ready) {
                TaskState::NotReady(waker) => waker.wake(),
                TaskState::Finished => panic!("Called 'wake' twice on task: {}", id),
                _ => unreachable!()
            }
        }).unwrap();
    }

    // 用这个反应器注册一个新任务。在这个特殊的例子里，如果一个具有相同 id 的任务被注册两次将导致 panic
    //
    // > Register a new task with the reactor. In this particular example
    // > we panic if a task with the same id get's registered twice
    fn register(&mut self, duration: u64, waker: Waker, id: usize) {
        if self.tasks.insert(id, TaskState::NotReady(waker)).is_some() {
            panic!("Tried to insert a task with id: '{}', twice!", id);
        }
        self.dispatcher.send(Event::Timeout(duration, id)).unwrap();
    }

    // 我们简单地检查一个对应这个 id 的任务是 `TaskState::Ready` 状态
    //
    // > We simply checks if a task with this id is in the state `TaskState::Ready`
    fn is_ready(&self, id: usize) -> bool {
        self.tasks.get(&id).map(|state| match state {
            TaskState::Ready => true,
            _ => false,
        }).unwrap_or(false)
    }
}

impl Drop for Reactor {
    fn drop(&mut self) {
        // 我们发送一个关闭事件给反应器，以使它关闭反应器线程。
        // 如果不这么做，最后我们会永远等待新的事件。
        //
        // > We send a close event to the reactor so it closes down our reactor-thread.
        // > If we don't do that we'll end up waiting forever for new events.
        self.dispatcher.send(Event::Close).unwrap();
        self.handle.take().map(|h| h.join().unwrap()).unwrap();
    }
}
```

虽然有很多代码，但本质上我们只是生成一个新的线程，并让它睡眠我们在创建任务时指定的一段时间。

> It's a lot of code though, but essentially we just spawn off a new thread and make it sleep for some time which we specify when we create a Task.

现在，让我们测试一下我们的代码，看看它是否有效。因为我们在这里睡了几秒钟，所以要给它一些时间来运行。

> Now, let's test our code and see if it works. Since we're sleeping for a couple of seconds here, just give it some time to run.

在最后一章中，我们[在一个可编辑的窗口中列出了全部的 200 行](https://cfsamson.github.io/books-futures-explained/8_finished_example.html)，你可以按照你喜欢的方式进行编辑和修改。

> In the last chapter we have the whole 200 lines in an editable window which you can edit and change the way you like.

```rust
fn main() {
    // 这是为了方便看到我们的 Future 被处理了
    //
    // > This is just to make it easier for us to see when our Future was resolved
    let start = Instant::now();

    // 很多运行时会创建一个全局的 `reactor` 但我们把它作为参数传递
    //
    // > Many runtimes create a global `reactor` we pass it as an argument
    let reactor = Reactor::new();

    // 我们创建了 2 个任务：
    // - 第一个参数是那个 `reactor`
    // - 第二个参数是用秒表示的超时时间
    // - 第三个参数是用来区分任务的 `id`
    //
    // > We create two tasks:
    // > - first parameter is the `reactor`
    // > - the second is a timeout in seconds
    // > - the third is an `id` to identify the task
    let future1 = Task::new(reactor.clone(), 1, 1);
    let future2 = Task::new(reactor.clone(), 2, 2);

    // `async` 块和 `async fn` 工作方式相同，它被编译成状态机，在每个 `await` 点让出。
    //
    // > an `async` block works the same way as an `async fn` in that it compiles
    // > our code into a state machine, `yielding` at every `await` point.
    let fut1 = async {
        let val = future1.await;
        println!("Got {} at time: {:.2}.", val, start.elapsed().as_secs_f32());
    };

    let fut2 = async {
        let val = future2.await;
        println!("Got {} at time: {:.2}.", val, start.elapsed().as_secs_f32());
    };

    // 我们的只能运行一个执行器，执行器只能运行一个 future，这倒是很正常。
    // 你有一组包含很多 future 的操作，最后作为一个单一的 future，驱动它们全部完成。
    //
    // > Our executor can only run one and one future, this is pretty normal
    // > though. You have a set of operations containing many futures that
    // > ends up as a single future that drives them all to completion.
    let mainfut = async {
        fut1.await;
        fut2.await;
    };

    // 这个执行器会阻塞主线程知道所有 future 都处理完。
    //
    // > This executor will block the main thread until the futures are resolved
    block_on(mainfut);
}
```

我添加了一些调试打印，以便我们可以观察到几件事情。

> I added a some debug printouts so we can observe a couple of things:

1. Waker 对象看起来就像我们在前一章谈到的特质对象一样
2. 程序从开始到结束的流程

> 1. How the Waker object looks just like the trait object we talked about in an earlier chapter
> 2. The program flow from start to finish

第二点与我们即将进入的最后一段有关。

> The last point is relevant when we move on the the last paragraph.

> 在我们的例子中，有一件微妙的事情需要注意。如果我们为两个事件传入相同的 id 会发生什么？
>
> > There is one subtle thing to note about our example. What happens if we pass in the same id for both events?
>
> ```rust
> let future1 = Task::new(reactor.clone(), 1, 1);
> let future2 = Task::new(reactor.clone(), 2, 1);
> ```
>
> 我们将在最后一章的练习中更多地讨论这个问题，在那里我们还将研究如何解决这个问题。现在，只需把它记下来，以便你能意识到这个问题。
>
> > We'll discuss this a bit more under exercises in the last chapter where we also look at ways to fix it. For now, just make a note of it so you're aware of the problem.

## async/await 和并发性

> Async/Await and concurrency

`async` 关键字可以用在函数上，如 `async fn(...)`，也可以用在一个块上，如 `async { ... }`。两者都会把你的函数或块变成一个 Future。

> The async keyword can be used on functions as in async fn(...) or on a block as in async { ... }. Both will turn your function, or block, into a Future.

这些 Future 是相当简单的。想象一下我们前几章的生成器。每个等待点就是一个让出点。

> These Futures are rather simple. Imagine our generator from a few chapters back. Every await point is like a yield point.

我们不是让出一个我们传入的值，而是让出我们正在等待的下一个 Future 上调用 `poll` 的结果。

> Instead of yielding a value we pass in, we yield the result of calling poll on the next Future we're awaiting.

我们的 `mainfut` 包含两个非叶子 future，它将对其调用 `poll`。**非叶子 future**有一个 `poll` 方法，简单地轮询其内部的期货，这些状态机被轮询，直到最后某个“叶子 future”返回 `Ready` 或 `Pending`。

> Our mainfut contains two non-leaf futures which it will call poll on. Non-leaf-futures has a poll method that simply polls their inner futures and these state machines are polled until some "leaf future" in the end either returns Ready or Pending.

我们的例子现在的样子，并不比普通的同步代码好多少。为了让我们能够同时等待多个 future，我们需要以某种方式发射它们，以便执行器开始同时运行它们。

> The way our example is right now, it's not much better than regular synchronous code. For us to actually await multiple futures at the same time we somehow need to spawn them so the executor starts running them concurrently.

我们的例子现在产生这样的结果：

> Our example as it stands now returns this:

```bash
Future got 1 at time: 1.00.
Future got 2 at time: 3.00.
```

如果这些 future 是异步执行的，我们会期望看到：

> If these Futures were executed asynchronously we would expect to see:

```bash
Future got 1 at time: 1.00.
Future got 2 at time: 2.00.
```

> 请注意，这并不意味着它们需要并行运行。它们可以并行运行，但并非必须。记住，我们在等待一些外部资源，所以我们可以在一个线程上发射许多这样的调用，并在事件解决时处理每个事件。
>
> > Note that this doesn't mean they need to run in parallel. They can run in parallel but there is no requirement. Remember that we're waiting for some external resource so we can fire off many such calls on a single thread and handle each event as it resolves.

现在，是时候向你推荐一些更好的资源来实现一个更好的执行器了。到目前为止，你应该对 Futures 的概念有了相当好的理解。

> Now, this is the point where I'll refer you to some better resources for implementing a better executor. You should have a pretty good understanding of the concept of Futures by now helping you along the way.

下一步应该是了解更高级的运行时是如何工作的，以及它们是如何实现以不同方式运行并完成 Futures 的。

> The next step should be getting to know how more advanced runtimes work and how they implement different ways of running Futures to completion.

[如果我是你，我接下来会阅读这个，并尝试为我们的例子实现它。](https://cfsamson.github.io/books-futures-explained/conclusion.html#building-a-better-exectuor)

> If I were you I would read this next, and try to implement it for our example..

实际上现在就这些了。可能还有很多东西要学，今天就到这里吧。

> That's actually it for now. There as probably much more to learn, this is enough for today.

我希望在读完这本书后，探索 Futures 和异步会变得更容易，我真的希望你能继续进一步探索。

> I hope exploring Futures and async in general gets easier after this read and I do really hope that you do continue to explore further.

不要忘记最后一章的练习😊。

> Don't forget the exercises in the last chapter 😊.

## 补充章节——暂停线程的正确方法

> Bonus Section - a Proper Way to Park our Thread

正如我们在本章前面所解释的，简单地调用 `thread::park` 其实并不足以实现一个合理的反应器。你还可以实现一个像 crossbeam 中的 `Parker` 那样的工具：[crossbeam::sync::Parker](https://docs.rs/crossbeam/0.7.3/crossbeam/sync/struct.Parker.html)

> As we explained earlier in our chapter, simply calling thread::park is not really sufficient to implement a proper reactor. You can also reach a tool like the Parker in crossbeam: crossbeam::sync::Parker

由于不需要很多行代码就能自己创建一个有效的解决方案，我们将展示我们如何通过使用条件变量和互斥锁来解决这个问题。

> Since it doesn't require many lines of code to create a working solution ourselves we'll show how we can solve that by using a Condvar and a Mutex instead.

先这样实现我们自己的 `Parker` 吧：

> Start by implementing our own Parker like this:

```rust
#[derive(Default)]
struct Parker(Mutex<bool>, Condvar);

impl Parker {
    fn park(&self) {

        // 我们锁定互斥锁，它保护着表示我们是否应该恢复执行的标记。
        //
        // > We aquire a lock to the Mutex which protects our flag indicating if we
        // > should resume execution or not.
        let mut resumable = self.0.lock().unwrap();

            // 我们把这个放在一个循环里，因为有可能我们被唤醒了但标记并没有变。如果出现这种情况我们继续睡。
            //
            // > We put this in a loop since there is a chance we'll get woken, but
            // > our flag hasn't changed. If that happens, we simply go back to sleep.
            while !*resumable {

                // 我们睡到有人发送通知。
                //
                // We sleep until someone notifies us
                resumable = self.1.wait(resumable).unwrap();
            }

        // 我们立即将条件设为 false，这样下次调用 `park` 时我们就能正确进入睡眠。
        //
        // > We immidiately set the condition to false, so that next time we call `park` we'll
        // > go right to sleep.
        *resumable = false;
    }

    fn unpark(&self) {
        // 我们简单地锁定标记的锁，并把条件设为可执行。
        //
        // > We simply acquire a lock to our flag and sets the condition to `runnable` when we
        // get it.
        *self.0.lock().unwrap() = true;

        // 通知条件变量，使其唤醒并恢复执行。
        //
        // > We notify our `Condvar` so it wakes up and resumes.
        self.1.notify_one();
    }
}
```

Rust 中的 `Condvar` 被设计为与 `Mutex` 一起工作。通常情况下，你会以为我们在睡眠前不会释放在 `self.0.lock().unwrap();` 中获得的互斥锁。这意味着我们的 `unpark` 函数永远拿不到标记的锁，我们就会陷入死锁。

> The Condvar in Rust is designed to work together with a Mutex. Usually, you'd think that we don't release the mutex-lock we acquire in self.0.lock().unwrap(); before we go to sleep. Which means that our unpark function never will acquire a lock to our flag and we deadlock.

使用 `Condvar` 可以避免这种情况，因为 `Condvar` 会消耗我们的锁，所以进入睡眠状态的时候锁就被释放。

> Using Condvar we avoid this since the Condvar will consume our lock so it's released at the moment we go to sleep.

当再次恢复时，`Condvar` 会返回锁，所以可以继续操作它。

> When we resume again, our Condvar returns our lock so we can continue to operate on it.

这意味着我们需要对执行器做一些非常细微的改变，比如这样：

> This means we need to make some very slight changes to our executor like this:

```rust
fn block_on<F: Future>(mut future: F) -> F::Output {
    let parker = Arc::new(Parker::default()); // <--- NB!
    let mywaker = Arc::new(MyWaker { parker: parker.clone() }); // <--- NB!
    let waker = mywaker_into_waker(Arc::into_raw(mywaker));
    let mut cx = Context::from_waker(&waker);

    // 安全性：遮蔽 `future`，这样它就不能再次访问。
    //
    // > SAFETY: we shadow `future` so it can't be accessed again.
    let mut future = unsafe { Pin::new_unchecked(&mut future) };
    loop {
        match Future::poll(future.as_mut(), &mut cx) {
            Poll::Ready(val) => break val,
            Poll::Pending => parker.park(), // <--- NB!
        };
    }
}
```

然后我们需要这样修改 `Waker`：

> And we need to change our Waker like this:

```rust
#[derive(Clone)]
struct MyWaker {
    parker: Arc<Parker>,
}

fn mywaker_wake(s: &MyWaker) {
    let waker_arc = unsafe { Arc::from_raw(s) };
    waker_arc.parker.unpark();
}
```

这就是全部内容。

> And that's really all there is to it.

> 如果你查看了展示暂停/恢复如何[导致微妙问题](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=b2343661fe3d271c91c6977ab8e681d0)的 playground 链接，你可以[查看这个示例](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=bebef0f8a8ce6a9d0d32442cc8381595)，它显示了我们的最终版本如何避免了这个问题。
>
> > If you checked out the playground link that showcased how park/unpark could cause subtle problems you can check out this example which shows how our final version avoids this problem.

下一章展示了我们完成的代码，如果你愿意的话，可以进一步探索这个改进。

> The next chapter shows our finished code with this improvement which you can explore further if you wish.

---

[下一章：完成的示例（可编辑）](finished-example-editable.md)
