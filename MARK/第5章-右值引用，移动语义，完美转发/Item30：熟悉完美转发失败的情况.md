---
title: Item 30 熟悉完美转发的失败 case
tags: ["完美转发"]
categories: [Effective Modern C++]
description: '熟悉完美转发失败的情况'
date: 2024-08-17 17:00:40
cover: https://images.pexels.com/photos/20410939/pexels-photo-20410939.jpeg
---

# 前言

C++11 最显眼的功能之⼀就是完美转发功能。开始使⽤，你就发现“完美”，理想与现实还是有差距。C++11 的完美转发是⾮常好⽤，但是只有当你愿意忽略⼀些失败情况，这个章节就是使你熟悉这些情形。

在我们开始探索之前，有必要回顾⼀下“完美转发”的含义。

> **“转发”仅表⽰将⼀个函数的参数传递给另⼀个函数**。对于被传递的第⼆个函数⽬标是**收到与第⼀个函数完全相同的对象**。**这就排除了按值传递参数，因为它们是原始调⽤者传⼊内容的副本**。我们希望被转发的函数能够可以与原始函数⼀起使⽤对象。指着参数也被排除在外，因为我们不想强迫调⽤者传⼊指针。关于通⽤转发，我们将处理引⽤参数。

完美转发意味着我们不仅转发对象，我们还转发显著的特征：它们的类型，是左值还是右值，是 `const` 还是 `volatile`。结合到我们会处理引⽤参数，这意味着我们将使⽤通⽤引⽤（参⻅Item24），因为通⽤引⽤参数被传⼊参数时才确定是左值还是右值

假定我们有⼀些函数f，然后想编写⼀个转发给它的函数（就使⽤⼀个函数模板）。我们需要的核⼼看起来像是这样：

```cpp
template<typename T>
void fwd(T&& param) // accept any argument
{
    f(std::forward<T>(param)); // forward it to f
}
```

从本质上说，转发功能是通⽤的。例如 `fwd` 模板，接受任何类型的采纳数，并转发得到的任何参数。这种通⽤性的逻辑扩展是转发函数不仅是模板，而且是可变模板，因此可以接受任何数量的参数。`fwd` 的可变个是如下：

```cpp
template<typename... Ts>
void fwd(Ts&&... params) // accept any arguments
{
    f(std::forward<Ts>(params)...); // forward them to f
}
```

这种形式你会在标准化容器 `emplace` 中（参⻅Item42）和只能容器的⼯⼚函数 `std::make_unique` 和 `std::make_shared` 中（参⻅Item21）看到。

给定我们的⽬标函数f和被转发的函数 `fwd`，如果f使⽤特定参数做⼀件事，但是 `fwd` 使⽤相同的参数做另⼀件事，完美转发就会失败：

```cpp
f(expression); // if this does one thing
fwd(expression); // but this does something else, fwd fails to perfectly forward expression to f
```

导致这种失败的原因有很多。知道它们是什么以及如何解决它们很重要，因此让我们来看看那种参数⽆法做到完美转发。

# Braced initializers（⽀撑初始化器）

假定 `f` 这样声明：

```cpp
void f(const std::vector<int>& v);
```

在这个例子中，通过列表初始化器，

```cpp
f({1,2,3});     // fine "{1,2,3}" implicitly converted to std::vector<int>
```

但是传递相同的列表初始化器给 `fwd` 不能编译

```cpp
fwd({1,2,3});       // error! doesn't compile
```

这是因为这是完美转发失效的⼀种情况。

所有这种错误有相同的原因。在对 `f` 的直接调⽤（例如 `f({1,2,3})`），编译器看到传⼊的参数是声明中的类型。如果类型不匹配，就会执⾏隐式转换操作使得调⽤成功。在上⾯的例⼦中，从 `{1,2,3}` ⽣成了临时变量 `std::vector<int>` 对象，因此f的参数会绑定到 `std::vector<int>` 对象上

当通过调⽤函数模板 `fwd` 调⽤f时，编译器不再⽐较传⼊给 `fwd` 的参数和f的声明中参数的类型。代替的是，推导传⼊给 `fwd` 的参数类型，然后⽐较推导后的参数类型和 `f` 的声明类型。当下⾯情况任何⼀个发⽣时，完美转发就会失败：

- **编译器不能推导出⼀个或者多个 `fwd` 的参数类型**，编译器就会报错
- **编译器将⼀个或者多个fwd的参数类型推导错误**。在这⾥，“错误”可能意味着 `fwd` 将⽆法使⽤推导出的类型进⾏编译，但是也可能意味着调⽤者 `f` 使⽤ `fwd` 的推导类型对⽐直接传⼊参数类型表现出不⼀致的⾏为。这种不同⾏为的原因可能是因为 `f` 的函数重载定义，并且由于是“不正确的”类型推导，在 `fwd` 内部调⽤f和直接调⽤ `f` 将重载不同的函数。

在上⾯的 `f({1,2,3})` 例⼦中，问题在于，如标准所⾔，将括号初始化器传递给未声明为 `std::initializer_list` 的函数模板参数，该标准规定为“⾮推导上下⽂”。简单来讲，这意味着编译器在对 `fwd` 的调⽤中推导表达式 `{1,2,3}` 的类型，因为 `fwd` 的参数没有声明为 `std::initializer_list`。对于 `fwd` 参数的推导类型被阻⽌，编译器只能拒绝该调⽤。

有趣的是，`Item2` 说明了使⽤ **braced initializer** 的 `auto` 的变量初始化的类型推导是成功的。这种变量被视为 `std::initializer_list` 对象，在转发函数应推导为 `std::initializer_list` 类型的情况，这提供了⼀种简单的解决⽅法----使⽤ `auto` 声明⼀个局部变量，然后将局部变量转发：

```cpp
auto il = {1,2,3}; // il's type deduced to be std::initializer_list<int>
fwd(il); // fine, perfect-forwards il to f
```

# 0或者NULL作为空指针

Item8 说明当你试图传递 0 或者 `NULL` 作为空指针给模板时，类型推导会出错，推导为⼀个整数类型而不是指针类型。结果就是不管是 0 还是 `NULL` 都不能被完美转发为空指针。解决⽅法⾮常简单，使⽤ `nullptr` 就可以了，具体的细节，参考Item 8.

# 仅声明的整数静态 `const` 数据成员

通常，⽆需在类中定义整数静态 `const` 数据成员；声明就可以了。这是因为编译器会对此类成员

```cpp
class Widget {
public:
    static const std::size_t MinVals = 28; // MinVal's declaration
    ...
};

... // no defn. for MinVals
std::vector<int> widgetData;
widgetData.reserve(Widget::MinVals); // use of MinVals
```

这⾥，我们使⽤ `Widget::MinVals` （或者简单点 `MinVals`）来确定 `widgetData` 的初始容量，即使 `MinVals` 缺少定义。编译器通过将值 28 放⼊所有位置来补充缺少的定义。没有为 `MinVals` 的值留存储空间是没有问题的。如果要使⽤ `MinVals` 的地址（例如，有⼈创建了 `MinVals` 的指针），则 `MinVals` 需要存储（因为指针总要有⼀个地址），尽管上⾯的代码仍然可以编译，但是链接时就会报错，直到为 `MinVals` 提供定义。

按照这个思路，想象下f（转发参数给 `fwd` 的函数）这样声明：

```cpp
void f(std::size_t val);
```

使⽤ `MinVals` 调⽤ `f` 是可以的，因为编译器直接将值 28 代替 `MinVals`：

```cpp
f(widget::MinVals);     // fine, treated as "28"
```

同样的，如果尝试通过 `fwd` 来调⽤ `f`

```cpp
fwd(Widget::MinVals); // error! shouldn't link
```

代码可以编译，但是不能链接。就像使⽤ `MinVals` 地址表现⼀样，确实，底层的问题是⼀样的。

尽管代码中没有使⽤ `MinVals` 的地址，但是 `fwd` 的参数是通⽤引⽤，而引⽤，在编译器⽣成的代码中，通常被视作指针。在程序的⼆进制底层代码中指针和引⽤是⼀样的。在这个⽔平下，引⽤只是可以⾃动取消引⽤的指针。在这种情况下，通过引⽤传递 `MinVals` 实际上与通过指针传递 `MinVals` 是⼀样的，因此，必须有内存使得指针可以指向。通过引⽤传递整型 `static const` 数据成员，必须定义它们，这个要求可能会造成完美转发失败，即使等效不使⽤完美转发的代码成功。（译者注：这⾥意思应该是没有定义，完美转发就会失败）

可能你也注意到了在上述讨论中我使⽤了⼀些模棱两可的词。代码“不应该”链接，引⽤“通常”被看做指针。传递整型 `static const` 数据成员“通常”要求定义。看起来就像有些事情我没有告诉你......

确实，根据标准，通过引⽤传递 `MinVals` 要求有定义。但不是所有的实现都强制要求这⼀点。所以，取决于你的编译器和链接器，你可能发现你可以在未定义的情况使⽤完美转发，恭喜你，但是这不是那样做的理由。为了具有可移植性，只要给整型 `static const` 提供⼀个定义，⽐如这样：

```cpp
const std::size_t Widget::MinVals; // in Widget's .cpp file
```

注意定义中不要重复初始化（这个例⼦中就是赋值 28）。不要忽略这个细节，否则，编译器就会报错，提醒你只初始化⼀次。

# 重载的函数名称和模板名称

假定我们的函数 `f`（通过 `fwd` 完美转发参数给 `f`）可以通过向其传递执⾏某些功能的函数来定义其⾏为。假设这个函数参数和返回值都是整数，`f` 声明就像这样：

```cpp
void f(int (*pf)(int)); // pf = "process function"
```

值得注意的是，也可以使⽤更简单的⾮指针语法声明。这种声明就像这样，含义与上⾯是⼀样的：

```cpp
void f(int pf(int)); // declares same f as above
```

⽆论哪种写法，我们都有了⼀个重载函数，`processVal`:

```cpp
int processVal(int value);
int processVal(int value, int priority);
```

我们可以传递 `processVal` 给 `f`

```cpp
f(processVal); // fine
```

但是有⼀点要注意，`f` 要求⼀个函数指针，但是 `processVal` 不是⼀个函数指针或者⼀个函数，它是两个同名的函数。但是，编译器可以知道它需要哪个：通过参数类型和数量来匹配。因此选择了⼀个 `int` 参数的 `processVal` 地址传递给f

⼯作的基本机制是让编译器帮选择f的声明选择⼀个需要的 `processVal`。但是，`fwd` 是⼀个函数模板，没有需要的类型信息，使得编译器不可能帮助⾃动匹配⼀个合适的函数：

```cpp
fwd(processVal); // error! which processVal?
```

`processVal` 没有类型信息，就不能类型推导，完美转发失败。同样的问题会发⽣在如果我们试图使⽤函数模板代替重载的函数名。⼀个函数模板是未实例化的函数，表⽰⼀个函数族：

```cpp
template<typename T>
T workOnVal(T param) { ... } // template for processing values
fwd(workOnVal); // error! which workOnVal instantiation ?
```

获得像 `fwd` 的完美转发接受⼀个重载函数名或者模板函数名的⽅式是指定转发的类型。⽐如，你可以创造与 `f` 相同参数类型的函数指针，通过 `processVal` 或者 `workOnVal` 实例化这个函数指针（可以引导⽣成代码时正确选择函数实例），然后传递指针给 `f`：

```cpp
using ProcessFuncType = int (*)(int); // make typedef; see Item 9
PorcessFuncType processValPtr = processVal; // specify needed signature for
processVal
fwd(processValPtr); // fine
fwd(static_cast<ProcessFuncType>(workOnVal)); // alse fine
```

当然，这要求你知道 `fwd` 转发的函数指针的类型。对于完美转发来说这⼀点并不合理，毕竟，完美转发被设计为转发任何内容，如果没有⽂档告诉你转发的类型，你如何知道？（译者注：这⾥应该想表达，这是解决重载函数名或者函数模板的解决⽅案，但是这是完美转发本⾝的问题）

# 位域

完美转发最后⼀种失败的情况是函数参数使⽤位域这种类型。为了更直观的解释，IPv4 的头部可以如下定义：

```cpp
struct IPv4Header {
    std::uint32_t version:4, IHL:4,DSCP:6,ECN:2,totalLength:16;
    ...
};
```

如果声明我们的函数 `f`（转发函数fwd的⽬标）为接收⼀个 `std::size_t` 的参数，则使⽤ `IPv4Header` 对象的 `totalLength` 字段进⾏调⽤没有问题：

```cpp
void f(std::size_t sz);
IPv4Header h;
...
f(h.totalLength);// fine
```

如果通过 `fwd` 转发 `h.totalLength` 给 `f` 呢，那就是⼀个不同的情况了：

```cpp
fwd(h.totalLength); // error!
```

问题在于 `fwd` 的参数是引⽤，而 `h.totalLength` 是⾮常量位域。听起来并不是那么糟糕，但是 C++ 标准⾮常清楚地谴责了这种组合：⾮常量引⽤不应该绑定到位域。禁⽌的理由很充分。位域可能包含了机器字节的任意部分（⽐如 32 位 `int` 的 3-5 位），但是⽆法直接定位。我之前提到了在硬件层⾯引⽤和指针时⼀样的，所以没有办法创建⼀个指向任意 `bit` 的指针（C++规定你可以指向的最小单位是char），所以就没有办法绑定引⽤到任意 `bit` 上。

⼀旦意识到接收位域作为参数的函数都将接收位域的副本，就可以轻松解决位域不能完美转发的问题。毕竟，没有函数可以绑定引⽤到位域，也没有函数可以接受指向位域的指针（不存在这种指针）。这种位域类型的参数只能按值传递，或者有趣的事，常量引⽤也可以。在按值传递时，被调⽤的函数接受了⼀个位域的副本，而且事实表明，位域的常量引⽤也是将其“复制”到普通对象再传递。

传递位域给完美转发的关键就是利⽤接收参数函数接受的是⼀个副本的事实。你可以⾃⼰创建副本然后利⽤副本调⽤完美转发。在 `IPv4Header` 的例⼦中，可以如下写法：

```cpp
// copy bitfield value; see Item6 for info on init. form
auto length = static_cast<std::uint16_t>(h.totalLength);
fwd(length); // forward the copy
```

# 总结

在⼤多数情况下，完美转发⼯作的很好。你基本不⽤考虑其他问题。但是当其不⼯作时，当看起来合理的代码⽆法编译，或者更糟的是，⽆法按照预期运⾏时，了解完美转发的缺陷就很重要了。同样重要的是如何解决它们。在⼤多数情况 下，都很简单

- 完美转发会失败当模板类型推导失败或者推导类型错误
- 导致完美转发失败的类型有 `braced initializers`，作为空指针的 0 或者 `NULL`，只声明的整型 `static const` 数据成员，模板和重载的函数名，位域