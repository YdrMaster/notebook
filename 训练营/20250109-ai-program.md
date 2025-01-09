# AI 程序的抽象

> 本文对[夏季文稿](20240806-ai-program.md)进行了修订。

大家好，上次课大家已经了解了组织中的两个 AI 编译器项目，今天我会带大家稍微深入细节，学习 AI 程序是如何在 AI 编译器中表示出来的。

通过今天的课程大家可以学到 4 项知识：

1. AI 程序的特点和计算图设计的理论基础；
2. 如何阅读 ONNX 模型和文档；
3. InfiniTensor 中的计算图；
4. RefactorGraph 是如何重构计算图的；

要开始今天的话题，需要首先理解 AI 程序和一般的程序究竟有什么区别？换句话说，为什么我们不会像写一般的程序一样，使用一种图灵完全的高级语言来实现一个一般的程序，用这个程序进行深度学习计算？

即使仅通过冬季训练营中提供的信息，大家应该也可以意识到，用高级语言直接实现深度学习程序无疑本身是可行的。大家已经见到的，无论是 C++ 习题中的几个算法，还是专业阶段 AI 编译器或者大模型方向的习题，以及 InfiniLM、llama.cpp 这样的具有实用性的专业大模型框架，都是直接使用高级语言实现的。无疑它们也具有加载模型和参数、执行推理的能力。在编译这些程序的过程中，高级语言编译器也正确发挥了它们的生成和优化作用。既然如此，为什么我们还需要 AI 编译器、AI 编程框架这样的东西呢？

我不知道大家有多深入思考过这个问题。实际上，就好像随着电子计算机的发展，我们经历了打孔卡、机器码、汇编语言、C 语言、再到现在的 C++、Rust、Python 等语法形式特别丰富的现代编程语言，我们能写出的功能越来越多，但是真正程序员对计算机的控制其实是越来越弱的。比如说，当使用机器码和汇编编写裸机程序时，我们可以践行冯诺依曼的理念，完全混淆程序与数据的界限，比如在某一个地址填入指令然后跳上去跑；当使用 C 语言的时候，我们需要提前规划堆、栈和静态区数据，但是至少对这些规划好的区域还可以随意地读写；但是在 C++ 中引入了类、命名空间和可见性，在 Rust 中程序员受到更多各种各样的约束，而 Python、Java 这样的语言则干脆是跑在虚拟机里。之所以有这样的发展规律，是因为随着计算机的计算能力增长，我们越来越倾向于把程序表达成有利于人类而不是计算机理解的形式。而人类的思维是建立在规律、秩序，或者说是约束、是对自我的限制之上的。而 AI 程序就是编程这件事的一个更加细致、受到更多约束的分支。因此，编写 AI 程序涉及到一种范式转变，就有了我们今天看到的独特的表示和对应的解读和优化工具。

在夏季的开营仪式上，豪杰老师介绍了人工智能的发展历程。

> 顺便说一下这些视频在[冬季导学](https://opencamp.cn/InfiniTensor/camp/2024winter/stage/0?tab=video)里，同时 [B 站](https://www.bilibili.com/video/BV12ekcYoEyU/?spm_id_from=333.1387.homepage.video_card.click)也发布了一份，没看过的同学可以去看一下。

今天我们所说的 AI 程序，实际上已经不再指向所有符合机器学习定义，具有学习和泛化能力的程序，而是特指基于深度神经网络、操作大规模数据且完全无副作用的一种程序。AI 程序这样的特点使得开发者可以接受相比通用程序的开发者严格地多的限制，以换取开发的便利性。

具体来说，AI 程序具有的 3 个特点使得 AI 程序被设计成以下的模式：

1. 基于深度神经网络，因此 AI 程序具有很强的时序性或者层序性。换句话说，AI 程序中几乎只有顺序执行，控制流极少。因此，AI 程序可以表示为有向无环图（DAG）；
2. 操作大规模数据，因此 AI 程序操作的基本数据类型不是单个变量，而是高维数组。在 AI 程序中，高维数组被称为张量（Tensor）；
3. 完全无副作用，因此 AI 程序可以完全由计算函数调用组成，不需要 IO 模块。在 AI 程序中，函数调用被称为算子（Operator）；

至此，我们就推导出了 AI 程序最常见的一种表示形式，也是 InfiniTensor、RefactorGraph、ONNX 以及许多其他 AI 程序格式和 AI 编译器使用的表示形式——计算图。我们来看 [resnet](https://github.com/onnx/models/tree/main/validated/vision/classification/resnet) 的例子（<https://netron.app/>）。

> 学习过程中给，可以使用开源的 netron 应用实现可视化浏览 ONNX 文件，但是要开发接受 ONNX 输入的人工智能程序还是需要熟悉 API 的。

在 netron 可视化系统中，可以看到整个计算图实际上仅由 3 种元素组成。我们从上往下看：

- 最上面的 data 是模型的输入，同理，最下面还有模型的输出；
- 之后是箭头，可以看到这第一个箭头上还标记了一个形状。而且这个箭头是可以点的，我们点上去可以发现它实际上是一个张量；
  - 作为张量，它有数据类型、形状信息，可能还有数据。注意形状可以是未知的，因为除了输入形状，其他张量的形状都是被计算约束的，因此可以由输入张量的形状推导得到。实际上 AI 编译器前端最主要的功能就是形状推导，因此我们作业题也包括大量的形状推导题目；
  - 作为图上的边（Edge），它有源点和目标点信息。注意看 MaxPool 下面的箭头，这两个箭头指示的是同一个张量，指针指上去它们也是一起高亮的。查看内容发现这是一个单个源点两个目标点的边，这样的边表示它由源点计算产生，但是同时被多个目标点使用；
- 这种彩色的方框是图上的节点（Node），同时可以看到它是算子。无论作为算子还是作为节点，它具有类型（type）、模块（module）（这个模块实际上就是命名空间，为了避免类型重名用的）、名字（name）、属性（attribute）、输入（inputs）和输出（outputs）。注意，节点中实际上还隐含了一种信息，就是节点的一些输入实际上是常量，利于卷积算子的卷积核。这些常量本质上仍然是一些张量，只不过张量常量像全图输入一样不是由算子产生的。后面我把这种常量张量叫做图的内部边。可以把它们当作一种独特的东西，也可以当作一些特殊的张量；

如果往下翻，我们会发现整个图是由重复的一些结构组成的。实际上这就是一个循环结构，由于我们课程中会讲到的计算图中都是不体现控制流的，因此整个循环被展开到许多重复的结构。我这里展示的是最小版本的 resnet18，所以它层数很少。事实上有许多更大型的模型，例如大家都用过的 tiny llama 就是 22 层的 Transformer 模型，我们还有 50 层甚至 80 层的九格模型，可以想象如果对这样的模型可视化的话会得到特别巨大的一个图。

这些东西就组成了一个 resnet 模型。事实上，几乎任何 AI 程序都可以用输入输出、张量、算子这 3 种元素组成。由于这种可行性是我们刚刚根据那 3 个条件推导出来的，因此作为开发者可以信任这个结论。同时也要注意，一旦这 3 个条件不再满足，AI 编译器的这个设计也就立即会从合理简化转变为阻碍。同理，现在的现状是大语言模型经常需要一个独立的推理引擎，而不是接入一般的 AI 编译器，正是因为大语言模型引入了以 KV Cache 为首的许多针对生成类模型服务特点的优化手段，这些优化打破了 AI 程序顺序、无状态的假设，从而不再适于用这样的形式来表述了。

把这个描述很直接地映射到代码上，就得到了 InfiniTensor 的抽象。InfiniTensor 中，张量定义在 [tensor_base.h](https://github.com/InfiniTensor/InfiniTensor/blob/master/include/core/tensor_base.h) 和 [tensor.h](https://github.com/InfiniTensor/InfiniTensor/blob/master/include/core/tensor.h)：

```c++
protected:
    int dim;

    DataType dtype;
    vector<WRef<OperatorObj>> targets;
    WRef<OperatorObj> source;
    Blob data;
    Runtime runtime;
```

```c++
private:
    Shape shape;
    size_t _size; // Cache of Π(shape).
```

这些字段存储着刚刚模型中体现出的各项内容：数据类型、形状、数据、输入、输出。可以看到输入是唯一的，输出可以有多个。

算子定义在 [operator.h](https://github.com/InfiniTensor/InfiniTensor/blob/master/include/core/operator.h)：

```c++
protected:
    OpType type;
    TensorVec inputs;
    TensorVec outputs;
    vector<WRef<OperatorObj>> predecessors;
    vector<WRef<OperatorObj>> successors;
```

可以看到这里有算子的类型、输入张量、输出张量，额外还加入了前驱算子和后继算子，这两个字段实际上通过输入输出张量也可以找到，保存一份相当于是一种缓存，在图优化中可能会用到。

最后，在一张[图](https://github.com/InfiniTensor/InfiniTensor/blob/master/include/core/graph.h)中，显然就需要张量列表、算子列表：

```c++
protected:
    Runtime runtime;
    TensorVec tensors;
    OpVec ops;
```

这就是非常直观的设计，可以说没有什么讲的空间。所以 InfiniTensor 其实还真是一个很适合教学的系统，整体设计上就是顺理成章的，只要理解了 AI 程序的一些约束，人人都能写得出来。那么关于 AI 程序的抽象表示就讲完了，今天的课就到这里。

但是真的吗？如果 AI 程序的定义就是这样简单，为什么 InfiniTensor 的张量、算子和图代码却如此复杂呢？可以看到除了这些定义之外还有大量代码。这是因为作为一个带有后端和执行能力的 AI 编译器，显然要做的不仅仅是把 AI 程序表示出来而已。AI 编译器还应该具有两个方面的能力：程序的优化和执行。

优化，意味着对程序结构的变换、以及对参数细节的调整，使得原来的 AI 程序转换成另一个具有不同表示的程序，同时对相同的输入保持输出一致或者误差足够小（因为浮点计算程序要让输出保持绝对一致的条件是很苛刻的）。这意味着图不是能建出来、能遍历访问就完事了，而是必须支持插入删除节点和边。如果是学过数据结构的同学都应该可以意识到，一个有向无环图这样的高级数据结构，建和查的难度和增删的难度完全是两个概念。

而执行，意味着程序需要一个运行时，负责分配和释放各种资源。由于这个后端实际上是解释执行每个算子的，所以运行时还需要一种能够逐个取出要计算的算子、传入正确的参数并计算的结构。以及要执行，就需要每个算子的算法实现，而且对于支持硬件加速的后端来说还需要每个算子在每种硬件上都有对应的实现。这就是 InfiniTensor 以及任何其他 AI 编译器代码量总是如此大的原因。当然另一方面原因是它们都是 C++ 写的，C++ 搞依赖太麻烦了导致大家喜欢造很多轮子而且全写一起。

我们可以看到 [graph.h](https://github.com/InfiniTensor/InfiniTensor/blob/master/include/core/graph.h) 中关于增删算子的代码：

```c++
template <typename T, typename... Args> Ref<T> addOp(Args &&...args) {
    Ref<T> op = infini::make_ref<T>(this, std::forward<Args>(args)...);
    addOperatorAndConnect(op);
    return op;
}
```

```c++
void removeOperator(Operator op) {
    auto it = std::find(ops.begin(), ops.end(), op);
    if (it != ops.end())
        ops.erase(it);
}
```

关于增删张量的代码：

```c++
Tensor addTensor(Shape dim, DataType dtype = DataType::Float32);
Tensor addTensor(const Tensor &tensor);
TensorVec addTensor(const TensorVec &tensors);

void removeTensor(Tensor tensor) {
    auto it = std::find(tensors.begin(), tensors.end(), tensor);
    if (it != tensors.end())
        tensors.erase(it);
}
```

关于分配空间的代码：

```c++
void dataMalloc(bool useNaiveAllocator = false, size_t memPoolSize = 0);
```

本来应该还有用于执行的代码，但是因为设计原因执行被放在了这个 [Runtime](https://github.com/InfiniTensor/InfiniTensor/blob/b9699b0e7a868832203438a9f52a0fda76ad6ecb/include/core/runtime.h#L58) 里：

```c++
/**
 * @brief Execute a graph.
 *
 * @param graph
 * @param tune If there is no performance record, whether to tune it. These
 * can be independent method.
 * @param profiling Whether to print breakdown of time
 */
virtual void run(const Graph &graph, bool tune = false,
                 bool profiling = false) const = 0;
```

实话说我并不完全理解这些东西为啥要这样做。如果完全理解的话我也不会再去写 RefactorGraph 了。Anyway，这些代码总归是都在的，需要的同学可以自己来看。

到现在为止，总结一下，我们已经知道了为什么 AI 程序与通用编程遵循不同的范式，从而需要更适用的 AI 编译器。以及为什么 AI 程序很简单，AI 编译器却不简单。今天要讲的关于 AI 程序的概念和 InfiniTensor 实现的内容就是这些。

（提问时间）

好的，那么讲到现在，不知道大家是否觉得 AI 程序在实现中被表示为这个样子有些奇怪。用大量智能指针把图上的各个元素连接起来，不失为一种可行的方法，但这种表示只有利于改而不利于增删查。由于在存储结构中包含大量的冗余和缓存，对这种图表示进行增删需要大量的关联操作。例如，要从图中移除一个节点，意味着不但要从图的节点列表中移除节点，而且要移除节点输入输出中对节点的引用，还要移除节点输入输出的输入输出对节点的前驱后继引用。同时，要窥见图的拓扑全貌，则必须在大量相互连接的对象之间使用复杂的方法跳跃，要实现灵活的查找几乎必须依赖递归实现的图遍历算法，局部性相当差，不利于现代 CPU 处理。

因此，今天要讲的第二个部分是关于 RefactorGraph 这个项目是如何 refactor graph，来改善这些问题的。

RefactorGraph 有 3 个主要的设计目标：

1. 第一是解耦。我希望 AI 程序依赖的计算图表示是纯粹的，功能上可以专注于图的表示，剥离那些运行时之类的东西；
2. 第二是优化数据结构。使用更加缓存友好的数据结构来加速访问，尤其是加速大型计算图的检索和遍历；
3. 第三是希望支持分层不同的表示。InfiniTensor 的计算图从构造到执行都是同一个，这导致无论是算子还是图都变得比一般的需求更为复杂。因为所有结构都需要同时保存编译时需要的所有信息和运行时需要的所有信息。此外，对于支持多种硬件的情况，无论是硬件相关的优化还是硬件无关的优化都需要一直维持完整的硬件信息，这也带来了复杂性；

要实现这 3 个目标，最直接的方式就是按照数据结构的一般设计方式，进一步对数据结构进行抽象，把**数据**与**结构**分离。我们回顾一下，计算图的本质结构是什么？之前已经介绍过，计算图上包含三种元素，输入输出、张量和算子。其中，显然算子是图的节点，张量是图的边。一个节点可以接受多个入边和多个出边，这一点和教科书中的、一般的图是一致的，但是也有一些与一般的图不一致的地方：

1. 首先，算子本质上是一个函数调用，因此其输入和输出并不是集合，而是列表或者说元组，也就是说其输入输出是有序的，虽然也有部分算子（例如相加）满足交换律，可以互换其输入，但一般情况下，认为输入和输出的顺序不可随意调换；
2. 其次，张量也不同与一般的图中的边。标准的有向无环图中，我们认为在两个节点之间至多只有一条边，而输入输出不同的边则一定是两条不同的边。即使是支持重边的实现中，也仅仅是允许多个边有相同的输入输出。然而，计算图中的张量信息实际上是由作为节点的第几个输出决定的，换句话说，作为 N 节点的第 i 个出边，无论连接到那个节点的哪个输入，都具有相同的信息；

因此，这个图并非一般的图，必须经过特殊设计才能实现最优的表示。也就是说，无法调用别人的实现，必须自己造轮子。下面来看一下我造的轮子：

<https://github.com/InfiniTensor/RefactorGraph/blob/master/src/01graph_topo/include/graph_topo/container.h>

网上有一种说法，说 Rust 是一种语法噪音很大的语言。我在这里就不讨论关于语法噪音的概念了，显然这个代码已经尽力简洁了，然而相信大家都能从这一篇代码里体会到好的 C++ 代码写出来能有多啰嗦……

这一页出现的 GraphTopo 类，就是计算图拓扑的核心结构。GraphTopo 这个名字表示这个类的功能是提供计算图的拓扑结构存储。注意，这个类设计为存储纯粹的拓扑，这意味着这个类型中是**不保存**图上的数据的。节点和张量的信息被额外存储在两个 `vector` 中。一份图拓扑，一组节点和一组张量，这 3 个对象共同组成了一张完整的图。这个文件中定义了 Graph 这个结构模板：

```c++
template<class Node, class Edge>
struct Graph {
    GraphTopo topology;
    std::vector<Node> nodes;
    std::vector<Edge> edges;
};
```

通过这个结构模板定义，相信大家可以感受到这个设定的简洁性和通用性。

那么图拓朴这个类型是怎么实现对拓扑结构的高效存储呢？我们先来看图拓扑类的公开部分：

```c++
class NodeRef;
class Iterator;

Iterator begin() const noexcept;
Iterator end() const noexcept;
size_t globalInputsCount() const noexcept;
size_t globalOutputsCount() const noexcept;
size_t nodeCount() const noexcept;
size_t edgeCount() const noexcept;
range_t<count_t> globalInputs() const noexcept;
slice_t<count_t> globalOutputs() const noexcept;
slice_t<count_t> connections() const noexcept;

std::string toString() const;
```

首先出现的是两个内部类型，看名字可以知道一个是节点的引用，一个是迭代器。在现代高级语言中，自定义迭代器是特别重要的一种功能，同时通常也特别麻烦。关于 Rust 自定义迭代器的方法我在基础阶段讲了，但是 C++ 的我没有讲，因为我认为这个写法完全不可理喻……所以关于 C++ 迭代器这个部分我依然不讲，包括这里的代码，大家只要知道这个 `Iterator` 类加上下面的 `begin`、`end` 方法一起可以让这个类型支持 C++ 的 for each 语法就行了。从 `end` 下面开始看，图拓扑提供了 `globalInputsCount`、`globalOutputsCount`、`nodeCount`、`edgeCount`，也就是整个图的输入数量、输出数量、节点数量和边数量。要注意的是全图的输入和输出实际上就是一些边，因此实际上这个边数是覆盖输入输出数量的。接下来是 `globalInputs`、`globalOutputs` 和 `connections`，这 3 个是用于遍历全图输入、全图输出和全图中所有连接关系的。注意，由于一些原因，虽然整个 RefactorGraph 是适用 C++20 标准的，但 GraphTopo 这个类型维持了 C++17 兼容性，所以它用不了 C++20 更新的 range 库和 `std::span`，所以这里用了我手写的 range 和 slice。可以看到这里的全图输入、全图输出和连接关系都是用这个 `count_t` 类型表示的，实际上这个类型就是一个无符号整数的别名。也就是说，图拓扑中实际上是用序号数字表示拓扑结构的。这个设定事实上锁死了图结构中节点和边必须用可以随机访问的容器来表示，这差不多意味着必须是 `std::vector`。最后还有一个 `toString()`，方便调试用的，可以可视化整张图。

我们现在来记住一些内部类型的定义和字段的类型：

```c++
struct Node {
    count_t
        _localEdgesCount,
        _inputsCount,
        _outputsCount;
};
using OutputEdge = count_t;

count_t _lenIn, _lenOut;
std::vector<OutputEdge> _connections;
std::vector<Node> _nodes;
```

来看这个类型的实现：

<https://github.com/InfiniTensor/RefactorGraph/blob/master/src/01graph_topo/src/container.cc>

```c++
auto GraphTopo::globalInputsCount() const noexcept -> size_t { return _lenIn; }
auto GraphTopo::globalOutputsCount() const noexcept -> size_t { return _lenOut; }
auto GraphTopo::nodeCount() const noexcept -> size_t { return static_cast<count_t>(_nodes.size()); }
auto GraphTopo::edgeCount() const noexcept -> size_t {
    return std::accumulate(_nodes.begin(), _nodes.end(), _lenIn,
                            [](auto const acc, auto const &n) { return acc + n._localEdgesCount + n._outputsCount; });
}
auto GraphTopo::globalInputs() const noexcept -> range_t<count_t> {
    return range0_(_lenIn);
}
auto GraphTopo::globalOutputs() const noexcept -> slice_t<count_t> {
    return slice(_connections.data(), static_cast<size_t>(_lenOut));
}
auto GraphTopo::connections() const noexcept -> slice_t<count_t> {
    return slice(_connections.data(), _connections.size());
}
```

可以看到这几个方法的实现都非常简单。首先，前 3 行告诉我们字段中 `_lenIn` 就是全图输入的数量，`_lenOut` 就是全图输出的数量，`_nodes` 就是所有节点的集合，所以它的长度就是节点数量，而边的数量则有些奇怪。可以看到这里用了 `std::accumulate`，所以它是在累加。累加的内容是全图输入的数量加上所有节点的 `_localEdgesCount` 和 `_outputsCount`。然后继续看，所有全图输入的序号是从 0 到全图输入的数量，所有全图输出的序号是这个 `_connections` 集合中最前面的 `_lenOut` 个元素的值。其实看到这里，整个数据结构已经比较清晰了。

实际上整个图拓扑本质上是由 3 个列表组成，节点表、边表和连接关系表，三个表的元素都是严格有序的。但是在数据结构定义中只能看到 2 个表，是因为边表实际上不需要存储任何东西，所以省略掉了，或者也可以理解成这里还有一个由边数个大小为零的元素组成的虚拟的数组：

```c++
Node       nodes      [node_count];
Edge       edges      [edge_count];       // size = 0
Connection connections[connection_count];
```

其中，节点表中保存着每个节点对应的边和连接关系。对应的边指的是由节点产生的边，包括首次被使用的常量张量以及由这个节点产生的输出边。这些边按照顺序存放在虚拟的边表上。所以每个节点为边表连接 `_localEdgesCount + _outputsCount` 个边。另外，由于全图数据边既不由任何算子产生，又需要有序，不能随着使用保存，所以定义边表的前 `_lenIn` 个边是全图入边。而每个节点的入边则保存在连接关系表中，每个节点占用连接关系表的 `_inputsCount` 项。连接关系表里的项是边的序号，因此定位某个算子占用连接关系表的哪些位置，就可以快速找到它有顺序的入边是哪些。另外，全图输出也是有序的连接关系，但不是任何节点的入边，所以定义连接关系表的前 `_lenOut` 个序号是全图输出。

可以看到，这个数据结构是一种高度压缩的数据结构，实际上并不方便随机访问任何元素的信息，但是它对从前到后的遍历极度友好，而实际上在大部分场景中确实只需要遍历访问所有节点。由于这个数据结构定义非常精简，所以对它的分析和操作也是很容易的，通过在它上面扩展 searcher、builder 等结构，就可以实现进一步的快速访问、用户友好的构建和修改等功能。另外，由于图结构采用了拓扑与数据分离的设计，在许多不依赖拓扑的操作中，可以直接遍历节点表和边表，完全不触碰拓扑结构，这进一步优化了访问性能。
