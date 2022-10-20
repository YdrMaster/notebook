# 200 行 Rust 解读 Futures

> Futures Explained in 200 Lines of Rust

本书旨在用实例驱动的方法解释 Rust 中的 Futures，探讨它们为什么被设计成这样，以及它们是如何工作的。我们还将介绍还有什么方法能处理编程中的并发问题。

> This book aims to explain Futures in Rust using an example driven approach, exploring why they're designed the way they are, and how they work. We'll also take a look at some of the alternatives we have when dealing with concurrency in programming.

理解这本书中描述的细节，不需要使用 Rust 中的 `futures` 或 `async/await`。这是为那些想知道这一切是如何运作的好奇者准备的。

> Going into the level of detail I do in this book is not needed to use futures or async/await in Rust. It's for the curious out there that want to know how it all works.

## 这本书涵盖的内容

> What this book covers

本书将尝试解释所有你可能想知道的东西，直到不同类型的执行器和运行时的话题。在本书中，我们只是实现了一个非常简单的运行时，介绍了一些概念，但已经足够入门了。

> This book will try to explain everything you might wonder about up until the topic of different types of executors and runtimes. We'll just implement a very simple runtime in this book introducing some concepts but it's enough to get started.

*Stjepan Glavina* 发表了一系列关于异步运行时和执行器的优秀文章。

> Stjepan Glavina has made an excellent series of articles about async runtimes and executors.

> **补充说明** 原文 “Stjepan Glavina”，以及下文 “Stjepan 的文章”、“构建你自己的`block_on()`”、“构建你自己的执行器” 带有已失效的链接。

你可以先阅读本书，然后继续阅读 *Stjepan* 的文章，以了解更多关于运行时和它们如何工作，特别是：

> The way you should go about it is to read this book first, then continue reading Stjepan's articles to learn more about runtimes and how they work, especially:

1. 构建你自己的 `block_on()`
2. 构建你自己的执行器

> 1. Build your own block_on()
> 2. Build your own executor

你也应该看看 [smol](https://github.com/smol-rs/smol) 运行时，因为它是由同一作者制作的真正的运行时。它有很好的注释，而且很容易学习。

> You should also check out the smol runtime as it's a real runtime made by the same author. It's well commented and made to be easy to learn from.

我把自己限制在一个 200 行的主要示例中（因此有了这个标题），以限制范围并介绍一个容易进一步探索的例子。

> I've limited myself to a 200 line main example (hence the title) to limit the scope and introduce an example that can easily be explored further.

然而，有很多东西需要消化，这并不像我所说的那么简单，但是我们会一步一步来，所以喝杯茶，放松一下。

> However, there is a lot to digest and it's not what I would call easy, but we'll take everything step by step so get a cup of tea and relax.

希望你能享受这段旅程。

> I hope you enjoy the ride.

> 本书是开源编写的，欢迎大家投稿。你可以在这里找到[这本书本身的仓库](https://github.com/cfsamson/books-futures-explained)。你可以在[这里](https://github.com/cfsamson/examples-futures)找到最后的例子，以便克隆、分叉或拷贝。任何建议或改进都可以以 PR 或 issue 形式提交给本书。
>
> 一如既往，我们欢迎各种反馈。
>
> > This book is developed in the open, and contributions are welcome. You'll find the repository for the book itself here. The final example which you can clone, fork or copy can be found here. Any suggestions or improvements can be filed as a PR or in the issue tracker for the book.
>
> > As always, all kinds of feedback is welcome.

## 读者练习和拓展阅读

> Reader exercises and further reading

在[最后一章](conclusion-and-exercises.md)中，如果你想进一步探索，我冒昧地提出了一些小练习。

本书也是我写的第四本关于 Rust 并发编程的书。如果你喜欢它，你可能也想看看其他的：

> This book is also the fourth book I have written about concurrent programming in Rust. If you like it, you might want to check out the others as well:

- [200 行 Rust 解读绿色线程](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/)
- [The Node Experiment - 用 Rust 探索异步基础](https://cfsamson.github.io/book-exploring-async-basics/)
- [用 Rust 解读 Epoll, Kqueue and IOCP](https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/)

> - Green Threads Explained in 200 lines of rust
> - The Node Experiment - Exploring Async Basics with Rust
> - Epoll, Kqueue and IOCP Explained with Rust

## 功劳和感谢

> Credits and thanks

我想借此机会感谢 `mio`、`tokio`、`async_std`、`futures`、`libc`、`crossbeam` 背后的人们，他们对异步生态贡献如此之大，我却很少听到对他们足够的赞美。

> I'd like to take this chance to thank the people behind mio, tokio, async_std, futures, libc, crossbeam which underpins so much of the async ecosystem and and rarely gets enough praise in my eyes.

特别感谢 [jonhoo](https://twitter.com/jonhoo)，他为对本书的早期草稿给予了我一些宝贵的反馈。他没有读过成品，但绝对要感谢他。

> A special thanks to jonhoo who was kind enough to give me some valuable feedback on a very early draft of this book. He has not read the finished product, but a big thanks is definitely due.

感谢 [@ckaran](https://github.com/ckaran) 对第二和第三章的建议和贡献。

> Thanks to @ckaran for suggestions and contributions in chapter 2 and 3.

翻译

> Translations

本书已由 [nkbai](https://github.com/nkbai) 翻译成[中文](https://stevenbai.top/rust/futures_explained_in_200_lines_of_rust/)。

> This book has been translated to Chinese by nkbai.

---

[下一节：一些背景知识](some-background-information.md)
