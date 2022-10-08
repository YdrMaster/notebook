# unikraft 启动流程分析

## 实验方法

用 `kraft menuconfig`，可以配置 library-configuration/ukdebug/kernel-message-level，把日志级别配置成全部显示，以观察启动流程。qemu 上运行 helloworld 输出：

```bash
Booting from ROM..
[    0.000000] Info: [libkvmplat] <setup.c @  268> Entering from KVM (x86)...
[    0.000000] Info: [libkvmplat] <setup.c @  269>      multiboot: 0x9500
[    0.000000] dbg:  [libkvmplat] <setup.c @  129> No initrd present
[    0.000000] Info: [libkvmplat] <setup.c @  282>     heap start: 0x140000
[    0.000000] Info: [libkvmplat] <setup.c @  287>      stack top: 0x7fd0000
[    0.000000] Info: [libkvmplat] <setup.c @  301> Switch from bootstrap stack to stack @0x7fe0000
[    0.000000] Info: [libukboot] <boot.c @  199> Unikraft constructor table at 0x112000 - 0x112010
[    0.000000] dbg:  [libukboot] <boot.c @  204> Call constructor: 0x1082f0())...
[    0.000000] dbg:  [libukbus] <bus.c @   54> Register bus handler: 0x117000
[    0.000000] dbg:  [libukboot] <boot.c @  204> Call constructor: 0x108a80())...
[    0.000000] dbg:  [libukbus] <bus.c @   54> Register bus handler: 0x117060
[    0.000000] Info: [libukboot] <boot.c @  222> Initialize memory allocator...
[    0.000000] dbg:  [libukboot] <boot.c @  231> Try memory region: 0x140000 - 0x7fd0000 (flags: 0x31)...
[    0.000000] Info: [libukallocbbuddy] <bbuddy.c @  491> Initialize binary buddy allocator 140000
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 143000 - 144000 (order 0)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 144000 - 148000 (order 2)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 148000 - 150000 (order 3)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 150000 - 160000 (order 4)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 160000 - 180000 (order 5)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 180000 - 200000 (order 7)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 200000 - 400000 (order 9)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 400000 - 800000 (order 10)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 800000 - 1000000 (order 11)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 1000000 - 2000000 (order 12)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 2000000 - 4000000 (order 13)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 4000000 - 6000000 (order 13)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 6000000 - 7000000 (order 12)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 7000000 - 7800000 (order 11)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 7800000 - 7c00000 (order 10)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 7c00000 - 7e00000 (order 9)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 7e00000 - 7f00000 (order 8)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 7f00000 - 7f80000 (order 7)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 7f80000 - 7fc0000 (order 6)
[    0.000000] dbg:  [libukallocbbuddy] <bbuddy.c @  447> 140000: Add allocate unit 7fc0000 - 7fd0000 (order 4)
[    0.000000] Info: [libukboot] <boot.c @  265> Initialize IRQ subsystem...
[    0.000000] Info: [libukboot] <boot.c @  272> Initialize platform time...
[    0.000000] Info: [libkvmplat] <tscclock.c @  253> Calibrating TSC clock against i8254 timer
[    0.100032] Info: [libkvmplat] <tscclock.c @  274> Clock source: TSC, frequency estimate is 3112027770 Hz
[    0.101047] Info: [libukboot] <boot.c @   90> Init Table @ 0x112010 - 0x112018
[    0.102102] dbg:  [libukboot] <boot.c @   95> Call init function: 0x1110a0()...
[    0.103365] Info: [libukbus] <bus.c @  136> Initialize bus handlers...
[    0.104122] dbg:  [libukbus] <bus.c @   78> Initialize bus handler 0x117000...
[    0.105153] dbg:  [libukbus] <bus.c @   78> Initialize bus handler 0x117060...
[    0.105779] Info: [libukbus] <bus.c @  138> Probe buses...
[    0.106737] dbg:  [libukbus] <bus.c @   90> Probe bus 0x117000...
[    0.107478] dbg:  [libkvmpci] <pci_bus.c @  339> Probe PCI
[    0.108196] Info: [libkvmpci] <pci_bus.c @  284> PCI 00:00.00 (0600 8086:1237): <no driver>
[    0.108983] Info: [libkvmpci] <pci_bus.c @  284> PCI 00:01.00 (0600 8086:7000): <no driver>
[    0.110052] Info: [libkvmpci] <pci_bus.c @  284> PCI 00:02.00 (0300 1234:1111): <no driver>
[    0.110643] Info: [libkvmpci] <pci_bus.c @  284> PCI 00:03.00 (0200 8086:100e): <no driver>
[    0.111645] dbg:  [libukbus] <bus.c @   90> Probe bus 0x117060...
Powered by
o.   .o       _ _               __ _
Oo   Oo  ___ (_) | __ __  __ _ ' _) :_
oO   oO ' _ `| | |/ /  _)' _` | |_|  _)
oOo oOO| | | | |   (| | | (_) |  _) :_
 OoOoO ._, ._:_:_,\_._,  .__,_:_, \___)
            Tethys 0.5.0~b8be82b-custom
[    0.114346] Info: [libukboot] <boot.c @  117> Pre-init table at 0x1161d0 - 0x1161d0
[    0.115178] Info: [libukboot] <boot.c @  128> Constructor table at 0x1161d0 - 0x1161d0
[    0.116964] Info: [libukboot] <boot.c @  139> Calling main(1, ['build/helloworld_kvm-x86_64'])
Hello world!
Arguments:  "build/helloworld_kvm-x86_64"
[    0.119050] Info: [libukboot] <boot.c @  148> main returned 0, halting system
[    0.120498] Info: [libkvmplat] <shutdown.c @   35> Unikraft halted
```

## 分析

### plat

第一次打印出日志的位置是 libkvmplat 的 setup.c:268，实际位于 `unikraft/plat/kvm/x86/setup.c`，259-268 行：

```c
void _libkvmplat_entry(void *arg)
{
    struct multiboot_info *mi = (struct multiboot_info *)arg;

    _init_cpufeatures();
    _libkvmplat_init_console();
    traps_init();
    intctrl_init();

    uk_pr_info("Entering from KVM (x86)...\n");
```

x86 的汇编不怎么能看懂，不过 `_libkvmplat_entry` 基本上就是第一次进 C 的位置了，前面大概也没那么重要，可以不管。接下来：

1. `_init_cpufeatures`：填写一个 `_x86_features` 结构体，大概是 cpu 功能探测之类的；
2. `_libkvmplat_init_console`：初始化 console；
3. `traps_init`：初始化陷入处理；
4. `intctrl_init`：里面调了一个 PIC_remap，PIC=`Programmable Interrupt Controller` 可编程中断控制器；

这 4 个函数都是 `plat/kvm/x86` 内部定义的，不管函数名带不带下划线。plat 库的函数导出可以查看 `unikraft/include/uk/plat` 目录。

接下来的这一段和 x86_64 引导的 multiboot 规范有关，以及打印了一些日志，不用仔细看：

```c
uk_pr_info("     multiboot: %p\n", mi);

/*
 * The multiboot structures may be anywhere in memory, so take a copy of
 * everything necessary before we initialise memory allocation.
 */
_mb_get_cmdline(mi);
_mb_init_mem(mi);
_mb_init_initrd(mi);

if (_libkvmplat_cfg.initrd.len)
    uk_pr_info("        initrd: %p\n", (void *) _libkvmplat_cfg.initrd.start);
uk_pr_info("    heap start: %p\n", (void *) _libkvmplat_cfg.heap.start);
if (_libkvmplat_cfg.heap2.len)
    uk_pr_info(" heap start (2): %p\n", (void *) _libkvmplat_cfg.heap2.start);
uk_pr_info("     stack top: %p\n", (void *) _libkvmplat_cfg.bstack.start);
```

最后：

```c
#ifdef CONFIG_HAVE_SYSCALL
_init_syscall();
#endif /* CONFIG_HAVE_SYSCALL */

#if CONFIG_HAVE_X86PKU
_check_ospke();
#endif /* CONFIG_HAVE_X86PKU */

/*
 * Switch away from the bootstrap stack as early as possible.
 */
uk_pr_info("Switch from bootstrap stack to stack @%p\n", (void *) _libkvmplat_cfg.bstack.end);
_libkvmplat_newstack(_libkvmplat_cfg.bstack.end, _libkvmplat_entry2, 0);
```

`CONFIG_HAVE_SYSCALL` 应该是为 x86 支持的平行发送的系统调用做准备。`_init_syscall()` 的源码上有一个版权声明，提到这个函数来自 [hermitux](https://ssrg-vt.github.io/hermitux/)，另一个 x86 幺内核。

`CONFIG_HAVE_X86PKU` 不知道是干什么用的，大概是 x86 上的某种硬件功能。

`_libkvmplat_newstack` 是一个汇编程序，大约是在一个新的栈上执行函数的意思，总之，它调用了 `_libkvmplat_entry2`：

```c
static void _libkvmplat_entry2(void *arg __attribute__((unused))) {
    ukplat_entry_argp(NULL, cmdline, sizeof(cmdline));
}
```

继续调用 `ukplat_entry_argp`，实际上离开了 plat 库。这个函数的声明在 `unikraft/include/uk/plat/bootstrap.h`：

```c
/**
 * Called by platform library during initialization
 * This function has to be provided by a non-platform library for bootstrapping
 * It is intended to do the argument parsing only and then call ukplat_entry()
 * @param arg0     Optional buffer for first argument (argv[0]), can be __NULL
 *                 Arguments found on argb will be placed on argv[1] onwards
 * @param argb     Buffer of arguments, can be __NULL
 * @param argb_len Length of arg buffer (set to __SZ_MAX, when argb should be
 *                 handled as zero-terminated)
 */
void ukplat_entry_argp(char *arg0, char *argb, __sz argb_len) __noreturn;
```

文档翻译：

> 在初始化过程中被平台库调用。该函数必须由非平台库提供，用于引导。用于参数解析，并调用 `ukplat_entry()`
>
> @param arg0 第一个参数的可选缓冲区（`argv[0]`），可以是 `__NULL`。在 `argb` 上找到的参数将放置到 `argv[1]`。
>
> @param argb 参数的缓冲区，可以是 `__NULL`。
>
> @param argb_len 参数缓冲区的长度（当 `argb` 应该被处理为零结尾时，设置为 `__SZ_MAX`）。

这里实际调用的是静态链接进来的 ukboot 提供的实现，根据[论文描述](/tranlation/20220923-unikraft.md#3-Unikraft-架构和-API)，ukboot 是一个默认的启动流程，可以由接口相同的另一个库替换掉。

### ukboot

在 `unikraft/lib/ukboot/boot.c`：

```c
void ukplat_entry_argp(char *arg0, char *argb, __sz argb_len) {
    static char *argv[CONFIG_LIBUKBOOT_MAXNBARGS];
    int argc = 0;

    if (arg0) {
        argv[0] = arg0;
        argc += 1;
    }
    if (argb && argb_len) {
        argc += uk_argnparse(
            argb,
            argb_len,
            arg0 ? &argv[1] : &argv[0],
            arg0 ? (CONFIG_LIBUKBOOT_MAXNBARGS - 1) : CONFIG_LIBUKBOOT_MAXNBARGS
        );
    }
    ukplat_entry(argc, argv);
}
```

`CONFIG_LIBUKBOOT_MAXNBARGS` 是 `ukboot` 的一个可配置的项，在 menuconfig 里可以修改，默认值是 60。这个函数计算 `argc` 并把参数都填进 `argv`，然后调用 `ukplat_entry`。`ukplat_entry` 的声明和定义都是挨着 `ukplat_entry_argp` 的，声明：

```c
/**
 * Called by platform library during initialization
 * This function has to be provided by a non-platform library for bootstrapping
 * It is called directly by platform libraries that have parsed arguments
 * already, otherwise a platform library will call ukplat_entry_argp() to let
 * the arguments parsed from a string buffer
 * @param argc Number of arguments
 * @param args Array to '\0'-terminated arguments
 */
void ukplat_entry(int argc, char *argv[]) __noreturn;
```

文档翻译：

> 在初始化过程中被平台库调用。这个函数必须由非平台库提供，用于引导。它被已经解析了参数的平台库直接调用，否则平台库会调用 `ukplat_entry_argp()` 来让参数从字符串缓冲区解析出来。

就是说如果平台库能直接提供解析好的参数就不用 `ukplat_entry_argp()` 了，估计是给 linuxu 用的。感觉封装一个参数解析也没啥大用，还不如只留一个 `ukplat_entry` 得了。

`ukplat_entry` 的定义都是一段一段的条件编译，可以一段一段地读。

首先是 3 个不受编译选项控制的变量：

```c
struct thread_main_arg tma;
int kern_args = 0;
int rc __maybe_unused = 0;
```

在这声明是为了照顾传统 C 写法，`tma` 在最后才会用到；`rc` 实际上是一个存错误码的局部变量，相当于 `errno`。

---

然后，如果 `ukalloc`（分配器接口）存在的话，初始化一个变量保存分配器链表：

```c
#if CONFIG_LIBUKALLOC
struct uk_alloc *a = NULL;
#endif
```

`uk_alloc` 既要描述分配器结构同时还是一个侵入式单链表，在 `unikraft/lib/ukalloc/include/uk/alloc.h:79`。

---

`CONFIG_LIBUKBOOT_NOALLOC` 是 `ukboot` 库的一个选项，可能是 `ukboot` 自己用什么分配器。除非 `ukboot` 不使用分配器，否则声明一个 `ukplat_memregion_desc` 结构体：

```c
#if !CONFIG_LIBUKBOOT_NOALLOC
struct ukplat_memregion_desc md;
#endif
```

---

如果有调度器，定义调度器和主线程指针（和分配器一样，调度器是一个侵入式单链表）：

```c
#if CONFIG_LIBUKSCHED
struct uk_sched *s = NULL;
struct uk_thread *main_thread = NULL;
#endif
```

---

`uksp` 是一个库，自述为“栈保护器”（uksp: Stack protector），不知道具体是干什么用的：

```c
/* We use a macro because if we were to use a function we
 * would not be able to return from the function if we have
 * changed the stack protector inside the function */
#if CONFIG_LIBUKSP
UKSP_INIT_CANARY();
#endif
```

---

然后是一段不受编译选项控制的代码：

```c
uk_ctor_func_t *ctorfn;

uk_pr_info("Unikraft constructor table at %p - %p\n", &uk_ctortab_start[0], &uk_ctortab_end);
uk_ctortab_foreach(ctorfn, uk_ctortab_start, uk_ctortab_end) {
    UK_ASSERT(*ctorfn);
    uk_pr_debug("Call constructor: %p())...\n", *ctorfn);
    (*ctorfn)();
}
```

这是 unikraft 定义的一种模块间交互的方式，即每个模块指定自己的一部分链接到某个段上，然后调用者直接去找那个段，相当于一个链接时动态表。因为操作系统需要定制链接脚本，所以可以这么弄。这里的用法是是经典的控制反转+依赖注入，需要动态初始化的模块直接把自己的构造器链接到一个 `.uk_ctortab` 段上，然后由 `ukboot` 依次调用。修改迭代中的日志级别之后，helloworld 会打印出：

```bash
[    0.000000] Info: [libukboot] <boot.c @  202> Unikraft constructor table at 0x113000 - 0x113010
[    0.000000] Warn: [libukboot] <boot.c @  207> Call constructor: 0x1082b0())...
[    0.000000] Warn: [libukboot] <boot.c @  207> Call constructor: 0x108a20())...
```

调用了 2 个注入的构造器，但这个没设计名字，不容易知道是谁注入的。构造静态链接的方法见[文档翻译](20221008-add-section.md)。ctor 这一组宏的定义在 `unikraft/include/uk/ctors.h`，类似文档的描述（格式经过调整）：

```c
extern const uk_ctor_func_t uk_ctortab_start[];
extern const uk_ctor_func_t uk_ctortab_end;

/**
 * Register a Unikraft constructor function that is called during bootstrap (uk_ctortab)
 *
 * @param fn
 *   Constructor function to be called
 * @param prio
 *   Priority level (0 (earliest) to 9 (latest))
 *   Use the UK_PRIO_AFTER() helper macro for computing priority dependencies.
 *   Note: Any other value for level will be ignored
 */
#define __UK_CTORTAB(fn, prio)            \
    static const uk_ctor_func_t           \
    __used __section(".uk_ctortab" #prio) \
    __uk_ctortab ## prio ## _ ## fn = (fn)

#define _UK_CTORTAB(fn, prio) __UK_CTORTAB(fn, prio)

#define UK_CTOR_PRIO(fn, prio) _UK_CTORTAB(fn, prio)

/**
 * Similar interface without priority.
 */
#define UK_CTOR(fn) UK_CTOR_PRIO(fn, UK_PRIO_LATEST)

/* DELETEME: Compatibility wrapper for existing code, to be removed! */
#define UK_CTOR_FUNC(lvl, ctorf) _UK_CTORTAB(ctorf, lvl)

/**
 * Helper macro for iterating over constructor pointer tables
 * Please note that the table may contain NULL pointer entries
 *
 * @param itr
 *   Iterator variable (uk_ctor_func_t *) which points to the individual
 *   table entries during iteration
 * @param ctortab_start
 *   Start address of table (type: const uk_ctor_func_t[])
 * @param ctortab_end
 *   End address of table (type: const uk_ctor_func_t)
 */
#define uk_ctortab_foreach(itr, ctortab_start, ctortab_end) \
    for ((itr) = DECONST(uk_ctor_func_t*, ctortab_start);   \
         (itr) < &(ctortab_end);                            \
         (itr)++)
```

注册和迭代的宏都和文档描述基本一致，只是加了几个不同的包装。

---

```c
#ifdef CONFIG_LIBUKLIBPARAM
rc = (argc > 1) ? uk_libparam_parse(argv[0], argc - 1, &argv[1]) : 0;
if (unlikely(rc < 0))
    uk_pr_crit("Failed to parse the kernel argument\n");
else {
    kern_args = rc;
    uk_pr_info("Found %d library args\n", kern_args);
}
#endif /* CONFIG_LIBUKLIBPARAM */
```

这一段判断有没有加载 `uklibparam` 库。这个库自述“库参数”（uklibparam: Library arguments），大概是用于给用户程序传参？没用到可以跳过。

---

下一段是嵌套的条件编译，为了更好看懂，缩进调整过。整段基于 `CONFIG_LIBUKBOOT_NOALLOC` 标记，声明变量时遇到过，大概是 ukboot 用不用动态内存分配，需要时编译这段代码：

```c
#if !CONFIG_LIBUKBOOT_NOALLOC
/* initialize memory allocator
 * FIXME: allocators are hard-coded for now
 */
uk_pr_info("Initialize memory allocator...\n");
ukplat_memregion_foreach(&md, UKPLAT_MEMRF_ALLOCATABLE) {

    #if CONFIG_UKPLAT_MEMRNAME
    uk_pr_debug("Try memory region: %p - %p (flags: 0x%02x, name: %s)...\n",
        md.base,
        (void *)((size_t)md.base + md.len),
        md.flags,
        md.name
    );
    #else
    uk_pr_debug("Try memory region: %p - %p (flags: 0x%02x)...\n",
        md.base,
        (void *)((size_t)md.base + md.len),
        md.flags
    );
    #endif

    /* try to use memory region to initialize allocator
     * if it fails, we will try again with the next region.
     * As soon we have an allocator, we simply add every
     * subsequent region to it
     */
    if (!a) {
        #if CONFIG_LIBUKBOOT_INITBBUDDY
        a = uk_allocbbuddy_init(md.base, md.len);
        #elif CONFIG_LIBUKBOOT_INITREGION
        a = uk_allocregion_init(md.base, md.len);
        #elif CONFIG_LIBUKBOOT_INITTLSF
        a = uk_tlsf_init(md.base, md.len);
        #endif
    } else {
        uk_alloc_addmem(a, md.base, md.len);
    }
}
if (unlikely(!a))
    uk_pr_warn("No suitable memory region for memory allocator. Continue without heap\n");
else {
    rc = ukplat_memallocator_set(a);
    if (unlikely(rc != 0))
        UK_CRASH("Could not set the platform memory allocator\n");
}
#endif
```

对于 helloworld，这段代码产生这个日志（后面还有一堆伙伴分配器插入产生的日志）：

```bash
[    0.000000] Info: [libukboot] <boot.c @  225> Initialize memory allocator...
[    0.000000] dbg:  [libukboot] <boot.c @  234> Try memory region: 0x140000 - 0x7fd0000 (flags: 0x31)...
[    0.000000] Info: [libukallocbbuddy] <bbuddy.c @  491> Initialize binary buddy allocator 140000
```

就是找到了 `0x140000 - 0x7fd0000` 这个可用地址段，然后把整个地址段插入伙伴分配器了。

---

```c
#if CONFIG_LIBUKALLOC
uk_pr_info("Initialize IRQ subsystem...\n");
rc = ukplat_irq_init(a);
if (unlikely(rc != 0))
    UK_CRASH("Could not initialize the platform IRQ subsystem\n");
#endif
```

`ukplat_irq_init` 函数是 plat 导出的，kvm 下的定义是：

```c
int ukplat_irq_init(struct uk_alloc *a)
{
    UK_ASSERT(allocator == NULL);
    allocator = a;
    return 0;
}
```

由于它依赖动态内存分配，所以在 `ukalloc` 存在时才配置。

---

初始化平台时间操作不受编译选项控制。

```c
/* On most platforms the timer depend on an initialized IRQ subsystem */
uk_pr_info("Initialize platform time...\n");
ukplat_time_init();
```

`ukplat_time_init` 函数的定义也是平台相关的，kvm/x86 的定义是：

```c
/*must be called before interrupts are enabled*/
void ukplat_time_init(void)
{
    int rc;

    rc = ukplat_irq_register(0, timer_handler, NULL);
    if (rc < 0)
        UK_CRASH("Failed to register timer interrupt handler\n");

    rc = tscclock_init();
    if (rc < 0)
        UK_CRASH("Failed to initialize TSCCLOCK\n");
}
```

---

如果调度器存在，初始化调度器：

```c
#if CONFIG_LIBUKSCHED
/* Init scheduler. */
s = uk_sched_default_init(a);
if (unlikely(!s))
    UK_CRASH("Could not initialize the scheduler\n");
#endif
```

---

启动参数将传递给主线程，但 `kern_args` 实际上是常数 0：

```c
tma.argc = argc - kern_args;
tma.argv = &argv[kern_args];
```

---

最后一步，如果有调度器，就让调度器加载主线程，并启动调度器；否则直接启动主线程。直接启动主线程时开了中断，可以推断出有调度器时中断应该是托管给调度器控制了：

```c
#if CONFIG_LIBUKSCHED
main_thread = uk_thread_create("main", main_thread_func, &tma);
if (unlikely(!main_thread))
    UK_CRASH("Could not create main thread\n");
uk_sched_start(s);
#else
/* Enable interrupts before starting the application */
ukplat_lcpu_enable_irq();
main_thread_func(&tma);
#endif
```

---

主线程的定义（有一些语法修改）：

```c
static void main_thread_func(struct thread_main_arg *tma)
{
    int i;
    int ret;
    uk_ctor_func_t *ctorfn;
    uk_init_func_t *initfn;

    /**
     * Run init table
     */
    uk_pr_info("Init Table @ %p - %p\n", &uk_inittab_start[0], &uk_inittab_end);
    uk_inittab_foreach(initfn, uk_inittab_start, uk_inittab_end) {
        UK_ASSERT(*initfn);
        uk_pr_debug("Call init function: %p()...\n", *initfn);
        ret = (*initfn)();
        if (ret < 0) {
            uk_pr_err("Init function at %p returned error %d\n", *initfn, ret);
            ret = UKPLAT_CRASH;
            goto exit;
        }
    }

    print_banner(stdout);
    fflush(stdout);

    /*
     * Application
     *
     * We are calling the application constructors right before calling
     * the application's main(). All of our Unikraft systems, VFS,
     * networking stack is initialized at this point. This way we closely
     * mimic what a regular user application (e.g., BSD, Linux) would expect
     * from its OS being initialized.
     */
    uk_pr_info("Pre-init table at %p - %p\n", &__preinit_array_start[0], &__preinit_array_end);
    uk_ctortab_foreach(ctorfn, __preinit_array_start, __preinit_array_end) {
        if (!*ctorfn)
        continue;

        uk_pr_debug("Call pre-init constructor: %p()...\n", *ctorfn);
        (*ctorfn)();
    }

    uk_pr_info("Constructor table at %p - %p\n", &__init_array_start[0], &__init_array_end);
    uk_ctortab_foreach(ctorfn, __init_array_start, __init_array_end) {
        if (!*ctorfn)
        continue;

        uk_pr_debug("Call constructor: %p()...\n", *ctorfn);
        (*ctorfn)();
    }

    uk_pr_info("Calling main(%d, [", tma->argc);
    for (i = 0; i < tma->argc; ++i) {
        uk_pr_info("'%s'", tma->argv[i]);
        if ((i + 1) < tma->argc)
            uk_pr_info(", ");
    }
    uk_pr_info("])\n");

    ret = main(tma->argc, tma->argv);
    uk_pr_info("main returned %d, halting system\n", ret);
    ret = (ret != 0) ? UKPLAT_CRASH : UKPLAT_HALT;

exit:
    ukplat_terminate(ret); /* does not return */
}
```
