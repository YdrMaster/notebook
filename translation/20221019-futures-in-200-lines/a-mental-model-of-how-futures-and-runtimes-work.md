# Futures 和运行时工作原理的心智模型

> A mental model of how Futures and runtimes work

这一部分的主要目标是建立一个概括性的心智模型，说明我们在前一章中读到的不同部分是如何一起工作的。我希望这将使我们在接下来的几章中深入研究特质对象和生成器等主题之前，更容易理解高层概念。

> The main goal in this part is to build a high level mental model of how the different pieces we read about in the previous chapter works together. I hope this will make it easier to understand the high level concepts before we take a deep dive into topics like trait objects and generators in the next few chapters.

这并不是创建一个异步系统模型的唯一方法，因为我们要对运行时的具体情况进行假设，而这些情况可能会有很大的不同。这是我认为最容易建立的方式，而且对于理解你在异步生态系统中发现的很多真实的实现也很有意义。

> This is not the only way to create a model of an async system since we're making assumptions on runtime specifics that can vary a great deal. It's the way I found it easiest to build upon and it's relevant for understanding a lot of real implementations you'll find in the async ecosystem.

最后，请注意，由于需要简洁明了，代码本身是"伪 rust"的。

> Finally, please note that the code itself is "pseudo-rust" due to the need for brevity and clarity.

> [本章](https://cfsamson.github.io/books-futures-explained/2_a_mental_model_for_futures.html)只有图

---

[下一章：Waker 和上下文](waker-and-context.md)
