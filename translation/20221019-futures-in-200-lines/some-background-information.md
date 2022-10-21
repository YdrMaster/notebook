# 一些背景知识

> Some Background Information

在我们深入探究 Rust 中的 Futures 的细节之前，让我们快速了解一下一般处理并发编程的方法以及它们各自的优缺点。

> Before we go into the details about Futures in Rust, let's take a quick look at the alternatives for handling concurrent programming in general and some pros and cons for each of them.

当我们做这事的时候，我们还将解释一些涉及到并发的东西，这将使我们在具体研究 Futures 时更加容易。

> While we do that we'll also explain some aspects when it comes to concurrency which will make it easier for us when we dive into Futures specifically.

为了好玩，我在大多数示例中加入了一小段可运行的代码。如果你像我一样，事情会变得更加有趣，也许你会看到一些你以前没有见过的东西。

> For fun, I've added a small snippet of runnable code with most of the examples. If you're like me, things get way more interesting then and maybe you'll see some things you haven't seen before along the way.

## 操作系统提供的线程

> Threads provided by the operating system

如今，完成并发编程的一种方法是全部交给操作系统处理。我们只需简单地为我们想要完成的每项任务生成一个新的操作系统线程，并像通常那样编写代码。

> Now, one way of accomplishing concurrent programming is letting the OS take care of everything for us. We do this by simply spawning a new OS thread for each task we want to accomplish and write code like we normally would.

我们用来处理并发的运行时就是操作系统本身。

> The runtime we use to handle concurrency for us is the operating system itself.

**优点：**

> Advantages:

- 简单
- 易用
- 任务切换速度相当快
- 自动获得并行性

> - Simple
> - Easy to use
> - Switching between tasks is reasonably fast
> - You get parallelism for free

**缺点：**

> Drawbacks:

- 操作系统级别的线程有一个相当大的栈。如果你有许多任务同时等待（就像在负载很重的网络服务器中那样），内存会很快耗尽。
- 有很多系统调用参与其中。当任务数量较多时，这可能是相当昂贵的。
- 操作系统有很多事情需要处理。它可能不会像你希望的那样快速切换回你的线程。
- 某些系统可能不支持线程

> - OS level threads come with a rather large stack. If you have many tasks waiting simultaneously (like you would in a web server under heavy load) you'll run out of memory pretty fast.
> - There are a lot of syscalls involved. This can be pretty costly when the number of tasks is high.
> - The OS has many things it needs to handle. It might not switch back to your thread as fast as you'd wish.
> - Might not be an option on some systems

在 Rust 中使用操作系统线程看起来像这样：

> Using OS threads in Rust looks like this:

```rust
use std::thread;

fn main() {
    println!("So we start the program here!");
    let t1 = thread::spawn(move || {
        thread::sleep(std::time::Duration::from_millis(200));
        println!("We create tasks which gets run when they're finished!");
    });

    let t2 = thread::spawn(move || {
        thread::sleep(std::time::Duration::from_millis(100));
        println!("We can even chain callbacks...");
        let t3 = thread::spawn(move || {
            thread::sleep(std::time::Duration::from_millis(50));
            println!("...like this!");
        });
        t3.join().unwrap();
    });
    println!("While our tasks are executing we can do other stuff here.");

    t1.join().unwrap();
    t2.join().unwrap();
}
```

操作系统线程确实有一些相当大的优势。所以首先要说明，为什么我们要谈论 `async` 和并发性呢？

> OS threads sure have some pretty big advantages. So why all this talk about "async" and concurrency in the first place?

首先，计算机要提高[效率](https://zh.wikipedia.org/wiki/%E6%95%88%E7%8E%87)，就需要多任务。一旦你开始深入研究（比如[操作系统是如何工作的](https://os.phil-opp.com/async-await/)），你会发现并发无处不在。这是我们做任何事情的基础。

> First, for computers to be efficient they need to multitask. Once you start to look under the covers (like how an operating system works) you'll see concurrency everywhere. It's very fundamental in everything we do.

其次，我们有网络。

> Secondly, we have the web.

网络服务器都是关于 I/O 和处理小任务（请求）的。当小任务的数量很大时就不适合如今的操作系统线程，因为它们需要的内存和创建新线程时涉及的开销。

> Web servers are all about I/O and handling small tasks (requests). When the number of small tasks is large it's not a good fit for OS threads as of today because of the memory they require and the overhead involved when creating new threads.

如果负载是可变的，问题就更大了，这意味着一个程序在任何时间点上的当前任务数是不可预测的。这就是为什么你今天会看到这么多异步网络框架和数据库驱动。

> This gets even more problematic when the load is variable which means the current number of tasks a program has at any point in time is unpredictable. That's why you'll see so many async web frameworks and database drivers today.

然而，对于大量的问题，标准的操作系统线程往往会是正确的解决方案。所以，在你启用一个异步库之前，请三思你的问题。

> However, for a huge number of problems, the standard OS threads will often be the right solution. So, just think twice about your problem before you reach for an async library.

现在，让我们来看看其他一些多任务的选择。它们的共同点是都实现了某种方法，通过“用户态”运行时执行多任务。

> Now, let's look at some other options for multitasking. They all have in common that they implement a way to do multitasking by having a "userland" runtime.

## 绿色线程/有栈协程

> Green threads/stackful coroutines

在本书中，我将使用“绿色线程”这个术语来指代有栈协程，以区别于本章中描述的其他续体机制。然而，你可以看到“绿色线程”这个术语在不同的文献或互联网上的讨论中被用来描述一大类续体机制。

> In this book I'll use the term "green threads" to mean stackful coroutines to differentiate them from the other continuation mechanisms described in this chapter. You can, however, see the term "green threads" be used to describe a broader set of continuation mechanisms in different littrature or discussions on the internet.

绿色线程使用与操作系统相同的机制——为每个任务创建一个线程，建立一个栈，保存 CPU 的状态，并通过“上下文切换”从一个任务（线程）跳到另一个。

> Green threads use the same mechanism as an OS - creating a thread for each task, setting up a stack, saving the CPU's state, and jumping from one task(thread) to another by doing a "context switch".

我们把控制权让给调度器（它是这种系统中运行时的核心部分），然后它继续运行不同的任务。

> We yield control to the scheduler (which is a central part of the runtime in such a system) which then continues running a different task.

Rust 曾经有绿色线程，但在到达 1.0 之间又删除了。执行的状态被存储在不同的栈中，所以这样的解决方案不需要 `async`、`await`、`Future` 或 `Pin`。在许多方面，绿色线程模仿了操作系统支持并发的方式，实现它们是一个很好的学习经历。

> Rust had green threads once, but they were removed before it hit 1.0. The state of execution is stored in each stack so in such a solution there would be no need for async, await, Future or Pin. In many ways, green threads mimics how an operating system facilitates concurrency, and implementing them is a great learning experience.

**典型的流程是这样的：**

> The typical flow looks like this:

1. 运行一些非阻塞的代码。
2. 对某些外部资源进行阻塞调用。
3. CPU “跳转”到“主”线程，该线程调度了一个不同的线程来运行，并“跳转”到新的栈。
4. 在新的线程上运行一些非阻塞的代码，直到再次需要阻塞调用或任务完成。
5. CPU “跳转”回“主”线程，调度一个就绪的新线程，并“跳转”到该线程。

> 1. Run some non-blocking code.
> 2. Make a blocking call to some external resource.
> 3. CPU "jumps" to the "main" thread which schedules a different thread to run and "jumps" to that stack.
> 4. Run some non-blocking code on the new thread until a new blocking call or the task is finished.
> 5. CPU "jumps" back to the "main" thread, schedules a new thread which is ready to make progress, and "jumps" to that thread.

这些“跳转”被称为上下文切换。当你读到这里时，你的操作系统每秒都在做许多次这样的事情。

**优点：**

> Advantages:

1. 易用。代码会和使用操作系统线程时差不多。
2. 一次“上下文切换”是相当快的。
3. 每个栈开始时只得到一点内存，所以你可以同时运行成百上千的绿色线程。
4. 很容易加入[抢占](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/green-threads#preemptive-multitasking)功能，这让运行时实现者掌握了极大的控制权。

> 1. Simple to use. The code will look like it does when using OS threads.
> 2. A "context switch" is reasonably fast.
> 3. Each stack only gets a little memory to start with so you can have hundreds of thousands of green threads running.
> 4. It's easy to incorporate preemption which puts a lot of control in the hands of the runtime implementors.

**缺点：**

> Drawbacks:

1. 栈可能需要增长。解决这个问题并不容易，而且会有成本。
2. 每次切换都需要保存 CPU 状态。
3. 这不是一个零成本的抽象（Rust 早期有绿色线程，这也是它们被移除的原因之一）。
4. 如果想支持许多不同的平台，正确实现起来就很复杂。

> 1. The stacks might need to grow. Solving this is not easy and will have a cost.
> 2. You need to save the CPU state on every switch.
> 3. It's not a zero cost abstraction (Rust had green threads early on and this was one of the reasons they were removed).
> 4. Complicated to implement correctly if you want to support many different platforms.

一个绿色线程的示例可能看起来是这样的：

> A green threads example could look something like this:

> 下面展示的示例是从我早先写的一本关于绿色线程的 gitbook 中改编的，名为 [200 行 Rust 解读绿色线程](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/)。如果你想知道发生了什么，你会发现书中详细地解释了一切。下面的代码非常不安全，只是为了展示一个真实的例子。它绝不是为了展示“最佳实践”。只是为了让我们站在同一起跑线上。
>
> > The example presented below is an adapted example from an earlier gitbook I wrote about green threads called Green Threads Explained in 200 lines of Rust. If you want to know what's going on you'll find everything explained in detail in that book. The code below is wildly unsafe and it's just to show a real example. It's not in any way meant to showcase "best practice". Just so we're on the same page.

```rust
pub fn main() {
    let mut runtime = Runtime::new();
    runtime.init();
    Runtime::spawn(|| {
        println!("I haven't implemented a timer in this example.");
        yield_thread();
        println!("Finally, notice how the tasks are executed concurrently.");
    });
    Runtime::spawn(|| {
        println!("But we can still nest tasks...");
        Runtime::spawn(|| {
            println!("...like this!");
        })
    });
    runtime.run();
}
```

> **补充说明** 包含绿色线程实现的完整代码，可以跳过：
>
> ```rust
> #![feature(naked_functions)]
> use std::{arch::asm, ptr};
>
> const DEFAULT_STACK_SIZE: usize = 1024 * 1024 * 2;
> const MAX_THREADS: usize = 4;
> static mut RUNTIME: usize = 0;
>
> pub struct Runtime {
>     threads: Vec<Thread>,
>     current: usize,
> }
>
> #[derive(PartialEq, Eq, Debug)]
> enum State {
>     Available,
>     Running,
>     Ready,
> }
>
> struct Thread {
>     id: usize,
>     stack: Vec<u8>,
>     ctx: ThreadContext,
>     state: State,
>     task: Option<Box<dyn Fn()>>,
> }
>
> #[derive(Debug, Default)]
> #[repr(C)]
> struct ThreadContext {
>     rsp: u64,
>     r15: u64,
>     r14: u64,
>     r13: u64,
>     r12: u64,
>     rbx: u64,
>     rbp: u64,
>     thread_ptr: u64,
> }
>
> impl Thread {
>     fn new(id: usize) -> Self {
>         Thread {
>             id,
>             stack: vec![0_u8; DEFAULT_STACK_SIZE],
>             ctx: ThreadContext::default(),
>             state: State::Available,
>             task: None,
>         }
>     }
> }
>
> impl Runtime {
>     pub fn new() -> Self {
>         let base_thread = Thread {
>             id: 0,
>             stack: vec![0_u8; DEFAULT_STACK_SIZE],
>             ctx: ThreadContext::default(),
>             state: State::Running,
>             task: None,
>         };
>
>         let mut threads = vec![base_thread];
>         threads[0].ctx.thread_ptr = &threads[0] as *const Thread as u64;
>         let mut available_threads: Vec<Thread> = (1..MAX_THREADS).map(|i| Thread::new(i)).collect();
>         threads.append(&mut available_threads);
>
>         Runtime {
>             threads,
>             current: 0,
>         }
>     }
>
>     pub fn init(&self) {
>         unsafe {
>             let r_ptr: *const Runtime = self;
>             RUNTIME = r_ptr as usize;
>         }
>     }
>
>     pub fn run(&mut self) -> ! {
>         while self.t_yield() {}
>         std::process::exit(0);
>     }
>
>     fn t_return(&mut self) {
>         if self.current != 0 {
>             self.threads[self.current].state = State::Available;
>             self.t_yield();
>         }
>     }
>
>     #[inline(never)]
>     fn t_yield(&mut self) -> bool {
>         let mut pos = self.current;
>         while self.threads[pos].state != State::Ready {
>             pos += 1;
>             if pos == self.threads.len() {
>                 pos = 0;
>             }
>             if pos == self.current {
>                 return false;
>             }
>         }
>
>         if self.threads[self.current].state != State::Available {
>             self.threads[self.current].state = State::Ready;
>         }
>
>         self.threads[pos].state = State::Running;
>         let old_pos = self.current;
>         self.current = pos;
>
>         unsafe {
>            let old: *mut ThreadContext = &mut self.threads[old_pos].ctx;
>            let new: *const ThreadContext = &self.threads[pos].ctx;
>            asm!("call switch", in("rdi") old, in("rsi") new, clobber_abi("C"));
>        }
>        self.threads.len() > 0
>     }
>
>     pub fn spawn<F: Fn() + 'static>(f: F){
>         unsafe {
>             let rt_ptr = RUNTIME as *mut Runtime;
>             let available = (*rt_ptr)
>                 .threads
>                 .iter_mut()
>                 .find(|t| t.state == State::Available)
>                 .expect("no available thread.");
>
>             let size = available.stack.len();
>             let s_ptr = available.stack.as_mut_ptr().offset(size as isize);
>             let s_ptr = (s_ptr as usize & !15) as *mut u8;
>             available.task = Some(Box::new(f));
>             available.ctx.thread_ptr = available as *const Thread as u64;
>             //ptr::write(s_ptr.offset((size - 8) as isize) as *mut u64, guard as u64);
>             std::ptr::write(s_ptr.offset(-16) as *mut u64, guard as u64);
>             std::ptr::write(s_ptr.offset(-24) as *mut u64, skip as u64);
>             std::ptr::write(s_ptr.offset(-32) as *mut u64, call as u64);
>             available.ctx.rsp = s_ptr.offset(-32) as u64;
>             available.state = State::Ready;
>         }
>     }
> }
>
> fn call(thread: u64) {
>     let thread = unsafe { &*(thread as *const Thread) };
>     if let Some(f) = &thread.task {
>         f();
>     }
> }
>
> #[naked]
> unsafe fn skip() {
>     asm!("ret", options(noreturn))
> }
>
> fn guard() {
>     unsafe {
>         let rt_ptr = RUNTIME as *mut Runtime;
>         let rt = &mut *rt_ptr;
>         println!("THREAD {} FINISHED.", rt.threads[rt.current].id);
>         rt.t_return();
>     };
> }
>
> pub fn yield_thread() {
>     unsafe {
>         let rt_ptr = RUNTIME as *mut Runtime;
>         (*rt_ptr).t_yield();
>     };
> }
> #[naked]
> #[no_mangle]
> unsafe fn switch() {
>     asm!(
>        "mov 0x00[rdi], rsp",
>        "mov 0x08[rdi], r15",
>        "mov 0x10[rdi], r14",
>        "mov 0x18[rdi], r13",
>        "mov 0x20[rdi], r12",
>        "mov 0x28[rdi], rbx",
>        "mov 0x30[rdi], rbp",
>        "mov rsp, 0x00[rsi]",
>        "mov r15, 0x08[rsi]",
>        "mov r14, 0x10[rsi]",
>        "mov r13, 0x18[rsi]",
>        "mov r12, 0x20[rsi]",
>        "mov rbx, 0x28[rsi]",
>        "mov rbp, 0x30[rsi]",
>        "mov rdi, 0x38[rsi]",
>        "ret", options(noreturn)
>     );
> }
> #[cfg(not(windows))]
> pub fn main() {
>     let mut runtime = Runtime::new();
>     runtime.init();
>     Runtime::spawn(|| {
>         println!("I haven't implemented a timer in this example.");
>         yield_thread();
>         println!("Finally, notice how the tasks are executed concurrently.");
>     });
>     Runtime::spawn(|| {
>         println!("But we can still nest tasks...");
>         Runtime::spawn(|| {
>             println!("...like this!");
>         })
>     });
>     runtime.run();
> }
> #[cfg(windows)]
> fn main() { }
> ```

还能坚持住？很好。如果上面的代码难以理解，请不要感到沮丧。如果不是我自己写的，我可能也会有同样的感觉。一会儿你还可以回头去读那本解释它的书。

## 基于回调的方法

> Callback based approaches

你可能已经知道我们接下来要谈论 Javascript，我猜大多数人都知道。

> You probably already know what we're going to talk about in the next paragraphs from JavaScript which I assume most know.

> 如果你早先因暴露于 JavaScript 回调而患上 PTSD，那么现在闭上眼睛，向下滚动 2-3 秒。你会发现那里有一个链接，可以带你到安全地带。
>
> If your exposure to JavaScript callbacks has given you any sorts of PTSD earlier in life, close your eyes now and scroll down for 2-3 seconds. You'll find a link there that takes you to safety.

某个基于回调的方法背后的整个思路就是保存一个指针，指向一组我们希望以后能根据任何必要的状态信息运行的指令。在 Rust 中，这将是一个闭包。在下面的示例中，我们将这些信息保存在一个 `HashMap` 中，但这并不是唯一的选择。

> The whole idea behind a callback based approach is to save a pointer to a set of instructions we want to run later together with whatever state is needed. In Rust this would be a closure. In the example below, we save this information in a HashMap but it's not the only option.

不把线程作为实现并发的主要方式，这一基本思想是其他方法的共同点。包括 Rust 今天使用的方法，我们很快就会讲到。

> The basic idea of not involving threads as a primary way to achieve concurrency is the common denominator for the rest of the approaches. Including the one Rust uses today which we'll soon get to.

**优点：**

> Advantages:

- 易于在大多数语言中实现
- 没有上下文切换
- 相对较低的内存开销（在大多数情况下）

> - Easy to implement in most languages
> - No context switching
> - Relatively low memory overhead (in most cases)

**缺点：**

> Drawbacks:

- 由于每个任务都必须为以后保存它所需要的状态，内存用量将随着一系列计算中回调次数的增加而线性增长。
- 可能很难理解。许多人已经知道这就是“回调地狱”。
- 这是一种非常不同的程序编写方式，需要进行大量的重写，才能从“正常”的程序流程变成“基于回调”的流程。
- 由于Rust的所有权模型，在任务之间共享状态是一个困难的问题。

> - Since each task must save the state it needs for later, the memory usage will grow linearly with the number of callbacks in a chain of computations.
> - Can be hard to reason about. Many people already know this as "callback hell".
> - It's a very different way of writing a program, and will require a substantial rewrite to go from a "normal" program flow to one that uses a "callback based" flow.
> - Sharing state between tasks is a hard problem in Rust using this approach due to its ownership model.

一个极度简化的基于回调的方法的示例看起来是这样：

> An extremely simplified example of a how a callback based approach could look like is:

```rust
fn program_main() {
    println!("So we start the program here!");
    set_timeout(200, || {
        println!("We create tasks with a callback that runs once the task finished!");
    });
    set_timeout(100, || {
        println!("We can even chain sub-tasks...");
        set_timeout(50, || {
            println!("...like this!");
        })
    });
    println!("While our tasks are executing we can do other stuff instead of waiting.");
}

fn main() {
    RT.with(|rt| rt.run(program_main));
}

use std::sync::mpsc::{channel, Receiver, Sender};
use std::{cell::RefCell, collections::HashMap, thread};

thread_local! {
    static RT: Runtime = Runtime::new();
}

struct Runtime {
    callbacks: RefCell<HashMap<usize, Box<dyn FnOnce() -> ()>>>,
    next_id: RefCell<usize>,
    evt_sender: Sender<usize>,
    evt_receiver: Receiver<usize>,
}

fn set_timeout(ms: u64, cb: impl FnOnce() + 'static) {
    RT.with(|rt| {
        let id = *rt.next_id.borrow();
        *rt.next_id.borrow_mut() += 1;
        rt.callbacks.borrow_mut().insert(id, Box::new(cb));
        let evt_sender = rt.evt_sender.clone();
        thread::spawn(move || {
            thread::sleep(std::time::Duration::from_millis(ms));
            evt_sender.send(id).unwrap();
        });
    });
}

impl Runtime {
    fn new() -> Self {
        let (evt_sender, evt_receiver) = channel();
        Runtime {
            callbacks: RefCell::new(HashMap::new()),
            next_id: RefCell::new(1),
            evt_sender,
            evt_receiver,
        }
    }

    fn run(&self, program: fn()) {
        program();
        for evt_id in &self.evt_receiver {
            let cb = self.callbacks.borrow_mut().remove(&evt_id).unwrap();
            cb();
            if self.callbacks.borrow().is_empty() {
                break;
            }
        }
    }
}
```

我们把这个示例做的超级简单，你可能想知道这种方法和使用操作系统线程并直接将回调传递给操作系统线程的方法有什么区别。

> We're keeping this super simple, and you might wonder what's the difference between this approach and the one using OS threads and passing in the callbacks to the OS threads directly.

区别在于，这个示例中的回调是在同一个线程上运行的。我们创建的操作系统线程基本上只是作为计时器使用，但可以代表我们必须等待的任何种类的资源。

> The difference is that the callbacks are run on the same thread using this example. The OS threads we create are basically just used as timers but could represent any kind of resource that we'll have to wait for.

## 从回调到承诺

> From callbacks to promises

你可能已经开始疑惑，我们什么时候才能谈及 Futures？

> You might start to wonder by now, when are we going to talk about Futures?

好吧，我们就要到了。你看，Promises、Futures 和其他表示推迟计算的名字经常被互换使用。

> Well, we're getting there. You see Promises, Futures, and other names for deferred computations are often used interchangeably.

它们之间有正式的区别，但我们不会在这里讨论这些。值得解释一下承诺，它们用于 JavaScript，因而广为人知。承诺与 Rust 的 Futures 也有很多共同之处。

> There are formal differences between them, but we won't cover those here. It's worth explaining promises a bit since they're widely known due to their use in JavaScript. Promises also have a lot in common with Rust's Futures.

总的来说，许多语言都有承诺的概念，但在下面的示例中我将将使用 JavaScript 的那种。

> First of all, many languages have a concept of promises, but I'll use the one from JavaScript in the examples below.

承诺是解决基于回调的方法所带来的复杂性的一种方式。

> Promises are one way to deal with the complexity which comes with a callback based approach.

以前是这样：

> Instead of:

```JavaScript
setTimer(200, () => {
  setTimer(100, () => {
    setTimer(50, () => {
      console.log("I'm the last one");
    });
  });
});
```

可以这样做：

> We can do this:

```JavaScript
function timer(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms));
}

timer(200)
.then(() => timer(100))
.then(() => timer(50))
.then(() => console.log("I'm the last one"));
```

深入原理，可以看到变化更为显著。你可以看到，承诺会返回一个状态机，它可以处于三种状态之一：`pending`、`fulfilled` 或 `rejected`。

> The change is even more substantial under the hood. You see, promises return a state machine which can be in one of three states: pending, fulfilled or rejected.

当我们在上面的示例中调用 `timer(200)` 时，我们得到的是一个 `pending` 状态的承诺。

> When we call timer(200) in the sample above, we get back a promise in the state pending.

由于承诺被重写为状态机，还有一种更好的语法让我们可以像下面这样编写最后一个示例：

> Since promises are re-written as state machines, they also enable an even better syntax which allows us to write our last example like this:

```rust
async function run() {
    await timer(200);
    await timer(100);
    await timer(50);
    console.log("I'm the last one");
}
```

你可以把 `run` 函数看作是一个由几个子任务组成的可暂停的任务。在每个“await”点上，它将控制权交给调度器（在本例中是众所周知的 Javascript 事件循环）。

> You can consider the run function as a pausable task consisting of several sub-tasks. On each "await" point it yields control to the scheduler (in this case it's the well-known JavaScript event loop).

一旦其中一个子任务的状态变为 `fulfilled` 或 `rejected`，该任务就会被调度继续进行下一步。

> Once one of the sub-tasks changes state to either fulfilled or rejected, the task is scheduled to continue to the next step.

从语法上看，Rust 的 Futures 0.1 很像上面的承诺的示例，而 Rust 的 Futures 0.3 很像我们上一个示例中的 async/await。

> Syntactically, Rust's Futures 0.1 was a lot like the promises example above, and Rust's Futures 0.3 is a lot like async/await in our last example.

现在，JavaScript 承诺和 Rust 的 Futures 之间的相似之处也到此为止。我们之所以讲到这些，是为了做一个介绍，并建立一个正确的思维模式来探索 Rust 的 Futures。

> Now this is also where the similarities between JavaScript promises and Rust's Futures stop. The reason we go through all this is to get an introduction and get into the right mindset for exploring Rust's Futures.

> 为了避免以后的混乱：有一个区别你应该知道。JavaScript 的承诺严格求值。这意味着一旦它被创建，它就开始运行一个任务。与此相反，Rust 的 Futures 惰性求值。在做任何工作之前，它需要先被轮询一次。
>
> > To avoid confusion later on: There's one difference you should know. JavaScript promises are eagerly evaluated. That means that once it's created, it starts running a task. Rust's Futures on the other hand are lazily evaluated. They need to be polled once before they do any work.

---

[下一章：Rust 中的 Futures](futures-in-rust.md)
