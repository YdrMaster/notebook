# unikraft 启动流程分析

## 实验方法

用 `kraft menuconfig`，可以配置 library-configuration/ukdebug/kernel-message-level，把日志级别配置成全部显示，以观察启动流程。

qemu 上运行一个改过一些日志级别的 helloworld 效果大致为（格式经过微调）：

```bash
Booting from ROM..
[    0.000000] Info: [libkvmplat] <setup.c @  268> Entering from KVM (x86)...
[    0.000000] Info: [libkvmplat] <setup.c @  269>      multiboot: 0x9500
[    0.000000] Info: [libkvmplat] <setup.c @  282>     heap start: 0x140000
[    0.000000] Info: [libkvmplat] <setup.c @  287>      stack top: 0x7fd0000
[    0.000000] Info: [libkvmplat] <setup.c @  301> Switch from bootstrap stack to stack @0x7fe0000
[    0.000000] Info: [libukboot] <boot.c @  202> Unikraft constructor table at 0x113000 - 0x113010
[    0.000000] Warn: [libukboot] <boot.c @  207> Call constructor: 0x1082b0())...
[    0.000000] Warn: [libukboot] <boot.c @  207> Call constructor: 0x108a20())...
[    0.000000] Info: [libukboot] <boot.c @  225> Initialize memory allocator...
[    0.000000] Info: [libukallocbbuddy] <bbuddy.c @  491> Initialize binary buddy allocator 140000
[    0.000000] Info: [libukboot] <boot.c @  268> Initialize IRQ subsystem...
[    0.000000] Info: [libukboot] <boot.c @  275> Initialize platform time...
[    0.000000] Info: [libkvmplat] <tscclock.c @  253> Calibrating TSC clock against i8254 timer
[    0.100061] Info: [libkvmplat] <tscclock.c @  274> Clock source: TSC, frequency estimate is 3113081910 Hz
[    0.101248] Info: [libukboot] <boot.c @   93> Init Table @ 0x113010 - 0x113018
[    0.102075] Info: [libukbus] <bus.c @  136> Initialize bus handlers...
[    0.103270] Info: [libukbus] <bus.c @  138> Probe buses...
[    0.104907] Info: [libkvmpci] <pci_bus.c @  284> PCI 00:00.00 (0600 8086:1237): <no driver>
[    0.106652] Info: [libkvmpci] <pci_bus.c @  284> PCI 00:01.00 (0600 8086:7000): <no driver>
[    0.108106] Info: [libkvmpci] <pci_bus.c @  284> PCI 00:02.00 (0300 1234:1111): <no driver>
[    0.109337] Info: [libkvmpci] <pci_bus.c @  284> PCI 00:03.00 (0200 8086:100e): <no driver>
Powered by
o.   .o       _ _               __ _
Oo   Oo  ___ (_) | __ __  __ _ ' _) :_
oO   oO ' _ `| | |/ /  _)' _` | |_|  _)
oOo oOO| | | | |   (| | | (_) |  _) :_
 OoOoO ._, ._:_:_,\_._,  .__,_:_, \___)
            Tethys 0.5.0~b8be82b-custom
[    0.115208] Info: [libukboot] <boot.c @  120> Pre-init table at 0x117080 - 0x117080
[    0.119628] Info: [libukboot] <boot.c @  131> Constructor table at 0x117080 - 0x117080
[    0.121724] Info: [libukboot] <boot.c @  142> Calling main(1, ['build/helloworld_kvm-x86_64'])
Hello world!
Arguments:  "build/helloworld_kvm-x86_64"
[    0.125712] Info: [libukboot] <boot.c @  151> main returned 0, halting system
[    0.126816] Info: [libkvmplat] <shutdown.c @   35> Unikraft halted
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

x86 的汇编不怎么能看懂，不过 `_libkvmplat_entry` 基本上就是第一次进 c 的位置了，前面大概也没那么重要，可以不管。接下来：

1. `_init_cpufeatures`：保存 tp 指针，填写一个 `_x86_features` 结构体，大概是 cpu 功能探测之类的；
2. `_libkvmplat_init_console`：初始化 console；
3. `traps_init`：初始化陷入处理；
4. `intctrl_init`：里面调了一个 PIC_remap，PIC=`Programmable Interrupt Controller` 可编程中断控制器；

这 4 个函数都是 `plat/kvm/x86` 内部定义的，不管函数名带不带下划线。plat 库似乎没有函数导出表，不那么好查。

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

`CONFIG_HAVE_SYSCALL` 应该是为 x86 支持的平行发送的系统调用做准备。`CONFIG_HAVE_X86PKU` 不知道是干什么用的，可能是 x86 上的某种功能，不管它。`_libkvmplat_newstack` 是一个汇编程序，大约是在一个新的栈上执行函数的意思，总之，它调用了 `_libkvmplat_entry2`：

```c
static void _libkvmplat_entry2(void *arg __attribute__((unused)))
{
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

这里实际调用的是静态链接进来的 ukboot 提供的实现，根据论文描述，ukboot 是一个默认的启动流程，可以由接口相同的另一个库替换掉。

### ukboot

在 `unikraft/lib/ukboot/boot.c`：

```c
void ukplat_entry_argp(char *arg0, char *argb, __sz argb_len)
{
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

构造了 `argc` 和 `argv` 之后调用 `ukplat_entry`。这个函数的声明和定义都是挨着 `ukplat_entry_argp` 的，声明：

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

然后，如果 `ukalloc`（分配器接口）存在的话，初始化一个变量保存分配器链表：

```c
#if CONFIG_LIBUKALLOC
struct uk_alloc *a = NULL;
#endif
```

`CONFIG_LIBUKBOOT_NOALLOC` 是 `ukboot` 库的一个选项，应该是 `ukboot` 自己用什么分配器。除非 `ukboot` 不使用分配器，否则声明一个 `ukplat_memregion_desc` 结构体：

```c
#if !CONFIG_LIBUKBOOT_NOALLOC
struct ukplat_memregion_desc md;
#endif
```

如果有调度器，定义调度器和主线程指针（和分配器一样，调度器是一个侵入式单链表）：

```c
#if CONFIG_LIBUKSCHED
struct uk_sched *s = NULL;
struct uk_thread *main_thread = NULL;
#endif
```

`uksp` 是一个库，自述为“栈保护器”（uksp: Stack protector），不知道具体是干什么用的：

```c
/* We use a macro because if we were to use a function we
 * would not be able to return from the function if we have
 * changed the stack protector inside the function */
#if CONFIG_LIBUKSP
UKSP_INIT_CANARY();
#endif
```

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

这是 unikraft 定义的一种模块间通信的方式，即每个模块指定自己的一部分链接到某个段上，然后调用者直接去找那个段。因为操作系统需要定制链接脚本，所以可以这么弄。这里是经典的控制反转+依赖注入，需要动态初始化的模块直接把自己的构造器链接到一个 `.uk_ctortab` 段上，然后由 `ukboot` 依次调用。修改迭代中的日志级别之后，helloworld 会打印出：

```bash
[    0.000000] Info: [libukboot] <boot.c @  202> Unikraft constructor table at 0x113000 - 0x113010
[    0.000000] Warn: [libukboot] <boot.c @  207> Call constructor: 0x1082b0())...
[    0.000000] Warn: [libukboot] <boot.c @  207> Call constructor: 0x108a20())...
```

调用了 2 个注入的构造器，但不知道是什么。
