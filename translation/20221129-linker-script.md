# 3 链接器脚本

> 3 Linker Scripts

> [原文](https://sourceware.org/binutils/docs/ld/Scripts.html)

每次链接都是由一个链接器脚本控制的。这个脚本是用链接器命令语言编写的。

> Every link is controlled by a linker script. This script is written in the linker command language.

链接器脚本的主要功能是描述输入文件中的节应该如何被映射到输出文件中，并控制输出文件的内存布局。大多数链接器脚本的作用不外乎如此。然而，在必要的时候，链接器脚本也可以使用下面描述的命令，指导链接器执行许多其他操作。

> The main purpose of the linker script is to describe how the sections in the input files should be mapped into the output file, and to control the memory layout of the output file. Most linker scripts do nothing more than this. However, when necessary, the linker script can also direct the linker to perform many other operations, using the commands described below.

链接器总是使用一个链接器脚本。如果你没有自己提供一个，链接器将使用一个默认的脚本，它被编译到链接器的可执行文件中。你可以使用 `--verbose` 命令行选项来显示默认的链接器脚本。某些命令行选项，如 `-r` 或 `-N`，将影响默认的链接器脚本。

> The linker always uses a linker script. If you do not supply one yourself, the linker will use a default script that is compiled into the linker executable. You can use the ‘--verbose’ command-line option to display the default linker script. Certain command-line options, such as ‘-r’ or ‘-N’, will affect the default linker script.

你可以通过使用 `-T` 命令行选项提供你自己的链接器脚本。当你这样做时，你的链接器脚本将取代默认的链接器脚本。

> You may supply your own linker script by using the ‘-T’ command line option. When you do this, your linker script will replace the default linker script.

你也可以隐式地使用链接器脚本，把它们命名为链接器的输入文件，就像它们是要被链接的文件一样。参见[隐式链接器脚本](#311-隐式链接器脚本)。

> You may also use linker scripts implicitly by naming them as input files to the linker, as though they were files to be linked. See Implicit Linker Scripts.

- [链接器脚本的基本概念](#31-链接器脚本的基本概念)
- [链接器脚本格式](#32-链接器脚本格式)
- [简单的链接器脚本示例](#33-简单的链接器脚本示例)
- [简单的链接器脚本命令](#34-简单的链接器脚本命令)
- [为符号赋值](#35-为符号赋值)
- [SECTIONS 命令](#36-sections-命令)
- [MEMORY 命令](#37-memory-命令)
- [PHDRS 命令](#38-phdrs-命令)
- [VERSION 命令](#39-version-命令)
- [链接器脚本中的表达式](#310-链接器脚本中的表达式)
- [隐式链接器脚本](#311-隐式链接器脚本)

> - Basic Linker Script Concepts
> - Linker Script Format
> - Simple Linker Script Example
> - Simple Linker Script Commands
> - Assigning Values to Symbols
> - SECTIONS Command
> - MEMORY Command
> - PHDRS Command
> - VERSION Command
> - Expressions in Linker Scripts
> - Implicit Linker Scripts

## 3.1 链接器脚本的基本概念

> 3.1 Basic Linker Script Concepts

我们需要定义一些基本的概念和词汇，以便描述链接器的脚本语言。

> We need to define some basic concepts and vocabulary in order to describe the linker script language.

链接器将输入文件合并成一个输出文件。输出文件和每个输入文件都采用一种特殊的数据格式，称为对象文件格式。每个文件都被称为一个对象文件。输出文件通常被称为可执行文件，但为了我们的目的，我们也将其称为对象文件。所有对象文件都有一个节表。我们有时把输入文件中的一个节称为输入节；类似地，输出文件中的一个节是输出节。

> The linker combines input files into a single output file. The output file and each input file are in a special data format known as an object file format. Each file is called an object file. The output file is often called an executable, but for our purposes we will also call it an object file. Each object file has, among other things, a list of sections. We sometimes refer to a section in an input file as an input section; similarly, a section in the output file is an output section.

一个对象文件中的每个节都有一个名称和一个大小。大多数节也有一个相关的数据块，被称为节内容。一个节可以被标记为可加载，这意味着当输出文件被运行时，其内容应该被加载到内存中。一个没有内容的节可能是可分配的，这意味着内存中的一个区域应该被预留出来，但没有特别的东西应该被加载到那里（在某些情况下，这个内存必须被清空）。一个既非可加载也非可分配的节通常包含某种调试信息。

> Each section in an object file has a name and a size. Most sections also have an associated block of data, known as the section contents. A section may be marked as loadable, which means that the contents should be loaded into memory when the output file is run. A section with no contents may be allocatable, which means that an area in memory should be set aside, but nothing in particular should be loaded there (in some cases this memory must be zeroed out). A section which is neither loadable nor allocatable typically contains some sort of debugging information.

每个可加载或可分配的输出节都有两个地址。第一个是 VMA，即虚拟内存地址。这是输出文件运行时该节的地址。第二个是 LMA，即加载内存地址。这是该节将被加载的地址。在大多数情况下，这两个地址是相同的。一个可能不同的例子是，数据节被加载到 ROM 中，然后在程序启动时被复制到 RAM 中（这种技术经常被用来初始化基于 ROM 的系统中的全局变量）。在这种情况下，ROM 地址是 LMA，而 RAM 地址是 VMA。

> Every loadable or allocatable output section has two addresses. The first is the VMA, or virtual memory address. This is the address the section will have when the output file is run. The second is the LMA, or load memory address. This is the address at which the section will be loaded. In most cases the two addresses will be the same. An example of when they might be different is when a data section is loaded into ROM, and then copied into RAM when the program starts up (this technique is often used to initialize global variables in a ROM based system). In this case the ROM address would be the LMA, and the RAM address would be the VMA.

你可以通过使用带有 `-h` 选项的 `objdump` 程序看到对象文件中的各个节。

> You can see the sections in an object file by using the objdump program with the ‘-h’ option.

每个对象文件也有一个符号列表，被称为符号表。一个符号可以是已定义的，也可以是未定义的。每个符号都有一个名称，每个已定义的符号都有一个地址，以及其他信息。如果你把一个 C 或 C++ 程序编译成一个对象文件，你将为每个已定义的函数和全局或静态变量得到一个已定义的符号。在输入文件中被引用的每个未定义的函数或全局变量都将成为一个未定义的符号。

> Every object file also has a list of symbols, known as the symbol table. A symbol may be defined or undefined. Each symbol has a name, and each defined symbol has an address, among other information. If you compile a C or C++ program into an object file, you will get a defined symbol for every defined function and global or static variable. Every undefined function or global variable which is referenced in the input file will become an undefined symbol.

你可以通过使用 `nm` 程序，或通过使用带有 `-t` 选项的 `objdump` 程序来查看对象文件中的符号。

> You can see the symbols in an object file by using the nm program, or by using the objdump program with the ‘-t’ option.

## 3.2 链接器脚本格式

> 3.2 Linker Script Format

链接器脚本是文本文件。

> Linker scripts are text files.

你把链接器脚本写成一系列的命令。每个命令都是一个关键字，后面可能有参数，或者是对一个符号的赋值。你可以用分号来分隔命令。空白通常被忽略。

> You write a linker script as a series of commands. Each command is either a keyword, possibly followed by arguments, or an assignment to a symbol. You may separate commands using semicolons. Whitespace is generally ignored.

像文件或格式名称这样的字符串通常可以直接输入。如果文件名包含一个字符，如逗号，你可以把文件名放在双引号中，否则会起到分隔文件名的作用。文件名中不能使用双引号字符。

> Strings such as file or format names can normally be entered directly. If the file name contains a character such as a comma which would otherwise serve to separate file names, you may put the file name in double quotes. There is no way to use a double quote character in a file name.

你可以在链接器脚本中加入注释，就像在 C 语言中一样，用 `/*` 和 `*/` 来分隔。和 C 语言一样，注释在语法上等同于空白。

> You may include comments in linker scripts just as in C, delimited by ‘/*’ and ‘*/’. As in C, comments are syntactically equivalent to whitespace.

## 3.3 简单的链接器脚本示例

> 3.3 Simple Linker Script Example

许多链接器脚本是相当简单的。

> Many linker scripts are fairly simple.

最简单的链接器脚本只有一条命令：SECTIONS。你使用 SECTIONS 命令来描述输出文件的内存布局。

> The simplest possible linker script has just one command: ‘SECTIONS’. You use the ‘SECTIONS’ command to describe the memory layout of the output file.

SECTIONS 是一个强大的命令。这里我们将描述它的一个简单用法。假设你的程序只包括代码、初始化数据和未初始化数据。这些将分别在 `.text`、`.data` 和 `.bss` 节。让我们进一步假设，你的输入文件只有这些节。

> The ‘SECTIONS’ command is a powerful command. Here we will describe a simple use of it. Let’s assume your program consists only of code, initialized data, and uninitialized data. These will be in the ‘.text’, ‘.data’, and ‘.bss’ sections, respectively. Let’s assume further that these are the only sections which appear in your input files.

对于这个例子，我们假设代码应该在地址 0x10000 处加载，而数据应该从地址 0x8000000 处开始。下面是一个链接器脚本，它可以做到这一点：

> For this example, let’s say that the code should be loaded at address 0x10000, and that the data should start at address 0x8000000. Here is a linker script which will do that:

```ld
SECTIONS
{
  . = 0x10000;
  .text : { *(.text) }
  . = 0x8000000;
  .data : { *(.data) }
  .bss : { *(.bss) }
}
```

你把 SECTIONS 命令写成关键词 SECTIONS，后面是一系列的符号赋值和用大括号括起来的输出节描述。

> You write the ‘SECTIONS’ command as the keyword ‘SECTIONS’, followed by a series of symbol assignments and output section descriptions enclosed in curly braces.

上面例子中 SECTIONS 命令里面的第一行设置了特殊符号 `.` 的值，这是位置计数器。如果你没有以其他方式指定输出节的地址（其他方式将在后面描述），地址将从位置计数器的当前值开始设置。然后，位置计数器按输出节的大小递增。在 SECTIONS 命令的开始，位置计数器的值为 `0`。

> The first line inside the ‘SECTIONS’ command of the above example sets the value of the special symbol ‘.’, which is the location counter. If you do not specify the address of an output section in some other way (other ways are described later), the address is set from the current value of the location counter. The location counter is then incremented by the size of the output section. At the start of the ‘SECTIONS’ command, the location counter has the value ‘0’.

第二行定义了一个输出节，`.text`。冒号是必要的语法，现在可以忽略。在输出节名称后面的大括号内，你列出了应该被放入这个输出节的输入节的名称。`*` 是一个通配符，可以匹配任何文件名。表达式 `*(.text)` 意味着所有输入文件中的所有 `.text` 输入节。

> The second line defines an output section, ‘.text’. The colon is required syntax which may be ignored for now. Within the curly braces after the output section name, you list the names of the input sections which should be placed into this output section. The ‘\*’ is a wildcard which matches any file name. The expression ‘\*(.text)’ means all ‘.text’ input sections in all input files.

由于定义输出节 `.text` 时位置计数器是 `0x10000`，链接器将把输出文件中 `.text` 节的地址设为 `0x10000`。

> Since the location counter is ‘0x10000’ when the output section ‘.text’ is defined, the linker will set the address of the ‘.text’ section in the output file to be ‘0x10000’.

剩下的几行定义了输出文件中的 `.data` 和 `.bss` 节。链接器将把 `.data` 输出节放在地址 `0x8000000` 处。在链接器放置 `.data` 输出节后，位置计数器的值将是 `0x8000000` 加上 `.data` 输出节的大小。其结果是链接器将在内存中的 `.data` 输出节之后立即放置 `.bss` 输出节。

> The remaining lines define the ‘.data’ and ‘.bss’ sections in the output file. The linker will place the ‘.data’ output section at address ‘0x8000000’. After the linker places the ‘.data’ output section, the value of the location counter will be ‘0x8000000’ plus the size of the ‘.data’ output section. The effect is that the linker will place the ‘.bss’ output section immediately after the ‘.data’ output section in memory.

链接器将确保每个输出节都有必要的对齐方式，如果有必要的话，可以增加位置计数器。在这个例子中，为 `.text` 和 `.data` 节指定的地址可能会满足任何对齐限制，但是链接器可能要在 `.data` 和 `.bss` 节之间制造一个小间隙。

> The linker will ensure that each output section has the required alignment, by increasing the location counter if necessary. In this example, the specified addresses for the ‘.text’ and ‘.data’ sections will probably satisfy any alignment constraints, but the linker may have to create a small gap between the ‘.data’ and ‘.bss’ sections.

这就是了！这就是一个简单而完整的链接器脚本。

> That’s it! That’s a simple and complete linker script.

## 3.4 简单的链接器脚本命令

> 3.4 Simple Linker Script Commands

在这一节中，我们将描述简单的链接器脚本命令。

> In this section we describe the simple linker script commands.

- [设置入口点](#341-设置入口点)
- [处理文件的命令](#342-处理文件的命令)
- [处理对象文件格式的命令](#343-处理对象文件格式的命令)
- [为内存区域指定别名](#344-为内存区域指定别名)
- [其他链接器脚本命令](#345-其他链接器脚本命令)

> - Setting the Entry Point
> - Commands Dealing with Files
> - Commands Dealing with Object File Formats
> - Assign alias names to memory regions
> - Other Linker Script Commands

### 3.4.1 设置入口点

> 3.4.1 Setting the Entry Point

程序中执行的第一条指令被称为入口点。你可以使用 ENTRY 链接器脚本命令来设置入口点。参数是一个符号名称：

> The first instruction to execute in a program is called the entry point. You can use the ENTRY linker script command to set the entry point. The argument is a symbol name:

```ld
ENTRY(symbol)
```

有几种方法来设置入口点。链接器将通过依次尝试以下方法来设置入口点，并在其中一种方法成功时停止：

There are several ways to set the entry point. The linker will set the entry point by trying each of the following methods in order, and stopping when one of them succeeds:

- `-e` 入口命令行选项；
- 链接器脚本中的 ENTRY(*symbol*) 命令；
- 由目标决定的符号的值，如果它被定义的话；对于许多目标来说是 `start`，但是对于例如基于 PE- 和 BeOS- 的系统，将检查可能的入口符号的列表，使用找到的第一个符号。
- 代码节的第一个字节的地址，如果存在并且正在创建一个可执行文件的话——代码节通常是 `.text`，但也可能是其他东西；
- 地址 0；

> - the ‘-e’ entry command-line option;
> - the ENTRY(symbol) command in a linker script;
> - the value of a target-specific symbol, if it is defined; For many targets this is start, but PE- and BeOS-based systems for example check a list of possible entry symbols, matching the first one found.
> - the address of the first byte of the code section, if present and an executable is being created - the code section is usually ‘.text’, but can be something else;
> - The address 0.

### 3.4.2 处理文件的命令

> 3.4.2 Commands Dealing with Files

有几个链接器脚本命令涉及到文件。

> Several linker script commands deal with files.

#### INCLUDE *filename*

在这个位置插入的链接器脚本的文件名。该文件将在当前目录和用 `-L` 选项指定的任何目录中被搜索到。你可以将对 INCLUDE 的调用嵌套到最多 10 层。

> Include the linker script filename at this point. The file will be searched for in the current directory, and in any directory specified with the -L option. You can nest calls to INCLUDE up to 10 levels deep.

你可以把 INCLUDE 指令放在顶层、MEMORY 或 SECTIONS 命令中、或输出节的描述中。

> You can place INCLUDE directives at the top level, in MEMORY or SECTIONS commands, or in output section descriptions.

#### INPUT(*file*, *file*, ...), INPUT(*file* *file* ...)

INPUT 命令指示链接器将命名的文件包括在链接中，就像它们在命令行中传入一样。

> The INPUT command directs the linker to include the named files in the link, as though they were named on the command line.

例如，如果你每次做链接时都想包括 subr.o，但你又懒得把它放在 每个链接命令行上，那么你可以在链接器脚本中加上 INPUT(subr.o)。

> For example, if you always want to include subr.o any time you do a link, but you can’t be bothered to put it on every link command line, then you can put ‘INPUT (subr.o)’ in your linker script.

事实上，如果你愿意，你可以在链接器脚本中列出所有的输入文件，然后只用 `-T` 选项来调用链接器。

> In fact, if you like, you can list all of your input files in the linker script, and then invoke the linker with nothing but a ‘-T’ option.

如果配置了 sysroot 前缀，并且文件名以 `/` 字符开头，而被处理的脚本位于 sysroot 前缀内，那么该文件名将在 sysroot 前缀中被寻找到。也可以通过指定 = 作为文件名路径中的第一个字符来强制使用系统根前缀，或者在文件名路径前加上 `$SYSROOT`。参见命令行选项中对 `-L` 的描述。

> In case a sysroot prefix is configured, and the filename starts with the ‘/’ character, and the script being processed was located inside the sysroot prefix, the filename will be looked for in the sysroot prefix. The sysroot prefix can also be forced by specifying = as the first character in the filename path, or prefixing the filename path with $SYSROOT. See also the description of ‘-L’ in Command-line Options.

如果没有使用 sysroot 前缀，那么链接器将尝试在包含链接器脚本的目录中打开文件。如果没有找到，链接器将在当前目录中搜索。如果仍然没有找到，链接器将通过归档库搜索路径进行搜索。

> If a sysroot prefix is not used then the linker will try to open the file in the directory containing the linker script. If it is not found the linker will then search the current directory. If it is still not found the linker will search through the archive library search path.

如果你使用 INPUT (*-lfile*)，ld 将把这个名字转化为 lib*file*.a，就像命令行参数 `-l` 一样。

> If you use ‘INPUT (-lfile)’, ld will transform the name to libfile.a, as with the command-line argument ‘-l’.

当你在隐式链接器脚本中使用 INPUT 命令时，文件将在链接中包含链接器脚本文件的位置被包含。这可能会影响到存档搜索。

> When you use the INPUT command in an implicit linker script, the files will be included in the link at the point at which the linker script file is included. This can affect archive searching.

#### GROUP(*file*, *file*, ...), GROUP(*file* *file* ...)

GROUP 命令与 INPUT 类似，只是命名的文件都应该是归档文件，它们被反复搜索，直到没有新的未定义的引用产生。参见命令行选项中对 `-(` 的描述。

> The GROUP command is like INPUT, except that the named files should all be archives, and they are searched repeatedly until no new undefined references are created. See the description of ‘-(’ in Command-line Options.

#### AS_NEEDED(*file*, *file*, ...), AS_NEEDED(*file* *file* ...)

这个结构只能出现在 INPUT 或 GROUP 命令中，在其他文件名中。列出的文件将被处理，就像它们直接出现在 INPUT 或 GROUP 命令中一样，但 ELF 共享库除外，它们只有在实际需要时才被添加。这个结构本质上是为它里面列出的所有文件启用 `--as-need` 选项，并在之后恢复先前的 `--as-need` 或 `--no-as-need` 设置。

> This construct can appear only inside of the INPUT or GROUP commands, among other filenames. The files listed will be handled as if they appear directly in the INPUT or GROUP commands, with the exception of ELF shared libraries, that will be added only when they are actually needed. This construct essentially enables --as-needed option for all the files listed inside of it and restores previous --as-needed resp. --no-as-needed setting afterwards.

#### OUTPUT(*filename*)

OUTPUT 命令命名输出文件。在链接器脚本中使用 OUTPUT(*filename*) 与在命令行中使用 `-o filename` 完全一样（见[命令行选项](https://sourceware.org/binutils/docs/ld/Options.html)）。如果两者都使用，则以命令行选项为准。

> The OUTPUT command names the output file. Using OUTPUT(filename) in the linker script is exactly like using ‘-o filename’ on the command line (see Command Line Options). If both are used, the command-line option takes precedence.

你可以使用 OUTPUT 命令为输出文件定义一个默认的名字，而不是通常默认的 a.out。

#### SEARCH_DIR(*path*)

SEARCH_DIR 命令将路径添加到 ld 寻找存档库的路径列表中。使用 SEARCH_DIR(*path*) 就像在命令行中使用 `-L path` 一样（见[命令行选项](https://sourceware.org/binutils/docs/ld/Options.html)）。如果两者都使用，那么链接器将搜索两个路径。使用命令行选项指定的路径将首先被搜索。

> The SEARCH_DIR command adds path to the list of paths where ld looks for archive libraries. Using SEARCH_DIR(path) is exactly like using ‘-L path’ on the command line (see Command-line Options). If both are used, then the linker will search both paths. Paths specified using the command-line option are searched first.

#### STARTUP(*filename*)

STARTUP 命令与 INPUT 命令一样，只是文件名将成为第一个被链接的输入文件，就像它在命令行中被首先指定一样。这在使用系统时可能很有用，因为系统的入口点总是第一个文件的开始。

> The STARTUP command is just like the INPUT command, except that filename will become the first input file to be linked, as though it were specified first on the command line. This may be useful when using a system in which the entry point is always the start of the first file.

### 3.4.3 处理对象文件格式的命令

> 3.4.3 Commands Dealing with Object File Formats

有几个链接器脚本命令是处理对象文件格式的。

> A couple of linker script commands deal with object file formats.

#### OUTPUT_FORMAT(*bfdname*), OUTPUT_FORMAT(*default*, *big*, *little*)

OUTPUT_FORMAT 命令命名了用于输出文件的 BFD 格式（参见 [BFD](https://sourceware.org/binutils/docs/ld/BFD.html)）。使用 OUTPUT_FORMAT(*bfdname*) 与在命令行上使用 `--oformat bfdname` 完全一样（见[命令行选项](https://sourceware.org/binutils/docs/ld/Options.html)）。如果两者都使用，则以命令行选项为准。

> The OUTPUT_FORMAT command names the BFD format to use for the output file (see BFD). Using OUTPUT_FORMAT(bfdname) is exactly like using ‘--oformat bfdname’ on the command line (see Command-line Options). If both are used, the command line option takes precedence.

你可以使用带有三个参数的 OUTPUT_FORMAT 来使用基于 `-EB` 和 `-EL` 命令行选项的不同格式。这允许链接器脚本根据需要的字节数来设置输出格式。

> You can use OUTPUT_FORMAT with three arguments to use different formats based on the ‘-EB’ and ‘-EL’ command-line options. This permits the linker script to set the output format based on the desired endianness.

如果既没有使用 `-EB` 也没有使用 `-EL`，那么输出格式将是第一个参数，这是默认的。如果使用了 `-EB`，输出格式将是第二个参数，大。如果使用了 `-EL`，那么输出格式将是第三个参数，小。

> If neither ‘-EB’ nor ‘-EL’ are used, then the output format will be the first argument, default. If ‘-EB’ is used, the output format will be the second argument, big. If ‘-EL’ is used, the output format will be the third argument, little.

例如，MIPS ELF 目标的默认链接器脚本使用这个命令：

> For example, the default linker script for the MIPS ELF target uses this command:

```ld
OUTPUT_FORMAT(elf32-bigmips, elf32-bigmips, elf32-littlemips)
```

这表示输出文件的默认格式是 `elf32-bigmips`，但如果用户使用 `-EL` 命令行选项，输出文件将以 `elf32-littlemips` 格式创建。

> This says that the default format for the output file is ‘elf32-bigmips’, but if the user uses the ‘-EL’ command-line option, the output file will be created in the ‘elf32-littlemips’ format.

#### TARGET(*bfdname*)

TARGET 命令命名了读取输入文件时要使用的 BFD 格式。它影响到后续的 INPUT 和 GROUP 命令。这个命令就像在命令行上使用 `-b bfdname` 一样（见[命令行选项](https://sourceware.org/binutils/docs/ld/Options.html)）。如果使用了 TARGET 命令，但没有使用 OUTPUT_FORMAT，那么最后一条 TARGET 命令也会用来设置输出文件的格式。参见 [BFD](https://sourceware.org/binutils/docs/ld/BFD.html)。

> The TARGET command names the BFD format to use when reading input files. It affects subsequent INPUT and GROUP commands. This command is like using ‘-b bfdname’ on the command line (see Command-line Options). If the TARGET command is used but OUTPUT_FORMAT is not, then the last TARGET command is also used to set the format for the output file. See BFD.

### 3.4.4 为内存区域指定别名

> 3.4.4 Assign alias names to memory regions

可以向 [MEMORY 命令](#37-memory-命令) 创建的内存区域添加别名。每个名字最多对应一个内存区域。

> Alias names can be added to existing memory regions created with the MEMORY Command command. Each name corresponds to at most one memory region.

#### REGION_ALIAS(*alias*, *region*)

REGION_ALIAS 函数为内存区域 *region* 创建一个别名 *alias*。这允许灵活地将输出节映射到内存区域。下面是一个例子。

> The REGION_ALIAS function creates an alias name alias for the memory region region. This allows a flexible mapping of output sections to memory regions. An example follows.

假设我们有一个应用，适用于带有各种内存外存设备的嵌入式系统。所有的（嵌入式系统）都有一个通用的、易失性的内存 RAM，允许代码执行或数据存储。有些可能有一个只读的、非易失性内存 ROM，允许代码执行和只读数据访问。最后一种变体是只读、非易失性存储器 ROM2，支持只读数据访问但无代码执行能力。我们有四个输出节：

> Suppose we have an application for embedded systems which come with various memory storage devices. All have a general purpose, volatile memory RAM that allows code execution or data storage. Some may have a read-only, non-volatile memory ROM that allows code execution and read-only data access. The last variant is a read-only, non-volatile memory ROM2 with read-only data access and no code execution capability. We have four output sections:

- *.text* 程序代码；
- *.rodata* 只读数据；
- *.data* 读写已初始化数据；
- *.bss* 读写零初始化数据。

> - .text program code;
> - .rodata read-only data;
> - .data read-write initialized data;
> - .bss read-write zero initialized data.

我们的目标是提供一个链接器命令文件，其中包含一个定义输出节的系统无关部分和一个特定于系统的部分，将输出节映射到系统上可用的内存区域。我们的嵌入式系统有三种不同的内存设置 A、B 和 C。

> The goal is to provide a linker command file that contains a system independent part defining the output sections and a system dependent part mapping the output sections to the memory regions available on the system. Our embedded systems come with three different memory setups A, B and C:

| 节    | 变体 A | 变体 B  | 变体 C
|---------|-----|---------|-
| .text   | RAM | ROM     |  ROM
| .rodata | RAM | ROM     |  ROM2
| .data   | RAM | RAM/ROM |  RAM/ROM2
| .bss    | RAM | RAM     |  RAM

RAM/ROM 或 RAM/ROM2 的写法表示该节被分别加载到 ROM 或 ROM2 区域。请注意，*.data* 节的加载地址在所有三个变体中都是从 *.rodata* 节的末尾开始的。

> The notation RAM/ROM or RAM/ROM2 means that this section is loaded into region ROM or ROM2 respectively. Please note that the load address of the .data section starts in all three variants at the end of the .rodata section.

处理输出节的基本链接器脚本如下。它包含了特定于系统的 linkcmds.memory 文件，描述了内存布局。

> The base linker script that deals with the output sections follows. It includes the system dependent linkcmds.memory file that describes the memory layout:

```ld
INCLUDE linkcmds.memory

SECTIONS
  {
    .text :
      {
        *(.text)
      } > REGION_TEXT
    .rodata :
      {
        *(.rodata)
        rodata_end = .;
      } > REGION_RODATA
    .data : AT (rodata_end)
      {
        data_start = .;
        *(.data)
      } > REGION_DATA
    data_size = SIZEOF(.data);
    data_load_start = LOADADDR(.data);
    .bss :
      {
        *(.bss)
      } > REGION_BSS
  }
```

现在我们需要三个不同的 linkcmds.memory 文件来定义内存区域和别名。三个变体 A、B 和 C 的 linkcmds.memory 的内容：

> Now we need three different linkcmds.memory files to define memory regions and alias names. The content of linkcmds.memory for the three variants A, B and C:

- A

  这里所有的东西都进入 RAM。

  > Here everything goes into the RAM.

  ```ld
  MEMORY
    {
      RAM : ORIGIN = 0, LENGTH = 4M
    }

  REGION_ALIAS("REGION_TEXT", RAM);
  REGION_ALIAS("REGION_RODATA", RAM);
  REGION_ALIAS("REGION_DATA", RAM);
  REGION_ALIAS("REGION_BSS", RAM);
  ```

- B

  程序代码和只读数据进入 ROM。读写数据进入 RAM。初始化数据的镜像被加载到 ROM 中，并在系统启动时被复制到 RAM 中。

  > Program code and read-only data go into the ROM. Read-write data goes into the RAM. An image of the initialized data is loaded into the ROM and will be copied during system start into the RAM.

  ```ld
  MEMORY
    {
      ROM : ORIGIN = 0, LENGTH = 3M
      RAM : ORIGIN = 0x10000000, LENGTH = 1M
    }

  REGION_ALIAS("REGION_TEXT", ROM);
  REGION_ALIAS("REGION_RODATA", ROM);
  REGION_ALIAS("REGION_DATA", RAM);
  REGION_ALIAS("REGION_BSS", RAM);
  ```

- C

  程序代码进入 ROM 中。只读数据进入 ROM2。读写数据进入 RAM。初始化数据的镜像被加载到 ROM2 中，并在系统启动时被复制到 RAM 中。

  > Program code goes into the ROM. Read-only data goes into the ROM2. Read-write data goes into the RAM. An image of the initialized data is loaded into the ROM2 and will be copied during system start into the RAM.

  ```ld
  MEMORY
    {
      ROM : ORIGIN = 0, LENGTH = 2M
      ROM2 : ORIGIN = 0x10000000, LENGTH = 1M
      RAM : ORIGIN = 0x20000000, LENGTH = 1M
    }

  REGION_ALIAS("REGION_TEXT", ROM);
  REGION_ALIAS("REGION_RODATA", ROM2);
  REGION_ALIAS("REGION_DATA", RAM);
  REGION_ALIAS("REGION_BSS", RAM);
  ```

可以写一个通用的系统初始化例程，在必要时将 *.data* 节从 ROM 或 ROM2 复制到 RAM 中：

> It is possible to write a common system initialization routine to copy the .data section from ROM or ROM2 into the RAM if necessary:

```c
# include <string.h>

extern char data_start [];
extern char data_size [];
extern char data_load_start [];

void copy_data(void)
{
  if (data_start != data_load_start)
    {
      memcpy(data_start, data_load_start, (size_t) data_size);
    }
}
```

### 3.4.5 其他链接器脚本命令

> 3.4.5 Other Linker Script Commands

还有一些其他的链接器脚本命令。

> There are a few other linker scripts commands.

#### ASSERT(*exp*, *message*)

确保 *exp* 不为零。如果为零，则以一个错误代码退出链接器，并打印 *message*。

> Ensure that exp is non-zero. If it is zero, then exit the linker with an error code, and print message.

注意，断言在链接的最后阶段发生之前被检查。这意味着用户必须为节定义内标记 PROVIDE 的符号设置值，否则涉及这些符号的表达式将会失败。这条规则的唯一例外是只引用点号的 PROVIDE 符号。因此，像这样的断言：

> Note that assertions are checked before the final stages of linking take place. This means that expressions involving symbols PROVIDEd inside section definitions will fail if the user has not set values for those symbols. The only exception to this rule is PROVIDEd symbols that just reference dot. Thus an assertion like this:

```ld
.stack :
{
  PROVIDE (__stack = .);
  PROVIDE (__stack_size = 0x100);
  ASSERT ((__stack > (_end + __stack_size)), "Error: No room left for the stack");
}
```

将会失败，只要 `__stack_size` 没有在其他地方定义。在节定义之外的符号更早求值，所以它们可以在 ASSERT 中使用。因此：

> will fail if __stack_size is not defined elsewhere. Symbols PROVIDEd outside of section definitions are evaluated earlier, so they can be used inside ASSERTions. Thus:

```ld
PROVIDE (__stack_size = 0x100);
.stack :
{
  PROVIDE (__stack = .)。
  ASSERT ((__stack > (_end + __stack_size)), "Error: No room left for the stack"）。
}
```

则会成功。

#### EXTERN(*symbol* *symbol* ...)

强制将符号作为一个未定义的符号输入到输出文件中。这样做可能会，例如，触发连接标准库中的额外模块。你可以为每个EXTERN列出几个符号，而且你可以多次使用EXTERN。这个命令与‘-u’命令行选项的效果相同。

> Force symbol to be entered in the output file as an undefined symbol. Doing this may, for example, trigger linking of additional modules from standard libraries. You may list several symbols for each EXTERN, and you may use EXTERN multiple times. This command has the same effect as the ‘-u’ command-line option.

#### FORCE_COMMON_ALLOCATION

这条命令与 `-d` 命令行选项的效果相同：即使指定了可重定位的输出文件（`-r`），也要让 ld 将空间分配给公共符号。

> This command has the same effect as the ‘-d’ command-line option: to make ld assign space to common symbols even if a relocatable output file is specified (‘-r’).

#### INHIBIT_COMMON_ALLOCATION

这个命令与 `--no-define-common` 命令行选项的效果相同：即使是对不可重定位的输出文件，也禁止 ld 将空间分配给公共符号。

> This command has the same effect as the ‘--no-define-common’ command-line option: to make ld omit the assignment of addresses to common symbols even for a non-relocatable output file.

#### FORCE_GROUP_ALLOCATION

这个命令与 `--force-group-allocation` 命令行选项的效果相同：使 ld 像放置正常的输入节一样放置节组成员，并删除节组，即使指定了可重定位的输出文件（`-r`）。

> This command has the same effect as the ‘--force-group-allocation’ command-line option: to make ld place section group members like normal input sections, and to delete the section groups even if a relocatable output file is specified (‘-r’).

#### INSERT [ AFTER | BEFORE ] *output_section*

这条命令通常用在由 `-T` 指定的脚本中，以增加默认的 SECTIONS，例如，叠加。它在 *output_section* 之后（或之前）插入所有先前的链接器脚本语句，并使 `-T` 不覆盖默认的链接器脚本。确切的插入点与孤岛节相同。参见位置计数器。插入发生在链接器将输入节映射到输出节之后。在插入之前，由于 `-T` 脚本是在默认链接器脚本之前解析的，因此在脚本的内部链接器表示中，`-T` 脚本中的语句出现在默认链接器脚本的语句之前。特别是，输入节将先分配给 `-T` 输出节，后分配给默认脚本。下面是一个使用 INSERT 的 `-T` 脚本的例子：

> This command is typically used in a script specified by ‘-T’ to augment the default SECTIONS with, for example, overlays. It inserts all prior linker script statements after (or before) output_section, and also causes ‘-T’ to not override the default linker script. The exact insertion point is as for orphan sections. See The Location Counter. The insertion happens after the linker has mapped input sections to output sections. Prior to the insertion, since ‘-T’ scripts are parsed before the default linker script, statements in the ‘-T’ script occur before the default linker script statements in the internal linker representation of the script. In particular, input section assignments will be made to ‘-T’ output sections before those in the default script. Here is an example of how a ‘-T’ script using INSERT might look:

```ld
SECTIONS
{
  OVERLAY :
  {
    .ov1 { ov1*(.text) }
    .ov2 { ov2*(.text) }
  }
}
INSERT AFTER .text;
```

#### NOCROSSREFS(*section* *section* ...)

这个命令可以用来告诉 ld 对特定输出节中的任何引用报错。

> This command may be used to tell ld to issue an error about any references among certain output sections.

在某些类型的程序中，特别是在嵌入式系统中使用覆盖时，当一个节被加载到内存中时，另一个节一定不会被加载。这两个节之间的任何直接引用都是错误的。例如，如果一个节的代码调用了另一个节定义的函数，将产生一个错误。

> In certain types of programs, particularly on embedded systems when using overlays, when one section is loaded into memory, another section will not be. Any direct references between the two sections would be errors. For example, it would be an error if code in one section called a function defined in the other section.

NOCROSSREFS 命令接收一个输出节名称的列表。如果 ld 检测到这些节之间有任何交叉引用，它会报告一个错误并返回一个非零的退出状态。注意，NOCROSSREFS 命令使用的是输出节的名称，而不是输入节的名称。

> The NOCROSSREFS command takes a list of output section names. If ld detects any cross references between the sections, it reports an error and returns a non-zero exit status. Note that the NOCROSSREFS command uses output section names, not input section names.

#### NOCROSSREFS_TO(*tosection* *fromsection* ...)

这条命令可以用来告诉 ld，从节列表中的某个节发现对指定节的任何引用时产生一个错误。

> This command may be used to tell ld to issue an error about any references to one section from a list of other sections.

NOCROSSREFS 命令在确保两个或以上输出节完全独立时非常有用，但也有需要单向依赖的情况。例如，在一个多核应用程序中，可能有一些共享代码可以从每个核中调用，但为了安全起见，决不能回调。

> The NOCROSSREFS command is useful when ensuring that two or more output sections are entirely independent but there are situations where a one-way dependency is needed. For example, in a multi-core application there may be shared code that can be called from each core but for safety must never call back.

NOCROSSREFS_TO 命令需要一个输出节的名称列表。第一个节不能被其他任何节引用。如果 ld 检测到任何其他节对第一个节的引用，它会报告一个错误并返回一个非零的退出状态。注意，NOCROSSREFS_TO 命令使用的是输出节的名字，而不是输入节的名字。

> The NOCROSSREFS_TO command takes a list of output section names. The first section can not be referenced from any of the other sections. If ld detects any references to the first section from any of the other sections, it reports an error and returns a non-zero exit status. Note that the NOCROSSREFS_TO command uses output section names, not input section names.

#### OUTPUT_ARCH(*bfdarch*)

指定一个特定的输出机架构。参数是 BFD 库使用的名称之一（参见 [BFD](https://sourceware.org/binutils/docs/ld/BFD.html)）。你可以通过使用带有 `-f` 选项的 objdump 程序来查看一个对象文件的架构。

> Specify a particular output machine architecture. The argument is one of the names used by the BFD library (see BFD). You can see the architecture of an object file by using the objdump program with the ‘-f’ option.

#### LD_FEATURE(*string*)

这个命令可以用来修改 ld 的行为。如果字符串是 `SANE_EXPR`，那么脚本中的绝对符号和数字就会被当作数字处理。参见[表达式的节](#3108-表达式的节)。

> This command may be used to modify ld behavior. If string is "SANE_EXPR" then absolute symbols and numbers in a script are simply treated as numbers everywhere. See The Section of an Expression.

## 3.5 为符号赋值

> 3.5 Assigning Values to Symbols

你可以在链接器脚本中为一个符号赋值。这将定义该符号并将其放入具有全局作用域的符号表。

> You may assign a value to a symbol in a linker script. This will define the symbol and place it into the symbol table with a global scope.

- [简单赋值](#351-简单赋值)
- [HIDDEN](#352-hidden)
- [PROVIDE](#353-provide)
- [PROVIDE_HIDDEN](#354-provide_hidden)
- [源码引用](#355-源码引用)

> - Simple Assignments
> - HIDDEN
> - PROVIDE
> - PROVIDE_HIDDEN
> - Source Code Reference

### 3.5.1 简单赋值

> 3.5.1 Simple Assignments

你可以使用 C 语言中的任何一个赋值运算符对一个符号进行赋值。

> You may assign to a symbol using any of the C assignment operators:

```c
symbol = expression;
symbol += expression;
symbol -= expression;
symbol *= expression;
symbol /= expression;
symbol <<= expression;
symbol >>= expression;
symbol &= expression;
symbol |= expression;
```

第行将把符号定义为表达式的值。在其他情况下，符号必须已经被定义，并且其值将被相应调整。

> The first case will define symbol to the value of expression. In the other cases, symbol must already be defined, and the value will be adjusted accordingly.

特殊符号名称 `.` 表示位置计数器。你只能在 SECTIONS 命令中使用它。请参阅[位置计数器](#3105-位置计数器)。

> The special symbol name ‘.’ indicates the location counter. You may only use this within a SECTIONS command. See The Location Counter.

表达式后面的分号是必须的。

> The semicolon after expression is required.

表达式的定义在下面，请看[链接器脚本中的表达式](#310-链接器脚本中的表达式)。

> Expressions are defined below; see Expressions in Linker Scripts.

你可以把符号赋值作为命令本身来写，也可以作为 SECTIONS 命令中的语句来写，或者作为 SECTIONS 命令中输出节描述的一部分。

> You may write symbol assignments as commands in their own right, or as statements within a SECTIONS command, or as part of an output section description in a SECTIONS command.

符号的节将根据表达式的节来设置，更多信息请参见[表达式的节](#3108-表达式的节)。

> The section of the symbol will be set from the section of the expression; for more information, see The Section of an Expression.

这是一个例子，显示了符号赋值可能使用的三个不同地方：

> Here is an example showing the three different places that symbol assignments may be used:

```ld
floating_point = 0;
SECTIONS
{
  .text :
    {
      *(.text)
      _etext = .;
    }
  _bdata = (. + 3) & ~ 3;
  .data : {*(.data) }
}
```

在这个例子中，符号 `floating_point` 将被定义为零。符号 `_etext` 将被定义为最后一个 *.text* 输入节之后的地址。符号 `_bdata` 将被定义为 *.text* 输出节之后的地址，向上对齐到 4 字节的边界。

> In this example, the symbol ‘floating_point’ will be defined as zero. The symbol ‘_etext’ will be defined as the address following the last ‘.text’ input section. The symbol ‘_bdata’ will be defined as the address following the ‘.text’ output section aligned upward to a 4 byte boundary.

### 3.5.2 HIDDEN

对于以 ELF 为目标的 port 来说，定义一个将被隐藏而不会被导出的符号。其语法是 HIDDEN(*symbol* = *expression*)。

> For ELF targeted ports, define a symbol that will be hidden and won’t be exported. The syntax is HIDDEN(symbol = expression).

下面是来自[简单赋值](#351-简单赋值)的例子， 使用 HIDDEN 改写：

> Here is the example from Simple Assignments, rewritten to use HIDDEN:

```ld
HIDDEN(floating_point = 0);
SECTIONS
{
  .text :
    {
      *(.text)
      HIDDEN(_etext = .);
    }
  HIDDEN(_bdata = (. + 3) & ~ 3);
.data : {*(.data) }
}
```

在这种情况下，这三个符号在本模块之外都不可见。

> In this case none of the three symbols will be visible outside this module.

### 3.5.3 PROVIDE

在某些情况下，链接器脚本最好能做到这样的事：当且仅当一个符号被引用，并且没有被链接中包含的任何对象定义过，才定义这个符号。例如，传统的链接器定义了符号 `etext`。然而，ANSI C 要求用户能够使用 `etext` 作为函数名而不会遇到错误。PROVIDE 关键字可以用来有条件地定义一个符号，比如 `etext` ，当且仅当它被引用但没有被定义的时候。语法是 PROVIDE(*symbol* = *expression*)。

> In some cases, it is desirable for a linker script to define a symbol only if it is referenced and is not defined by any object included in the link. For example, traditional linkers defined the symbol ‘etext’. However, ANSI C requires that the user be able to use ‘etext’ as a function name without encountering an error. The PROVIDE keyword may be used to define a symbol, such as ‘etext’, only if it is referenced but not defined. The syntax is PROVIDE(symbol = expression).

下面是一个使用 PROVIDE 来定义 `etext` 的例子：

> Here is an example of using PROVIDE to define ‘etext’:

```ld
SECTIONS
{
  .text :
    {
      *(.text)
      _etext = .;
      PROVIDE(etext = .);
    }
}
```

在这个例子中，如果程序定义了 `_etext`（有一个前导下划线），链接器将给出一个重复定义的错误信息。另一方面，如果程序定义了 `etext`（没有前导下划线），链接器将默默地使用程序中的定义。如果程序引用了 `etext` 但没有定义它，链接器将使用链接器脚本中的定义。

> In this example, if the program defines ‘_etext’ (with a leading underscore), the linker will give a multiple definition diagnostic. If, on the other hand, the program defines ‘etext’ (with no leading underscore), the linker will silently use the definition in the program. If the program references ‘etext’ but does not define it, the linker will use the definition in the linker script.

注意 - PROVIDE 直接认为所有公共符号是已定义的，即使这样的符号可以与 PROVIDE 将创建的符号相结合。在考虑构造函数和析构函数列表符号时，这一点特别重要，例如 `__CTOR_LIST__`，因为这些符号经常被定义为公共符号。

> Note - the PROVIDE directive considers a common symbol to be defined, even though such a symbol could be combined with the symbol that the PROVIDE would create. This is particularly important when considering constructor and destructor list symbols such as ‘__CTOR_LIST__’ as these are often defined as common symbols.

### 3.5.4 PROVIDE_HIDDEN

类似于 PROVIDE。对于以 ELF 为目标的 port，这个符号会被隐藏，不会导出。

Similar to PROVIDE. For ELF targeted ports, the symbol will be hidden and won’t be exported.

### 3.5.5 源码引用

> 3.5.5 Source Code Reference

从源码中访问链接器脚本定义的变量是不直观的。特别是链接器脚本符号并不等同于高级语言中的变量声明，而是一个没有值的符号。

> Accessing a linker script defined variable from source code is not intuitive. In particular a linker script symbol is not equivalent to a variable declaration in a high level language, it is instead a symbol that does not have a value.

在进一步讨论之前，需要注意的是，编译器将源码中的名称存储到符号表中时，往往会将它们转化为不同的名称。例如，Fortran 编译器通常会前缀或后缀一个下划线，而 C++ 则会进行广泛的 *name mangling*。因此，在源码中使用的变量名和链接器脚本中定义的同一变量的名称之间可能存在差异。例如，在 C 语言中，一个链接器脚本变量可能被称为：

> Before going further, it is important to note that compilers often transform names in the source code into different names when they are stored in the symbol table. For example, Fortran compilers commonly prepend or append an underscore, and C++ performs extensive ‘name mangling’. Therefore there might be a discrepancy between the name of a variable as it is used in source code and the name of the same variable as it is defined in a linker script. For example in C a linker script variable might be referred to as:

```c
extern int foo;
```

但在链接器脚本中，它可能被定义为：

> But in the linker script it might be defined as:

```ld
_foo = 1000;
```

然而，在后续的例子中，我们假设没有发生名称转换的情况。

> In the remaining examples however it is assumed that no name transformation has taken place.

当高级语言（如 C 语言）声明一个符号时，会发生两件事。第一件事是，编译器在程序的内存中保留了足够的空间来容纳该符号的值。第二件事是，编译器在程序的符号表中创建一个条目，持有该符号的地址，即符号表包含持有该符号值的内存块的地址。因此，例如以下的 C 语言声明，在文件作用域中：

> When a symbol is declared in a high level language such as C, two things happen. The first is that the compiler reserves enough space in the program’s memory to hold the value of the symbol. The second is that the compiler creates an entry in the program’s symbol table which holds the symbol’s address. ie the symbol table contains the address of the block of memory holding the symbol’s value. So for example the following C declaration, at file scope:

```c
int foo = 1000;
```

在符号表中创建一个名为 `foo` 的条目。这个条目保存一个 `int` 大小的内存块的地址，数字 1000 最初被储存在这个内存块中。

> creates an entry called ‘foo’ in the symbol table. This entry holds the address of an ‘int’ sized block of memory where the number 1000 is initially stored.

当程序引用一个符号时，编译器会生成代码，首先访问符号表，找到符号的内存块地址，然后从该内存块中读取数值。所以：

When a program references a symbol the compiler generates code that first accesses the symbol table to find the address of the symbol’s memory block and then code to read the value from that memory block. So:

```c
foo = 1;
```

在符号表中查找符号 `foo`，获得与该符号相关的地址，然后将值 1 写入该地址。而：

> looks up the symbol ‘foo’ in the symbol table, gets the address associated with this symbol and then writes the value 1 into that address. Whereas:

```c
int * a = & foo;
```

在符号表中查找符号 `foo`，得到它的地址，然后把这个地址复制到与变量 `a` 相关的内存块中。

> looks up the symbol ‘foo’ in the symbol table, gets its address and then copies this address into the block of memory associated with the variable ‘a’.

相比之下，链接器脚本的符号声明在符号表中创建了一个条目，但没有给它们分配任何内存。因此，它们是一个没有值的地址。因此，举例来说，链接器脚本定义：

> Linker scripts symbol declarations, by contrast, create an entry in the symbol table but do not assign any memory to them. Thus they are an address without a value. So for example the linker script definition:

```ld
foo = 1000;
```

在符号表中创建了一个名为 `foo` 的条目，保存内存位置 1000 的地址，但在地址 1000 处没有存储任何特别的东西。这意味着你不能访问链接器脚本定义的符号的值--它没有值--你所能做的就是访问链接器脚本定义的符号的地址。

> creates an entry in the symbol table called ‘foo’ which holds the address of memory location 1000, but nothing special is stored at address 1000. This means that you cannot access the value of a linker script defined symbol - it has no value - all you can do is access the address of a linker script defined symbol.

因此，当你在源码中使用一个链接器脚本定义的符号时，你应该始终使用该符号的地址，而不要试图使用它的值。例如，假设你想把一段名为 `.ROM` 的内存内容复制到一段名为 `.FLASH` 的内存中，而链接器脚本包含这些声明：

> Hence when you are using a linker script defined symbol in source code you should always take the address of the symbol, and never attempt to use its value. For example suppose you want to copy the contents of a section of memory called .ROM into a section called .FLASH and the linker script contains these declarations:

```ld
start_of_ROM   = .ROM;
end_of_ROM     = .ROM + sizeof (.ROM);
start_of_FLASH = .FLASH;
```

则执行复制的 C 源代码应该是：

> Then the C source code to perform the copy would be:

```c
extern char start_of_ROM, end_of_ROM, start_of_FLASH;

memcpy (& start_of_FLASH, & start_of_ROM, & end_of_ROM - & start_of_ROM);
```

注意 `&` 运算符的使用。这些都是正确的。另外，这些符号可以被视为向量或数组的名称，然后代码将再次按预期工作：

> Note the use of the ‘&’ operators. These are correct. Alternatively the symbols can be treated as the names of vectors or arrays and then the code will again work as expected:

```c
extern char start_of_ROM[], end_of_ROM[], start_of_FLASH[];

memcpy (start_of_FLASH, start_of_ROM, end_of_ROM - start_of_ROM);
```

注意使用这种方法不需要使用 `&` 运算符。

> Note how using this method does not require the use of ‘&’ operators.

## 3.6 SECTIONS 命令

> 3.6 SECTIONS Command

SECTIONS 命令告诉链接器如何将输入节映射到输出部分，以及如何将输出节放在内存中。

> The SECTIONS command tells the linker how to map input sections into output sections, and how to place the output sections in memory.

SECTIONS 命令的格式是：

> The format of the SECTIONS command is:

```ld
SECTIONS
{
  sections-command
  sections-command
  ...
}
```

每个 *section-command* 可以是下列之一：

> Each sections-command may of be one of the following:

- 一个 ENTRY 命令（见 [ENTRY 命令](#341-设置入口点)）
- 一个符号赋值（见为[为符号赋值](#35-为符号赋值)）。
- 一个输出节的描述
- 一个覆盖描述

> - an ENTRY command (see Entry command)
> - a symbol assignment (see Assigning Values to Symbols)
> - an output section description
> - an overlay description

为了方便在这些命令中使用位置计数器，ENTRY 命令和符号赋值被允许放在 SECTIONS 命令中。这也可以使链接器脚本更容易理解，因为你可以在输出文件的布局中有意义的点上使用这些命令。

> The ENTRY command and symbol assignments are permitted inside the SECTIONS command for convenience in using the location counter in those commands. This can also make the linker script easier to understand because you can use those commands at meaningful points in the layout of the output file.

输出节的描述和覆盖节的描述将在下面描述。

> Output section descriptions and overlay descriptions are described below.

如果你在链接器脚本中没有使用 SECTIONS 命令，链接器将按照输入文件中的节第一次遇到的顺序，把每个输入节放入一个相同名称的输出节。例如，如果所有的输入节都出现在第一个文件中，那么输出文件中各节的顺序将与第一个输入文件中的顺序一致。第一个节将在地址 0 处。

> If you do not use a SECTIONS command in your linker script, the linker will place each input section into an identically named output section in the order that the sections are first encountered in the input files. If all input sections are present in the first file, for example, the order of sections in the output file will match the order in the first input file. The first section will be at address zero.

- [输出节描述](#361-输出节描述)
- [输出节名称](#362-输出节名称)
- [输出节地址](#363-输出节地址)
- [输入节描述](#364-输入节描述)
- [输出节数据](#365-输出节数据)
- [输出节关键字](#366-输出节关键字)
- [输出节丢弃](#367-输出节丢弃)
- [输出节属性](#368-输出节属性)
- [覆盖描述](#369-覆盖描述)

> - Output Section Description
> - Output Section Name
> - Output Section Address
> - Input Section Description
> - Output Section Data
> - Output Section Keywords
> - Output Section Discarding
> - Output Section Attributes
> - Overlay Description

### 3.6.1 输出节描述

> 3.6.1 Output Section Description

一个输出节的完整描述看起来像这样：

> The full description of an output section looks like this:

```ld
section [address] [(type)] :
  [AT(lma)]
  [ALIGN(section_align) | ALIGN_WITH_INPUT]
  [SUBALIGN(subsection_align)]
  [constraint]
  {
    output-section-command
    output-section-command
    ...
  } [>region] [AT>lma_region] [:phdr :phdr ...] [=fillexp] [,]
```

大多数输出节不使用大多数可选的节属性。

> Most output sections do not use most of the optional section attributes.

*section* 周围的空白是必须的，这样一来，*section* 的名字就不会有歧义了。冒号和大括号也是必须的。如果使用了 *fillexp*，并且下一个 *sections-command* 看起来像这个表达式的延续，那么末尾的逗号可能是必须的。换行符和其他空白是可选的。

> The whitespace around section is required, so that the section name is unambiguous. The colon and the curly braces are also required. The comma at the end may be required if a fillexp is used and the next sections-command looks like a continuation of the expression. The line breaks and other white space are optional.

每个 *output-section-command* 可以是以下的一种：

> Each output-section-command may be one of the following:

- 符号赋值（参见[为符号赋值](#35-为符号赋值)）
- 一个输入节描述（见[输入节描述](#364-输入节描述)）
- 直接包括的数据值（见[输出节数据](#365-输出节数据)）
- 一个特殊的输出节关键字（见[输出节关键字](#366-输出节关键字)）。

> - a symbol assignment (see Assigning Values to Symbols)
> - an input section description (see Input Section Description)
> - data values to include directly (see Output Section Data)
> - a special output section keyword (see Output Section Keywords)

### 3.6.2 输出节名称

> 3.6.2 Output Section Name

输出节的名称是 *section*。*section* 必须符合你的输出格式的限制。在只支持有限数量的节的格式中，比如 a.out，名称必须是格式所支持的名称之一（比如a.out，只允许 *.text*、*.data* 或 *.bss*）。如果输出格式支持任何数量的节，但使用数字而不是名称（如 Oasys），那么名称应该以带引号的数字字符串提供。一个节的名称可以由任何字符序列组成，但包含任何不寻常字符的名称，如逗号，必须加引号。

> The name of the output section is section. section must meet the constraints of your output format. In formats which only support a limited number of sections, such as a.out, the name must be one of the names supported by the format (a.out, for example, allows only ‘.text’, ‘.data’ or ‘.bss’). If the output format supports any number of sections, but with numbers and not names (as is the case for Oasys), the name should be supplied as a quoted numeric string. A section name may consist of any sequence of characters, but a name which contains any unusual characters such as commas must be quoted.

输出节的名称 */DISCARD/* 是特殊的；（表示）输出节丢弃。

> The output section name ‘/DISCARD/’ is special; Output Section Discarding.

### 3.6.3 输出节地址

> 3.6.3 Output Section Address

该地址是输出节的 VMA（虚拟内存地址）的表达式。这个地址是可选的，但是如果它存在，那么输出地址将被精确地设置为指定的地址。

> The address is an expression for the VMA (the virtual memory address) of the output section. This address is optional, but if it is provided then the output address will be set exactly as specified.

如果没有指定输出地址，那么将根据下面的启发式方法，为该部分选择一个地址。这个地址将被调整以适应输出节的对齐要求。对齐要求是指输出节所包含的任何输入节的最严格对齐。

> If the output address is not specified then one will be chosen for the section, based on the heuristic below. This address will be adjusted to fit the alignment requirement of the output section. The alignment requirement is the strictest alignment of any input section contained within the output section.

输出节的地址启发法如下：

> The output section address heuristic is as follows:

如果为该节设置了一个输出内存区域，那么它将被添加到该区域，其地址将是该区域的下一个空闲地址。

> If an output memory region is set for the section then it is added to this region and its address will be the next free address in that region.

如果 MEMORY 命令被用来创建一个内存区域的列表，那么会选择第一个与节的属性兼容的区域来包含它。该节的输出地址将是该区域的下一个空闲地址；[MEMORY 命令](#37-memory-命令)。

> If the MEMORY command has been used to create a list of memory regions then the first region which has attributes compatible with the section is selected to contain it. The section’s output address will be the next free address in that region; MEMORY Command.

如果没有指定内存区域，或者没有与该部分相匹配的内存区域，那么输出地址将基于位置计数器的当前值。

> If no memory regions were specified, or none match the section then the output address will be based on the current value of the location counter.

比如说：

> For example:

```ld
.text . : { *(.text) }
```

和

> and

```ld
.text : { *(.text) }
```

是有细微差别的。第一个将把 *.text* 输出节的地址设置为位置计数器的当前值。第二种将把它设置为位置计数器的当前值，并对齐到所有 *.text* 输入节的最严格对齐。

> are subtly different. The first will set the address of the ‘.text’ output section to the current value of the location counter. The second will set it to the current value of the location counter aligned to the strictest alignment of any of the ‘.text’ input sections.

该地址可以是一个任意的表达式；[链接器脚本中的表达式](#310-链接器脚本中的表达式)。例如，如果你想在 0x10 字节的边界上对齐该节，使该节地址的最低 4 位为 0，你可以这样做：

> The address may be an arbitrary expression; Expressions in Linker Scripts. For example, if you want to align the section on a 0x10 byte boundary, so that the lowest four bits of the section address are zero, you could do something like this:

```ld
.text ALIGN(0x10) : { *(.text) }
```

这是有效的，原因是 ALIGN 返回当前位置的计数器，并向上对齐到指定的值。

> This works because ALIGN returns the current location counter aligned upward to the specified value.

为一个节指定地址将改变位置计数器的值，只要该节不是空的（空的节会被忽略）。

> Specifying address for a section will change the value of the location counter, provided that the section is non-empty. (Empty sections are ignored).

### 3.6.4 输入节描述

> 3.6.4 Input Section Description

最常见的输出命令节是输入节描述。

> The most common output section command is an input section description.

输入节描述是最基本的链接器脚本操作。你用输出节来告诉链接器如何在内存中布局你的程序。你使用输入部分描述来告诉链接器如何将输入文件映射到你的内存布局中。

> The input section description is the most basic linker script operation. You use output sections to tell the linker how to lay out your program in memory. You use input section descriptions to tell the linker how to map the input files into your memory layout.

- [输入节基础知识](#3641-输入节基础知识)
- [输入节通配符模式](#3642-输入节通配符模式)
- [常见符号的输入节](#3643-公共符号的输入节)
- [输入节和垃圾收集](#3644-输入节和垃圾收集)
- [输入节示例](#3645-输入节示例)

> - Input Section Basics
> - Input Section Wildcard Patterns
> - Input Section for Common Symbols
> - Input Section and Garbage Collection
> - Input Section Example

#### 3.6.4.1 输入节基础知识

> 3.6.4.1 Input Section Basics

一个输入节的描述由一个文件名组成，并可选后缀一个节名列表，用括号括起。

> An input section description consists of a file name optionally followed by a list of section names in parentheses.

文件名和节名可以是通配符模式，我们将在下面进一步描述（见[输入节通配符模式](#3642-输入节通配符模式)）。

> The file name and the section name may be wildcard patterns, which we describe further below (see Input Section Wildcard Patterns).

最常见的输入节描述是在输出节包括所有具有特定名称的输入节。例如，要包括所有输入的 *.text* 节，你可以这样写：

> The most common input section description is to include all input sections with a particular name in the output section. For example, to include all input ‘.text’ sections, you would write:

```ld
*(.text)
```

这里的 `*` 是一个通配符，可以匹配任何文件名。要排除一个与文件名通配符匹配的文件列表，可以使用 EXCLUDE_FILE 来匹配除 EXCLUDE_FILE 列表中指定的文件以外的所有文件。例如：

> Here the ‘\*’ is a wildcard which matches any file name. To exclude a list of files from matching the file name wildcard, EXCLUDE_FILE may be used to match all files except the ones specified in the EXCLUDE_FILE list. For example:

```ld
EXCLUDE_FILE (*crtend.o *otherfile.o) *(.ctors)
```

将导致除 crtend.o 和 otherfile.o 之外的所有文件的 *.ctors* 节都被包括在内。EXCLUDE_FILE 也可以放在节列表里面，例如：

> will cause all .ctors sections from all files except crtend.o and otherfile.o to be included. The EXCLUDE_FILE can also be placed inside the section list, for example:

```ld
*(EXCLUDE_FILE (*crtend.o *otherfile.o) .ctors)
```

这样做的结果与前面的例子相同。如果节列表包含多个节，支持 EXCLUDE_FILE 的两种语法是很有用的，如下所述。

> The result of this is identically to the previous example. Supporting two syntaxes for EXCLUDE_FILE is useful if the section list contains more than one section, as described below.

有两种方法可以包括多于一个部分。

> There are two ways to include more than one section:

```ld
*(.text .rdata)
*(.text) *(.rdata)
```

这两者之间的区别在于 *.text* 和 *.rdata* 输入节在输出节出现的顺序。在第一个例子中，它们将被混合在一起，以它们在链接器输入中的相同顺序出现。在第二个例子中，所有 *.text* 输入节将先出现，然后是所有 *.rdata* 输入节。

> The difference between these is the order in which the ‘.text’ and ‘.rdata’ input sections will appear in the output section. In the first example, they will be intermingled, appearing in the same order as they are found in the linker input. In the second example, all ‘.text’ input sections will appear first, followed by all ‘.rdata’ input sections.

当使用 EXCLUDE_FILE 时，有一个以上的节，如果排除在节列表中，那么排除只适用于紧接着的部分，例如：

> When using EXCLUDE_FILE with more than one section, if the exclusion is within the section list then the exclusion only applies to the immediately following section, for example:

```ld
*(EXCLUDE_FILE (*somefile.o) .text .rdata)
```

将导致除 somefile.o 以外的所有文件中的所有 *.text* 部分被包括在内，而所有文件中的所有 *.rdata* 部分，包括 somefile.o，都将被包括在内。为了排除somefile.o 中的 *.rdata* 部分，这个例子可以修改为：

> will cause all ‘.text’ sections from all files except somefile.o to be included, while all ‘.rdata’ sections from all files, including somefile.o, will be included. To exclude the ‘.rdata’ sections from somefile.o the example could be modified to:

```ld
*(EXCLUDE_FILE (*somefile.o) .text EXCLUDE_FILE (*somefile.o) .rdata)
```

另外，把 EXCLUDE_FILE 放在节列表之外，在输入文件选择之前，会导致排除适用于所有节。因此，前面的例子可以改写为：

> Alternatively, placing the EXCLUDE_FILE outside of the section list, before the input file selection, will cause the exclusion to apply for all sections. Thus the previous example can be rewritten as:

```ld
EXCLUDE_FILE (*somefile.o) *(.text .rdata)
```

你可以指定一个文件名来包括某个特定文件的节。如果你的一个或多个文件包含需要在内存中某个特定位置的特殊数据，你会这样做。例如：

> You can specify a file name to include sections from a particular file. You would do this if one or more of your files contain special data that needs to be at a particular location in memory. For example:

```ld
data.o(.data)
```

为了根据输入节的节标记细化被包含的节，可以使用 INPUT_SECTION_FLAGS。

> To refine the sections that are included based on the section flags of an input section, INPUT_SECTION_FLAGS may be used.

下面是一个为 ELF 节使用节头标记的简单例子：

> Here is a simple example for using Section header flags for ELF sections:

```ld
SECTIONS {
  .text : { INPUT_SECTION_FLAGS (SHF_MERGE & SHF_STRINGS) *(.text) }
  .text2 : { INPUT_SECTION_FLAGS (!SHF_WRITE) *(.text) }
}
```

在这个例子中，输出节 *.text* 将由任何与名称 *\*(.text)* 相匹配的输入节组成，为其设置了 SHF_MERGE 和 SHF_STRINGS 节头标记。输出节 *.text2* 将由任何与名称 *\*(.text)* 相匹配的输入节组成，不带节头标记 SHF_WRITE。

> In this example, the output section ‘.text’ will be comprised of any input section matching the name \*(.text) whose section header flags SHF_MERGE and SHF_STRINGS are set. The output section ‘.text2’ will be comprised of any input section matching the name \*(.text) whose section header flag SHF_WRITE is clear.

> You can also specify files within archives by writing a pattern matching the archive, a colon, then the pattern matching the file, with no whitespace around the colon.
>
> ‘archive:file’
> matches file within archive
>
> ‘archive:’
> matches the whole archive
>
> ‘:file’
> matches file but not one in an archive
>
> Either one or both of ‘archive’ and ‘file’ can contain shell wildcards. On DOS based file systems, the linker will assume that a single letter followed by a colon is a drive specifier, so ‘c:myfile.o’ is a simple file specification, not ‘myfile.o’ within an archive called ‘c’. ‘archive:file’ filespecs may also be used within an EXCLUDE_FILE list, but may not appear in other linker script contexts. For instance, you cannot extract a file from an archive by using ‘archive:file’ in an INPUT command.
>
> If you use a file name without a list of sections, then all sections in the input file will be included in the output section. This is not commonly done, but it may by useful on occasion. For example:
>
> data.o
> When you use a file name which is not an ‘archive:file’ specifier and does not contain any wild card characters, the linker will first see if you also specified the file name on the linker command line or in an INPUT command. If you did not, the linker will attempt to open the file as an input file, as though it appeared on the command line. Note that this differs from an INPUT command, because the linker will not search for the file in the archive search path.

#### 3.6.4.2 输入节通配符模式

> 3.6.4.2 Input Section Wildcard Patterns

在一个输入节的描述中，文件名、节名或二者同时可以是通配符模式。

> In an input section description, either the file name or the section name or both may be wildcard patterns.

在许多例子中看到的 `*` 文件名是一个简单的文件名通配符模式。

> The file name of ‘\*’ seen in many examples is a simple wildcard pattern for the file name.

通配符模式和 Unix shell 使用的模式一样。

> The wildcard patterns are like those used by the Unix shell.

- `*`
  匹配任何数量的字符

  > matches any number of characters

- `?`
  匹配任何单个字符

  > matches any single character

- `[chars]`
  匹配任何一个字符的单个实例；`-` 字符可用于指定一个字符范围，如 `[a-z]` 可匹配任何小写字母

  > smatches a single instance of any of the chars; the ‘-’ character may be used to specify a range of characters, as in ‘[a-z]’ to match any lower case letter

- `\`
  引述后面的字符

  > quotes the following character

文件名通配符模式只匹配在命令行或 INPUT 命令中明确指定的文件。链接器不会搜索目录来扩展通配符。

> File name wildcard patterns only match files which are explicitly specified on the command line or in an INPUT command. The linker does not search directories to expand wildcards.

如果一个文件名与一个以上的通配符模式相匹配，或者一个文件名明确出现并且也同时被通配符模式所匹配，链接器将使用链接器脚本中的第一个匹配。例如，这个输入部分描述的序列可能是错误的，因为 data.o 规则将不会被使用：

> If a file name matches more than one wildcard pattern, or if a file name appears explicitly and is also matched by a wildcard pattern, the linker will use the first match in the linker script. For example, this sequence of input section descriptions is probably in error, because the data.o rule will not be used:

```ld
.data : { *(.data) }
.data1 : { data.o(.data) }
```

通常情况下，链接器会按照链接过程中看到的顺序放置通配符匹配的文件和节。你可以通过使用 SORT_BY_NAME 关键字来改变这一点，该关键字出现在括号中的通配符模式之前（例如，SORT_BY_NAME(.text\*)）。当使用 SORT_BY_NAME 关键字时，链接器将在把文件或节放在输出文件中之前，按名称升序排序。

> Normally, the linker will place files and sections matched by wildcards in the order in which they are seen during the link. You can change this by using the SORT_BY_NAME keyword, which appears before a wildcard pattern in parentheses (e.g., SORT_BY_NAME(.text\*)). When the SORT_BY_NAME keyword is used, the linker will sort the files or sections into ascending order by name before placing them in the output file.

SORT_BY_ALIGNMENT 与 SORT_BY_NAME 类似。SORT_BY_ALIGNMENT 将把各节按对齐降序排序，然后再把它们放在输出文件中。将较大的对齐方式放在较小的对齐方式之前，可以减少需要的填充量。

> SORT_BY_ALIGNMENT is similar to SORT_BY_NAME. SORT_BY_ALIGNMENT will sort sections into descending order of alignment before placing them in the output file. Placing larger alignments before smaller alignments can reduce the amount of padding needed.

SORT_BY_INIT_PRIORITY 也与 SORT_BY_NAME 类似。SORT_BY_INIT_PRIORITY 将把节按节名中编码的 GCC init_priority 属性的数字升序排序，然后再把它们放到输出文件中。在 .init_array.NNNN 和 .fini_array.NNNN 中，NNNN 是 init_priority。在 .ctors.NNNNN 和 .dtors.NNNNN 中，NNNNN 是 65535 减去 init_priority。

> SORT_BY_INIT_PRIORITY is also similar to SORT_BY_NAME. SORT_BY_INIT_PRIORITY will sort sections into ascending numerical order of the GCC init_priority attribute encoded in the section name before placing them in the output file. In .init_array.NNNNN and .fini_array.NNNNN, NNNNN is the init_priority. In .ctors.NNNNN and .dtors.NNNNN, NNNNN is 65535 minus the init_priority.

SORT 是 SORT_BY_NAME 的一个别名。

> SORT is an alias for SORT_BY_NAME.

当链接器脚本中存在嵌套的节排序命令时，最多可以有 1 层嵌套。

> When there are nested section sorting commands in linker script, there can be at most 1 level of nesting for section sorting commands.

1. SORT_BY_NAME (SORT_BY_ALIGNMENT (wildcard section pattern))。它将首先按名称对输入节进行排序，如果两个节有相同的名称，则按对齐排序。
2. SORT_BY_ALIGNMENT (SORT_BY_NAME (wildcard section pattern))。它将首先按对齐对输入节进行排序，如果两个部分有相同的对齐，则按名称排序。
3. SORT_BY_NAME (SORT_BY_NAME (wildcard section pattern)) 的处理方法与 SORT_BY_NAME (wildcard section pattern) 相同。
4. SORT_BY_ALIGNMENT (SORT_BY_ALIGNMENT (wildcard section pattern)) 的处理方法与 SORT_BY_ALIGNMENT (wildcard section pattern) 相同。
5. 所有其他的节排序命令嵌套都是无效的。

> 1. SORT_BY_NAME (SORT_BY_ALIGNMENT (wildcard section pattern)). It will sort the input sections by name first, then by alignment if two sections have the same name.
> 2. SORT_BY_ALIGNMENT (SORT_BY_NAME (wildcard section pattern)). It will sort the input sections by alignment first, then by name if two sections have the same alignment.
> 3. SORT_BY_NAME (SORT_BY_NAME (wildcard section pattern)) is treated the same as SORT_BY_NAME (wildcard section pattern).
> 4. SORT_BY_ALIGNMENT (SORT_BY_ALIGNMENT (wildcard section pattern)) is treated the same as SORT_BY_ALIGNMENT (wildcard section pattern).
> 5. All other nested section sorting commands are invalid.

当同时使用命令行节排序选项和链接器脚本节排序命令时，节排序命令总是优先于命令行选项。

> When both command-line section sorting option and linker script section sorting command are used, section sorting command always takes precedence over the command-line option.

如果链接器脚本中的节排序命令没有嵌套，命令行选项将使节排序命令被视为嵌套排序命令。

> If the section sorting command in linker script isn’t nested, the command-line option will make the section sorting command to be treated as nested sorting command.

1. SORT_BY_NAME (wildcard section pattern) 加上 `--sort-sections alignment` 等同于 SORT_BY_NAME (SORT_BY_ALIGNMENT (wildcard section pattern))。
2. SORT_BY_ALIGNMENT (wildcard section pattern) 加上 `--sort-sections name` 等同于SORT_BY_ALIGNMENT (SORT_BY_NAME (wildcard section pattern))。

> 1. SORT_BY_NAME (wildcard section pattern) with --sort-sections alignment is equivalent to SORT_BY_NAME (SORT_BY_ALIGNMENT (wildcard section pattern)).
> 2. SORT_BY_ALIGNMENT (wildcard section pattern) with --sort-section name is equivalent to SORT_BY_ALIGNMENT (SORT_BY_NAME (wildcard section pattern)).

如果链接器脚本中的节排序命令是嵌套的，那么命令行选项将被忽略。

> If the section sorting command in linker script is nested, the command-line option will be ignored.

SORT_NONE 通过忽略命令行的节排序选项来禁用节排序。

> SORT_NONE disables section sorting by ignoring the command-line section sorting option.

如果你对输入的节去向感到困惑，可以使用 `-M` 链接器选项来生成一个映射文件。映射文件精确地显示了输入节是如何映射到输出节的。

> If you ever get confused about where input sections are going, use the ‘-M’ linker option to generate a map file. The map file shows precisely how input sections are mapped to output sections.

这个例子显示了如何使用通配符模式来划分文件。这个链接器脚本指示链接器将所有 *.text* 节放在 *.text* 中，所有 *.bss* 节放在 *.bss* 中。链接器将把所有以大写字母开头的文件中的 *.data* 节放在 *.DATA* 中；对于所有其他文件，链接器将把 *.data* 节放在 *.data* 中。

> This example shows how wildcard patterns might be used to partition files. This linker script directs the linker to place all ‘.text’ sections in ‘.text’ and all ‘.bss’ sections in ‘.bss’. The linker will place the ‘.data’ section from all files beginning with an upper case character in ‘.DATA’; for all other files, the linker will place the ‘.data’ section in ‘.data’.

```ld
SECTIONS {
  .text : { *(.text) }
  .DATA : { [A-Z]*(.data) }
  .data : { *(.data) }
  .bss : { *(.bss) }
}
```

#### 3.6.4.3 公共符号的输入节

3.6.4.3 Input Section for Common Symbols

公共符号需要一个特殊的写法，因为在许多对象文件格式中，公共符号没有一个特定的输入节。链接器认为公共符号在一个名为 *COMMON* 的输入节。

> A special notation is needed for common symbols, because in many object file formats common symbols do not have a particular input section. The linker treats common symbols as though they are in an input section named ‘COMMON’.

你可以在 *COMMON* 节使用文件名，就像在其他输入节一样。你可以利用这一点将某个特定输入节的公共符号放在一个节，而将其他输入文件的公共符号放在另一个节。

> You may use file names with the ‘COMMON’ section just as with any other input sections. You can use this to place common symbols from a particular input file in one section while common symbols from other input files are placed in another section.

在大多数情况下，输入文件中的公共符号将被放置在输出文件的 *.bss* 节。比如说：

> In most cases, common symbols in input files will be placed in the ‘.bss’ section in the output file. For example:

```ld
.bss { *(.bss) *(COMMON) }
```

一些对象文件格式有多于一种类型的公共符号。例如，MIPS ELF 对象文件格式区分了标准通用符号和小型通用符号。在这种情况下，链接器将对其他类型的公共符号使用不同的特殊节名称。在 MIPS ELF 的情况下，链接器对标准公共符号使用 *COMMON*，对小型公共符号使用 *.scommon*。这允许你将不同类型的公共符号映射到内存的不同位置。

> Some object file formats have more than one type of common symbol. For example, the MIPS ELF object file format distinguishes standard common symbols and small common symbols. In this case, the linker will use a different special section name for other types of common symbols. In the case of MIPS ELF, the linker uses ‘COMMON’ for standard common symbols and ‘.scommon’ for small common symbols. This permits you to map the different types of common symbols into memory at different locations.

你有时会在老的链接器脚本中看到 *\[COMMON\]*。这种符号现在已经被认为是过时的了。它等同于 *\*(COMMON)*。

> You will sometimes see ‘[COMMON]’ in old linker scripts. This notation is now considered obsolete. It is equivalent to ‘*(COMMON)’.

#### 3.6.4.4 输入节和垃圾收集

> 3.6.4.4 Input Section and Garbage Collection

当使用链接时垃圾收集时（`--gc-sections`），标记不应该被消除的节往往是有用的。这可以通过在输入节的通配符周围加上 KEEP() 来实现，如 KEEP(\*(.init)) 或 KEEP(SORT_BY_NAME(\*)(.ctors))。

> When link-time garbage collection is in use (‘--gc-sections’), it is often useful to mark sections that should not be eliminated. This is accomplished by surrounding an input section’s wildcard entry with KEEP(), as in KEEP(\*(.init)) or KEEP(SORT_BY_NAME(\*)(.ctors)).

#### 3.6.4.5 输入节示例

> 3.6.4.5 Input Section Example

下面的例子是一个完整的链接器脚本。它告诉链接器从文件 all.o 中读取所有节，并把它们放在输出节 *outputa* 的开始位置 *0x10000* 处。文件 foo.o 中的所有 *.input1* 节紧随其后，在同一个输出节。所有来自 foo.o 的 *.input2* 节都进入输出节 *outputb*，然后是来自 foo1.o 的 *.input1* 节。剩下任何文件的所有 *.input1* 和 *.input2* 节都写入输出节 *outputc*。

> The following example is a complete linker script. It tells the linker to read all of the sections from file all.o and place them at the start of output section ‘outputa’ which starts at location ‘0x10000’. All of section ‘.input1’ from file foo.o follows immediately, in the same output section. All of section ‘.input2’ from foo.o goes into output section ‘outputb’, followed by section ‘.input1’ from foo1.o. All of the remaining ‘.input1’ and ‘.input2’ sections from any files are written to output section ‘outputc’.

```ld
SECTIONS {
  outputa 0x10000 :
    {
    all.o
    foo.o (.input1)
    }
  outputb :
    {
    foo.o (.input2)
    foo1.o (.input1)
    }
  outputc :
    {
    *(.input1)
    *(.input2)
    }
}
```

如果一个输出节的名称与输入节的名称相同，并且可以用 C 语言标识符表示，那么链接器将自动看到 PROVIDE 两个符号：`__start_SECNAME` 和 `__stop_SECNAME`，其中 *SECNAME* 是该节的名称。它们分别表示输出节的起始地址和结束地址。注意：大多数节的名称不能作为 C 语言的标识符来表示，因为它们包含一个 `.` 字符。

> If an output section’s name is the same as the input section’s name and is representable as a C identifier, then the linker will automatically see PROVIDE two symbols: __start_SECNAME and__stop_SECNAME, where SECNAME is the name of the section. These indicate the start address and end address of the output section respectively. Note: most section names are not representable as C identifiers because they contain a ‘.’ character.

### 3.6.5 输出节数据

> 3.6.5 Output Section Data

你可以通过使用 BYTE、SHORT、LONG、QUAD 或者 SQUAD 作为输出节命令，在输出节包含明确的字节数据。每个关键字后面都有一个括号内的表达式，提供了要存储的值（参见[链接器脚本中的表达式](#310-链接器脚本中的表达式)）。表达式的值被存储在位置计数器的当前值上。

> You can include explicit bytes of data in an output section by using BYTE, SHORT, LONG, QUAD, or SQUAD as an output section command. Each keyword is followed by an expression in parentheses providing the value to store (see Expressions in Linker Scripts). The value of the expression is stored at the current value of the location counter.

BYTE、SHORT、LONG 和 QUAD 命令分别存储一个、两个、四个和八个字节。在存储字节后，位置计数器按存储的字节数递增。

> The BYTE, SHORT, LONG, and QUAD commands store one, two, four, and eight bytes (respectively). After storing the bytes, the location counter is incremented by the number of bytes stored.

例如，这将存储字节 1，然后是符号 *addr* 的四个字节的值。

> For example, this will store the byte 1 followed by the four byte value of the symbol ‘addr’:

```ld
BYTE(1)
LONG(addr)
```

当使用 64 位主机或目标时，QUAD 和 SQUAD 是一样的；它们都存储一个 8 字节，或 64 位的值。当主机和目标机都是 32 位时，一个表达式被计算为 32 位。在这种情况下，QUAD 存储一个零扩展到 64 位的 32 位值，而 SQUAD 存储一个符号扩展到 64 位的 32 位值。

> When using a 64 bit host or target, QUAD and SQUAD are the same; they both store an 8 byte, or 64 bit, value. When both host and target are 32 bits, an expression is computed as 32 bits. In this case QUAD stores a 32 bit value zero extended to 64 bits, and SQUAD stores a 32 bit value sign extended to 64 bits.

如果输出文件的对象文件格式有一个明确的节模式，这是一般的情况，值按这个端模式存储。当对象文件格式没有明确的端模式时，例如 S-recoreds，值按第一个输入对象文件的端模式存储。

> If the object file format of the output file has an explicit endianness, which is the normal case, the value will be stored in that endianness. When the object file format does not have an explicit endianness, as is true of, for example, S-records, the value will be stored in the endianness of the first input object file.

注意--这些命令只在节描述内起作用，在它们之间不起作用，所以下面的命令会使链接器产生一个错误：

> Note—these commands only work inside a section description and not between them, so the following will produce an error from the linker:

```ld
SECTIONS { .text : { *(.text) } LONG(1) .data : {*(.data) } }
```

而这句话则可以工作：

> whereas this will work:

```ld
SECTIONS { .text : { *(.text) ; LONG(1) } .data : {*(.data) } }
```

你可以使用 FILL 命令来设置当前节的填充模式。它后面是括号中的表达式。节内任何未指定的内存区域（例如，由于输入节需要对齐而留下的空隙）都会用表达式的值来填充，必要时重复。一个 FILL 语句影响它在节定义中出现的那个点之后的内存位置；通过包含一个以上的 FILL 语句，你可以在输出节的不同部分有不同的填充模式。

> You may use the FILL command to set the fill pattern for the current section. It is followed by an expression in parentheses. Any otherwise unspecified regions of memory within the section (for example, gaps left due to the required alignment of input sections) are filled with the value of the expression, repeated as necessary. A FILL statement covers memory locations after the point at which it occurs in the section definition; by including more than one FILL statement, you can have different fill patterns in different parts of an output section.

这个例子显示了如何用值 *0x90* 来填充未指定的内存区域。

> This example shows how to fill unspecified regions of memory with the value ‘0x90’:

```ld
FILL(0x90909090)
```

FILL 命令与 `=fillexp` 输出节属性类似，但它只影响 FILL 命令之后的部分，而不是整个节。如果两者都使用，则以 FILL 命令为准。参见[输出节填充](#3688-输出节填充)，了解填充表达式的细节。

> The FILL command is similar to the ‘=fillexp’ output section attribute, but it only affects the part of the section following the FILL command, rather than the entire section. If both are used, the FILL command takes precedence. See Output Section Fill, for details on the fill expression.

### 3.6.6 输出节关键字

> 3.6.6 Output Section Keywords

有几个关键字可以作为输出节的命令出现。

> There are a couple of keywords which can appear as output section commands.

- CREATE_OBJECT_SYMBOLS

  这个命令告诉链接器为每个输入文件创建一个符号。每个符号的名称是相应的输入文件的名称。这些符号的节是 CREATE_OBJECT_SYMBOLS 命令所出现的输出节。

  > The command tells the linker to create a symbol for each input file. The name of each symbol will be the name of the corresponding input file. The section of each symbol will be the output section in which the CREATE_OBJECT_SYMBOLS command appears.

  这是对 a.out 对象文件格式的常规做法。它通常不用于任何其他对象文件格式。

  > This is conventional for the a.out object file format. It is not normally used for any other object file format.

- CONSTRUCTORS

  当使用 a.out 对象文件格式进行链接时，链接器使用一个不寻常的集合结构来支持 C++ 全局构造函数和析构函数。当链接不支持任意节的对象文件格式时，例如 ECOFF 和 XCOFF，链接器将自动识别 C++ 全局构造函数和析构函数的名称。对于这些对象文件格式，CONSTRUCTORS 命令告诉链接器将构造函数信息放在出现 CONSTRUCTORS 命令的输出节中。对于其他对象文件格式，CONSTRUCTORS 命令被忽略。

  > When linking using the a.out object file format, the linker uses an unusual set construct to support C++ global constructors and destructors. When linking object file formats which do not support arbitrary sections, such as ECOFF and XCOFF, the linker will automatically recognize C++ global constructors and destructors by name. For these object file formats, the CONSTRUCTORS command tells the linker to place constructor information in the output section where the CONSTRUCTORS command appears. The CONSTRUCTORS command is ignored for other object file formats.

  符号 `__CTOR_LIST__` 标志着全局构造函数的开始，而符号 `__CTOR_END__` 标志着结束。类似地，`__DTOR_LIST__` 和 `__DTOR_END__` 标志着全局析构函数的开始和结束。列表中的第一个字是条目的数量，接着是每个构造函数或析构函数的地址，然后是一个零。编译器必须安排实际运行该代码。对于这些对象文件格式，GNU C++ 通常从子程序 `__main` 调用构造函数；对 `__main` 的调用会自动插入 main 的启动代码中。GNU C++ 通常通过使用 `atexit` 或直接从函数 `exit` 来运行析构器。

  > The symbol __CTOR_LIST__ marks the start of the global constructors, and the symbol __CTOR_END__ marks the end. Similarly, __DTOR_LIST__ and __DTOR_END__ mark the start and end of the global destructors. The first word in the list is the number of entries, followed by the address of each constructor or destructor, followed by a zero word. The compiler must arrange to actually run the code. For these object file formats GNU C++ normally calls constructors from a subroutine __main; a call to__main is automatically inserted into the startup code for main. GNU C++ normally runs destructors either by using atexit, or directly from the function exit.

  对于像 COFF 或 ELF 这样支持任意节名的对象文件格式，GNU C++ 通常会安排将全局构造函数和析构函数的地址放到 .ctors 和 .dtors 节中。将下面的序列放入你的链接器脚本将建立 GNU C++ 运行时代码期望看到的那种表格。

  > For object file formats such as COFF or ELF which support arbitrary section names, GNU C++ will normally arrange to put the addresses of global constructors and destructors into the .ctors and .dtors sections. Placing the following sequence into your linker script will build the sort of table which the GNU C++ runtime code expects to see.

  ```ld
  __CTOR_LIST__ = .;
  LONG((__CTOR_END__ - __CTOR_LIST__) / 4 - 2)
  *(.ctors)
  LONG(0)
  __CTOR_END__ = .;
  __DTOR_LIST__ = .;
  LONG((__DTOR_END__ - __DTOR_LIST__) / 4 - 2)
  *(.dtors)
  LONG(0)
  __DTOR_END__ = .;
  ```

  如果你使用 GNU C++ 对初始化优先级的支持，它提供了对全局构造函数运行顺序的一些控制，你必须在链接时对构造函数进行排序，以确保它们以正确的顺序被执行。在使用 CONSTRUCTORS 命令时，请使用 *SORT_BY_NAME(CONSTRUCTORS)* 来代替。当使用 *.ctors* 和 *.dtors* 节时，使用 *\*(SORT_BY_NAME(.ctors))* 和 *\*(SORT_BY_NAME(.dtors))*，而不仅仅是 *\*(.ctors)* 和 *\*(.dtors)*。

  > If you are using the GNU C++ support for initialization priority, which provides some control over the order in which global constructors are run, you must sort the constructors at link time to ensure that they are executed in the correct order. When using the CONSTRUCTORS command, use ‘SORT_BY_NAME(CONSTRUCTORS)’ instead. When using the .ctors and .dtors sections, use ‘\*(SORT_BY_NAME(.ctors))’ and ‘\*(SORT_BY_NAME(.dtors))’ instead of just ‘\*(.ctors)’ and ‘\*(.dtors)’.

  通常，编译器和链接器会自动处理这些问题，你不需要关心。然而，如果你使用 C++ 并编写自己的链接器脚本，你可能需要考虑这个问题。

  > Normally the compiler and linker will handle these issues automatically, and you will not need to concern yourself with them. However, you may need to consider this if you are using C++ and writing your own linker scripts.

### 3.6.7 输出节丢弃

> 3.6.7 Output Section Discarding

链接器通常不会创建没有内容的输出节。这是为了在引用输入节时的方便，这些节可能存在于任何输入文件中，也可能不存在。比如说：

> The linker will not normally create output sections with no contents. This is for convenience when referring to input sections that may or may not be present in any of the input files. For example:

```ld
.foo : { *(.foo) }
```

只有在至少一个输入文件中有 *.foo* 节，并且输入节不全是空的情况下，才会在输出文件中创建一个 *.foo* 节。其他在输出节分配空间的链接脚本指令也将创建输出节。给点赋值也会创建输出节，即使这次赋值并未创建空间，除了 *sym* 在脚本中定义为 0 的情况下使用 `. = 0`、`. = . + 0`、`. = sym`、`. = . + sym` 和 `. = ALIGN (. != 0, expr, 1)` 的情况。这允许你用 `. = .` 强制输出一个空的节。

> will only create a ‘.foo’ section in the output file if there is a ‘.foo’ section in at least one input file, and if the input sections are not all empty. Other link script directives that allocate space in an output section will also create the output section. So too will assignments to dot even if the assignment does not create space, except for ‘. = 0’, ‘. = . + 0’, ‘. = sym’, ‘. = . + sym’ and ‘. = ALIGN (. != 0, expr, 1)’ when ‘sym’ is an absolute symbol of value 0 defined in the script. This allows you to force output of an empty section with ‘. = .’.

链接器将忽略被丢弃的输出节的地址分配（见[输出节地址](#363-输出节地址)），除非链接器脚本在输出节定义了符号。在这种情况下，链接器将服从地址分配，可能会推进点，即使该节被丢弃。

> The linker will ignore address assignments (see Output Section Address) on discarded output sections, except when the linker script defines symbols in the output section. In that case the linker will obey the address assignments, possibly advancing dot even though the section is discarded.

特殊的输出节名称 */DISCARD/* 可以用来丢弃输入节。任何被分配到名为 */DISCARD/* 的输出节的输入节都不包括在输出文件中。

> The special output section name ‘/DISCARD/’ may be used to discard input sections. Any input sections which are assigned to an output section named ‘/DISCARD/’ are not included in the output file.

这可以用来丢弃标有 ELF 标志 SHF_GNU_RETAIN的 输入节，否则这些节将从链接器的垃圾收集中被保存。

> This can be used to discard input sections marked with the ELF flag SHF_GNU_RETAIN, which would otherwise have been saved from linker garbage collection.

注意，与 */DISCARD/* 输出节相匹配的节将被丢弃，即使它们在一个 ELF 节组中，而该组的其他成员没有被丢弃。这是故意的。丢弃优先于分组。

> Note, sections that match the ‘/DISCARD/’ output section will be discarded even if they are in an ELF section group which has other members which are not being discarded. This is deliberate. Discarding takes precedence over grouping.

### 3.6.8 输出节属性

> 3.6.8 Output Section Attributes

我们已经展示过，一个输出节的完整描述是这样的：

> We showed above that the full description of an output section looked like this:

```ld
section [address] [(type)] :
  [AT(lma)]
  [ALIGN(section_align) | ALIGN_WITH_INPUT]
  [SUBALIGN(subsection_align)]
  [constraint]
  {
    output-section-command
    output-section-command
    ...
  } [>region] [AT>lma_region] [:phdr :phdr ...] [=fillexp]
```

我们已经描述了 *section*、*address* 和 *output-section-command*。在这一节中，我们将描述其余的节属性。

> We’ve already described section, address, and output-section-command. In this section we will describe the remaining section attributes.

- [输出节类型](#3681-输出节类型)
- [输出节 LMA](#3682-输出节-lma)
- [强制输出对齐](#3683-强制输出对齐)
- [强制输入对齐](#3684-强制输入对齐)
- [输出节约束](#3685-输出节约束)
- [输出节区域](#3686-输出节区域)
- [输出节 Phdr](#3687-输出节-phdr)
- [输出节填充](#3688-输出节填充)

> - Output Section Type
> - Output Section LMA
> - Forced Output Alignment
> - Forced Input Alignment
> - Output Section Constraint
> - Output Section Region
> - Output Section Phdr
> - Output Section Fill

#### 3.6.8.1 输出节类型

> 3.6.8.1 Output Section Type

每个输出节都可以有一个类型。该类型是括号中的一个关键字。已经定义了下列类型：

> Each output section may have a type. The type is a keyword in parentheses. The following types are defined:

- NOLOAD

  该节应该被标记为不可加载，这样在程序运行时它就不会被加载到内存中。

  > The section should be marked as not loadable, so that it will not be loaded into memory when the program is run.

- READONLY

  该节应该被标记为只读。

  > The section should be marked as read-only.

- DSECT
- COPY
- INFO
- OVERLAY

  这些类型的名称是为了向后兼容而支持的，但很少使用。它们都有相同的效果：该节应该被标记为不可分配，这样在程序运行时就不会为该部分分配内存。

  > These type names are supported for backward compatibility, and are rarely used. They all have the same effect: the section should be marked as not allocatable, so that no memory is allocated for the section when the program is run.

- TYPE = *type*

  将节的类型设置为整数 *type*。在生成 ELF 输出文件时，类型名称 SHT_PROGBITS、SHT_STRTAB、SHT_NOTE、SHT_NOBITS、SHT_INIT_ARRAY、SHT_FINI_ARRAY 和 SHT_PREINIT_ARRAY 也允许用于 *type*。确保满足节类型的任何特殊要求是用户的责任。

  > Set the section type to the integer type. When generating an ELF output file, type names SHT_PROGBITS, SHT_STRTAB, SHT_NOTE, SHT_NOBITS, SHT_INIT_ARRAY, SHT_FINI_ARRAY, and SHT_PREINIT_ARRAY are also allowed for type. It is the user’s responsibility to ensure that any special requirements of the section type are met.

- READONLY ( TYPE = *type* )

  这种形式的语法结合了 READONLY 类型和 *type* 指定的类型。

  > This form of the syntax combines the READONLY type with the type specified by type.

链接器通常根据映射到它的输入节来设置输出节的属性。你可以通过使用节类型来覆盖这一点。例如，在下面的脚本示例中，*ROM* 节在内存位置 *0* 被寻址，在程序运行时不需要加载。

> The linker normally sets the attributes of an output section based on the input sections which map into it. You can override this by using the section type. For example, in the script sample below, the ‘ROM’ section is addressed at memory location ‘0’ and does not need to be loaded when the program is run.

```ld
SECTIONS {
  ROM 0 (NOLOAD) : { ... }
  ...
}
```

#### 3.6.8.2 输出节 LMA

> 3.6.8.2 Output Section LMA

每个节都有一个虚拟地址（VMA）和一个加载地址（LMA）；参见[链接器脚本的基本概念](#31-链接器脚本的基本概念)。虚拟地址是由前述的[输出节地址](#363-输出节地址)指定的。加载地址是由 AT 或 AT> 关键字指定的。指定一个加载地址是可选的。

> Every section has a virtual address (VMA) and a load address (LMA); see Basic Linker Script Concepts. The virtual address is specified by the see Output Section Address described earlier. The load address is specified by the AT or AT> keywords. Specifying a load address is optional.

AT 关键字需要一个表达式作为参数。这指定了该节的精确加载地址。AT> 关键字以一个内存区域的名称作为参数。参见 [MEMORY 命令](#37-memory-命令)。该节的加载地址被设置为该区域的下一个空闲地址，按照该节的对齐要求进行对齐。

> The AT keyword takes an expression as an argument. This specifies the exact load address of the section. The AT> keyword takes the name of a memory region as an argument. See MEMORY Command. The load address of the section is set to the next free address in the region, aligned to the section’s alignment requirements.

如果对于可分配的节既没有指定 AT 也没有指定 AT>，链接器将使用以下启发式方法来确定加载地址：

> If neither AT nor AT> is specified for an allocatable section, the linker will use the following heuristic to determine the load address:

- 如果为该节指定了一个 VMA 地址，那么它也会被用作 LMA 地址。
- 如果该节是不可分配的，那么它的 LMA 将被设置为其 VMA。
- 否则，如果能找到一个与该节兼容的内存区域，并且该区域至少包含一个节，则该节的 LMA 被设置，以满足：VMA 和 LMA 之间的差值与位于该区域的最后一个节的 VMA 和 LMA 之间的差值相同。
- 如果没有声明任何内存区域，那么在上一步中会使用一个覆盖整个地址空间的默认区域。
- 如果找不到合适的区域，或者找到的区域里没有节，那么 LMA 被设置为等于 VMA。

> - If the section has a specific VMA address, then this is used as the LMA address as well.
> - If the section is not allocatable then its LMA is set to its VMA.
> - Otherwise if a memory region can be found that is compatible with the current section, and this region contains at least one section, then the LMA is set so the difference between the VMA and LMA is the same as the difference between the VMA and LMA of the last section in the located region.
> - If no memory regions have been declared then a default region that covers the entire address space is used in the previous step.
> - If no suitable region could be found, or there was no previous section then the LMA is set equal to the VMA.

设计这个功能是为了使建立一个 ROM 映像变得容易。例如，下面的链接器脚本创建了三个输出节：一个叫 *.text*，从 0x1000 开始，一个叫 *.mdata*，在 *.text* 区的末尾加载，尽管其 VMA 是 0x2000，还有一个叫 *.bss*，在地址 0x3000 存放未初始化的数据。符号 *_data* 的定义值为 0x2000，这表明位置计数器持有 VMA 值，而不是 LMA 值。

> This feature is designed to make it easy to build a ROM image. For example, the following linker script creates three output sections: one called ‘.text’, which starts at 0x1000, one called ‘.mdata’, which is loaded at the end of the ‘.text’ section even though its VMA is 0x2000, and one called ‘.bss’ to hold uninitialized data at address 0x3000. The symbol _data is defined with the value 0x2000, which shows that the location counter holds the VMA value, not the LMA value.

```ld
SECTIONS
  {
  .text 0x1000 : { *(.text) _etext = . ; }
  .mdata 0x2000 :
    AT ( ADDR (.text) + SIZEOF (.text) )
    { _data = . ; *(.data); _edata = . ;  }
  .bss 0x3000 :
    { _bstart = . ;  *(.bss) *(COMMON) ; _bend = . ;}
}
```

用这个链接器脚本生成的程序的运行时初始化代码将包括如下内容，将初始化数据从 ROM 映像复制到其运行时地址。注意这节代码是如何利用链接器脚本所定义的符号的。

> The run-time initialization code for use with a program generated with this linker script would include something like the following, to copy the initialized data from the ROM image to its runtime address. Notice how this code takes advantage of the symbols defined by the linker script.

```c
extern char _etext,_data, _edata,_bstart, _bend;
char *src = &_etext;
char *dst = &_data;

/*ROM has data at end of text; copy it.*/
while (dst < &_edata)
  *dst++ = *src++;

/*Zero bss.*/
for (dst = &_bstart; dst< &_bend; dst++)
  *dst = 0;
```

#### 3.6.8.3 强制输出对齐

> 3.6.8.3 Forced Output Alignment

你可以通过使用 ALIGN 来增加一个输出节的对齐方式。另外你可以通过 ALIGN_WITH_INPUT 属性强制要求 VMA 和 LMA 之间的差异在整个输出节保持不变。

> You can increase an output section’s alignment by using ALIGN. As an alternative you can enforce that the difference between the VMA and LMA remains intact throughout this output section with the ALIGN_WITH_INPUT attribute.

#### 3.6.8.4 强制输入对齐

> 3.6.8.4 Forced Input Alignment

你可以通过使用 SUBALIGN 在输出节强制输入节对齐。指定的值会覆盖输入节给出的任何对齐方式，无论是大还是小。

> You can force input section alignment within an output section by using SUBALIGN. The value specified overrides any alignment given by input sections, whether larger or smaller.

#### 3.6.8.5 输出节约束

> 3.6.8.5 Output Section Constraint

你可以通过使用关键字 ONLY_IF_RO 和 ONLY_IF_RW 分别指定，只有当一个输出节的所有输入节都是只读的，或者所有输入节都是读写的时候才可以创建。

> You can specify that an output section should only be created if all of its input sections are read-only or all of its input sections are read-write by using the keyword ONLY_IF_RO and ONLY_IF_RW respectively.

#### 3.6.8.6 输出节区域

> 3.6.8.6 Output Section Region

你可以通过使用 *>region* 将一个节分配到一个先前定义的内存区域。参见 [MEMORY 命令](#37-memory-命令)。

> You can assign a section to a previously defined region of memory by using ‘>region’. See MEMORY Command.

下面是一个简单的例子：

> Here is a simple example:

```ld
MEMORY { rom : ORIGIN = 0x1000, LENGTH = 0x1000 }
SECTIONS { ROM : { *(.text) } >rom }
```

#### 3.6.8.7 输出节 Phdr

> 3.6.8.7 Output Section Phdr

你可以通过使用 *:phdr* 将一个节分配给先前定义的程序节。参见 [PHDRS 命令](#38-phdrs-命令)。如果一个节被分配到一个或多个程序节，那么所有后续分配的节也将被分配到这些程序节，除非它们明确使用了 *:phdr* 修饰符。你可以使用 *:NONE* 来告诉链接器不要把这个节放在任何节中。

> You can assign a section to a previously defined program segment by using ‘:phdr’. See PHDRS Command. If a section is assigned to one or more segments, then all subsequent allocated sections will be assigned to those segments as well, unless they use an explicitly :phdr modifier. You can use :NONE to tell the linker to not put the section in any segment at all.

下面是一个简单的例子：

> Here is a simple example:

```ld
PHDRS { text PT_LOAD ; }
SECTIONS { .text : { *(.text) } :text }
```

#### 3.6.8.8 输出节填充

> 3.6.8.8 Output Section Fill

你可以通过使用 *=fillexp* 来设置整个节的填充模式。 *fillexp* 是一个表达式（参见[链接器脚本中的表达式](#310-链接器脚本中的表达式)）。在输出节的任何其他未指定的内存区域（例如，由于输入节的对齐要求而留下的空隙）将被填入该值，必要时重复。如果填充表达式是一个简单的十六进制数字，即一串以 *0x* 开头的十六进制数字，没有尾部的 *k* 或 *M*，那么可以使用任意长的十六进制数字序列来指定填充模式；前导零也成为模式的一部分。对于所有其他情况，包括额外的括号或单目 +，填充模式是表达式值的四个最小有效字节。在所有情况下，数字都是大端的。

> You can set the fill pattern for an entire section by using ‘=fillexp’. fillexp is an expression (see Expressions in Linker Scripts). Any otherwise unspecified regions of memory within the output section (for example, gaps left due to the required alignment of input sections) will be filled with the value, repeated as necessary. If the fill expression is a simple hex number, ie. a string of hex digit starting with ‘0x’ and without a trailing ‘k’ or ‘M’, then an arbitrarily long sequence of hex digits can be used to specify the fill pattern; Leading zeros become part of the pattern too. For all other cases, including extra parentheses or a unary +, the fill pattern is the four least significant bytes of the value of the expression. In all cases, the number is big-endian.

你也可以用输出节命令中的 FILL 命令改变填充值；（见[输出节数据](#365-输出节数据)）。

> You can also change the fill value with a FILL command in the output section commands; (see Output Section Data).

下面是一个简单的例子：

> Here is a simple example:

```ld
SECTIONS { .text : { *(.text) } =0x90909090 }
```

### 3.6.9 覆盖描述

> 3.6.9 Overlay Description

覆盖描述提供了一种简单的方法来描述作为单个内存映像的一部分被加载但在同一内存地址运行的多个节。在运行时，某种覆盖管理器会根据需要将重叠部分复制到运行时的内存地址中，也许是通过简单地操作地址位。这种方法很有用，例如，当某一区域的内存比另一区域快时。

> An overlay description provides an easy way to describe sections which are to be loaded as part of a single memory image but are to be run at the same memory address. At run time, some sort of overlay manager will copy the overlaid sections in and out of the runtime memory address as required, perhaps by simply manipulating addressing bits. This approach can be useful, for example, when a certain region of memory is faster than another.

覆盖是用 OVERLAY 命令描述的。OVERLAY 命令是在 SECTIONS 命令中使用的，就像输出节的描述。OVERLAY 命令的完整语法如下：

> Overlays are described using the OVERLAY command. The OVERLAY command is used within a SECTIONS command, like an output section description. The full syntax of the OVERLAY command is as follows:

```ld
OVERLAY [start] : [NOCROSSREFS] [AT ( ldaddr )]
  {
    secname1
      {
        output-section-command
        output-section-command
        ...
      } [:phdr...] [=fill]
    secname2
      {
        output-section-command
        output-section-command
        ...
      } [:phdr...] [=fill]
    ...
  } [>region] [:phdr...] [=fill] [,]
```

除了 OVERLAY（一个关键字），其他都是可选的，每个节都必须有一个名称（上面的 *secname1* 和 *secname2*）。OVERLAY 构造中的节定义与一般的 SECTIONS 结构（见 [SECTIONS 命令](#36-sections-命令)）中的定义相同，只是在一个 OVERLAY 中不能为节定义地址和内存区域。

> Everything is optional except OVERLAY (a keyword), and each section must have a name (secname1 and secname2 above). The section definitions within the OVERLAY construct are identical to those within the general SECTIONS construct (see SECTIONS Command), except that no addresses and no memory regions may be defined for sections within an OVERLAY.

如果使用了填充，并且下一个 section-command 看起来像表达式的延续，那么末尾的逗号可能是必需的。

> The comma at the end may be required if a fill is used and the next sections-command looks like a continuation of the expression.

这些节都定义在相同的起始地址。各部分的加载地址是这样安排的：它们在内存中是连续的，从用于整个 OVERLAY 的加载地址开始（与普通的节定义一样，加载地址是可选的，默认为起始地址；起始地址也是可选的，默认为位置计数器的当前值）。

> The sections are all defined with the same starting address. The load addresses of the sections are arranged such that they are consecutive in memory starting at the load address used for the OVERLAY as a whole (as with normal section definitions, the load address is optional, and defaults to the start address; the start address is also optional, and defaults to the current value of the location counter).

如果使用了 NOCROSSREFS 关键字，并且在各节之间存在任何引用，链接器将报告一个错误。由于各节都在同一地址运行，通常一个节直接引用另一个节是没有意义的。参见 [NOCROSSREFS](#345-其他链接器脚本命令)。

> If the NOCROSSREFS keyword is used, and there are any references among the sections, the linker will report an error. Since the sections all run at the same address, it normally does not make sense for one section to refer directly to another. See NOCROSSREFS.

对于 OVERLAY 中的每个节，链接器会自动提供两个符号。符号 `__load_start_secname` 定义为该部分的起始加载地址。符号 `__load_stop_secname` 定义为该部分的结束装载地址。*secname* 中任何在 C 语言标识符中不合法的字符都被删除。C（或汇编器）代码可以使用这些符号在必要时移动重叠的部分。

> For each section within the OVERLAY, the linker automatically provides two symbols. The symbol __load_start_secname is defined as the starting load address of the section. The symbol__load_stop_secname is defined as the final load address of the section. Any characters within secname which are not legal within C identifiers are removed. C (or assembler) code may use these symbols to move the overlaid sections around as necessary.

在覆盖结束时，位置计数器的值被设置为覆盖的起始地址加上最大节的大小。

> At the end of the overlay, the value of the location counter is set to the start address of the overlay plus the size of the largest section.

下面是一个例子。请记住，这将出现在一个 SECTIONS 结构中。

> Here is an example. Remember that this would appear inside a SECTIONS construct.

```ld
OVERLAY 0x1000 : AT (0x4000)
  {
    .text0 { o1/*.o(.text) }
    .text1 { o2/*.o(.text) }
  }
```

这将定义 *.text0* 和 *.text1* 都从地址 0x1000 开始。*.text0* 可以从地址 0x4000 处加载，而 *.text1* 可以从 *.text0* 之后加载。如果被引用，以下符号将被定义：\__load_start_text0, \__load_stop_text0, \__load_start_text1, \__load_stop_text1`。

> This will define both ‘.text0’ and ‘.text1’ to start at address 0x1000. ‘.text0’ will be loaded at address 0x4000, and ‘.text1’ will be loaded immediately after ‘.text0’. The following symbols will be defined if referenced: \__load_start_text0, \__load_stop_text0, \__load_start_text1, \__load_stop_text1.

复制覆盖 *.text1* 覆盖区的 C 语言代码可能看起来像下面这样。

> C code to copy overlay .text1 into the overlay area might look like the following.

```c
extern char __load_start_text1,__load_stop_text1;
memcpy ((char *) 0x1000, &__load_start_text1,
        &__load_stop_text1 - &__load_start_text1);
```

注意，OVERLAY 命令只是语法上的糖，因为它所做的一切都可以用更基本的命令来完成。上面的例子可以写成以下形式，产生相同的效果。

> Note that the OVERLAY command is just syntactic sugar, since everything it does can be done using the more basic commands. The above example could have been written identically as follows.

```ld
.text0 0x1000 : AT (0x4000) { o1/*.o(.text) }
PROVIDE (__load_start_text0 = LOADADDR (.text0));
PROVIDE (__load_stop_text0 = LOADADDR (.text0) + SIZEOF (.text0));
.text1 0x1000 : AT (0x4000 + SIZEOF (.text0)) { o2/*.o(.text) }
PROVIDE (__load_start_text1 = LOADADDR (.text1));
PROVIDE (__load_stop_text1 = LOADADDR (.text1) + SIZEOF (.text1));
. = 0x1000 + MAX (SIZEOF (.text0), SIZEOF (.text1));
```

## 3.7 MEMORY 命令

> 3.7 MEMORY Command

链接器的默认配置允许分配所有可用的内存。你可以通过使用 MEMORY 命令来覆盖它。

> The linker’s default configuration permits allocation of all available memory. You can override this by using the MEMORY command.

MEMORY 命令描述了目标中内存块的位置和大小。你可以用它来描述哪些内存区域可以被链接器使用，哪些内存区域它必须避开。然后，你可以将节分配给特定的内存区域。链接器将根据内存区域来设置节地址，并对变得太满的区域发出警告。链接器不会为了适应可用的区域而把节弄乱。

> The MEMORY command describes the location and size of blocks of memory in the target. You can use it to describe which memory regions may be used by the linker, and which memory regions it must avoid. You can then assign sections to particular memory regions. The linker will set section addresses based on the memory regions, and will warn about regions that become too full. The linker will not shuffle sections around to fit into the available regions.

一个链接器脚本可以多次使用 MEMORY 命令，但是，所有定义的内存块都被当作是在一个 MEMORY 命令中指定的。MEMORY 的语法是：

> A linker script may contain many uses of the MEMORY command, however, all memory blocks defined are treated as if they were specified inside a single MEMORY command. The syntax for MEMORY is:

```ld
MEMORY
  {
    name [(attr)] : ORIGIN = origin, LENGTH = len
    ...
  }
```

*name* 是链接器脚本中用来指代区域的名称。区域名称在链接器脚本之外没有任何意义。区域名称存储在一个单独的名称空间中，不会与符号名称、文件名称或章节名称冲突。在 MEMORY 命令中，每个内存区域必须有一个独占的名字。然而你可以用给内存区域分配别名的命令给现有的内存区域添加别名。

> The name is a name used in the linker script to refer to the region. The region name has no meaning outside of the linker script. Region names are stored in a separate name space, and will not conflict with symbol names, file names, or section names. Each memory region must have a distinct name within the MEMORY command. However you can add later alias names to existing memory regions with the Assign alias names to memory regions command.

*attr* 字符串是一个可选的属性列表，如果输入节在链接器脚本中没有明确地映射，属性用于指定是否为该节使用某个特定的内存区域。正如在 [SECTIONS 命令](#36-sections-命令)中所描述的，如果你没有为某个输入节指定一个输出节，链接器将创建一个与输入节同名的输出节。如果你定义了区域属性，链接器将使用它们来选择它所创建的输出节的内存区域。

> The attr string is an optional list of attributes that specify whether to use a particular memory region for an input section which is not explicitly mapped in the linker script. As described in SECTIONS Command, if you do not specify an output section for some input section, the linker will create an output section with the same name as the input section. If you define region attributes, the linker will use them to select the memory region for the output section that it creates.

*attr* 字符串必须只由以下字符组成：

> The attr string must consist only of the following characters:

- ‘R’
  只读节
  > Read-only section

- ‘W’
  读写节
  > Read/write section

- ‘X’
  可执行节
  > Executable section

- ‘A’
  可分配节
  > Allocatable section

- ‘I’
  已初始化节
  > Initialized section

- ‘L’
  和 ‘I’ 一样
  > Same as ‘I’

- ‘!’
  反转后续任何属性的意义
  > Invert the sense of any of the attributes that follow

如果一个未映射的节与除 ‘!’ 之外的任何列出的属性相匹配，它将被放置在内存区域中。‘!’ 属性反转了后面字符的意义，所以只有当一个未映射的节不匹配后面列出的任何属性时，它才会被放在内存区域中。因此，一个 ‘RW!X’ 的属性字符串将匹配任何具有 ‘R’ 和 ‘W’ 属性或两者，但没有 ‘X’ 的未映射节。

> If an unmapped section matches any of the listed attributes other than ‘!’, it will be placed in the memory region. The ‘!’ attribute reverses the test for the characters that follow, so that an unmapped section will be placed in the memory region only if it does not match any of the attributes listed afterwards. Thus an attribute string of ‘RW!X’ will match any unmapped section that has either or both of the ‘R’ and ‘W’ attributes, but only as long as the section does not also have the ‘X’ attribute.

*origin* 是内存区域起始地址的数字表达式。该表达式必须求值为一个常数，并且不能涉及任何符号。关键字 ORIGIN 可以缩写为 org 或 o（但不能缩写为 ORG）。

> The origin is an numerical expression for the start address of the memory region. The expression must evaluate to a constant and it cannot involve any symbols. The keyword ORIGIN may be abbreviated to org or o (but not, for example, ORG).

*len* 是一个关于内存区域大小的表达式，单位是字节。与 origin 表达式一样，该表达式必须是数字，并且必须求值为一个常数。关键词 LENGTH 可以简写为 len 或 l。

> The len is an expression for the size in bytes of the memory region. As with the origin expression, the expression must be numerical only and must evaluate to a constant. The keyword LENGTH may be abbreviated to len or l.

在下面的例子中，我们指定有两个可供分配的内存区域：一个从 ‘0’ 开始，为 256 KB，另一个从 ‘0x40000000’ 开始，为 4M。链接器将把每一个没有明确映射到内存区域，并且是只读或可执行的节放入 ‘rom’ 内存区域。链接器将把其他没有明确映射到内存区域的节放入 ‘ram’ 内存区域。

> In the following example, we specify that there are two memory regions available for allocation: one starting at ‘0’ for 256 kilobytes, and the other starting at ‘0x40000000’ for four megabytes. The linker will place into the ‘rom’ memory region every section which is not explicitly mapped into a memory region, and is either read-only or executable. The linker will place other sections which are not explicitly mapped into a memory region into the ‘ram’ memory region.

```ld
MEMORY
  {
    rom (rx)  : ORIGIN = 0, LENGTH = 256K
    ram (!rx) : org = 0x40000000, l = 4M
  }
```

一旦你定义了一个内存区域，你就可以通过使用 ‘>region’ 输出节属性来指示链接器将特定的输出节放入该内存区域。例如，如果你有一个名为 ‘mem’的内存区域，你可以在输出节定义中使用 ‘>mem’。参见[输出节区域](#3686-输出节区域)。如果没有为输出节指定地址，链接器将把地址设置为内存区域内的下一个可用地址。如果指向一个内存区域的所有输出节对该区域来说太大，链接器将发出错误信息。

> Once you define a memory region, you can direct the linker to place specific output sections into that memory region by using the ‘>region’ output section attribute. For example, if you have a memory region named ‘mem’, you would use ‘>mem’ in the output section definition. See Output Section Region. If no address was specified for the output section, the linker will set the address to the next available address within the memory region. If the combined output sections directed to a memory region are too large for the region, the linker will issue an error message.

可以通过 ORIGIN(memory) 和 LENGTH(memory) 函数访问表达式中内存的起点和长度：

> It is possible to access the origin and length of a memory in an expression via the ORIGIN(memory) and LENGTH(memory) functions:

```ld
_fstack = ORIGIN(ram) + LENGTH(ram) - 4;
```

## 3.8 PHDRS 命令

> 3.8 PHDRS Command

ELF 对象文件格式使用程序头，也被称为段。程序头描述了程序应该如何被加载到内存中。你可以通过使用带有 ‘-p’ 选项的 objdump 程序将它们打印出来。

> The ELF object file format uses program headers, also knows as segments. The program headers describe how the program should be loaded into memory. You can print them out by using the objdump program with the ‘-p’ option.

当你在原生 ELF 系统上运行一个 ELF 程序时，系统加载器会读取程序头以确定如何加载该程序。这只有在程序头文件设置正确的情况下才会起作用。本手册没有描述系统加载器如何解释程序头的细节，更多信息请参见 ELF ABI。

> When you run an ELF program on a native ELF system, the system loader reads the program headers in order to figure out how to load the program. This will only work if the program headers are set correctly. This manual does not describe the details of how the system loader interprets program headers; for more information, see the ELF ABI.

链接器将默认创建合理的程序头。然而，在某些情况下，你可能需要更精确地指定程序头。你可以为此目的使用 PHDRS 命令。当链接器在链接器脚本中看到 PHDRS 命令时，它只会创建指定的程序头。

> The linker will create reasonable program headers by default. However, in some cases, you may need to specify the program headers more precisely. You may use the PHDRS command for this purpose. When the linker sees the PHDRS command in the linker script, it will not create any program headers other than the ones specified.

链接器只有在生成 ELF 输出文件时才会注意到 PHDRS 命令。在其他情况下，链接器将直接忽略 PHDRS。

> The linker only pays attention to the PHDRS command when generating an ELF output file. In other cases, the linker will simply ignore PHDRS.

这就是 PHDRS 命令的语法。PHDRS、FILEHDR、AT 和 FLAGS 这些词是关键字。

> This is the syntax of the PHDRS command. The words PHDRS, FILEHDR, AT, and FLAGS are keywords.

```ld
PHDRS
{
  name type [ FILEHDR ] [ PHDRS ] [ AT ( address ) ]
        [ FLAGS ( flags ) ] ;
}
```

*name* 只用于链接器脚本的 SECTIONS 命令中的引用。它不会被放到输出文件中。程序头名被存储在一个单独的名称空间中，不会与符号名、文件名或节名相冲突。每个程序头必须有一个独特的名称。程序头是按顺序处理的，通常它们是按加载地址的升序映射到节。

> The name is used only for reference in the SECTIONS command of the linker script. It is not put into the output file. Program header names are stored in a separate name space, and will not conflict with symbol names, file names, or section names. Each program header must have a distinct name. The headers are processed in order and it is usual for them to map to sections in ascending load address order.

某些程序头类型描述了系统加载器将从文件中加载的内存段。在链接器脚本中，你通过将可分配的输出节放在这些段中来指定这些段的内容。你可以使用‘:phdr’输出节属性来把一个节放在一个特定的段中。参见[输出节 Phdr](#3687-输出节-phdr)。

> Certain program header types describe segments of memory which the system loader will load from the file. In the linker script, you specify the contents of these segments by placing allocatable output sections in the segments. You use the ‘:phdr’ output section attribute to place a section in a particular segment. See Output Section Phdr.

将某些节放在一个以上的段中是正常的。这只是意味着一个内存段包含另一个。你可以重复‘:phdr’，对每个应该包含该节的段使用一次。

> It is normal to put certain sections in more than one segment. This merely implies that one segment of memory contains another. You may repeat ‘:phdr’, using it once for each segment which should contain the section.

如果你用‘:phdr’将一个节放在一个或多个段中，那么链接器将把所有没有指定‘:phdr’的后续可分配节放在相同段中。这是为了方便起见，因为一般来说，一整组连续的节会被放在一个段中。你可以使用 :NONE 来覆盖默认的段，告诉链接器不要把该节放在任何段中。

> If you place a section in one or more segments using ‘:phdr’, then the linker will place all subsequent allocatable sections which do not specify ‘:phdr’ in the same segments. This is for convenience, since generally a whole set of contiguous sections will be placed in a single segment. You can use :NONE to override the default segment and tell the linker to not put the section in any segment at all.

你可以在程序头类型之后使用 FILEHDR 和 PHDRS 关键字来进一步描述段的内容。FILEHDR 关键字意味着该段应该包括 ELF 文件头。PHDRS 关键字意味着该段应该包括 ELF 程序头本身。如果应用于可加载段（PT_LOAD），所有先前的可加载段必须有这些关键字之一。

> You may use the FILEHDR and PHDRS keywords after the program header type to further describe the contents of the segment. The FILEHDR keyword means that the segment should include the ELF file header. The PHDRS keyword means that the segment should include the ELF program headers themselves. If applied to a loadable segment (PT_LOAD), all prior loadable segments must have one of these keywords.

*type* 可以是以下之一。数字表示关键字的值。

> The type may be one of the following. The numbers indicate the value of the keyword.

- PT_NULL (0)
  表示一个未使用的程序头。
  > Indicates an unused program header.

- PT_LOAD (1)
  表示这个程序头描述了一个要从文件中加载的段。
  > Indicates that this program header describes a segment to be loaded from the file.

- PT_DYNAMIC (2)
  表示一个可以找到动态链接信息的段。
  > Indicates a segment where dynamic linking information can be found.

- PT_INTERP (3)
  表示一个可以找到程序解释器名称的段。
  > Indicates a segment where the name of the program interpreter may be found.

- PT_NOTE (4)
  表示一个存放注释信息的段。
  > Indicates a segment holding note information.

- PT_SHLIB (5)
  一个保留的程序头类型，由 ELF ABI 定义但未指定。
  > A reserved program header type, defined but not specified by the ELF ABI.

- PT_PHDR (6)
  表示一个可以找到程序头的段。
  > Indicates a segment where the program headers may be found.

- PT_TLS (7)
  表示一个包含线程本地存储的段。
  > Indicates a segment containing thread local storage.

- *expression*
  一个表达式，给出程序头的类型数字。可以用于上面没有定义的类型。
  > An expression giving the numeric type of the program header. This may be used for types not defined above.

你可以通过使用 AT 表达式来指定一个段应该在内存中的特定地址加载。这与作为输出节属性的 AT 命令相同（见[输出节 LMA](#3682-输出节-lma)）。程序头的 AT 命令覆盖了输出节的属性。

> You can specify that a segment should be loaded at a particular address in memory by using an AT expression. This is identical to the AT command used as an output section attribute (see Output Section LMA). The AT command for a program header overrides the output section attribute.

链接器通常会根据组成段的节来设置段的标记。你可以使用 FLAGS 关键字来明确指定段标记。flags 的值必须是一个整数。它被用来设置程序头的 p_flags 字段。

> The linker will normally set the segment flags based on the sections which comprise the segment. You may use the FLAGS keyword to explicitly specify the segment flags. The value of flags must be an integer. It is used to set the p_flags field of the program header.

下面是一个 PHDRS 的例子。这显示了在一个原生 ELF 系统上使用的一组典型的程序头。

> Here is an example of PHDRS. This shows a typical set of program headers used on a native ELF system.

```ld
PHDRS
{
  headers PT_PHDR PHDRS ;
  interp PT_INTERP ;
  text PT_LOAD FILEHDR PHDRS ;
  data PT_LOAD ;
  dynamic PT_DYNAMIC ;
}

SECTIONS
{
  . = SIZEOF_HEADERS;
  .interp : { *(.interp) } :text :interp
  .text : { *(.text) } :text
  .rodata : { *(.rodata) } /* defaults to :text */
  ...
  . = . + 0x1000; /* move to a new page in memory */
  .data : { *(.data) } :data
  .dynamic : { *(.dynamic) } :data :dynamic
  ...
}
```

## 3.9 VERSION 命令

> 3.9 VERSION Command

当使用 ELF 时，链接器支持符号版本。符号版本只在使用共享库时有用。当动态链接器运行一个可能已经与早期版本的共享库链接的程序时，可以使用符号版本来选择一个特定版本的函数。

> The linker supports symbol versions when using ELF. Symbol versions are only useful when using shared libraries. The dynamic linker can use symbol versions to select a specific version of a function when it runs a program that may have been linked against an earlier version of the shared library.

> 动态链接用不到，See [原文](https://sourceware.org/binutils/docs/ld/VERSION.html)。

## 3.10 链接器脚本中的表达式

> 3.10 Expressions in Linker Scripts

链接器脚本语言中表达式的语法与 C 语言表达式的语法相同，只是在某些地方需要使用空格来解决语法上的歧义。所有的表达式都求值为整数。所有的表达式都以相同的大小求值，如果主机和目标机都是 32 位，那么就是 32 位，否则就是 64 位。

> The syntax for expressions in the linker script language is identical to that of C expressions, except that whitespace is required in some places to resolve syntactic ambiguities. All expressions are evaluated as integers. All expressions are evaluated in the same size, which is 32 bits if both the host and target are 32 bits, and is otherwise 64 bits.

你可以在表达式中使用和设置符号值。

> You can use and set symbol values in expressions.

链接器定义了几个特殊用途的内置函数，以便在表达式中使用。

> The linker defines several special purpose builtin functions for use in expressions.

- [常量](#3101-常量)
- [符号常量](#3102-符号常量)
- [符号名](#3103-符号名)
- [孤岛节](#3104-孤岛节)
- [位置计数器](#3105-位置计数器)
- [运算符](#3106-运算符)
- [求值](#3107-求值)
- [表达式的节](#3108-表达式的节)
- [内置函数](#3109-内置函数)

> - Constants
> - Symbolic Constants
> - Symbol Names
> - Orphan Sections
> - The Location Counter
> - Operators
> - Evaluation
> - The Section of an Expression
> - Builtin Functions

### 3.10.1 常量

> 3.10.1 Constants

所有常量都是整数。

> All constants are integers.

和 C 语言一样，链接器认为以‘0’开头的整数是八进制，以‘0x’或‘0X’开头的整数是十六进制。另外，链接器接受后缀‘h’或‘H’代表十六进制，‘o’或‘O’代表八进制，‘b’或‘B’代表二进制，‘d’或‘D’代表十进制。没有前缀或后缀的任何整数值都被认为是十进制的。

> As in C, the linker considers an integer beginning with ‘0’ to be octal, and an integer beginning with ‘0x’ or ‘0X’ to be hexadecimal. Alternatively the linker accepts suffixes of ‘h’ or ‘H’ for hexadecimal, ‘o’ or ‘O’ for octal, ‘b’ or ‘B’ for binary and ‘d’ or ‘D’ for decimal. Any integer value without a prefix or a suffix is considered to be decimal.

此外，你可以使用后缀 K 和 M 分别将常数按 1024 或 1024*1024 进行缩放。例如，下面这些都是指同一个数量：

> In addition, you can use the suffixes K and M to scale a constant by 1024 or 1024*1024 respectively. For example, the following all refer to the same quantity:

```ld
_fourk_1 = 4K;
_fourk_2 = 4096;
_fourk_3 = 0x1000;
_fourk_4 = 10000o;
```

注意 - 后缀 K 和 M 不能与上述的基本后缀一起使用。

> Note - the K and M suffixes cannot be used in conjunction with the base suffixes mentioned above.

### 3.10.2 符号常量

> 3.10.2 Symbolic Constants

可以通过使用 CONSTANT(*name*) 操作符来引用目标特定的常量，其中 *name* 是下列之一：

> It is possible to refer to target-specific constants via the use of the CONSTANT(name) operator, where name is one of:

#### MAXPAGESIZE

目标的最大页大小。

> The target’s maximum page size.

#### COMMONPAGESIZE

目标的默认页面大小。

> The target’s default page size.

因此，举例来说：

> So for example:

```ld
.text ALIGN (CONSTANT (MAXPAGESIZE)) : { *(.text) }
```

将创建一个与目标支持的最大页边界对齐的代码节。

> will create a text section aligned to the largest page boundary supported by the target.

### 3.10.3 符号名

> 3.10.3 Symbol Names

除非有引号，否则符号名以字母、下划线或点号开始，可以包括字母、数字、下划线、点号和连字符。未加引号的符号名不能与任何关键字冲突。你可以用双引号包围符号名称，指定一个包含特殊字符或与关键字同名的符号：

> Unless quoted, symbol names start with a letter, underscore, or period and may include letters, digits, underscores, periods, and hyphens. Unquoted symbol names must not conflict with any keywords. You can specify a symbol which contains odd characters or has the same name as a keyword by surrounding the symbol name in double quotes:

```ld
"SECTION" = 9;
"with a space" = "also with a space" + 10;
```

由于符号可能包含许多非字母字符，所以用空格来限定符号是最安全的。例如，‘A-B’ 是一个符号，而 ‘A - B’ 是一个涉及减法的表达式。

> Since symbols can contain many non-alphabetic characters, it is safest to delimit symbols with spaces. For example, ‘A-B’ is one symbol, whereas ‘A - B’ is an expression involving subtraction.

### 3.10.4 孤岛节

> 3.10.4 Orphan Sections

孤岛节是指在输入文件中存在，但没有被链接器脚本明确地放到输出文件中的节。链接器仍然会将这些节复制到输出文件中，方法是找到或者创建一个合适的输出节来放置孤岛输入节。

> Orphan sections are sections present in the input files which are not explicitly placed into the output file by the linker script. The linker will still copy these sections into the output file by either finding, or creating a suitable output section in which to place the orphaned input section.

如果孤岛输入节的名称与现有输出节的名称完全匹配，那么孤岛输入节将被放置在该输出节的末尾。

> If the name of an orphaned input section exactly matches the name of an existing output section, then the orphaned input section will be placed at the end of that output section.

如果没有名称匹配的输出节，那么将创建新的输出节。每个新的输出节的名称将与放置在其中的孤岛节相同。如果有多个具有相同名称的孤岛节，这些节将被合并为一个新的输出节。

> If there is no output section with a matching name then new output sections will be created. Each new output section will have the same name as the orphan section placed within it. If there are multiple orphan sections with the same name, these will all be combined into one new output section.

如果创建新的输出节来容纳孤岛输入节，那么链接器必须决定将这些新的输出节放在与现有输出节相关的位置。在大多数现代目标上，链接器试图将孤岛节放在具有相同属性的节之后，例如代码与数据、可加载与不可加载等等。如果没有找到具有匹配属性的节，或者你的目标缺乏这种支持，那么孤岛节就会被放在文件的最后。

> If new output sections are created to hold orphaned input sections, then the linker must decide where to place these new output sections in relation to existing output sections. On most modern targets, the linker attempts to place orphan sections after sections of the same attribute, such as code vs data, loadable vs non-loadable, etc. If no sections with matching attributes are found, or your target lacks this support, the orphan section is placed at the end of the file.

命令行选项‘--orphan-handling’ 和 ‘--unique’（见[命令行选项](https://sourceware.org/binutils/docs/ld/Options.html)）可以用来控制孤岛被放置在哪些输出节。

> The command-line options ‘--orphan-handling’ and ‘--unique’ (see Command-line Options) can be used to control which output sections an orphan is placed in.

### 3.10.5 位置计数器

> 3.10.5 The Location Counter

特殊的链接器变量点‘.’总是包含当前的输出位置计数器。由于‘.’总是指的是输出节的一个位置，所以它只能出现在 SECTIONS 命令的表达式中。‘.’符号可以出现在表达式中允许使用普通符号的任何地方。

> The special linker variable dot ‘.’ always contains the current output location counter. Since the . always refers to a location in an output section, it may only appear in an expression within a SECTIONS command. The . symbol may appear anywhere that an ordinary symbol is allowed in an expression.

为‘.’赋值将导致位置计数器被移动。这可以用来在输出节创建空洞。位置计数器不能在输出节内向后移动，也不能在输出节外向后移动，如果这样做会产生重叠的 LMA 区域。

> Assigning a value to . will cause the location counter to be moved. This may be used to create holes in the output section. The location counter may not be moved backwards inside an output section, and may not be moved backwards outside of an output section if so doing creates areas with overlapping LMAs.

```ld
SECTIONS
{
  output :
    {
      file1(.text)
      . = . + 1000;
      file2(.text)
      . += 1000;
      file3(.text)
    } = 0x12345678;
}
```

在前面的例子中，来自 file1 的‘.text’节位于输出节‘output’的开头。它后面有一个 1000 字节的空白。然后，来自 file2 的‘.text’节出现，在来自 file3 的‘.text’节之前也有一个 1000 字节的空白。符号‘=0x12345678’指定了在空白处写入什么数据（见[输出节填充](#3688-输出节填充)）。

> In the previous example, the ‘.text’ section from file1 is located at the beginning of the output section ‘output’. It is followed by a 1000 byte gap. Then the ‘.text’ section from file2 appears, also with a 1000 byte gap following before the ‘.text’ section from file3. The notation ‘= 0x12345678’ specifies what data to write in the gaps (see Output Section Fill).

注意：‘.’实际上是指从当前包含对象开始的字节偏移。通常这是 SECTIONS 语句，其起始地址为 0，因此‘.’可以作为绝对地址使用。然而，如果‘.’被用在一个节的描述中，它指的是从该节开始的字节偏移，而不是绝对地址。因此，在这样一个脚本中：

> Note: . actually refers to the byte offset from the start of the current containing object. Normally this is the SECTIONS statement, whose start address is 0, hence . can be used as an absolute address. If . is used inside a section description however, it refers to the byte offset from the start of that section, not an absolute address. Thus in a script like this:

```ld
SECTIONS
{
    . = 0x100
    .text: {
      *(.text)
      . = 0x200
    }
    . = 0x500
    .data: {
      *(.data)
      . += 0x600
    }
}
```

‘.text’ 节将被分配一个 0x100 的起始地址和一个正好是 0x200 字节的大小，即使在‘.text’输入节中没有足够的数据来填充这个区域。（如果有太多的数据，将产生一个错误，因为这将是一个向后移动的尝试）。‘.data’节将从 0x500 开始，在‘.data’输入节的值结束后，‘.data’输出节本身结束前，将有一个额外的 0x600 字节的空间。

> The ‘.text’ section will be assigned a starting address of 0x100 and a size of exactly 0x200 bytes, even if there is not enough data in the ‘.text’ input sections to fill this area. (If there is too much data, an error will be produced because this would be an attempt to move . backwards). The ‘.data’ section will start at 0x500 and it will have an extra 0x600 bytes worth of space after the end of the values from the ‘.data’ input sections and before the end of the ‘.data’ output section itself.

如果链接器需要放置孤岛节，在输出节语句之外将符号设置为位置计数器的值会导致意外的值。例如，给出以下内容：

> Setting symbols to the value of the location counter outside of an output section statement can result in unexpected values if the linker needs to place orphan sections. For example, given the following:

```ld
SECTIONS
{
    start_of_text = . ;
    .text: { *(.text) }
    end_of_text = . ;

    start_of_data = . ;
    .data: { *(.data) }
    end_of_data = . ;
}
```

如果链接器需要放置一些脚本中没有提到的输入节，例如 .rodata，它可能会选择将该节放在 .text 和 .data 之间。你可能认为链接器应该把 .rodata 放在上述脚本的空行上，但是空行对链接器来说并没有什么特别的意义。同样，链接器也没有将上述符号名称与它们的节联系起来。相反，它假定所有的赋值或其他语句都属于前面的输出节，除了对‘.’的赋值这种特殊情况。也就是说，链接器会将孤岛 .rodata 节放在脚本中，如同下面的写法一样：

> If the linker needs to place some input section, e.g. .rodata, not mentioned in the script, it might choose to place that section between .text and .data. You might think the linker should place .rodata on the blank line in the above script, but blank lines are of no particular significance to the linker. As well, the linker doesn’t associate the above symbol names with their sections. Instead, it assumes that all assignments or other statements belong to the previous output section, except for the special case of an assignment to .. I.e., the linker will place the orphan .rodata section as if the script was written as follows:

```ld
SECTIONS
{
    start_of_text = . ;
    .text: { *(.text) }
    end_of_text = . ;

    start_of_data = . ;
    .rodata: { *(.rodata) }
    .data: { *(.data) }
    end_of_data = . ;
}
```

这可能是也可能不是脚本作者对 start_of_data 的值的意图。影响孤岛节位置的一个方法是将位置计数器赋值给它自己，因为链接器认为对‘.’的赋值是在设置下面一个输出节的起始地址，因此应该与该节一组。所以你可以写：

> This may or may not be the script author’s intention for the value of start_of_data. One way to influence the orphan section placement is to assign the location counter to itself, as the linker assumes that an assignment to . is setting the start address of a following output section and thus should be grouped with that section. So you could write:

```ld
SECTIONS
{
    start_of_text = . ;
    .text: { *(.text) }
    end_of_text = . ;

    . = . ;
    start_of_data = . ;
    .data: { *(.data) }
    end_of_data = . ;
}
```

现在，孤岛 .rodata 节将被放在 end_of_text 和 start_of_data 之间。

> Now, the orphan .rodata section will be placed between end_of_text and start_of_data.

### 3.10.6 运算符

> 3.10.6 Operators

链接器可以识别标准的 C 语言算术运算符集，并具有标准的结合律和优先级：

> The linker recognizes the standard C set of arithmetic operators, with the standard bindings and precedence levels:

| 优先级 | 结合律 | 运算符 | 备注
|-|-|-|-
|（最高）| | |
| 1  | 左 | ! - ~           | (1)
| 2  | 左 | * / %           |
| 3  | 左 | + -             |
| 4  | 左 | >> <<           |
| 5  | 左 | == != > < <= >= |
| 6  | 左 | &               |
| 7  | 左 | \|              |
| 8  | 左 | &&              |
| 9  | 左 | \|\|            |
| 10 | 右 | ? :             |
| 11 | 右 | &= += -=*= /=   | (2)
|（最低）| | |

注意：(1)前缀运算符 (2)见[为符号赋值](#35-为符号赋值)。

> Notes: (1) Prefix operators (2) See Assigning Values to Symbols.

### 3.10.7 求值

> 3.10.7 Evaluation

链接器对表达式进行懒求值。它只在绝对必要时计算表达式的值。

> The linker evaluates expressions lazily. It only computes the value of an expression when absolutely necessary.

链接器需要一些信息，比如第一个节的起始地址的值，以及内存区域的起始和长度，才能进行任何链接。这些值在链接器读入链接器脚本时被尽快计算出来。

> The linker needs some information, such as the value of the start address of the first section, and the origins and lengths of memory regions, in order to do any linking at all. These values are computed as soon as possible when the linker reads in the linker script.

然而，其他的值（如符号值）在存储分配之后才知道或需要。这些值将在其他信息（比如输出节的大小）可以在符号分配表达式中使用后才求值。

> However, other values (such as symbol values) are not known or needed until after storage allocation. Such values are evaluated later, when other information (such as the sizes of output sections) is available for use in the symbol assignment expression.

节的大小在分配之后才能知道，所以依赖于它们的赋值在分配之后才会执行。

> The sizes of sections cannot be known until after allocation, so assignments dependent upon these are not performed until after allocation.

一些表达式，比如那些依赖于位置计数器‘.’的表达式，必须在节分配期间进行求值。

> Some expressions, such as those depending upon the location counter ‘.’, must be evaluated during section allocation.

如果需要一个表达式的结果时其值仍不可求，那么就会产生一个错误。例如，像下面这样的脚本

> If the result of an expression is required, but the value is not available, then an error results. For example, a script like the following

```ld
SECTIONS
  {
    .text 9+this_isnt_constant :
      { *(.text) }
  }
```

将导致错误信息“初始地址不是常量表达式”。

> will cause the error message ‘non constant expression for initial address’.

### 3.10.8 表达式的节

> 3.10.8 The Section of an Expression

地址和符号可以是相对于节的，也可以是绝对的。一个相对于节的符号是可重定位的。如果你使用‘-r’选项要求可重定位的输出，进一步的链接操作可能会改变相对于节符号的值。另一方面，一个绝对符号将在任何进一步的链接操作中保持相同的值。

> Addresses and symbols may be section relative, or absolute. A section relative symbol is relocatable. If you request relocatable output using the ‘-r’ option, a further link operation may change the value of a section relative symbol. On the other hand, an absolute symbol will retain the same value throughout any further link operations.

链接器表达式中的一些术语是地址。对于相对于节符号和返回地址的内置函数，例如 ADDR、LOADADDR、ORIGIN 和 SEGMENT_START，就是如此。其他术语是简单的数字，或者是返回非地址值的内置函数，如 LENGTH。一个复杂的问题是，除非你设置 LD_FEATURE ("SANE_EXPR")（见[其他链接器脚本命令](#345-其他链接器脚本命令)），否则数字和绝对符号会根据它们的位置被不同的处理，以便与旧版本的 ld 兼容。出现在输出节定义之外的表达式将所有数字视为绝对地址。出现在输出节定义内的表达式将绝对符号作为数字处理。如果设置 LD_FEATURE ("SANE_EXPR")，那么绝对符号和数字在任何地方都被当作数字处理。

> Some terms in linker expressions are addresses. This is true of section relative symbols and for builtin functions that return an address, such as ADDR, LOADADDR, ORIGIN and SEGMENT_START. Other terms are simply numbers, or are builtin functions that return a non-address value, such as LENGTH. One complication is that unless you set LD_FEATURE ("SANE_EXPR") (see Other Linker Script Commands), numbers and absolute symbols are treated differently depending on their location, for compatibility with older versions of ld. Expressions appearing outside an output section definition treat all numbers as absolute addresses. Expressions appearing inside an output section definition treat absolute symbols as numbers. If LD_FEATURE ("SANE_EXPR") is given, then absolute symbols and numbers are simply treated as numbers everywhere.

在下面这个简单的例子中，

> In the following simple example,

```ld
SECTIONS
  {
    . = 0x100;
    __executable_start = 0x100;
    .data :
    {
      . = 0x10;
      __data_start = 0x10;
      *(.data)
    }
    ...
  }
```

在前两次赋值中，‘.’和 \__executable_start 都被设置为绝对地址 0x100，然后在后两次赋值中，. 和 \__data_start 都被设置为相对于 .data 节的 0x10。

> both . and \__executable_start are set to the absolute address 0x100 in the first two assignments, then both . and __data_start are set to 0x10 relative to the .data section in the second two assignments.

对于涉及数字、相对地址和绝对地址的表达式，ld 遵循这些规则求值：

> For expressions involving numbers, relative addresses and absolute addresses, ld follows these rules to evaluate terms:

- 对一个绝对地址或数字进行单目运算，对两个绝对地址或两个数字进行双目运算，或在一个绝对地址和一个数字之间进行双目运算，计算值。
- 对一个相对地址的单目运算，以及对同一节的两个相对地址或一个相对地址与一个数字之间的双目，计算地址的偏移部分。
- 其他双目运算，即在不同节的两个相对地址之间，或在一个相对地址和一个绝对地址之间，在运算之前，首先将任何非绝对地址转换为绝对地址。

> - Unary operations on an absolute address or number, and binary operations on two absolute addresses or two numbers, or between one absolute address and a number, apply the operator to the value(s).
> - Unary operations on a relative address, and binary operations on two relative addresses in the same section or between one relative address and a number, apply the operator to the offset part of the address(es).
> - Other binary operations, that is, between two relative addresses not in the same section, or between a relative address and an absolute address, first convert any non-absolute term to an absolute address before applying the operator.

每个子表达式的结果节如下：

> The result section of each sub-expression is as follows:

- 只涉及数字的运算结果是一个数字。
- 比较、‘&&’和‘||’的结果也是一个数字。
- 当 LD_FEATURE ("SANE_EXPR") 或在输出节定义内时，对同一节的两个相对地址或两个绝对地址（经过上述转换）进行其他双目算术和逻辑操作的结果也是一个数字，否则就是一个绝对地址。
- 对相对地址或一个相对地址和一个数字进行其他操作的结果，是一个与相对参数相同节的相对地址。
- 对绝对地址进行其他操作的结果（经过上述转换）是一个绝对地址。

> - An operation involving only numbers results in a number.
> - The result of comparisons, ‘&&’ and ‘||’ is also a number.
> - The result of other binary arithmetic and logical operations on two relative addresses in the same section or two absolute addresses (after above conversions) is also a number when LD_FEATURE ("SANE_EXPR") or inside an output section definition but an absolute address otherwise.
> - The result of other operations on relative addresses or one relative address and a number, is a relative address in the same section as the relative operand(s).
> - The result of other operations on absolute addresses (after above conversions) is an absolute address.

你可以使用内置函数 ABSOLUTE 来强制表达式为绝对地址，否则它将是相对地址。例如，创建一个绝对符号，设置为输出节‘.data’的末地址：

> You can use the builtin function ABSOLUTE to force an expression to be absolute when it would otherwise be relative. For example, to create an absolute symbol set to the address of the end of the output section ‘.data’:

```ld
SECTIONS
  {
    .data : { *(.data) _edata = ABSOLUTE(.); }
  }
```

如果不使用‘ABSOLUTE’，‘_edata’将是相对于‘.data’节的。

> If ‘ABSOLUTE’ were not used, ‘_edata’ would be relative to the ‘.data’ section.

使用 LOADADDR 也会强制表达式为绝对，因为这个特殊的内置函数返回一个绝对地址。

> Using LOADADDR also forces an expression absolute, since this particular builtin function returns an absolute address.

### 3.10.9 内置函数

> 3.10.9 Builtin Functions

链接器脚本语言包括一些内置函数，用于链接器脚本表达式中。

> The linker script language includes a number of builtin functions for use in linker script expressions.

#### ABSOLUTE(*exp*)

返回表达式 *exp* 的绝对值（不可重定位，不是非负值的意思）。主要用于为节定义中的符号分配一个绝对值，在这里符号值通常是相对于节的。参见[表达式的节](#3108-表达式的节)。

> Return the absolute (non-relocatable, as opposed to non-negative) value of the expression exp. Primarily useful to assign an absolute value to a symbol within a section definition, where symbol values are normally section relative. See The Section of an Expression.

#### ADDR(*section*)

返回用名字表示的 *section* 的地址（VMA）。你的脚本必须事先定义好该节的位置。在下面的例子中，start_of_output_1、symbol_1 和 symbol_2 被分配了相等的值，只不过 symbol_1 是相对于 .output1 节的，而另外两个则是绝对值：

> Return the address (VMA) of the named section. Your script must previously have defined the location of that section. In the following example, start_of_output_1, symbol_1 and symbol_2 are assigned equivalent values, except that symbol_1 will be relative to the .output1 section while the other two will be absolute:

```ld
SECTIONS { ...
  .output1 :
    {
    start_of_output_1 = ABSOLUTE(.);
    ...
    }
  .output :
    {
    symbol_1 = ADDR(.output1);
    symbol_2 = start_of_output_1;
    }
... }
```

#### ALIGN(*align*) ALIGN(*exp*,*align*)

返回位置计数器（.）或任意表达式对齐到下一个 *align* 边界的结果。单目 ALIGN 不改变位置计数器的值--只是用它进行运算。双目 ALIGN 允许任意表达式向上对齐（ALIGN(*align*) 等同于 ALIGN(ABSOLUTE(.), *align*)）。

> Return the location counter (.) or arbitrary expression aligned to the next align boundary. The single operand ALIGN doesn’t change the value of the location counter—it just does arithmetic on it. The two operand ALIGN allows an arbitrary expression to be aligned upwards (ALIGN(align) is equivalent to ALIGN(ABSOLUTE(.), align)).

下面是一个例子，它将 .data 输出节对齐到前一节后的下一个 0x2000 字节边界，并将该节中的一个变量设置到输入节后的下一个 0x8000 边界：

> Here is an example which aligns the output .data section to the next 0x2000 byte boundary after the preceding section and sets a variable within the section to the next 0x8000 boundary after the input sections:

```ld
SECTIONS { ...
  .data ALIGN(0x2000): {
    *(.data)
    variable = ALIGN(0x8000);
  }
... }
```

这个例子中 ALIGN 的第一次使用指定了一个节的位置，因为它被用作节定义的可选地址属性（参见[输出节地址](#363-输出节地址)）。ALIGN 的第二次使用是用来定义一个符号的值。

> The first use of ALIGN in this example specifies the location of a section because it is used as the optional address attribute of a section definition (see Output Section Address). The second use of ALIGN is used to defines the value of a symbol.

内置函数 NEXT 与 ALIGN 密切相关。

> The builtin function NEXT is closely related to ALIGN.

#### ALIGNOF(*section*)

如果用名字表示的 *section* 已经分配，返回该节的对齐方式，用字节表示。如果在求值时该节还没有分配，链接器将报一个错。在下面的例子中，*.output* 节的对齐方式被存储为该节的第一个值。

> Return the alignment in bytes of the named section, if that section has been allocated. If the section has not been allocated when this is evaluated, the linker will report an error. In the following example, the alignment of the .output section is stored as the first value in that section.

```ld
SECTIONS { ...
  .output {
    LONG (ALIGNOF (.output))
    ...
    }
... }
```

#### BLOCK(*exp*)

这是 ALIGN 的同义词，为了与旧的链接器脚本兼容。它最常出现在设置输出节的地址时。

> This is a synonym for ALIGN, for compatibility with older linker scripts. It is most often seen when setting the address of an output section.

#### DATA_SEGMENT_ALIGN(*maxpagesize*, *commonpagesize*)

This is equivalent to either

```ld
(ALIGN(maxpagesize) + (. & (maxpagesize - 1)))
```

or

```ld
(ALIGN(maxpagesize)
 + ((. + commonpagesize - 1) & (maxpagesize - commonpagesize)))
```

depending on whether the latter uses fewer commonpagesize sized pages for the data segment (area between the result of this expression and DATA_SEGMENT_END) than the former or not. If the latter form is used, it means commonpagesize bytes of runtime memory will be saved at the expense of up to commonpagesize wasted bytes in the on-disk file.

This expression can only be used directly in SECTIONS commands, not in any output section descriptions and only once in the linker script. commonpagesize should be less or equal to maxpagesize and should be the system page size the object wants to be optimized for while still running on system page sizes up to maxpagesize. Note however that ‘-z relro’ protection will not be effective if the system page size is larger than commonpagesize.

Example:

```ld
. = DATA_SEGMENT_ALIGN(0x10000, 0x2000);
```

#### DATA_SEGMENT_END(*exp*)

This defines the end of data segment for DATA_SEGMENT_ALIGN evaluation purposes.

```ld
. = DATA_SEGMENT_END(.);

```

#### DATA_SEGMENT_RELRO_END(*offset*, *exp*)

This defines the end of the PT_GNU_RELRO segment when ‘-z relro’ option is used. When ‘-z relro’ option is not present, DATA_SEGMENT_RELRO_END does nothing, otherwise DATA_SEGMENT_ALIGN is padded so that exp + offset is aligned to the commonpagesize argument given to DATA_SEGMENT_ALIGN. If present in the linker script, it must be placed between DATA_SEGMENT_ALIGN and DATA_SEGMENT_END. Evaluates to the second argument plus any padding needed at the end of the PT_GNU_RELRO segment due to section alignment.

```ld
. = DATA_SEGMENT_RELRO_END(24, .);
```

#### DEFINED(*symbol*)

如果符号在链接器全局符号表中，并且在脚本中使用 DEFINED 的语句前定义，则返回 1，否则返回 0。你可以使用这个函数来为符号提供默认值。例如，下面的脚本片段显示了如何将全局符号 ‘begin’ 设置为 ‘.text’ 节的起始地址--但如果一个名为 ‘begin’ 的符号已经存在，其值将被保留。

> Return 1 if symbol is in the linker global symbol table and is defined before the statement using DEFINED in the script, otherwise return 0. You can use this function to provide default values for symbols. For example, the following script fragment shows how to set a global symbol ‘begin’ to the first location in the ‘.text’ section—but if a symbol called ‘begin’ already existed, its value is preserved:

```ld
SECTIONS { ...
  .text : {
    begin = DEFINED(begin) ? begin : . ;
    ...
  }
  ...
}
```

#### LENGTH(*memory*)

返回名为 *memory* 的内存区域的长度。

> Return the length of the memory region named memory.

#### LOADADDR(*section*)

返回名为 *section* 的节的绝对 LMA。（见[输出节 LMA](#3682-输出节-lma)）。

> Return the absolute LMA of the named section. (see Output Section LMA).

#### LOG2CEIL(*exp*)

返回 *exp* 以 2 为底的对数，向上取整。LOG2CEIL(0) 返回 0。

> Return the binary logarithm of exp rounded towards infinity. LOG2CEIL(0) returns 0.

#### MAX(*exp1*, *exp2*)

返回 *exp1* 和 *exp2* 的最大值。

> Returns the maximum of exp1 and exp2.

#### MIN(*exp1*, *exp2*)

返回 *exp1* 和 *exp2* 的最小值。

> Returns the minimum of exp1 and exp2.

#### NEXT(*exp*)

返回下一个未分配的、是 *exp* 的倍数的地址。这个函数与 ALIGN(*exp*) 近似；除非你使用 MEMORY 命令为输出文件定义不连续的内存，否则这两个函数是等同的。

> Return the next unallocated address that is a multiple of exp. This function is closely related to ALIGN(exp); unless you use the MEMORY command to define discontinuous memory for the output file, the two functions are equivalent.

#### ORIGIN(*memory*)

返回名为 *memory* 的内存区域的起点。

> Return the origin of the memory region named memory.

#### SEGMENT_START(*segment*, *default*)

返回名为 *segment* 的段的基址。如果这个段已经有一个明确的值（通过命令行的‘-T’选项），那么这个值将被返回，否则这个值设为 *default*。目前，‘-T’命令行选项只能用于设置“text”、“data”和“bss”段的基址，但你可以对任何段名使用 SEGMENT_START。

> Return the base address of the named segment. If an explicit value has already been given for this segment (with a command-line ‘-T’ option) then that value will be returned otherwise the value will be default. At present, the ‘-T’ command-line option can only be used to set the base address for the “text”, “data”, and “bss” sections, but you can use SEGMENT_START with any segment name.

#### SIZEOF(*section*)

如果名为 *section* 的节已被分配，则返回该节以字节为单位的大小。如果在求值时该节还没有被分配，链接器将报一个错。在下面的例子中，*symbol_1* 和 *symbol_2* 被分配了相同的值：

> Return the size in bytes of the named section, if that section has been allocated. If the section has not been allocated when this is evaluated, the linker will report an error. In the following example, symbol_1 and symbol_2 are assigned identical values:

```ld
SECTIONS { ...
  .output {
    .start = . ;
    ...
    .end = . ;
    }
  symbol_1 = .end - .start ;
  symbol_2 = SIZEOF(.output);
... }
```

#### SIZEOF_HEADERS

返回输出文件头的大小，以字节为单位。这是出现在输出文件开头的信息。你可以在设置第一个节的起始地址时使用这个数字，以方便分页。

> Return the size in bytes of the output file’s headers. This is information which appears at the start of the output file. You can use this number when setting the start address of the first section, if you choose, to facilitate paging.

在制作 ELF 输出文件时，如果链接器脚本使用 SIZEOF_HEADERS 内置函数，链接器必须在确定所有节的地址和大小之前计算出程序头的数量。如果链接器后来发现它需要额外的程序头，它将报告一个错误‘not enough room for program headers’。为了避免这个错误，你必须避免使用 SIZEOF_HEADERS 函数，或者你必须重新编写你的链接器脚本以避免强迫链接器使用额外的程序头，或者你必须使用 PHDRS 命令自己定义程序头（参见 [PHDRS 命令](#38-phdrs-命令)）。

> When producing an ELF output file, if the linker script uses the SIZEOF_HEADERS builtin function, the linker must compute the number of program headers before it has determined all the section addresses and sizes. If the linker later discovers that it needs additional program headers, it will report an error ‘not enough room for program headers’. To avoid this error, you must avoid using the SIZEOF_HEADERS function, or you must rework your linker script to avoid forcing the linker to use additional program headers, or you must define the program headers yourself using the PHDRS command (see PHDRS Command).

## 3.11 隐式链接器脚本

> 3.11 Implicit Linker Scripts

如果你指定了一个链接器输入文件，而链接器不能将其识别为对象文件或归档文件，它将尝试将该文件作为一个链接器脚本来读取。如果该文件不能被解析为链接器脚本，链接器将报告一个错误。

> If you specify a linker input file which the linker can not recognize as an object file or an archive file, it will try to read the file as a linker script. If the file can not be parsed as a linker script, the linker will report an error.

隐式链接器脚本不会取代默认的链接器脚本。

> An implicit linker script will not replace the default linker script.

通常情况下，隐式链接器脚本只包含符号赋值，或 INPUT、GROUP 或 VERSION 命令。

> Typically an implicit linker script would contain only symbol assignments, or the INPUT, GROUP, or VERSION commands.

由于隐式链接器脚本而读取的任何输入文件将从命令行中读取隐式链接器脚本的位置读取。这可能会影响归档搜索。

> Any input files read because of an implicit linker script will be read at the position in the command line where the implicit linker script was read. This can affect archive searching.
