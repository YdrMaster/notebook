# learning-cxx 讲义

## 00_hello_world

首先是我们最熟悉的输出 hello world。现在还没讲到重载运算符，所以我们不讲原理，把 `std::cout` 直接当作一种语法现象来理解即可。C++ 的输出语法是在目标之间用左箭头 `<<` 连接：

```c++
std::cout << "Hello, InfiniTensor!" << std::endl;
```

后面的 `std::endl` 表示输出一个换行符。

我猜很多 C 语言用户习惯写 `printf`，C++ 里也可以直接用的。

```c++
printf("Hello, InfiniTensor!\n");
```

这个函数和 C++ 的流式语法有什么区别呢？

最主要的区别在于流式语法是类型安全的，比如在输出一个整数，一个小数都是正确的：

```c++
std::cout << 123 << std::endl
          << 3.1415926535 << std::endl;
```

但是 `printf` 需要你在格式串里直接标注输出的类型，如果类型标错：

```c++
printf("%d\n", 3.14); // %d 表示整数，却对应一个浮点数
```

虽然现代编译器能提出警告，但这不是语法错误，真的会把这个地址里的东西当作整数打印出来。

---

但是在可读性方面，流式语法就比较蠢了，比如我想输出一个 `a + b = c`，`printf` 形式写成：

```c++
printf("%d + %d = %d\n", 23, 19, 23 + 19);
```

这一看就是一段完整的话，很舒服。但是流式语法就得写成：

```c++
std::cout << 23 << " + " << 19 << " = " << 23 + 19 << std::endl;
```

支离破碎了属于是。

还有一个要注意的问题是流式格式化显示 bool 变量的格式：

```c++
std::cout << true << ' '<< false << std::endl;
```

可以看到它居然是当作整数显示的，相当奇怪。必须给它挂上流修饰符 `std::boolalpha` 才行：

```c++
std::cout << std::boolalpha << true << ' ' << false << std::endl;
```

但是这个流修饰符的行为又很奇怪，它的影响竟然是全局的！比如我写成：

```c++
{
    std::cout << std::boolalpha << true << ' ' << false << std::endl;
}
{
    std::cout << true << ' ' << false << std::endl;
}
```

可以观察到修饰一次之后，再输出给 std::cout 的所有 bool 全部被转化了。这种一不留神改了全局状态的设计实在是太逆天了。关于标准库中其他的流修饰符大家可以自行查看 [cppreference](https://zh.cppreference.com/w/cpp/io/manip)。虽然我的评价是能不用就算了。

所以在实践中，我们尽量使用 [{fmt}](https://fmt.dev) 库完成格式化输出。这个库有可能是 C++ 世界里使用最广泛的库之一了，功能相当强大。我们 RefactorGraph 项目里也是用的这个库。具体这个库的用法大家可以自己看文档学习。

由于这个库实在是太好用了，以至于它在 C++20 版本进入了标准。由于主流编译器实现这个库普遍比较晚，为了避免版本问题，我们就没有在习题里使用，这里可以展示一下它是怎么用的：

```c++
#include "../exercise.h"
#include <format>

int main(int argc, char **argv) {
    std::cout << std::format("Hello, {}!", "InfiniTensor") << std::endl
              << std::format("{} + {} = {}", 23, 19, 23 + 19) << std::endl;
    return 0;
}
```

推荐所有有条件选择项目运行环境的同学用这套输出。详细信息见 [cppreference](https://zh.cppreference.com/w/cpp/utility/format/format)。

最后要讲的是一些跟操作系统进程相关的东西，不是 C++ 本身的知识了。现代操作系统的进程设计中，为每个进程提供了 3 个管道，分别是标准输入、标准输出和标准错误。这个设定被映射到 C++ 标准库中，当然 C/Rust 或者其他语言，只要是能生成应用程序的基本都有这个设定。例如：

```c++
std::cout << "Hello, ";
std::cerr << "InfiniTensor!" << std::endl;
```

就是向输出流发送 `Hello,`，向错误流发送 `InfiniTensor!`。运行这个示例，看起来它们很正常地输出了。但是我们还是可以使用控制台的管道操作符来把标准输出和标准错误分别重定向到文件：

```shell
xmake run learn 0 > out.txt      # 输出流到文件，错误流到控制台
xmake run learn 0 > out.txt 2>&1 # 输出流和错误流都到文件
xmake run learn 0 2>err.txt      # 输出流到控制台，错误流到文件
```

好的，跟 C++ 输出相关的内容就讲这么多。标准输入流 `std::cin` 在我们的项目里完全用不到，这里就不讲了，需要的同学可以到 [cppreference](https://zh.cppreference.com/w/cpp/io/cin) 自学。

## 01_variable&add

第二题是变量定义。

做题还是非常容易的，随便给 x 一个定义就好。C++ 能出现的所有运算符见 [cppreference](https://zh.cppreference.com/w/cpp/language/expressions#.E8.BF.90.E7.AE.97.E7.AC.A6)。它们的优先级在这里列出了，大家可以自行学习。我们看到其中有两个算术运算符 `<<` 左移和 `>>` 右移，它们具有比计算更低的优先级，但是具有比比较和其他位运算更高的优先级，这意味着什么呢？

现在我们应该可以发现，C++ 的流操作实际上是流对象上复用这个左移运算符形成的语法！而流运算符的优先级是不会随着它操作的对象变化而变化的。所以，如果流操作里穿插了较低优先级的运算，比如：

```c++
std::cout << (1 << 10) << ' ' << (64 >> 3) << ' ' << (42 != 10) << std::endl;
```

必须要加上括号才是正确的。这样直接写出来似乎不难写对，然而当这些操作叠加在宏里的时候就不一定了，比如：

```c++
#define LEFT(x, w) (x) << (w)
    std::cout << LEFT(7, 10) << std::endl;
```

这就提醒我们宏很危险，流式操作也没它看起来那么安全，两个叠起来可以说是险象环生了。写 C++ 的时候一定要注意这些细节。

## 02_function

第 2 题我们来定义函数。函数的语法不复杂，返回值类型放前面，然后是一个名字，括号里是参数列表，每个参数都是类型加名字。基本所有语言的函数签名都差不多是这么定义的。然而对于没接触过 C/C++ 的同学来说，搞明白声明和定义就比较痛苦了。

简单来说，C/C++ 中，一个语法实体，无论是变量、类型、函数还是宏，都严格必须遵循先出现后使用的原则。只有已知是什么的东西才能出现在表达式里。这个函数只是存在在文件中是无法使用的，它必须要么出现在所有使用它的地方前面，比如：

```c++
int add(int a, int b) {
    return a + b;
}

int main(int argc, char **argv) {
    ASSERT(add(123, 456) == 123 + 456, "add(123, 456) should be 123 + 456");

    auto x = 1, y = 2;
    std::cout << x << " + " << y << " = " << add(x, y) << std::endl;
    return 0;
}
```

要么先利用一个声明来说清它是个什么东西，即：

```c++
int add(int, int);
```

这就是说 add 是一个接受 2 个 int 参数，返回一个 int 值的函数。到 `main` 里，就已知它是个什么函数了，可以编译下去，直到在下面看到定义。由于在声明的时候我们只关心 `add` 是什么东西，所以这些参数是不需要名字的。

> **NOTICE** 虽然把函数视作完全特殊的一种语法现象是很自然的，但是更好的理解是把函数、数组、指针和 `const`、`volatile` 都视作类型修饰符，它们共同作用在预定义和自定义类型上形成了对标识符的类型声明。阅读 [cppreference](https://zh.cppreference.com/w/cpp/language/declarations) 中的示例和链接来学习复杂类型解读法。

## 03_argument&parameter

这题讲的是经典的形参和实参问题。但是需要关注参数传递的问题：值传递、左值引用传递、左值常引用传递、右值引用传递、完美转发，各有什么特点，如何选择？

## 04_static

直接看 static 关键字的解释即可。

## 05_constexpr

这题要学习的是编译期计算的问题。使用 C++/Rust 要分清编译期和运行期。编译期计算有 2 种情况：

1. 编译器利用常量传播把运行期的常量计算优化到编译期；
2. 直接指定计算可能或者必须发生在编译期；

第一个是编译器的能力问题，现代编译器几乎都会选择传播所有常量。第二个则是语法现象，在 C++ 中，constexpr 关键字表示这种行为。当 constexpr 修饰函数时，表示这个函数可以在编译期执行。当 constexpr 修饰变量或者成员时，表示这个变量或者成员一定在编译期求值。所以在这一题中，需要同时在函数上和值上修饰 constexpr，才会导致所有计算发生在编译期。

同时也是因为计算发生在编译期，编译器执行代码的逻辑和运行时执行代码的逻辑不同，编译器会有一个最大递归深度限制。所以这个故意写成递归的斐波那契计算会算不了。所以这题的第一个解法是去掉 constexpr，这样解结果是对的但是会计算极慢。我们可以打印一下看看它在算啥：

```c++
#include "../exercise.h"

unsigned long long fibonacci(int i, int level = 0) {
    for (int j = 0; j < level; ++j) {
        std::cout << ": ";
    }
    std::cout << i << std::endl;
    switch (i) {
        case 0:
            return 0;
        case 1:
            return 1;
        default:
            return fibonacci(i - 1, level + 1) + fibonacci(i - 2, level + 1);
    }
}

int main(int argc, char **argv) {
    constexpr auto ANS_N = 5;
    auto ANS = fibonacci(ANS_N);
    std::cout << "fibonacci(" << ANS_N << ") = " << ANS << std::endl;
    return 0;
}
```

我这里改一改代码，我们看一个详细的过程。我给斐波那契函数加了一个参数 level，并且提供默认值 0，这样调用的时候就不用再传了，然后这个 level 每次递归就 +1，这就标记了递归的深度。然后搞一个小循环来打印一些缩进和标线出来好看。我们看一下这个结果：

```plaintext
5
: 4
: : 2
: : : 0
: : : 1
: : 3
: : : 1
: : : 2
: : : : 0
: : : : 1
: 3
: : 1
: : 2
: : : 0
: : : 1
fibonacci(5) = 5
```

可以看到为什么递归很慢。计算 5 需要计算 4 和 3，计算 4 又需要 3 和 2，所以相当于每多算一个数，计算量就大了接近 1 倍。

以及，我们再看这里计算的顺序。可以看到算 5 是先算 4 再算 3，但是算 4 的时候就是先算 2 再算 3 了。这说明什么 C++ 是不保证 `+` 两边求值的顺序的。实际上，C++ 只保证短路运算的顺序，其他的包括 `<<` 的顺序都是不保证的；函数调用的时候，函数的每个参数的求值顺序也是不保证的。所以**任何依赖参数求值顺序的操作都是未定义行为**。

当然，虽然很慢，但是这个计算最终是可以完成的。所以我在这里写的是修改一处使代码编译运行，但这可不一定能运行出结果😂。要快速算出结果的话，最简单的改法当然还是直接减小这个 N。这题大家只要理解为什么原来不能编译运行就行了。我看到群里有同学也自己探索了用模板特化的手法实现编译时的缓存来加速，这个方法是最好的，但是比较比较超纲我们这里就不讲了。

## 06_loop

第六题就是直白地考察数组初始化和 for 循环语法，以及展示一下斐波那契是怎么用缓存来加速的。说到缓存加速，顺便问大家一个问题，这个函数是一个纯函数吗？

（缓存不改变函数的“纯”性）

## 07_enum&union

第七题就是关于枚举和联合体。关于这两个东西本身我在内联的讲义上已经说的比较清楚了我就不再解释了。但是我要额外讲一下类型双关的问题。

最近我也更新了题目上的很多说明文字和阅读材料，已经做完的同学有时间的话可以回来读一读。

我们来看这段关于类型双关的[阅读材料](https://tttapa.github.io/Pages/Programming/Cpp/Practices/type-punning.html)。这里它又引用了 Wikipedia 的说法，说的是“类型双关是绕开编程语言类型系统的编程技术，以便实现在一般情况下无法实现的效果”。但是我使用“类型双关”这个词的时候通常取的是一个更狭窄的意思，就是我在习题说明里写的这句话：“将一种类型的值转换为另一种无关类型的值”。比如说在一般的机器上，C++ `int` 关键字对应的类型是 32 位有符号的整型，`float` 关键字对应的类型是 32 位满足 IEEE745 规范的浮点型，这两个类型就是所谓的无关类型。如果要把一个整型逐比特重新解释为浮点型，就要用到类型双关。

那这种类型双关怎么实现呢？

首先看这个转换：

```c++
float a = 2.5;
std::cout << (int) a << std::endl
          << static_cast<int>(a) << std::endl;
```

这两种转换都是在值之间转换，所以能够打印出 int 版本的 2。比特级别转换，最容易想到的是通过重新解释指针。

```c++
float a = 2.5, *p = &a;
std::cout << *(int *) p << std::endl
          << *reinterpret_cast<int *>(p) << std::endl;
```

我们会介绍到的类型双关全都是未定义行为，所以不用纠结这些代码的性质了。加入我们认为这些代码都按照字面意思生效，这个代码有什么问题呢？

float 和 int 是大小相同，对齐方式也相同的类型，因此直接转换看不出问题。但是如果转换的是大小不同、对齐不同的类型，直接指针转换本身就是错的了。比如：

```c++
const char *s = "Hello, InfiniTensor";
std::cout << *(int *) (s + 0) << std::endl
          << *(int *) (s + 1) << std::endl
          << *(int *) (s + 2) << std::endl
          << *(int *) (s + 3) << std::endl;
```

当然在大家的机器里这个通常都是可以运行的，因为 cpu 可能是支持非对齐访存的，只是性能受限。但是 rcore 训练营出来的同学肯定知道 RISCV 非对齐访存是直接会触发硬件异常的。所以在高级语言层面直接转换两个对齐不同的类型的指针肯定是有问题的。

所以我们需要基于联合体的类型双关。虽然我这里写了，这个写法在 C++ 中仍是未定义行为，但是这差不多只是一个设定问题，而不是特别本质的问题，因为这种写法在 C 语言中是良定义的，而且是被鼓励的。假设我们站在 C 语言的视角看这个问题，应该可以发现联合体类型双关至少解决了直接转换指针面临的长度和对齐不匹配的问题。因为当我们定义一个联合体：

```c++
union TypePun {
    char c[7];
    float f;
    int i;
    double d;
};
```

显然，编译器会保证这个联合体类型的变量被定义出来时，它的每个成员都是可以安全访问的，因此联合体整体一定具有这些成员中要求最高的对齐。而当程序员设置其中一个成员而访问另一个成员时，即使访问的成员比设置的成员更大，至少仍然位于整个联合体的范围内，不至于溢出到其他对象、甚至是未分配的空间。因此，这样的访问至少从原理上看是安全的，可以正确反映开发者的意图。

那么，在 C++ 中良定义的类型双关如何实现呢？这个文档也有介绍。第一个方法是 `std::bit_cast`，这个方法各方面堪称和谐，唯一的缺点是需要 C++20。第二个方法是 `std::memcpy`，这个操作在 C 语言中也是良定义的，所以兼容 C 和 C++ 的库建议写成这样，唯一的缺点是这个操作不能在编译期执行。然而向大家通报一个不幸的消息，联合体和直接转换指针的类型双关同样无法用于编译期，因为 C++ 规范要求编译期执行的所有操作都良定义。举个例子：

```c++
constexpr float type_pun(int i) {
    union U {
        int i;
        float f;
    } u{i};
    return u.f;
}

int main(int argc, char **argv) {
    constexpr float f = type_pun(0);
    std::cout << f << std::endl;
    return 0;
}
```

所以在 C++20 之前，没有任何办法能做到在编译期取出浮点数的编码。

## 08_trivial_type

这题就是操作简单结构体，没什么需要讲的。可以读一下阅读材料了解这些名称，不怎么影响开发。

这里的方法的形式就是很典型的 C 语言使用结构体的用法。C 语言没有类，没法定义方法，就只能这样写。

## 09_method

第九题就是把上一题的函数挪到解构里面定义成方法，和上一题没区别，过。

## 10_method_const

第十题是带有 const 修饰的方法。名字和参数都相同，但是 const 修饰符不同的方法是重载关系的两个方法，可以根据调用者是否 const 来决议。举个例子：

```c++
struct A {
    void f() {
        std::cout << "mutable" << std::endl;
    }
    void f() const {
        std::cout << "const" << std::endl;
    }
} a;
a.f();
auto const &a_ = a;
a_.f();
```

## 11_class

十一题主要介绍访问修饰符。注意一下 C++ 里类和结构体的唯一区别是结构体默认 public，类默认 private。

注意一下下面的说明，之前的类型，包括基本类型、数组和简单解构体，声明之后不写直接读是未定义行为。但是类型有无参构造器之后，声明时会直接调用无参构造器来初始化，所以声明直接用不是未定义行为了。但是无参构造器里面可能没有设置所有的成员，对这些成员不写直接读依然是未定义行为。

## 12_class_destruct

十二题是析构的问题，记住凡是调用 new 构造出来的，除非所有权被转让给了特别说明会释放对象的结构体，比如智能指针，否则一定需要调用 delete 才会释放。如果发现你的 new 和 delete 不成对，多半发生了内存泄露。所以 C++ 里安全使用内存是特别难的事，几乎只有在构造器中 new，在析构器中 delete 才能保证正确，所以写大型的 C++ 程序最好保证不在构造器里就不要 new 任何东西。

另外我们看一下 new 的语法：

```c++
DynFibonacci(int capacity)
    : cache(new size_t[capacity]{0, 1}),
      cached(2) {}

~DynFibonacci() {
    delete[] cache;
}
```

如果要把这里出现的 `new size_t[capacity]{0, 1}` 分解成几段，正确的分法应该是：`new / size_t[capacity] / {0, 1}`，其中：

- `new` 是运算符，表示创建满足指定大小和对齐要求的内存空间；
- `size_t[capacity]` 是类型。上节课讲过复杂声明的解读法问题，这里的基本类型+方括号是明显的省略标识符的类型声明；
- 最后 `{0, 1}` 是初始化，表示为刚刚创建出来的存储空间填入一些值；

我注意到群里有些同学不容易分清 C++ 里的小括号中括号和大括号，容易用错。这个实际上是对语法死记硬背，没搞明白语法成分的分类和分解。对于没有明显标点符号分隔的句子必须学会分解句子成分，不然就会“句读之不知，惑之不解……小学而大疑”。我这里写了 10 个例子，大家可以自己分析分析都是什么含义：

1. ```c++
   new T
   ```

2. ```c++
   new T[c]
   ```

3. ```c++
   new T()
   ```

4. ```c++
   new T{}
   ```

5. ```c++
   new T({})
   ```

6. ```c++
   new T{{}}
   ```

7. ```c++
   new T[]{t0, t1}
   ```

8. ```c++
   new T[c]{t0, t1}
   ```

9. ```c++
   new T(a0, a1)
   ```

10. ```c++
    new T{a0, a1}
    ```

## 13_class_clone

第十三题是关于拷贝的语义问题。如果将故意写错的构造器先改对或注释掉：

```c++
DynFibonacci(int capacity);//: cache(new ?), cached(?) {}
DynFibonacci(DynFibonacci const &) = delete;
```

编译产生的编译错误是：

```plaintext
error: main.cpp
13_class_clone\main.cpp(42): error C2280: “DynFibonacci::DynFibonacci(const DynFibonacci &)”: 尝试引用已删除的函数
13_class_clone\main.cpp(14): note: 参见“DynFibonacci::DynFibonacci”的声明
13_class_clone\main.cpp(14): note: “DynFibonacci::DynFibonacci(const DynFibonacci &)”: 已隐式删除函数

  > in 13_class_clone\main.cpp
```

这个叫做显式弃置函数定义，也是 C++11 新增的语法。注意这个 `= delete` 就是这个构造函数的函数体，并且这样的函数体不仅可以用于类成员函数。随便举个例子：

```c++
void f() = delete;
```

这样写就是对的，不会产生编译错误。但是如果调用：

```plaintext
error: main.cpp
13_class_clone\main.cpp(55): error C2280: “void test(void)”: 尝试引用已删除的函数
13_class_clone\main.cpp(47): note: 参见“test”的声明
13_class_clone\main.cpp(47): note: “void test(void)”: 已隐式删除函数

  > in 13_class_clone\main.cpp
```

报错说调用已删除的函数。

那么这个语法有什么实际作用呢？结合特化它就用出来了：

```c++
template<class T> T add(T const &a, T const &b) { return a + b; }
template<> int add(int const &, int const &) = delete;

...

ASSERT(add(1., 2.) == 3., "add(1, 2) should be 3");
ASSERT(add(1, 2) == 3, "add(1, 2) should be 3");
```

虽然目前还没讲到模板，可以先看个意思。这里有个函数模板 `add` 可以把任何类型的两实例加起来，但是我希望这个模板不能加 int 型，就可以通过特化弃置，在模板的函数族里挖个洞。调用的话，可以看到 int 是不能调到的。

现在把复制构造器实现出来：

```c++
DynFibonacci(DynFibonacci const &others)
    : cache(new size_t[others.cached]),
      cached(others.cached) {
    std::memcpy(cache, others.cache, cached * sizeof(size_t));
}
```

实现的时候要注意 3 个点：

1. 复制构造器也是构造器，记得可以用 : 初始化列表；
2. 成员复制完了别忘了把成员的内容也复制了；我这里用的 memcpy 需要加头文件，也可以循环赋值；
3. 复制构造器和移动构造器里的参数名字习惯上是用 others，因为是和隐式的 this 对立，但是如果觉得对于移动和复制来说 others 不直观的话也可以叫做 source 或者 origin；

下面调用的时候可以看到，fib 这个对象已经填写到了 12 项，所以拷贝一个 const 应该可以直接查第 10 项，没有问题。

## 14_class_move

第十四题是关于移动语义。移动语义是 C++11 加入的最重要概念，可以说没有之一，在我看来比什么 lambda 或者新增容器类型都重要多了。甚至移动语义直接构成了 Rust 语言的最基础概念。但是为了解释移动语义，C++ 引入了一个特别抽象的概念叫做左值右值亡值……这个概念的自洽性我没有疑问，但是必要性和合理性我觉得存疑，所以这里就不讲细节了。我补充了很多阅读材料，需要的同学可以自己看。我们这里只说一个词：由于 C++ 引入的这套左值右值体系，C++ 里要触发移动语义，需要一个“右值引用”。

抛开概念不谈，所谓一个“右值引用”，就是一个非字面量的、可变的、准备好了被移动而失效的值。

这里可以对比一下 Rust 的概念。Rust 里，由于移动是一个缺省的、基础的情况，所以移动行为看起来是双向的。只要源码出现基于参数或者模式匹配的重新绑定，移动就直接发生了。

但是 C++ 里的移动是非缺省的、复杂的，由 3 个阶段构成：

1. 接受移动的一方先在编译时提供接收移动的渠道，也就是移动构造和移动赋值，对应自身当前状态是空白的还是非空白的；
2. 运行时被移动的一方要转换成“右值引用”，相当于变成准备移动的状态；
3. 右值引用传递给接收机制，移动发生；

具体到这个例子里，我们先在这个移动构造器和移动赋值里打印，方便追踪行为：

```c++
DynFibonacci(int capacity)
    : cache(new size_t[capacity]{0, 1}),
        cached(2) {
    std::cout << "Constructor called" << std::endl;
}
DynFibonacci(DynFibonacci const &others)
    : cache(new size_t[others.cached]),
        cached(others.cached) {
    std::cout << "Copy constructor called" << std::endl;
    std::memcpy(cache, others.cache, cached * sizeof(size_t));
}
~DynFibonacci() {
    std::cout << "Destructor called" << std::endl;
    delete[] cache;
}
DynFibonacci(DynFibonacci &&) noexcept {
    std::cout << "Move constructor called" << std::endl;
}
DynFibonacci &operator=(DynFibonacci &&) noexcept {
    std::cout << "Move assignment called" << std::endl;
}
```

> **NOTICE** 纯粹是示例！

下面这样打印看看：

```c++
std::cout << "step 0" << std::endl;
DynFibonacci fib(12);
std::cout << "step 1" << std::endl;
DynFibonacci &&fib_1 = std::move(fib);
std::cout << "step 2" << std::endl;
DynFibonacci fib_2(fib_1);
```

顺便点进来看一下这个 `std::move` 做了什么事，可以发现它就是一个 `static_cast` 直接类型转换到右值引用，然后返回的也是右值引用。

> **NOTICE** 这个实现是共识。实践中，标准库和编译器的关系是很暧昧的。虽然理论上标准库不是编译器的一部分，但是还没有什么语言的实现中，标准库和编译器不是一起分发的；同时大概也没有什么语言的实现中编译器不为标准库开洞的。但是这个 `std::move` 公认的就是这么实现，没有哪套工具链写成别的样子。

大家觉得这个能会打印出什么呢？跑一下看看，会发现：

```plaintext
step 0
Constructor called
step 1
step 2
Copy constructor called
Destructor called
Destructor called
```

这个居然发生的还是复制构造！而且由于发生了复制，产生了两个对象，析构函数也调用了两次。会发生这个事是因为“右值引用”作为右值，无法出现在 `=` 左边。然而你直接声明等号左边要一个右值引用，C++ 编译器一句话都不说，也不警告，但是跑起来就直接给你一个左值引用。非暴力不合作了属于是。

如果直接去掉 `std::move`，显然还是会调用到复制构造：

```c++
std::cout << "step 0" << std::endl;
DynFibonacci fib(12);
std::cout << "step 1" << std::endl;
// DynFibonacci &&fib_1 = std::move(fib);
std::cout << "step 2" << std::endl;
DynFibonacci fib_2(fib);
```

所以要触发移动语义必须这样写：

```c++
std::cout << "step 0" << std::endl;
DynFibonacci fib(12);
std::cout << "step 1" << std::endl;
// DynFibonacci &&fib_1 = std::move(fib);
std::cout << "step 2" << std::endl;
DynFibonacci fib_2(std::move(fib));
```

然后，可以看到这样的输出：

```plaintext
step 0
Constructor called
step 1
step 2
Move constructor called
Destructor called
Destructor called
```

可以看到，这里重载决议触发了移动构造器。但是还是发生了两次析构。这是因为 C++ 中，一个被移动的对象处于什么状态到今天也没一个明确的说法。在 [cppreference](https://zh.cppreference.com/w/cpp/language/move_constructor) 中是这样描述的：

> 典型的移动构造函数“窃取”实参曾保有的资源（例如指向动态分配对象的指针，文件描述符，TCP 套接字，输入输出流，运行的线程，等等），而非复制它们，并使它的实参遗留在**某个合法但不确定的状态**。例如，从 `std::string` 或从 `std::vector` 移动可以导致实参被置为空。但是**不应该依赖此类行为**。对于**某些类型**，例如 `std::unique_ptr`，移动后的状态是完全指定的。

这就是说，C++ 仅仅约定了移动构造应该比复制对象更轻量，但是具体做什么事完全是实现自己决定的。*典型*的移动构造函数这么干，那非典型的移动构造函数可不可以比复制还慢还重呢？至少这样做不是未定义的行为。以及被移动之后的对象要怎么处理，也没有统一的规范。有时候是可以复用的，比如说一个对象容器，从中移走了一项，为了避免对容器本身大动干戈，触发了什么 O(n) 复杂度的增删移动操作，最好是能再从别的地方移动来一个东西复用这个对象。但是如果是栈上对象被移动了，那复用它一般就没有什么好处，可能仅仅是节约一些栈空间。但是即使不复用它，由于编译器无法确认这个对象是不是已经移动干净了，能不能直接析构，所以通常来说编译器也不敢直接复用这块栈空间干别的事。也就是说，有时候 Rust 的一些行为和语义更明确，能做的优化也更深，这就是为什么有时候 Rust 程序性能比 C++ 的还要高。

接下来我们正式开始实现移动构造和移动赋值。先看移动构造：

```c++
DynFibonacci(DynFibonacci &&others) noexcept
    : cache(others.cache),
      cached(others.cached) {
    std::cout << "Move constructor called" << std::endl;
    others.cache = nullptr;
    others.cached = 0;
}
```

通过实现移动构造，同学们可以理解移动语义的好处在哪。显然，现在不需要重新分配空间，然后把 cache 的内容整个复制一遍了，只需要复制一个指针。注意 3 点：

1. 移动构造器也是构造器，记得可以用 : 初始化列表；
2. 移动之后记得把原对象标记到你选择的**合法但不确定的状态**，不一定要写 0 和空指针，但至少要和被移走的资源断开联系，以免出现争用和重复释放问题；
3. 注意这后面有个 noexcept，意思是说这个函数绝不抛出异常。这个东西是和 C++ 异常体系联系的，这个体系本来就十分扭曲，推荐完全不用就完事了。C++ 约定了移动构造和移动赋值不应该抛异常，所以直接挂上就行；

> 同样地，如果不喜欢 `others` 这个参数名可以换成别的。

使用 : 初始化列表实现移动构造有一个缺陷，就是复制 others 的字段和断开资源连接不得不分开写，所以对于简单但是非默认的移动构造比较推荐用一个标准库函数叫做 [`std::exchange`](https://zh.cppreference.com/w/cpp/utility/exchange) 的，写成这样：

```c++
DynFibonacci(DynFibonacci &&others) noexcept
    : cache(std::exchange(others.cache, nullptr)),
      cached(std::exchange(others.cached, 0)) {
    std::cout << "Move constructor called" << std::endl;
}
```

> **NOTICE** `std::exchange` 可能需要 `#include <utility>`。

这样移动的语义看起来更明确一些。

接下来实现移动赋值。首先注意，移动赋值是一种运算符重载。关于运算符重载的语法可以看上方的阅读材料。关于运算符重载的实现，关键点是 3 个：

1. `operator` 关键字 + 要重载的符号做为函数标识符；
2. 参数和返回类型要写对，具体每个运算符写成什么算对，请勤于查文档，这么多运算符真记不住；
3. 有许多看起来不像一般意义上的运算符的东西，实际上都是运算符，都能重载。比如这里的赋值运算符 `=`；还有之前提过的类型转换，类型转换是一个单目运算符，可以重载；还有访问指针成员的 `->`，调用函数的小括号，都算运算符，都能重载；以及左边的 ++ -- 和右边的 ++ --，可以分别重载成不一样的；具体这些东西都怎么重载，大家可以自己查文档，有些并没有那么直觉。

这个题目里移动赋值重载的签名已经写好了，注意参数是右值引用，返回值是左值引用，这里固定写自己。整体实现：

```c++
DynFibonacci &operator=(DynFibonacci &&others) noexcept {
    ~if (this != &others) {
        std::cout << "Move assignment called" << std::endl;
        delete[] cache;
        cache = std::exchange(others.cache, nullptr);
        cached = std::exchange(others.cached, 0);
    } else {
        std::cerr << "Warning: self-assignment detected" << std::endl;
    }
    return *this;
}
```

移动赋值起手，直接判断不是移动到自己。这些全是约定：

1. 移动到自己是无法避免的，虽然直接写是警告行为但动态条件下编译器不能发现；
2. 移动到自己会怎么样是实现决定的，实践中大家倾向于什么也不做，但是根据业务也可以干别的事。无论如何不判断一定是不对的；

如果不是移动到自己，记得干两件事：

1. 释放原本 this 持有的资源；
2. 做移动构造同款操作；

注意移动赋值是左值也现存右值也现存的情况，所以一旦右值移动到左值，左值原来的内容就被覆盖了，所以要在覆盖之前直接释放。所以一个移动赋值实际上基本上等价于先在 this 上原地析构，再调用移动构造。而且 Rust 就是这么实现的。C++ 需要开发者自己去写这个逻辑；

最后记得 `reture *this`。运行：

```plaintext
Constructor called
Move constructor called
Constructor called
Constructor called
Move assignment called
Warning: self-assignment detected
Destructor called
Destructor called
Destructor called
Destructor called
```

移动到自己检测出来了。以及注意声明了 4 个对象一定调用 4 个析构，被移动的对象也要析构。

另外，我注意到有人会在析构里写这种代码：

```c++
~DynFibonacci() {
    if (cache) delete[] cache;
}
```

这是没有必要的。删除/释放空指针定义上就是什么都不做，不需要自己判断。

## 15 class_derive

第十五题是类派生。这个题主要是提醒大家继承和派生的时候字段是什么关系。可以看到 X 里有一个 int，A 里有一个 int，然后 B 继承 A 同时持有一个 X。那么它们各自多大呢？

显然 X 和 A 都是一个 int 那么大。同时注意方法是不占体积的，因为方法本质上就是函数指针类型的类静态常量成员，作为静态成员存储不会占用堆栈段空间。

那么 B 的大小是多少呢？对于字段来说，实际上持有 A 和继承 A 是一回事，所以 B 的大小就是 X + A = 2 x int。注意两个 int 除了写成 `sizeof(int) * 2`，还可以这么写：

```c++
static_assert(sizeof(X) == sizeof(int), "There is an int in X");
static_assert(sizeof(A) == sizeof(int), "There is an int in A");
static_assert(sizeof(B) == sizeof(int[2]), "B is an A with an X");
```

其实对于复杂类型的数组更推荐这么写。

除此之外，还要讲一个 C++ 的坑点，就是下满这个代码为什么不是错误的问题。看题目下方的代码，先打开这个注释：

```c++
B ba = A(4);
```

这是试图把 A 类型的值赋值到 B 类型的变量里。显然这是不可能的，事实上也确实产生编译错误，说是 A 无法转换成 B。但是我们发现，B 转换到 A 却是正常的！这不是非常奇怪吗？B 是 A 的派生类，这意味着 B 拥有 A 的所有字段，还可能拥有比 A 更多的其他字段。也就是说 B 变量一定放得下 A 值，而 A 变量不一定能放下 B 值。这种情况下把 A 赋给 B 非法，把 B 赋给 A 却合法，这不是太奇怪了吗？

之所以会出现这种情况，是因为 C++ 过于复杂，以至于在设计中无法很难注意到，许多语法设计上具有“副作用”。什么叫副作用呢？比如说一个函数，一般的功能是输入参数返回结果。因此函数的作用就是返回的结果，而如果函数不通过返回的结果，直接影响了其他东西的状态，这些影响就是函数的副作用。

语法设计上也可以有副作用。跑这个代码，看看可以打印出什么：

```plaintext
1. A(1)
2. X(5)
3. B(1, X(5))
4. A(A const &) : a(1)
5. ~B(1, X(5))
6. ~X(5)
7. ~A(1)
```

注意，这一大串话都是一行打出来的。分析一下它都干了什么事：

1. 首先，赋值符是右侧优先的运算符，所以整个语句最先求值的是 `B(5)` 表达式；
2. 构造 `B(5)`，调用的是 `B` 的有参构造器。根据构造器实现，需要先构造 `A(1)` 和 `X(5)`；
3. 所以观察到，第 1 行打印出 A(1)，第 2 行打印了 X(5)；
4. 构造基类和成员之后可以构造派生类，于是第 3 行打印了 B(1, X(5))，这个时候 `B(5)` 就构造好了；
5. 但是明明构造了 `B`，第 4 行调用的却是 `A` 的复制构造，于是 `B` 作为 `A` 被复制到变量；
6. 然后构造出来的 `B` 由于没有地方存储，直接又被析构掉了；
7. 为了析构 `B`，需要按顺序析构 `B` 和作为 `B` 的成员的 `X` 和 `A`；

这么一分析，可以发现显然问题就出现在复制构造这一步，明明等号右边是 B，但是复制的时候却默默变成了 A。这个变换是怎么完成的呢？

实际上这个问题在 17 题提到了。

> 由于今天是最后一节课，16 题就跳过了，重点讲几个有坑的题目，以及张量相关的算法题目。

17 题写了，C++ 中，派生类指针可以随意转换到基类指针。由于引用就是指针，所以自然派生类引用就会自动转型成基类引用。本来这是个很自然的设计，任谁来设计都会这样设计。所以这个例子中，B 可以在接受 A 的场景中自动转型到自己的基类 A。然而在这个语句中，B 是一个临时的右值，而 C++ 里复制语义又是基于对赋值号和构造函数的重载实现的，这些功能碰到一起就出现了这种特别怪异的现象，实现类竟然可以被赋值到基类变量中！并且，这其中每一步都在 C++ 的设定之中，不存在任何未定义行为，因此也就不会引发警告和错误。但是，显然这个现象也不会是故意设计的结果，因此我把这个现象称为语法设计的副作用。

这个副作用是无关痛痒的，由于这个行为语义怪异，所以应该也不会有人主动利用这个行为。但是其他一些语法副作用可能产生了更加复杂的结果，比如著名的模板元编程，一开始就是作为模板——或者我们用更通行的名字——泛型的副作用产生的，这就是另一个故事了。

## 17_class_virtual_destruct

第十六题和十七题考察的是虚函数和函数重写的问题。C++ 的岗位面试我也参加过十几场吧，其中最日常的问题之一就是重写（override）和重载（overload）的区别。复读一下，重载指的是同名的函数，如果有不同的参数类型，则可以有不同的返回值类型，被视作两个函数。这个机制我们之前已经使用了很多次。而重写是类型派生中才会用到的功能。

类型派生的基本形式在 15 题中已经出现了。当一个类作为基类，它可以通过在函数签名之前增加 virtual 修饰符，声明一个函数是虚的。C++ 编译器会根据类及其基类中虚函数的数量为类生成虚表。因此，如果类包含任何虚函数，相当于其字段中额外增加了一个指针，就是虚表指针。我们可以做这样的简化：

```c++
#include "../exercise.h"

struct A {
    char name() const {
        return 'A';
    }
};
struct B final : public A {
    char name() const {
        return 'B';
    }
};

int main(int argc, char **argv) {
    std::cout << sizeof(A) << " " << sizeof(B) << std::endl;
    return 0;
}
```

由于这个时候，A 和 B 中没有任何字段，因此它们实际上的大小应该是 0，但是 C++ 禁止 0 长度的结构体存在，没有任何字段的结构体长度会设置为 1 字节，以免为它们分配空间时产生不正常的指针。但是一旦我把 `A::name` 方法声明为虚，就会发现结构体的长度变为 8，这就是因为结构体里多了一个虚表指针。每个对象的虚函数实现的函数指针会存放在虚表指针中，这样实现了通过对象调用方法时，可以调用到具体实现类的对应方法。

这就是所谓虚函数重写的意思，就是说对于这个类型的对象，类型提供了另一个函数指针，替代继承自基类的虚表中同名同参的函数指针。所以这个“重写”的称呼还是很准确的。对于重写的方法，实现类中同名同参就会发生重写，但是为了保证重写正确，一般要求在后面增加 override 或 final 修饰符。override 表示一般重写，final 表示终结重写，即不可以在被以后的孙子类重写。注意这里如果写 override 的话是没有语义的，和不写的效果完全一样，它唯一的功能是避免因为名字或参数写错，导致无法命中基类中的函数。如果对基类中无对应虚函数的方法添加 override 和 final 修饰符，会直接产生编译错误。同理，类也支持 final 修饰符，表示类不能派生。

最后要注意的还有一点，由于静态函数不对应对象，无法写入虚表的，也就无法设置为虚以及重写。

现在我们来完成第十七题。首先是类静态字段初始化的问题。这个问题无解，是个规定。我推荐大家就放弃在类中初始化任何东西比较好，不用较劲什么东西可以类内初始化什么东西不行了。总之只要是静态字段，一律在源文件中单独初始化，所以这里要这样写：

```c++
int A::num_a = 0, B::num_b = 0;
```

而只要是非静态的字段，一律在构造函数里初始化。构造函数有多种形式的，尽量写一个完整版本，然后将重载版本转发给完整版本实现。这都是经验之谈，这块特别难写对，用上这些约束会比较有好处。

好的，正确初始化静态成员之后我们看这个静态成员的值。这里的问题是构造 B 是否一定构造 A？之前说过，B 派生自 A，意味着 B 有 A 的所有字段和方法，就好像 B 持有一个 A 一样。因此从零开始构造 B 显然一定需要构造 A，因此即使这里没写，也默认会调用 A 的无参构造。因此，构造 B 意味着 num_b 和 num_a 都加了 1。所以这里 a 加了 1，B 给 a 和 b 各加了 1，a 就是 2，b 就是 1：

```c++
ASSERT(A::num_a == 2, "Fill in the correct value for A::num_a");
ASSERT(B::num_b == 1, "Fill in the correct value for B::num_b");
```

下面一处，不管指针是什么类型，只调用了一次 new B，所以 a 和 b 都是 1：

```c++
ASSERT(A::num_a == 1, "Fill in the correct value for A::num_a");
ASSERT(B::num_b == 1, "Fill in the correct value for B::num_b");
```

名字的话，由于通过指针总是可以找到虚表并调用到具体实现的方法，因此都是实现的名字。

最后为向下转型补全正确的转换语句。`dynamic_cast` 的特点是会通过查虚表动态发现一个实现到底能不能转换到目标类型。

运行这个代码，发现仍有问题，问题出在通过 A 的指针删除 B，换句话说，通过基类指针删除实现类。删除的时候，会调用析构函数，从而使 num 静态成员减 1。但是，如果是通过基类调用，那么要么调用到基类的方法，要么调用到虚方法。这个限制不但适用于一般方法，也适用于析构函数。因此，正确的写法是将析构函数声明为虚。则析构函数也会出现在虚表中，从而在派生时由子类的析构替换。那么即使通过基类指针，仍然可以调用到具体类型的析构函数。

这里有一个问题，如果调用的是子类析构，不会导致基类没有被析构而 a 不减 1 吗？

并不会，这里基类的析构是由子类析构自动调用的。之前多次说过，基类在字段上的行为与成员一致，因此析构子类必然也要析构基类，所以基类析构是自动调用的。

## 18_function_template

第十八题是我们首次正式引入模板。

模板，本质上就是由编译器直接理解的宏。模板可以添加一些参数，具体来说，可以是类型标识符，也可以是一种整数。就和宏的逻辑完全一样，模板参数会替换到模板代码的对应位置上，将模板变换成实现类型。注意，虽然模板当然不是宏，但是实际开发中，各方面它更像宏，因为它的所有行为都发生在编译时，这会带来各种问题。在下一题中我们会看到这种现象。

注意到十八题什么都不改的话直接编译，产生的是警告而不是错误，因为浮点到整数也是自动转型的……

给函数加上模板的结构还是很简单的：

```c++
template<class T>
T plus(T a, T b) {
    return a + b;
}
```

注意，这样改掉之后，实际上将 plus 由函数定义转换成立一个函数模板。模板就好像宏一样，本身是无法编译成二进制的，比如通过调用或特化，为它确定所有模板参数，才会将模板转换成实例编译。因此模板如同宏一样，必须出现在头文件中，整体为调用者可见，才能完成编译。这个设定进一步扭曲了 C++ 生态。

原本的 C++ 和 C 一样，是十分强调声明定义分离，声明对外可见，定义编译为二进制隐藏这一套逻辑的。然而引入模板之后，可以产生实现的模板却也要求可见。隐藏实现就变为无稽之谈了。客观上这促进了如今 C++ 开原生态的建立，因为大家不再思考开放什么隐藏什么，而是直接纯头一把梭。甚至有不止一两个人开发了单头压缩工具，能够将一整个项目压缩成一个头文件。这个一方面降低了调用开源库的难度，一方面却也增大了编译器的压力。因为这样的项目中，每个编译单元都变得极为庞大。

第十八题中，实际使用到 int、unsigned int、float 和 double 四种实际的函数实现。因此 C++ 为这 4 种类型各自生成了函数实现，再依照类似函数重载的方法选择调用。因此一方面，同一个类型多次调用只会生成一个实例，然而另一方面，确实会为每个类型都生成一个完整的实例。对于大型的函数或者非常复杂的模板，这造成了著名的二进制膨胀问题。以及即使是较小的修改，只要是修改模板，就可能造成大范围的影响，从而导致增量编译失效。

最后注意浮点数比较的问题。我看到有些人不理解浮点比较的原理，盲目认为浮点比较一定不是精确的，实则不然。C++ 和一般的其他常见语言，都把这个 `==` 认定为一般的浮点比较规则。IEEE754 浮点规则本身是各复杂的体系。简单而不精确地说法，就是浮点数中既包含一般的数，也包含特殊类型的不是数的东西。对于一般意义上的数字就直接比较其编码，编码相同则认为数相等。但浮点数还有正无穷值、负无穷值和多个零值，以及 NaN。它们各自的比较规则大家可以自行尝试。这里要说的是，如果是字面量比较，你需要确认目标数字是一个二进制表示有限的数字。例如 1 在二进制中是 1，0.5 在二进制中是 0.1，0.25 在二进制中是 0.01。因此，此处的 1、2、3.75 都是可以精确表示的浮点数，于是就可以使用 `==` 判等。而 `0.1 + 0.2 = 0.3` 则是标准的不可判等的情况。因此判断条件要修改为误差和小于指定的界限。

## 19_runtime_datatype

第十九题是关于经典的动态类型计算实现。这里的 TaggedUnion 是联合体一种常见的安全用法。

直接先按 TODO 描述把 sigmoid 计算模板化:

```c++
template<class T>
T sigmoid(T x) {
    return 1 / (1 + std::exp(-x));
}
```

然后麻烦的地方来了。请问这里类型能直接传给模板吗？

有些同学可能会说：“当然不行，这里的类型标签是一个枚举，又不是类型，写也写不上去啊。”那么，如果我利用模板元把类型搞出来就可以写了吗：

```c++
template<DataType DT>
struct data_type_t {};

template<>
struct data_type_t<DataType::Float> {
    using type = float;
};

template<>
struct data_type_t<DataType::Double> {
    using type = double;
};
```

通过写这样的代码，可以把枚举与实际类型对应，例如：

```c++
std::cout << sizeof(data_type_t<DataType::Float>::type) << std::endl;
```

有了这个机制，sigmoid_dyn 可以写成这样吗：

```c++
auto val = *((data_type_t<x.type>::ty *) &x.f);
return sigmoid(val);
```

因为 x 的 f 和 d 在相同地址，那么我只要把这个地址转换成目标类型指针，不就可以直接调用对应的实现函数了吗？

显然答案是不行的。x 的类型是运行时类型，而为模板填写类型发生在编译期。我们生活在一个大致满足因果规律的世界，未发生的事不可能决定已发生的事，所以这样的代码就不可能存在。正确的写法是直接 switch 判断：

```c++
TaggedUnion sigmoid_dyn(TaggedUnion x) {
    TaggedUnion ans{x.type};
    switch (x.type) {
        case DataType::Float:
            ans.f = sigmoid(x.f);
            break;
        case DataType::Double:
            ans.d = sigmoid(x.d);
            break;
    }
    return ans;
}
```

好的，测试通过。那么还有一个问题，这里的浮点数是任意的，不可能保证恰好是二进制有限位的，这里使用 `==` 判断相等性是正确的吗？

## 20_class_template

第二十题考察类模板。关于模板本身就不介绍了，重点是这个单向广播加法。首先把构造函数补全：

```c++
Tensor4D(unsigned int const shape_[4], T const *data_) {
    unsigned int size = 1;
    for (auto i = 0u; i < 4; ++i) {
        shape[i] = shape_[i];
        size *= shape[i];
    }
    data = new T[size];
    std::memcpy(data, data_, size * sizeof(T));
}
```

其实这里最通用的写法是求两个张量的跨步，然后利用跨步计算每个坐标的数据偏移完成加法，但是一方面第二十五题才介绍跨步计算，另一方面这题有两个点可以简化计算：

1. 张量的维数是确定的，一定是 4；
2. 张量计算是直接发生在 this 上的，others 可以广播，this 不能广播；

因此，实现这个特化版本会比通用版本简单不少，只需要用一个 4 重循环来遍历每个维度：

```c++
Tensor4D &operator+=(Tensor4D const &others) {
    //! 为了让函数看起来简洁，省略了一部分大括号

    // 预先存储每个阶是否需要广播
    bool broadcast[4];
    for (auto i = 0u; i < 4; ++i)
        if (broadcast[i] = shape[i] != others.shape[i]) // 如果形状不一致就需要广播
            ASSERT(others.shape[i] == 1, "!");          // 单向广播，others 的对应长度必须为 1

    auto dst = this->data;  // 要加到的元素地址
    auto src = others.data; // 要加上的元素地址
    T *marks[4]{src};       // 4 个阶的锚点
    for (auto i0 = 0u; i0 < shape[0]; ++i0) {

        if (broadcast[0]) src = marks[0]; // 如果这个阶是广播的，回到锚点位置
        marks[1] = src;                   // 记录下一阶锚点

        for (auto i1 = 0u; i1 < shape[1]; ++i1) {

            if (broadcast[1]) src = marks[1];
            marks[2] = src;

            for (auto i2 = 0u; i2 < shape[2]; ++i2) {

                if (broadcast[2]) src = marks[2];
                marks[3] = src;

                for (auto i3 = 0u; i3 < shape[3]; ++i3) {

                    if (broadcast[3]) src = marks[3];
                    *dst++ += *src++;

                }
            }
        }
    }

    return *this;
}
```

这里只是为了教学的正常顺序才这么写这个算法。真正的深度学习程序中，不管是推理引擎还是编译器都不可能限制张量不超过 4 阶。所以大家看看就好，当然这个写法本身确实也是对的，而且保证了相当的灵活性和效率。

## 21_template_const

第二十一题本身是关于模板常量。这个机制看似正常，但实际流毒无穷，直接导致了模板元编程的诞生，把 C++ 彻底推向了无人能懂的地步。但是我们这里只涉及最一般的、符合设计初衷的用法，也就是控制一个数组的长度。

首先仍然是补全构造器：

```c++
Tensor(unsigned int const shape_[N]) {
    unsigned int size = 1;
    for (auto i = 0u; i < N; ++i) {
        shape[i] = shape_[i];
        size *= shape[i];
    }
    data = new T[size];
    std::memset(data, 0, size * sizeof(T));
}
```

模板的数字参数之所以叫做模板常量，就是因为它从各方面看起来都和一个常量一样，也可以直接作为参数传递给运行时。所以这里的循环就是 N 阶而不是 4 阶了。

接下来实现这题要求的算法。其实这个计算坐标，跨步计算也是呼之欲出了。但是我们依然可以不显式使用跨步概念，而是直接计算这个坐标。关键点就在于：显然，最后一维是最小的维度，加要从小到大加，所以循环一定要改成从后往前循环。想明白这个事算法写起来非常轻松：

```c++
unsigned int data_index(unsigned int const indices[N]) const {
    unsigned int index = 0, mul = 1;
    for (auto i = N - 1; i < N; --i) {
        ASSERT(indices[i] < shape[i], "Invalid index");
        index += mul * indices[i];
        mul *= shape[i];
    }
    return index;
}
```

## 22、23、24

这三题考察的主要就是容器占用空间大小的问题：

- `std::array` 各方面就是数组，只不过数组要兼容 C 语法，设施不完善，所以又弄了一个标准库类型；
- `std::vector` 典型大小就是 24 字节或者说 3 个指针长度。这件事虽然理论上也是实现决定，但是实际上大家就当作事实就可以了。甚至这个事实在 Rust 中依然成立，Rust 的 Vec 大小也是 24 字节；
- `std::vector<bool>` 各方面就是奇葩，据说是因为当时刚发明模板特化，委员会想推这个技术，就特批在标准库里加一个特化类型给大家打个样，结果打歪了打到自己脸上了。这东西日常使用有 3 个大坑：

1. 用 `data()` 不能获取指针、映射为数组；
2. `at` 和 `[]` 得到的默认类型是类似左值引用的东西，不是值语义；
3. 栈上结构体大小不是 24 字节；

总之，使用这种东西一定要慎之又慎。

## 25_strides

终于到了我们盼望已久的跨步计算，Talk is cheap，here is the code:

```c++
std::vector<udim> strides(std::vector<udim> const &shape) {
    std::vector<udim> strides(shape.size());
    auto ans = strides.rbegin();
    *ans = 1;
    for (auto it = shape.rbegin();
         it + 1 != shape.rend();
         ++it, ++ans) {
        ans[1] = ans[0] * *it;
    }
    return strides;
}
```

感觉没什么可解释的吧，只要会反着迭代，这题没有任何难度了。
