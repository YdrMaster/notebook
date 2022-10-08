# 文档翻译：二进制结构

> Binary Structure

> [原文](https://unikraft.org/docs/develop/binary-structure/#adding-new-sections-to-an-elf)

## 向 ELF 添加新段

> Adding New Sections to an ELF

在有些情况下，我们想为我们的应用程序或库在可执行文件（ELF 格式--可执行和链接格式）中添加新的部分。这些部分有用的原因是，库（或应用程序）变得更容易配置，从而服务于更多的目的。例如，Unikraft 虚拟文件系统（即 `vfscore` 库）使用了这样一个部分，它在其中注册了使用的文件系统（`ramfs`，`9pfs`），我们将在下面的章节中讨论这个问题。另一个使用额外部分的组件是调度器。调度器接口允许我们在构建时注册一组函数，这些函数将在线程被创建或执行结束时被调用。

> There are situations in which we want to add new sections in the executable file (ELF format - *Executable and Linking Format*) for our application or library. The reason these sections are useful is that the library (or application) becomes much easier to configure, thus serving more purposes. For example, the Unikraft virtual filesystem (i.e. the `vfscore` library) uses such a section in which it registers the used filesystem (`ramfs`, `9pfs`), and we are going to discuss this in the following sections. Another component that makes use of additional sections is the scheduler. The scheduler interface allows us to register a set of functions at build time that will be called when a thread is created or at the end of its execution.

我们可以在我们的应用程序/库中添加这样一个部分，方法如下：

> The way we can add such a section in our application/library is the following:

1\. 创建一个扩展名为 `.ld` 的文件（如 `extra.ld`），内容如下：

> 1\. Create a file with the .ld extension (e.g. extra.ld) with the following content:

```ld
SECTIONS
{
    .my_section : {
        PROVIDE(my_section_start = .);
        KEEP (*(.my_section_entry))
        PROVIDE(my_section_end = .);
    }
}
INSERT AFTER .text;
```

2\. 在 `Makefile.uk` 中添加以这一行：

> 2\. Add the following line to `Makefile.uk`:

```makefile
LIBYOURAPPNAME_SRCS-$(CONFIG_LIBYOURAPPNAME) += $(LIBYOURAPPNAME_BASE)/extra.ld
```

> 这将在 ELF 文件的 `.text` 段之后添加 `.my_section` 段。`.my_section_entry` 字段将用于把对象注册到这个段，对它的访问通常通过遍历这个段获得（即从 `my_section_start` 到 `my_section_end`）。
>
> > This will add the `.my_section` section after the `.text` section in the ELF file. The `.my_section_entry` field will be used to register an entry in this section, and access to it is generally gained via traversing the section’s endpoints (i.e. from `my_section_start` to `my_section_end`).

在运行程序之前，让我们分析一下源代码。我们想在新增加的部分注册 `my-structure` 结构。在 Unikraft 核心库中，这通常是使用宏来完成的。所以我们也要这样做。

> Before running the program let’s analyze the source code. We want to register the my-structure structure in the newly added section. In Unikraft core libraries this is usually done using macros. So we will do the same.

```c
# define MY_REGISTER(s, f) static const struct my_structure \
         __section(".my_section_entry")                     \
         __my_section_var __used = {.name = (s), .func = (f)};
```

这个宏接收结构的字段，并在新增加的段定义一个名为 `__my_section_var` 的变量。这是通过 `__section()` 完成的。我们还使用了 `__used` 属性来告诉编译器不要优化掉这个变量。注意，这个宏使用了不同的编译器属性。这些属性大多在 `uk/essentials.h` 中，所以在使用宏时请确保包含它。

> This macro receives the fields of the structure and defines a variable called `__my_section_var` in the newly added section. This is done via `__section()`. We also use the `__used` attribute to tell the compiler not to optimize out the variable. Note that this macro uses different compiler attributes. Most of these are in `uk/essentials.h`, so please make sure you include it when working with macros.

接下来，让我们分析一下，我们可以通过什么方法去寻找这个段的对象。我们必须首先导入该段的端点。可以按以下方法进行：

> Next, let’s analyze the method by which we can go through this section to find the entries. We must first import the endpoints of the section. It can be done as follows:

```c
extern const struct my_structure my_section_start;
extern const struct my_structure my_section_end;
```

利用端点，我们可以编写宏来迭代这个段：

> Using the endpoints we can write the macro for iterating through the section:

```c
# define for_each_entry(iter)           \
        for (iter = &my_section_start;  \
             iter < &my_section_end;    \
             iter++)
```

> 如果你对宏不熟悉，你可以用 GCC 的预处理程序检查它们的扩展内容。删除所有包含的头文件，执行 `gcc -E main.c`。
>
> > If you’re not familiar with macros, you may check what they expand to with the GCC’s preprocessor. Remove all the included headers and run `gcc -E main.c`.

让我们来配置这个程序。使用 `make menuconfig` 命令来设置 KVM 平台，如下图所示。

> Let’s configure the program. Use the `make menuconfig` command to set the KVM platform as in the following image.

> \#!\[platform_configuration\]

保存配置，退出 menuconfig 标签并运行 `make`。现在，让我们来运行它。你可以使用下面的命令：

> Save the configuration, exit the menuconfig tab and run `make`. Now, let’s run it. You can use the following command:

```bash
qemu-guest -k build/01-extrald-app_kvm-x86_64
```

该程序的输出应该是以下内容：

> The program’s output should be the following:

> \#!\[01-extrald-app-output\]

为了查看关于段的大小和它的起始地址的信息是否正确，我们将使用 readelf 工具检查二进制文件。readelf 工具是用来显示 ELF 文件的信息，比如节或段。关于它的更多信息在[这里](https://man7.org/linux/man-pages/man1/readelf.1.html)。使用下面的命令来显示关于 ELF 段的信息：

> To see that the information about the section size and its start address is correct we will examine the binary using the readelf utility. The readelf utility is used to display information about ELF files, like sections or segments. More about it here. Use the following command to display information about the ELF sections:

```bash
readelf -S build/01-extrald-app_kvm-x86_64
```

输出应该看起来像这样：

> The output should look like this:

> \#!\[readelf_output\]

我们可以看到，`my_section` 确实在 ELF 的段中。看看它的大小，我们发现它是 0x10 字节（相当于十进制的 16 字节）。我们还注意到，该部分的起始地址是 0x1120f0，与我们运行程序得到的地址相同。

> We can see that my_section is indeed among the sections of the ELF. Looking at its size we see that it is 0x10 bytes (the equivalent of 16 in decimal). We also notice that the start address of the section is 0x1120f0, the same as the one we got from running the program.
