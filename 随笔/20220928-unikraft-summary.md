# unikraft 学习总结

## 幺内核的概念

幺内核和运行在幺内核上的应用程序与正常的应用程序或裸机应用程序都不同。

一般的应用程序会编译出一个基于特定操作系统的 ELF（可执行或链接文件），这个程序会被作为背景运行的操作系统加载和执行。因此应用程序可定制的启动主函数并不是整个应用程序的入口，适于特定操作系统的 libc 库会为应用程序添加供操作系统加载必要的序幕。

一般的裸机应用程序（包括操作系统）由于没有加载环境了，会自己指定自己的入口，所以可以在裸机上运行。

而幺内核像是二者的结合体。一方面，它的应用程序的可定制部分不会指定程序的入口，而是直接链接幺内核本身，由幺内核提供入口。另一方面，它的编译结果又可以在裸机上运行。

换句话说，幺内核是库操作系统这个定义是非常恰当的。它本质上是在实现一个适用于裸机的 libc 库。应用程序链接这个 libc 就能得到包括启动代码在内的各种加载步骤，最终得到裸机可执行文件。所以开发幺内核要注意：这个内核对于应用程序来说是库，可以被应用程序链接起来；同时，应用程序对于内核来说也是库，因为编译出的可执行文件的入口是幺内核提供的，应用程序被链接进来，其入口以库函数的形式被调用。

从对幺内核的理解出发，我们能得到另一种看待任何操作系统内核的视角：一切操作系统实际上都是幺内核，只不过那些类 Linux 内核运行的应用程序是一个 Linux Executable Loader 罢了。事实上，unikraft 也确实是这样做的：

```bash
APPLICATIONS            VERSION         RELEASED        LAST CHECKED
helloworld              0.10.0          29 Aug 22       28 Sep 22
httpreply               0.10.0          29 Aug 22       28 Sep 22
elfloader               36f0e9f         17 Aug 22       28 Sep 22
helloworld-go           0.10.0          20 Aug 22       28 Sep 22
rust                    95a1825         27 Aug 21       28 Sep 22
duktape                 0.10.0          20 Aug 22       28 Sep 22
wamr                    0.10.0          20 Aug 22       28 Sep 22
lua                     0.10.0          20 Aug 22       28 Sep 22
helloworld-cpp          0.10.0          29 Aug 22       28 Sep 22
python3                 0.10.0          29 Aug 22       28 Sep 22
redis                   0.10.0          4 days ago      28 Sep 22
click                   0.10.0          20 Aug 22       28 Sep 22
nginx                   0.10.0          20 Aug 22       28 Sep 22
ruby                    0.10.0          20 Aug 22       28 Sep 22
micropython             0.10.0          20 Aug 22       28 Sep 22
sqlite                  0.10.0          29 Aug 22       28 Sep 22
nettle-test             0.5             06 Feb 21       28 Sep 22
```

unikraft 的应用程序列表里有 elfloader（第四行），用于从其他地方加载应用程序。实际上，除非使用了 Hypervisor，一般的硬件也只能运行一个裸机应用程序，但其实只是不能同时运行，不代表不能更换。就好像 uboot 引导另一个裸机应用程序然后自己退出一样，幺内核也可以利用一些会退出的应用程序来实现以不同方式运行多个应用程序的目的。

换句话说，通过把各种加载器视作一些应用程序，幺内核可以变为各种类型的内核，一个批处理式加载多个应用程序的加载器可以将幺内核变为批处理系统，一个支持进程的加载器，自然也可以以多道甚至 Linux 的方式执行多个应用程序（这有点像微内核，但这一切都是运行在相同特权级和相同地址空间的以保证性能）。利用如此精妙又有效的抽象，足以将幺内核与其他内核统一起来。这是对模块化内核来说非常重要的认识。

## unikraft 的组织结构

> 参考文档：<https://unikraft.org/docs/concepts/>

参考文档提到，unikraft 包含**库组件**、**配置**和**构建工具** 3 种组件。由于 C 语言构建系统也很底层，不像 rust 有必要区分应用程序板块（bin crate）和库板块（lib crate），所以完全可以把它们全部称为库，反正其实只是是否指定了加载入口点的区别。

---

> 如果按 rust 的分类来看，幺内核包括 2 个应用程序板块，也就是应用程序板块和裸机入口板块，这其实是 cargo 所不能接受的。当然实际上应用程序是最终的，还是应该定义为幺内核形成一个库，链接到应用程序这个 `#[no_main]` 的应用程序板块上。

---

而配置（库的 `Config.uk`、应用的 `kraft.yml`）只是充当一个人机接口的作用，以文本方式呈现方便人工修改。构建系统会从配置文件获取这些信息并生成真正的集成化配置，然后根据配置完成应用程序和库操作系统的联合构建。但实际上这只是构建系统的细节设计而已，完全可以认为配置就是构建系统的一部分。

因此，实际上 unikraft 的架构表述完全可以修改为`构建系统 + 源码`，换句话说，只是为 C 重新包装了一个现代语言的标配结构，并没有任何特异之处。可以说，rust 基本上已经完成这个工作了。

## unikraft 的模块化

unikraft 的模块化实际上是很 C 的。模块隔离基本上通过统一的方式进行：函数关联利用函数名的链接时绑定、状态关联利用静态变量的依赖注入。

例如，作为内核内部模块的 nolibc，实际上只包含库级的实现代码。原本的“系统调用”现在只是函数声明，链接时实现会从其他库绑定过来。而 `ukalloc` 微库包含一个经典的依赖注入。

`unikraft/lib/ukalloc/alloc.c`：声明一个 `struct uk_alloc` 指针，这实际上是一个侵入式单链表。`uk_alloc_register` 函数向链表末尾压入一个新的分配器对象：

```c
struct uk_alloc *_uk_alloc_head;

int uk_alloc_register(struct uk_alloc *a)
{
 struct uk_alloc *this = _uk_alloc_head;

 if (!_uk_alloc_head) {
  _uk_alloc_head = a;
  a->next = NULL;
  return 0;
 }

 while (this && this->next)
  this = this->next;
 this->next = a;
 a->next = NULL;
 return 0;
}
```

`unikraft/lib/ukalloc/include/uk/alloc_impl.h`：用一个函数式宏调用 `uk_alloc_register` 完成分配器注册：

```c
/* Shortcut for doing a registration of an allocator that does not implement
 * palloc() or pfree()
 */
#define uk_alloc_init_malloc(a, malloc_f, calloc_f, realloc_f, free_f, \
    posix_memalign_f, memalign_f, addmem_f) \
 do {        \
  (a)->malloc         = (malloc_f);   \
  (a)->calloc         = (calloc_f);   \
  (a)->realloc        = (realloc_f);   \
  (a)->posix_memalign = (posix_memalign_f);  \
  (a)->memalign       = (memalign_f);   \
  (a)->free           = (free_f);    \
  (a)->palloc         = uk_palloc_compat;   \
  (a)->pfree          = uk_pfree_compat;   \
  (a)->addmem         = (addmem_f);   \
         \
  uk_alloc_register((a));     \
 } while (0)
```

`unikraft/lib/ukalloc/exportsyms.uk`：指定哪些函数定义需要导出：

```text
uk_alloc_register
uk_alloc_get_default
uk_malloc_ifpages
uk_free_ifpages
uk_realloc_ifpages
uk_posix_memalign_ifpages
uk_malloc_ifmalloc
uk_realloc_ifmalloc
uk_posix_memalign_ifmalloc
uk_free_ifmalloc
uk_calloc_compat
uk_memalign_compat
uk_realloc_compat
uk_palloc_compat
uk_pfree_compat
_uk_alloc_head
```
