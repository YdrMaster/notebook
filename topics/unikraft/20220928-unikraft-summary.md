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

换句话说，通过把一般内核的其他部分视作幺内核的应用程序，幺内核可以变为各种类型的内核，一个批处理式加载多个应用程序的加载器可以将幺内核变为批处理系统，一个支持进程的加载器，自然也可以以多道甚至 Linux 的方式执行多个应用程序（这有点像微内核，但这一切都是运行在相同特权级和相同地址空间的以保证性能）。利用如此精妙又有效的抽象，足以将幺内核与其他内核统一起来。这是对模块化内核来说非常重要的认识。

## unikraft 的组织结构

> 参考文档：<https://unikraft.org/docs/concepts/>

参考文档提到，unikraft 包含**库组件**、**配置**和**构建工具** 3 种组件。由于 C 语言构建系统也很底层，不像 rust 有必要区分应用程序板块（bin crate）和库板块（lib crate），所以完全可以把它们全部称为库，反正其实只是是否指定了加载入口点的区别。

---

> 如果按 rust 的分类来看，幺内核包括 2 个应用程序板块，也就是应用程序板块和裸机入口板块，这其实是 cargo 所不能接受的。当然实际上应用程序是最终的，还是应该定义为幺内核形成一个库，链接到应用程序这个 `#[no_main]` 的应用程序板块上。

---

而配置（库的 `Config.uk`、应用的 `kraft.yaml`）只是充当一个人机接口的作用，以文本方式呈现方便人工修改。构建系统会从配置文件获取这些信息并生成真正的集成化配置，然后根据配置完成应用程序和库操作系统的联合构建。但实际上这只是构建系统的细节设计而已，完全可以认为配置就是构建系统的一部分。

因此，实际上 unikraft 的架构表述完全可以修改为`构建系统 + 源码`。只是这个构建系统充分利用了 C 的底层特性，介入编译（设置编译选项）、链接过程（选择、替换链接库），实现了不修改源码就构造不同的内核。

一个只编译过 helloworld 应用程序的 kraft 工作区的结构是这样：

```bash
.
├── apps
│   └── helloworld
├── archs
├── libs
├── plats
└── unikraft
    ├── CODING_STYLE.md
    ├── CONTRIBUTING.md
    ├── COPYING.md
    ├── Config.uk
    ├── MAINTAINERS.md
    ├── Makefile
    ├── Makefile.uk
    ├── README.md
    ├── arch
    ├── doc
    ├── include
    ├── lib
    ├── plat
    ├── support
    └── version.mk
```

可以看到，helloworld 这个示例项目并不需要任何外部库，也没有设计特别的目标架构和平台，所以 archs、libs 和 plats 都是空的。apps 目录存放应用程序，unikraft 目录存放内核。unikraft 目录中又包含文档、构建系统和源码 3 部分。源码位于 arch、plat 和 lib 目录下。由于是 C 语言工程，还有一个 include 目录，里面放着 arch 和 plat 的头文件。lib 的头文件在每个 lib 内部，大概是为了方便互相引用。arch 里是少量平台相关的抽象，包括通用寄存器定义、同步原语之类的。plat 提供平台相关的引导、中断注册、驱动等。lib 提供各个内核模块。

## unikraft 的模块化

unikraft 的模块化实际上是很 C 的。模块隔离基本上通过统一的方式进行：函数关联利用函数名的链接时绑定、状态关联利用静态变量的依赖注入。

例如，作为内核内部模块的 nolibc，实际上只包含库级的实现代码。原本的“系统调用”现在只是函数声明，链接时实现会从其他库绑定过来。

`ukalloc` 微库则包含一套更完整的依赖注入流程。

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

rust 要想实现类似的抽象比较困难。rust 能用的依赖注入方式包括：

- 源码依赖

  优点是“正常”，缺点是子级依赖不方便替换。例如下图的情况：

  ```text
  α
   \
    β
     \
     γ/δ
  ```

  只定制 α 的依赖无法影响 β 依赖的是 γ 还是 δ。要做这个选择，要么同时修改 β，要么给 β 添加编译选项在 α 里选择。其实加编译选项确实是合理的，即使是 C 语言对稍微复杂点的情况还是需要编译选项，比如另一个库可能不存在的情况。

- 动态依赖注入

  优点是灵活、安全，缺点是需要在顶层改源码，并且有运行时开销。以 log 库为例，所有宏会调用最后一次 `set_logger` 设置的日志器，随时随地都可以调用。但是除了需要调用 `set_logger` 之外，这样做还导致每次调用额外的加锁解锁和两次寻址（指针寻址、虚表寻址），而且导致无法链接时优化。对于细粒度模块化的大量细碎函数来说，链接时优化应该影响会比较大。

- 静态链接

  包括 C++ 和 Rust 在内的所有会 mangle 函数名的现代语言都不提倡静态链接。并且静态链接会破坏同时使用一个库的多个版本的能力，强调工程性的现代语言很看重这一特性。另外还需要人为注意函数名对应，避免冲突。适用于使用极其广泛并且对性能要求较高的情况，比如 alloc。

## 源码阅读

### uktime

这个微库提供计时和延时。还应该提供定时器，但目前全部没有实现。模块的结构如下：

```bash
.
├── Config.uk
├── Makefile.uk
├── exportsyms.uk
├── include
│   └── uk
├── musl-imported
│   ├── COPYRIGHT
│   ├── include
│   └── src
├── time.c
└── timer.c
```

Config.uk 定义了微库元信息，可以看到它会提供 `HAVE_TIME`：

```uk
config LIBUKTIME
       bool "uktime: Time functions"
       default n
       select HAVE_TIME
```

exportsyms.uk 导出符号表：

| 符号 | 定义文件 | 备注
| --- | ------- | -
| clock_getres               | time.c   | `return 0`
| clock_gettime              | time.c   | 调用 `ukplat_wall_clock` 或 `ukplat_wall_clock`
| uk_syscall_e_clock_gettime |          |
| uk_syscall_r_clock_gettime |          |
| clock_settime              | time.c   | `return 0`
| gettimeofday               | time.c   | 调用 `ukplat_wall_clock`
| nanosleep                  | time.c   | !!!
| uk_syscall_e_nanosleep     |          |
| uk_syscall_r_nanosleep     |          |
| setitimer                  | time.c   | `WARN_STUBBED`
| sleep                      | time.c   | 调用 `nanosleep`
| timegm                     | timegm.c | 库函数
| times                      | time.c   | `errno = ENOTSUP`
| usleep                     | time.c   | 调用 `nanosleep`
| utime                      | time.c   | `return 0`
| timer_create               | timer.c  | `errno = ENOTSUP`
| timer_delete               | timer.c  | `errno = ENOTSUP`
| timer_settime              | timer.c  | `errno = ENOTSUP`
| timer_gettime              | timer.c  | `errno = ENOTSUP`
| timer_getoverrun           | timer.c  | `errno = ENOTSUP`

值得关注的只有 `nanosleep`：

```c
UK_SYSCALL_DEFINE(int, nanosleep, const struct timespec *, req, struct timespec *, rem)
{
  __nsec before, after, diff, nsec;

  if (!req || req->tv_nsec < 0 || req->tv_nsec > 999999999) {
    errno = EINVAL;
    return -1;
  }

  nsec = (__nsec)req->tv_sec * 1000000000L;
  nsec += req->tv_nsec;
  before = ukplat_monotonic_clock();

#if CONFIG_HAVE_SCHED
  uk_sched_thread_sleep(nsec);
#else
  __spin_wait(nsec);
#endif

  after = ukplat_monotonic_clock();
  diff = after - before;

  if (diff < nsec) {
    if (rem) {
      rem->tv_sec = ukarch_time_nsec_to_sec(nsec - diff);
      rem->tv_nsec = ukarch_time_subsec(nsec - diff);
    }
    errno = EINTR;
    return -1;
  }
  return 0;
}
```

还是比较简单的，如果有调度器就让调度器区休眠，否则原地自旋等待。`__spin_wait` 的定义是：

```c
# ifndef CONFIG_HAVE_SCHED
/* Workaround until Unikraft changes interface for something more
 * sensible
 */
static void __spin_wait(__nsec nsec)
{
  __nsec until = ukplat_monotonic_clock() + nsec;

  while (until > ukplat_monotonic_clock())
    ukplat_lcpu_halt_to(until);
}
# endif
```

就是反复获取时间。
