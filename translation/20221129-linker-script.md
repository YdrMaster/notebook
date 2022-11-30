# 3 链接器脚本

> 3 Linker Scripts

> [原文](https://sourceware.org/binutils/docs/ld/Scripts.html)

每次链接都是由一个链接器脚本控制的。这个脚本是用链接器命令语言编写的。

> Every link is controlled by a linker script. This script is written in the linker command language.

链接器脚本的主要功能是描述输入文件中的段应该如何被映射到输出文件中，并控制输出文件的内存布局。大多数链接器脚本的作用不外乎如此。然而，在必要的时候，链接器脚本也可以使用下面描述的命令，指导链接器执行许多其他操作。

> The main purpose of the linker script is to describe how the sections in the input files should be mapped into the output file, and to control the memory layout of the output file. Most linker scripts do nothing more than this. However, when necessary, the linker script can also direct the linker to perform many other operations, using the commands described below.

链接器总是使用一个链接器脚本。如果你没有自己提供一个，链接器将使用一个默认的脚本，它被编译到链接器的可执行文件中。你可以使用 `--verbose` 命令行选项来显示默认的链接器脚本。某些命令行选项，如 `-r` 或 `-N`，将影响默认的链接器脚本。

> The linker always uses a linker script. If you do not supply one yourself, the linker will use a default script that is compiled into the linker executable. You can use the ‘--verbose’ command-line option to display the default linker script. Certain command-line options, such as ‘-r’ or ‘-N’, will affect the default linker script.

你可以通过使用 `-T` 命令行选项提供你自己的链接器脚本。当你这样做时，你的链接器脚本将取代默认的链接器脚本。

> You may supply your own linker script by using the ‘-T’ command line option. When you do this, your linker script will replace the default linker script.

你也可以隐式地使用链接器脚本，把它们命名为链接器的输入文件，就像它们是要被链接的文件一样。参见[隐式链接器脚本]()。

> You may also use linker scripts implicitly by naming them as input files to the linker, as though they were files to be linked. See Implicit Linker Scripts.

- [链接器脚本的基本概念](#31-链接器脚本的基本概念)
- [链接器脚本格式](#32-链接器脚本格式)
- [简单的链接器脚本示例](#33-简单的链接器脚本示例)
- [简单的链接器脚本命令](#34-简单的链接器脚本命令)
- [为符号赋值](#35-为符号赋值)
- [SECTIONS 命令](#36-sections-命令)
- [MEMORY 命令]()
- [PHDRS 命令]()
- [VERSION 命令]()
- [链接器脚本中的表达式]()
- [隐式链接器脚本]()

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

链接器将输入文件合并成一个输出文件。输出文件和每个输入文件都采用一种特殊的数据格式，称为对象文件格式。每个文件都被称为一个对象文件。输出文件通常被称为可执行文件，但为了我们的目的，我们也将其称为对象文件。所有对象文件都有一个段表。我们有时把输入文件中的一个段称为输入段；类似地，输出文件中的一个段是输出段。

> The linker combines input files into a single output file. The output file and each input file are in a special data format known as an object file format. Each file is called an object file. The output file is often called an executable, but for our purposes we will also call it an object file. Each object file has, among other things, a list of sections. We sometimes refer to a section in an input file as an input section; similarly, a section in the output file is an output section.

一个对象文件中的每个段都有一个名称和一个大小。大多数段也有一个相关的数据块，被称为段内容。一个段可以被标记为可加载，这意味着当输出文件被运行时，其内容应该被加载到内存中。一个没有内容的段可能是可分配的，这意味着内存中的一个区域应该被预留出来，但没有特别的东西应该被加载到那里（在某些情况下，这个内存必须被清空）。一个既非可加载也非可分配的段通常包含某种调试信息。

> Each section in an object file has a name and a size. Most sections also have an associated block of data, known as the section contents. A section may be marked as loadable, which means that the contents should be loaded into memory when the output file is run. A section with no contents may be allocatable, which means that an area in memory should be set aside, but nothing in particular should be loaded there (in some cases this memory must be zeroed out). A section which is neither loadable nor allocatable typically contains some sort of debugging information.

每个可加载或可分配的输出段都有两个地址。第一个是 VMA，即虚拟内存地址。这是输出文件运行时该段的地址。第二个是 LMA，即加载内存地址。这是该段将被加载的地址。在大多数情况下，这两个地址是相同的。一个可能不同的例子是，数据段被加载到 ROM 中，然后在程序启动时被复制到 RAM 中（这种技术经常被用来初始化基于 ROM 的系统中的全局变量）。在这种情况下，ROM 地址是 LMA，而 RAM 地址是 VMA。

> Every loadable or allocatable output section has two addresses. The first is the VMA, or virtual memory address. This is the address the section will have when the output file is run. The second is the LMA, or load memory address. This is the address at which the section will be loaded. In most cases the two addresses will be the same. An example of when they might be different is when a data section is loaded into ROM, and then copied into RAM when the program starts up (this technique is often used to initialize global variables in a ROM based system). In this case the ROM address would be the LMA, and the RAM address would be the VMA.

你可以通过使用带有 `-h` 选项的 `objdump` 程序看到对象文件中的各个段。

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

SECTIONS 是一个强大的命令。这里我们将描述它的一个简单用法。假设你的程序只包括代码、初始化数据和未初始化数据。这些将分别在 `.text`、`.data` 和 `.bss` 段。让我们进一步假设，你的输入文件只有这些段。

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

你把 SECTIONS 命令写成关键词 SECTIONS，后面是一系列的符号赋值和用大括号括起来的输出段描述。

> You write the ‘SECTIONS’ command as the keyword ‘SECTIONS’, followed by a series of symbol assignments and output section descriptions enclosed in curly braces.

上面例子中 SECTIONS 命令里面的第一行设置了特殊符号 `.` 的值，这是位置计数器。如果你没有以其他方式指定输出段的地址（其他方式将在后面描述），地址将从位置计数器的当前值开始设置。然后，位置计数器按输出段的大小递增。在 SECTIONS 命令的开始，位置计数器的值为 `0`。

> The first line inside the ‘SECTIONS’ command of the above example sets the value of the special symbol ‘.’, which is the location counter. If you do not specify the address of an output section in some other way (other ways are described later), the address is set from the current value of the location counter. The location counter is then incremented by the size of the output section. At the start of the ‘SECTIONS’ command, the location counter has the value ‘0’.

第二行定义了一个输出段，`.text`。冒号是必要的语法，现在可以忽略。在输出段名称后面的大括号内，你列出了应该被放入这个输出段的输入段的名称。`*` 是一个通配符，可以匹配任何文件名。表达式 `*(.text)` 意味着所有输入文件中的所有 `.text` 输入段。

> The second line defines an output section, ‘.text’. The colon is required syntax which may be ignored for now. Within the curly braces after the output section name, you list the names of the input sections which should be placed into this output section. The ‘*’ is a wildcard which matches any file name. The expression ‘*(.text)’ means all ‘.text’ input sections in all input files.

由于定义输出段 `.text` 时位置计数器是 `0x10000`，链接器将把输出文件中 `.text` 段的地址设为 `0x10000`。

> Since the location counter is ‘0x10000’ when the output section ‘.text’ is defined, the linker will set the address of the ‘.text’ section in the output file to be ‘0x10000’.

剩下的几行定义了输出文件中的 `.data` 和 `.bss` 段。链接器将把 `.data` 输出段放在地址 `0x8000000` 处。在链接器放置 `.data` 输出段后，位置计数器的值将是 `0x8000000` 加上 `.data` 输出段的大小。其结果是链接器将在内存中的 `.data` 输出段之后立即放置 `.bss` 输出段。

> The remaining lines define the ‘.data’ and ‘.bss’ sections in the output file. The linker will place the ‘.data’ output section at address ‘0x8000000’. After the linker places the ‘.data’ output section, the value of the location counter will be ‘0x8000000’ plus the size of the ‘.data’ output section. The effect is that the linker will place the ‘.bss’ output section immediately after the ‘.data’ output section in memory.

链接器将确保每个输出段都有必要的对齐方式，如果有必要的话，可以增加位置计数器。在这个例子中，为 `.text` 和 `.data` 段指定的地址可能会满足任何对齐限制，但是链接器可能要在 `.data` 和 `.bss` 段之间制造一个小间隙。

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
- 代码段的第一个字节的地址，如果存在并且正在创建一个可执行文件的话——代码段通常是 `.text`，但也可能是其他东西；
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

你可以把 INCLUDE 指令放在顶层、MEMORY 或 SECTIONS 命令中、或输出段s的描述中。

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

OUTPUT 命令命名输出文件。在链接器脚本中使用 OUTPUT(*filename*) 与在命令行中使用 `-o filename` 完全一样（见命令行选项）。如果两者都使用，则以命令行选项为准。

> The OUTPUT command names the output file. Using OUTPUT(filename) in the linker script is exactly like using ‘-o filename’ on the command line (see Command Line Options). If both are used, the command-line option takes precedence.

你可以使用 OUTPUT 命令为输出文件定义一个默认的名字，而不是通常默认的 a.out。

#### SEARCH_DIR(*path*)

SEARCH_DIR 命令将路径添加到 ld 寻找存档库的路径列表中。使用 SEARCH_DIR(*path*) 就像在命令行中使用 `-L path` 一样（见命令行选项）。如果两者都使用，那么链接器将搜索两个路径。使用命令行选项指定的路径将首先被搜索。

> The SEARCH_DIR command adds path to the list of paths where ld looks for archive libraries. Using SEARCH_DIR(path) is exactly like using ‘-L path’ on the command line (see Command-line Options). If both are used, then the linker will search both paths. Paths specified using the command-line option are searched first.

#### STARTUP(*filename*)

STARTUP 命令与 INPUT 命令一样，只是文件名将成为第一个被链接的输入文件，就像它在命令行中被首先指定一样。这在使用系统时可能很有用，因为系统的入口点总是第一个文件的开始。

> The STARTUP command is just like the INPUT command, except that filename will become the first input file to be linked, as though it were specified first on the command line. This may be useful when using a system in which the entry point is always the start of the first file.

### 3.4.3 处理对象文件格式的命令

> 3.4.3 Commands Dealing with Object File Formats

有几个链接器脚本命令是处理对象文件格式的。

> A couple of linker script commands deal with object file formats.

#### OUTPUT_FORMAT(*bfdname*), OUTPUT_FORMAT(*default*, *big*, *little*)

OUTPUT_FORMAT 命令命名了用于输出文件的 BFD 格式（参见 [BFD](https://sourceware.org/binutils/docs/ld/BFD.html)）。使用 OUTPUT_FORMAT(*bfdname*) 与在命令行上使用 `--oformat bfdname` 完全一样（见命令行选项）。如果两者都使用，则以命令行选项为准。

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

TARGET 命令命名了读取输入文件时要使用的 BFD 格式。它影响到后续的 INPUT 和 GROUP 命令。这个命令就像在命令行上使用 `-b bfdname` 一样（见命令行选项）。如果使用了 TARGET 命令，但没有使用 OUTPUT_FORMAT，那么最后一条 TARGET 命令也会用来设置输出文件的格式。参见 [BFD](https://sourceware.org/binutils/docs/ld/BFD.html)。

> The TARGET command names the BFD format to use when reading input files. It affects subsequent INPUT and GROUP commands. This command is like using ‘-b bfdname’ on the command line (see Command-line Options). If the TARGET command is used but OUTPUT_FORMAT is not, then the last TARGET command is also used to set the format for the output file. See BFD.

### 3.4.4 为内存区域指定别名

> 3.4.4 Assign alias names to memory regions

可以向 [MEMORY 命令]() 创建的内存区域添加别名。每个名字最多对应一个内存区域。

> Alias names can be added to existing memory regions created with the MEMORY Command command. Each name corresponds to at most one memory region.

#### REGION_ALIAS(*alias*, *region*)

REGION_ALIAS 函数为内存区域 *region* 创建一个别名 *alias*。这允许灵活地将输出段映射到内存区域。下面是一个例子。

> The REGION_ALIAS function creates an alias name alias for the memory region region. This allows a flexible mapping of output sections to memory regions. An example follows.

假设我们有一个应用，适用于带有各种内存外存设备的嵌入式系统。所有的（嵌入式系统）都有一个通用的、易失性的内存 RAM，允许代码执行或数据存储。有些可能有一个只读的、非易失性内存 ROM，允许代码执行和只读数据访问。最后一种变体是只读、非易失性存储器 ROM2，支持只读数据访问但无代码执行能力。我们有四个输出段：

> Suppose we have an application for embedded systems which come with various memory storage devices. All have a general purpose, volatile memory RAM that allows code execution or data storage. Some may have a read-only, non-volatile memory ROM that allows code execution and read-only data access. The last variant is a read-only, non-volatile memory ROM2 with read-only data access and no code execution capability. We have four output sections:

- *.text* 程序代码；
- *.rodata* 只读数据；
- *.data* 读写已初始化数据；
- *.bss* 读写零初始化数据。

> - .text program code;
> - .rodata read-only data;
> - .data read-write initialized data;
> - .bss read-write zero initialized data.

我们的目标是提供一个链接器命令文件，其中包含一个定义输出段的系统无关部分和一个特定于系统的部分，将输出段映射到系统上可用的内存区域。我们的嵌入式系统有三种不同的内存设置 A、B 和 C。

> The goal is to provide a linker command file that contains a system independent part defining the output sections and a system dependent part mapping the output sections to the memory regions available on the system. Our embedded systems come with three different memory setups A, B and C:

| 段    | 变体 A | 变体 B  | 变体 C
|---------|-----|---------|-
| .text   | RAM | ROM     |  ROM
| .rodata | RAM | ROM     |  ROM2
| .data   | RAM | RAM/ROM |  RAM/ROM2
| .bss    | RAM | RAM     |  RAM

RAM/ROM 或 RAM/ROM2 的写法表示该段被分别加载到 ROM 或 ROM2 区域。请注意，*.data* 段的加载地址在所有三个变体中都是从 *.rodata* 段的末尾开始的。

> The notation RAM/ROM or RAM/ROM2 means that this section is loaded into region ROM or ROM2 respectively. Please note that the load address of the .data section starts in all three variants at the end of the .rodata section.

处理输出段的基本链接器脚本如下。它包含了特定于系统的 linkcmds.memory 文件，描述了内存布局。

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

可以写一个通用的系统初始化例程，在必要时将 *.data* 段从 ROM 或 ROM2 复制到 RAM 中：

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

注意，断言在链接的最后阶段发生之前被检查。这意味着用户必须为段定义内标记 PROVIDE 的符号设置值，否则涉及这些符号的表达式将会失败。这条规则的唯一例外是只引用点号的 PROVIDE 符号。因此，像这样的断言：

> Note that assertions are checked before the final stages of linking take place. This means that expressions involving symbols PROVIDEd inside section definitions will fail if the user has not set values for those symbols. The only exception to this rule is PROVIDEd symbols that just reference dot. Thus an assertion like this:

```ld
.stack :
{
  PROVIDE (__stack = .);
  PROVIDE (__stack_size = 0x100);
  ASSERT ((__stack > (_end + __stack_size)), "Error: No room left for the stack");
}
```

将会失败，只要 `__stack_size` 没有在其他地方定义。在段定义之外的符号更早求值，所以它们可以在 ASSERT 中使用。因此：

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

强制将符号作为一个未定义的符号输入到输出文件中。这样做可能会，例如，触发连接标准库中的额外模块。你可以为每个EXTERN列出几个符号，而且你可以多次使用EXTERN。这个命令与'-u'命令行选项的效果相同。

> Force symbol to be entered in the output file as an undefined symbol. Doing this may, for example, trigger linking of additional modules from standard libraries. You may list several symbols for each EXTERN, and you may use EXTERN multiple times. This command has the same effect as the ‘-u’ command-line option.

#### FORCE_COMMON_ALLOCATION

这条命令与 `-d` 命令行选项的效果相同：即使指定了可重定位的输出文件（`-r`），也要让 ld 将空间分配给普通符号。

> This command has the same effect as the ‘-d’ command-line option: to make ld assign space to common symbols even if a relocatable output file is specified (‘-r’).

#### INHIBIT_COMMON_ALLOCATION

这个命令与 `--no-define-common` 命令行选项的效果相同：即使是对不可重定位的输出文件，也禁止 ld 将空间分配给普通符号。

> This command has the same effect as the ‘--no-define-common’ command-line option: to make ld omit the assignment of addresses to common symbols even for a non-relocatable output file.

#### FORCE_GROUP_ALLOCATION

这个命令与 `--force-group-allocation` 命令行选项的效果相同：使 ld 像放置正常的输入段一样放置段组成员，并删除区段，即使指定了可重定位的输出文件（`-r`）。

> This command has the same effect as the ‘--force-group-allocation’ command-line option: to make ld place section group members like normal input sections, and to delete the section groups even if a relocatable output file is specified (‘-r’).

#### INSERT [ AFTER | BEFORE ] *output_section*

这条命令通常用在由 `-T` 指定的脚本中，以增加默认的 SECTIONS，例如，叠加。它在 *output_section* 之后（或之前）插入所有先前的链接器脚本语句，并使 `-T` 不覆盖默认的链接器脚本。确切的插入点与无主段相同。参见位置计数器。插入发生在链接器将输入段映射到输出段之后。在插入之前，由于 `-T` 脚本是在默认链接器脚本之前解析的，因此在脚本的内部链接器表示中，`-T` 脚本中的语句出现在默认链接器脚本的语句之前。特别是，输入段将先分配给 `-T` 输出段，后分配给默认脚本。下面是一个使用 INSERT 的 `-T` 脚本的例子：

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

这个命令可以用来告诉 ld 对特定输出段中的任何引用报错。

> This command may be used to tell ld to issue an error about any references among certain output sections.

在某些类型的程序中，特别是在嵌入式系统中使用覆盖时，当一个段被加载到内存中时，另一个段一定不会被加载。这两个段之间的任何直接引用都是错误的。例如，如果一个段的代码调用了另一个段定义的函数，将产生一个错误。

> In certain types of programs, particularly on embedded systems when using overlays, when one section is loaded into memory, another section will not be. Any direct references between the two sections would be errors. For example, it would be an error if code in one section called a function defined in the other section.

NOCROSSREFS 命令接收一个输出段名称的列表。如果 ld 检测到这些段之间有任何交叉引用，它会报告一个错误并返回一个非零的退出状态。注意，NOCROSSREFS 命令使用的是输出段的名称，而不是输入段的名称。

> The NOCROSSREFS command takes a list of output section names. If ld detects any cross references between the sections, it reports an error and returns a non-zero exit status. Note that the NOCROSSREFS command uses output section names, not input section names.

#### NOCROSSREFS_TO(*tosection* *fromsection* ...)

这条命令可以用来告诉 ld，从段列表中的某个段发现对指定段的任何引用时产生一个错误。

> This command may be used to tell ld to issue an error about any references to one section from a list of other sections.

NOCROSSREFS 命令在确保两个或以上输出段完全独立时非常有用，但也有需要单向依赖的情况。例如，在一个多核应用程序中，可能有一些共享代码可以从每个核中调用，但为了安全起见，决不能回调。

> The NOCROSSREFS command is useful when ensuring that two or more output sections are entirely independent but there are situations where a one-way dependency is needed. For example, in a multi-core application there may be shared code that can be called from each core but for safety must never call back.

NOCROSSREFS_TO 命令需要一个输出段的名称列表。第一个段不能被其他任何段引用。如果 ld 检测到任何其他段对第一个段的引用，它会报告一个错误并返回一个非零的退出状态。注意，NOCROSSREFS_TO 命令使用的是输出段的名字，而不是输入段的名字。

> The NOCROSSREFS_TO command takes a list of output section names. The first section can not be referenced from any of the other sections. If ld detects any references to the first section from any of the other sections, it reports an error and returns a non-zero exit status. Note that the NOCROSSREFS_TO command uses output section names, not input section names.

#### OUTPUT_ARCH(*bfdarch*)

指定一个特定的输出机架构。参数是 BFD 库使用的名称之一（参见 [BFD](https://sourceware.org/binutils/docs/ld/BFD.html)）。你可以通过使用带有 `-f` 选项的 objdump 程序来查看一个对象文件的架构。

> Specify a particular output machine architecture. The argument is one of the names used by the BFD library (see BFD). You can see the architecture of an object file by using the objdump program with the ‘-f’ option.

#### LD_FEATURE(*string*)

这个命令可以用来修改 ld 的行为。如果字符串是 `SANE_EXPR`，那么脚本中的绝对符号和数字就会被当作数字处理。参见[表达式的段]()。

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

特殊符号名称 `.` 表示位置计数器。你只能在 SECTIONS 命令中使用它。请参阅[位置计数器]()。

> The special symbol name ‘.’ indicates the location counter. You may only use this within a SECTIONS command. See The Location Counter.

表达式后面的分号是必须的。

> The semicolon after expression is required.

表达式的定义在下面，请看[链接器脚本中的表达式]()。

> Expressions are defined below; see Expressions in Linker Scripts.

你可以把符号赋值作为命令本身来写，也可以作为 SECTIONS 命令中的语句来写，或者作为 SECTIONS 命令中输出段描述的一部分。

> You may write symbol assignments as commands in their own right, or as statements within a SECTIONS command, or as part of an output section description in a SECTIONS command.

符号的段将根据表达式的段来设置，更多信息请参见[表达式的段]()。

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

在这个例子中，符号 `floating_point` 将被定义为零。符号 `_etext` 将被定义为最后一个 *.text* 输入段之后的地址。符号 `_bdata` 将被定义为 *.text* 输出段之后的地址，向上对齐到 4 字节的边界。

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

注意 - PROVIDE 直接认为每个普通的符号是已定义的，即使这样的符号可以与 PROVIDE 将创建的符号相结合。在考虑构造函数和析构函数列表符号时，这一点特别重要，例如 `__CTOR_LIST__`，因为这些符号经常被定义为普通符号。

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

SECTIONS 命令告诉链接器如何将输入段映射到输出部分，以及如何将输出段放在内存中。

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
- 一个输出段的描述
- 一个叠加描述

> - an ENTRY command (see Entry command)
> - a symbol assignment (see Assigning Values to Symbols)
> - an output section description
> - an overlay description

为了方便在这些命令中使用位置计数器，ENTRY 命令和符号赋值被允许放在 SECTIONS 命令中。这也可以使链接器脚本更容易理解，因为你可以在输出文件的布局中有意义的点上使用这些命令。

> The ENTRY command and symbol assignments are permitted inside the SECTIONS command for convenience in using the location counter in those commands. This can also make the linker script easier to understand because you can use those commands at meaningful points in the layout of the output file.

输出段的描述和覆盖段的描述将在下面描述。

> Output section descriptions and overlay descriptions are described below.

如果你在链接器脚本中没有使用 SECTIONS 命令，链接器将按照输入文件中的段第一次遇到的顺序，把每个输入段放入一个相同名称的输出段。例如，如果所有的输入段都出现在第一个文件中，那么输出文件中各段的顺序将与第一个输入文件中的顺序一致。第一个段将在地址 0 处。

> If you do not use a SECTIONS command in your linker script, the linker will place each input section into an identically named output section in the order that the sections are first encountered in the input files. If all input sections are present in the first file, for example, the order of sections in the output file will match the order in the first input file. The first section will be at address zero.

- [输出段描述](#361-输出段描述)
- [输出段名称](#362-输出段名称)
- [输出段地址](#363-输出段地址)
- [输入段描述](#364-输入段描述)
- [输出段数据]()
- [输出段关键字]()
- [输出段丢弃]()
- [输出段属性]()
- [叠加描述]()

> - Output Section Description
> - Output Section Name
> - Output Section Address
> - Input Section Description
> - Output Section Data
> - Output Section Keywords
> - Output Section Discarding
> - Output Section Attributes
> - Overlay Description

### 3.6.1 输出段描述

> 3.6.1 Output Section Description

一个输出段的完整描述看起来像这样：

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
    …
  } [>region] [AT>lma_region] [:phdr :phdr …] [=fillexp] [,]
```

大多数输出段不使用大多数可选的段属性。

> Most output sections do not use most of the optional section attributes.

*section* 周围的空白是必须的，这样一来，*section* 的名字就不会有歧义了。冒号和大括号也是必须的。如果使用了 *fillexp*，并且下一个 *sections-command* 看起来像这个表达式的延续，那么末尾的逗号可能是必须的。换行符和其他空白是可选的。

> The whitespace around section is required, so that the section name is unambiguous. The colon and the curly braces are also required. The comma at the end may be required if a fillexp is used and the next sections-command looks like a continuation of the expression. The line breaks and other white space are optional.

每个 *output-section-command* 可以是以下的一种：

> Each output-section-command may be one of the following:

- 符号赋值（参见[为符号赋值](#35-为符号赋值)）
- 一个输入段描述（见[输入段描述](#364-输入段描述)）
- 直接包括的数据值（见[输出段数据]()）
- 一个特殊的输出段关键字（见[输出段关键字]()）。

> - a symbol assignment (see Assigning Values to Symbols)
> - an input section description (see Input Section Description)
> - data values to include directly (see Output Section Data)
> - a special output section keyword (see Output Section Keywords)

### 3.6.2 输出段名称

> 3.6.2 Output Section Name

输出段的名称是 *section*。*section* 必须符合你的输出格式的限制。在只支持有限数量的段的格式中，比如 a.out，名称必须是格式所支持的名称之一（比如a.out，只允许 *.text*、*.data* 或 *.bss*）。如果输出格式支持任何数量的段，但使用数字而不是名称（如 Oasys），那么名称应该以带引号的数字字符串提供。一个段的名称可以由任何字符序列组成，但包含任何不寻常字符的名称，如逗号，必须加引号。

> The name of the output section is section. section must meet the constraints of your output format. In formats which only support a limited number of sections, such as a.out, the name must be one of the names supported by the format (a.out, for example, allows only ‘.text’, ‘.data’ or ‘.bss’). If the output format supports any number of sections, but with numbers and not names (as is the case for Oasys), the name should be supplied as a quoted numeric string. A section name may consist of any sequence of characters, but a name which contains any unusual characters such as commas must be quoted.

输出段的名称 */DISCARD/* 是特殊的；（表示）输出段丢弃。

> The output section name ‘/DISCARD/’ is special; Output Section Discarding.

### 3.6.3 输出段地址

> 3.6.3 Output Section Address

该地址是输出段的 VMA（虚拟内存地址）的表达式。这个地址是可选的，但是如果它存在，那么输出地址将被精确地设置为指定的地址。

> The address is an expression for the VMA (the virtual memory address) of the output section. This address is optional, but if it is provided then the output address will be set exactly as specified.

如果没有指定输出地址，那么将根据下面的启发式方法，为该部分选择一个地址。这个地址将被调整以适应输出段的对齐要求。对齐要求是指输出段所包含的任何输入段的最严格对齐。

> If the output address is not specified then one will be chosen for the section, based on the heuristic below. This address will be adjusted to fit the alignment requirement of the output section. The alignment requirement is the strictest alignment of any input section contained within the output section.

输出段的地址启发法如下：

> The output section address heuristic is as follows:

如果为该段设置了一个输出内存区域，那么它将被添加到该区域，其地址将是该区域的下一个空闲地址。

> If an output memory region is set for the section then it is added to this region and its address will be the next free address in that region.

如果 MEMORY 命令被用来创建一个内存区域的列表，那么会选择第一个与段的属性兼容的区域来包含它。该段的输出地址将是该区域的下一个空闲地址；[MEMORY 命令]()。

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

是有细微差别的。第一个将把 *.text* 输出段的地址设置为位置计数器的当前值。第二种将把它设置为位置计数器的当前值，并对齐到所有 *.text* 输入段的最严格对齐。

> are subtly different. The first will set the address of the ‘.text’ output section to the current value of the location counter. The second will set it to the current value of the location counter aligned to the strictest alignment of any of the ‘.text’ input sections.

该地址可以是一个任意的表达式；[链接器脚本中的表达式]()。例如，如果你想在 0x10 字节的边界上对齐该段，使该段地址的最低 4 位为 0，你可以这样做：

> The address may be an arbitrary expression; Expressions in Linker Scripts. For example, if you want to align the section on a 0x10 byte boundary, so that the lowest four bits of the section address are zero, you could do something like this:

```ld
.text ALIGN(0x10) : { *(.text) }
```

这是有效的，原因是 ALIGN 返回当前位置的计数器，并向上对齐到指定的值。

> This works because ALIGN returns the current location counter aligned upward to the specified value.

为一个区段指定地址将改变位置计数器的值，只要该区段不是空的（空的区段会被忽略）。

> Specifying address for a section will change the value of the location counter, provided that the section is non-empty. (Empty sections are ignored).

### 3.6.4 输入段描述

> 3.6.4 Input Section Description

最常见的输出命令段是输入段描述。

> The most common output section command is an input section description.

输入段描述是最基本的链接器脚本操作。你用输出段来告诉链接器如何在内存中布局你的程序。你使用输入部分描述来告诉链接器如何将输入文件映射到你的内存布局中。

> The input section description is the most basic linker script operation. You use output sections to tell the linker how to lay out your program in memory. You use input section descriptions to tell the linker how to map the input files into your memory layout.

- [输入段基础知识](#3641-输入段基础知识)
- [输入段通配符模式]()
- [常见符号的输入段]()
- [输入段和垃圾收集]()
- [输入段示例]()

> - Input Section Basics
> - Input Section Wildcard Patterns
> - Input Section for Common Symbols
> - Input Section and Garbage Collection
> - Input Section Example

#### 3.6.4.1 输入段基础知识

> 3.6.4.1 Input Section Basics

一个输入段的描述由一个文件名组成，并可选后缀一个段名列表，用括号括起。

> An input section description consists of a file name optionally followed by a list of section names in parentheses.

文件名和段名可以是通配符模式，我们将在下面进一步描述（见[输入段通配符模式]()）。

> The file name and the section name may be wildcard patterns, which we describe further below (see Input Section Wildcard Patterns).

最常见的输入段描述是在输出段包括所有具有特定名称的输入段。例如，要包括所有输入的 *.text* 段，你可以这样写：

> The most common input section description is to include all input sections with a particular name in the output section. For example, to include all input ‘.text’ sections, you would write:

```ld
*(.text)
```

这里的 `*` 是一个通配符，可以匹配任何文件名。要排除一个与文件名通配符匹配的文件列表，可以使用 EXCLUDE_FILE 来匹配除 EXCLUDE_FILE 列表中指定的文件以外的所有文件。例如：

> Here the ‘*’ is a wildcard which matches any file name. To exclude a list of files from matching the file name wildcard, EXCLUDE_FILE may be used to match all files except the ones specified in the EXCLUDE_FILE list. For example:

```ld
EXCLUDE_FILE (*crtend.o *otherfile.o) *(.ctors)
```

将导致除 crtend.o 和 otherfile.o 之外的所有文件的 *.ctors* 段都被包括在内。EXCLUDE_FILE 也可以放在段列表里面，例如：

> will cause all .ctors sections from all files except crtend.o and otherfile.o to be included. The EXCLUDE_FILE can also be placed inside the section list, for example:

```ld
*(EXCLUDE_FILE (*crtend.o *otherfile.o) .ctors)
```

这样做的结果与前面的例子相同。如果段列表包含多个段，支持 EXCLUDE_FILE 的两种语法是很有用的，如下所述。

> The result of this is identically to the previous example. Supporting two syntaxes for EXCLUDE_FILE is useful if the section list contains more than one section, as described below.

有两种方法可以包括多于一个部分。

> There are two ways to include more than one section:

```ld
*(.text .rdata)
*(.text) *(.rdata)
```

这两者之间的区别在于 *.text* 和 *.rdata* 输入段在输出段出现的顺序。在第一个例子中，它们将被混合在一起，以它们在链接器输入中的相同顺序出现。在第二个例子中，所有 *.text* 输入段将先出现，然后是所有 *.rdata* 输入段。

> The difference between these is the order in which the ‘.text’ and ‘.rdata’ input sections will appear in the output section. In the first example, they will be intermingled, appearing in the same order as they are found in the linker input. In the second example, all ‘.text’ input sections will appear first, followed by all ‘.rdata’ input sections.

当使用 EXCLUDE_FILE 时，有一个以上的段，如果排除在段列表中，那么排除只适用于紧接着的部分，例如：

> When using EXCLUDE_FILE with more than one section, if the exclusion is within the section list then the exclusion only applies to the immediately following section, for example:

```ld
*(EXCLUDE_FILE (*somefile.o) .text .rdata)
```

将导致除 somefile.o 以外的所有文件中的所有 *.text* 部分被包括在内，而所有文件中的所有 *.rdata* 部分，包括 somefile.o，都将被包括在内。为了排除somefile.o 中的 *.rdata* 部分，这个例子可以修改为：

> will cause all ‘.text’ sections from all files except somefile.o to be included, while all ‘.rdata’ sections from all files, including somefile.o, will be included. To exclude the ‘.rdata’ sections from somefile.o the example could be modified to:

```ld
*(EXCLUDE_FILE (*somefile.o) .text EXCLUDE_FILE (*somefile.o) .rdata)
```

另外，把 EXCLUDE_FILE 放在段列表之外，在输入文件选择之前，会导致排除适用于所有段。因此，前面的例子可以改写为：

> Alternatively, placing the EXCLUDE_FILE outside of the section list, before the input file selection, will cause the exclusion to apply for all sections. Thus the previous example can be rewritten as:

```ld
EXCLUDE_FILE (*somefile.o) *(.text .rdata)
```

你可以指定一个文件名来包括某个特定文件的段。如果你的一个或多个文件包含需要在内存中某个特定位置的特殊数据，你会这样做。例如：

> You can specify a file name to include sections from a particular file. You would do this if one or more of your files contain special data that needs to be at a particular location in memory. For example:

```ld
data.o(.data)
```

为了根据输入段的段标记细化被包含的段，可以使用 INPUT_SECTION_FLAGS。

> To refine the sections that are included based on the section flags of an input section, INPUT_SECTION_FLAGS may be used.

下面是一个为 ELF 段使用段头标记的简单例子：

> Here is a simple example for using Section header flags for ELF sections:

```ld
SECTIONS {
  .text : { INPUT_SECTION_FLAGS (SHF_MERGE & SHF_STRINGS) *(.text) }
  .text2 : { INPUT_SECTION_FLAGS (!SHF_WRITE) *(.text) }
}
```

在这个例子中，输出段 *.text* 将由任何与名称 *\*(.text)* 相匹配的输入段组成，为其设置了 SHF_MERGE 和 SHF_STRINGS 段头标记。输出段 *.text2* 将由任何与名称 *\*(.text)* 相匹配的输入段组成，不带段头标记 SHF_WRITE。

> In this example, the output section ‘.text’ will be comprised of any input section matching the name *(.text) whose section header flags SHF_MERGE and SHF_STRINGS are set. The output section ‘.text2’ will be comprised of any input section matching the name*(.text) whose section header flag SHF_WRITE is clear.

你也可以通过写一个与档案匹配的模式，冒号，然后写与文件匹配的模式，冒号周围没有空白，来指定档案中的文件。

You can also specify files within archives by writing a pattern matching the archive, a colon, then the pattern matching the file, with no whitespace around the colon.

‘archive:file’
matches file within archive

‘archive:’
matches the whole archive

‘:file’
matches file but not one in an archive

'archive:file'
匹配档案中的文件

归档:'
匹配整个归档文件

':file'
匹配文件，但不匹配档案中的一个文件

archive "和 "file "中的任何一个或两个都可以包含shell通配符。在基于DOS的文件系统中，链接器会认为单个字母后面的冒号是一个驱动器指定符，所以'c:myfile.o'是一个简单的文件规格，而不是在名为'c'的档案中的'myfile.o'。archive:file'文件规格也可以在EXCLUDE_FILE列表中使用，但不能出现在其他链接器脚本的上下文中。例如，你不能通过在 INPUT 命令中使用'archive:file'从档案中提取一个文件。

Either one or both of ‘archive’ and ‘file’ can contain shell wildcards. On DOS based file systems, the linker will assume that a single letter followed by a colon is a drive specifier, so ‘c:myfile.o’ is a simple file specification, not ‘myfile.o’ within an archive called ‘c’. ‘archive:file’ filespecs may also be used within an EXCLUDE_FILE list, but may not appear in other linker script contexts. For instance, you cannot extract a file from an archive by using ‘archive:file’ in an INPUT command.

如果你使用一个没有章节列表的文件名，那么输入文件中的所有章节都将包括在输出章节中。这种做法并不常见，但在某些情况下可能很有用。比如说

If you use a file name without a list of sections, then all sections in the input file will be included in the output section. This is not commonly done, but it may by useful on occasion. For example:

data.o

当你使用的文件名不是 "archive:file "指定符，也不包含任何通配符时，链接器将首先查看你是否在链接器命令行或INPUT命令中也指定了该文件名。如果没有，链接器将尝试把该文件作为一个输入文件打开，就像它出现在命令行上一样。注意，这与INPUT命令不同，因为链接器不会在归档搜索路径中搜索该文件。

When you use a file name which is not an ‘archive:file’ specifier and does not contain any wild card characters, the linker will first see if you also specified the file name on the linker command line or in an INPUT command. If you did not, the linker will attempt to open the file as an input file, as though it appeared on the command line. Note that this differs from an INPUT command, because the linker will not search for the file in the archive search path.
