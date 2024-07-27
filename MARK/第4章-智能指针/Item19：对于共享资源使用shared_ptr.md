---
title: Item 19 对于共享资源使⽤ std::shared_ptr
tags: ["智能指针", "std::shared_ptr"]
categories: [Effective Modern C++]
description: 'std::shared_ptr'
date: 2024-07-26 15:37:21
cover: https://images.pexels.com/photos/20623114/pexels-photo-20623114.jpeg
---

# 条款19：对于共享资源使用std::shared_ptr

一个很有意思的说法：程序员使⽤带垃圾回收的语⾔指着 C++ 笑看他们如何防⽌资源泄露。“真是原始啊！”他们嘲笑着说。“你们没有从 1960 年的 Lisp 那⾥得到启发吗，机器应该⾃⼰管理资源的⽣命周期而不应该依赖⼈类。” C++ 程序员翻⽩眼。“你得到的启发就是只有内存算资源，其他资源释放都是⾮确定性的你知道吗？我们更喜欢通⽤，可预料的销毁，谢谢你。”为什么我们不能同时有两个完美的世界：⼀个⾃动⼯作的世界（垃圾回收），⼀个销毁可预测的世界（析构）？

C++11中的 `std::shared_ptr` 将两者组合了起来。⼀个通过 `std::shared_ptr` 访问的对象其⽣命周期由指向它的指针们共享所有权（shared ownership）。没有特定的 `std::shared_ptr` 拥有该对象。相反，所有指向它的 `std::shared_ptr` 都能相互合作确保在它不再使⽤的那个点进⾏析构。**当最后⼀个 `std::shared_ptr` 到达那个点，`std::shared_ptr` 会销毁它所指向的对象。**就垃圾回收来说，客⼾端不需要关⼼指向对象的⽣命周期，而对象的析构是确定性的。

**`std::shared_ptr` 通过引⽤计数来确保它是否是最后⼀个指向某种资源的指针，引⽤计数关联资源并跟踪有多少 `std::shared_ptr` 指向该资源**。`std::shared_ptr` 构造函数递增引⽤计数值（注意是通常——原因参⻅下⾯），析构函数递减值，拷⻉赋值运算符可能递增也可能递减值。（如果 sp1 和 sp2 是`std::shared_ptr` 并且指向不同对象，赋值运算符 `sp1 = sp2` 会使 sp1 指向 sp2 指向的对象。直接效果就是 sp1 引⽤计数减⼀，sp2 引⽤计数加⼀。）如果 `std::shared_ptr` 发现引⽤计数值为零，没有其他 `std::shared_ptr` 指向该资源，它就会销毁资源。

引⽤计数暗⽰着性能问题：
- **`std::shared_ptr` ⼤小是原始指针的两倍，因为它内部包含⼀个指向资源的原始指针，还包含⼀个资源的引⽤计数值。**
- 引⽤计数必须动态分配。 理论上，引⽤计数与所指对象关联起来，但是被指向的对象不知道这件事情（译注：不知道有指向⾃⼰的指针）。因此它们没有办法存放⼀个引⽤计数值。Item21 会解释使⽤ `std::make_shared` 创建 `std::shared_ptr` 可以避免引⽤计数的动态分配，但是还存在⼀些 `std::make_shared` 不能使⽤的场景，这时候引⽤计数就会动态分配。
- **递增递减引⽤计数必须是原⼦性的**，因为多个 `reader、writer` 可能在不同的线程。⽐如，指向某种资源的 `std::shared_ptr` 可能在⼀个线程执⾏析构，在另⼀个不同的线程，`std::shared_ptr` 指向相同的对象，但是执⾏的确是拷⻉操作。原⼦操作通常⽐⾮原⼦操作要慢，所以即使是引⽤计数，你也应该假定读写它们是存在开销的。
  
上面说到 `std::shared_ptr` 构造函数只是 **“通常”递增指向对象的引⽤计数** 。创建⼀个指向对象的 `std::shared_ptr` ⾄少产⽣了⼀个指向对象的智能指针，为什么我没说总是增加引⽤计数值？

原因是移动构造函数的存在。从另⼀个 `std::shared_ptr` 移动构造新 `std::shared_ptr` 会将原来的 `std::shared_ptr` 设置为 `null`，那意味着⽼的 `std::shared_ptr` 不再指向资源，同时新的 `std::shared_ptr` 指向资源。这样的结果就是不需要修改引⽤计数值。因此移动 `std::shared_ptr` 会⽐拷⻉它要快：拷⻉要求递增引⽤计数值，移动不需要。移动赋值运算符同理，所以移动赋值运算符也⽐拷⻉赋值运算符快(通常情况下)。

类似 `std::unique_ptr` （参加Item18），`std::shared_ptr` 使⽤ `delete` 作为资源的默认销毁器，但是它也⽀持⾃定义的销毁器。这种⽀持有别于 `std::unique_ptr` 。对于 `std::unique_ptr` 来说，销毁器类型是智能指针类型的⼀部分。对于 `std::shared_ptr` 则不是：

```cpp
auto loggingDel = [](Widget *pw) //⾃定义销毁器
                { // (和Item 18⼀样)
                makeLogEntry(pw);
                delete pw;
                };
std::unique_ptr< // 销毁器类型是
    Widget, decltype(loggingDel) // ptr类型的⼀部分
    > upw(new Widget, loggingDel);
std::shared_ptr<Widget> // 销毁器类型不是
spw(new Widget, loggingDel); // ptr类型的⼀部分
```

`std::shared_ptr` 的设计更为灵活。考虑有两个 `std::shared_ptr`，每个⾃带不同的销毁器（⽐如通过 lambda 表达式⾃定义销毁器）：

```cpp
auto customDeleter1 = [](Widget *pw) { … };
auto customDeleter2 = [](Widget *pw) { … };
std::shared_ptr<Widget> pw1(new Widget, customDeleter1);
std::shared_ptr<Widget> pw2(new Widget, customDeleter2);
```

因为 pw1 和 pw2 有相同的类型，所以它们都可以放到存放那个类型的对象的容器中：

```cpp
std::vector<std::shared_ptr<Widget>> vpw{ pw1, pw2 };
```

它们也能相互赋值，也可以传⼊形参为 `std::shared_ptr<Widget>` 的函数。但是 `std::unique_ptr` 就不⾏，因为 `std::unique_ptr` 把销毁器视作类型的⼀部分。

另⼀个不同于 `std::unique_ptr` 的地⽅是，指定自定义销毁器不会改变 `std::shared_ptr` 对象的⼤小。**不管销毁器是什么，⼀个 `std::shared_ptr` 对象都是两个指针⼤小。**这是个好消息，但是它应该让你隐隐约约不安。⾃定义销毁器可以是函数对象，函数对象可以包含任意多的数据。它意味着函数对象是任意⼤的。`std::shared_ptr` 怎么能引⽤⼀个任意⼤的销毁器而不使⽤更多的内存？

它不能。它必须使⽤更多的内存。然而，那部分内存不是 `std::shared_ptr` 对象的⼀部分。那部分在堆上⾯，只要 `std::shared_ptr` ⾃定义了分配器，那部分内存随便在哪都⾏。我前⾯提到了`std::shared_ptr` 对象包含了所指对象的引⽤计数。没错，但是有点误导⼈。因为引⽤计数是另⼀个更⼤的数据结构的⼀部分，那个数据结构通常叫做控制块。控制块包含除了引⽤计数值外的⼀个⾃定义销毁器的拷⻉，当然前提是存在⾃定义销毁器。如果⽤⼾还指定了⾃定义分配器，控制器也会包含⼀个分配器的拷⻉。控制块可能还包含⼀些额外的数据，正如 Item21 提到的，⼀个次级引⽤计数 `weak count`，但是⽬前我们先忽略它。

当 `std::shared_ptr` 对象⼀创建，对象控制块就建⽴了。⾄少我们期望是如此。通常，对于⼀个创建指向对象的 `std::shared_ptr` 的函数来说不可能知道是否有其他 `std::shared_ptr` 早已指向那个对象，所以控制块的创建会遵循下⾯⼏条规则：

- **`std::make_shared` 总是创建⼀个控制块** (参⻅Item21)。它创建⼀个指向新对象的指针，所以可以肯定 `std::make_shared` 调⽤时对象不存在其他控制块。
- **当从独占指针上构造出 `std::shared_ptr` 时会创建控制块（即 `std::unique_ptr` 或者 `std::auto_ptr`）**。独占指针没有使⽤控制块，所以指针指向的对象没有关联其他控制块。（作为构造的⼀部分，`std::shared_ptr` 侵占独占指针所指向的对象的独占权，所以 `std::unique_ptr` 被设置为null）
- **当从原始指针上构造出 `std::shared_ptr` 时会创建控制块**。如果你想从⼀个早已存在控制块的对象上创建 `std::shared_ptr`，你将假定传递⼀个 `std::shared_ptr` 或者 `std::weak_ptr` 作为构造函数实参，而不是原始指针。⽤ `std::shared_ptr` 或者 `std::weak_ptr` 作为构造函数实参创建 `std::shared_ptr` 不会创建新控制块，因为它可以依赖传递来的智能指针指向控制块。

违反这些规则造成的后果就是**从原始指针上构造超过⼀个 std::shared_ptr 就会出现未定义⾏为**，因为指向的对象有多个控制块关联。多个控制块意味着多个引⽤计数值，多个引⽤计数值意味着对象将会被销毁多次（每个引⽤计数⼀次）。那意味着下⾯的代码是有问题的，很有问题，问题很⼤：

```cpp
auto pw = new Widget; // pw是原始指针
…
std::shared_ptr<Widget> spw1(pw, loggingDel); // 为*pw创建控制块
…
std::shared_ptr<Widget> spw2(pw, loggingDel); // 为*pw创建第⼆个控制块
```

现在，传给 `spw1` 的构造函数⼀个原始指针，它会为指向的对象创建⼀个控制块（引⽤计数值在⾥⾯）。这种情况下，指向的对象是 `*pw`。就其本⾝而⾔没什么问题，但是将同样的原始指针传递给 `spw2` 的构造函数会再次为 `*pw` 创建⼀个控制块。因此 `*pw` 有两个引⽤计数值，每⼀个最后都会变成零，然后最终导致 `*pw` 销毁两次。第⼆个销毁会产⽣未定义⾏为。

第⼀，避免传给 `std::shared_ptr` 构造函数原始指针。通常替代⽅案是使⽤ `std::make_shared` (参⻅Item21)，不过上⾯例⼦中，我们使⽤了⾃定义销毁器，⽤ `std::make_shared` 就没办法做到。第⼆，如果你必须传给 `std::shared_ptr` 构造函数原始指针，直接传 `new` 出来的结果，不要传指针变量。如果上⾯代码第⼀部分这样重写：

```cpp
std::shared_ptr 给我们上了两堂课。第⼀，避免传给 std::shared_ptr 构造函数原始指针。通常替
代⽅案是使⽤ std::make_shared (参⻅Item21)，不过上⾯例⼦中，我们使⽤了⾃定义销毁器，⽤
std::make_shared 就没办法做到。第⼆，如果你必须传给 std::shared_ptr 构造函数原始指针，直
接传new出来的结果，不要传指针变量。如果上⾯代码第⼀部分这样重写：

```cpp
std::shared_ptr<Widget> spw1(new Widget, // 直接使⽤new的结果
loggingDel);
```

会少了很多创建第⼆个从原始指针上构造 `std::shared_ptr` 的诱惑。相应的，创建 `spw2` 也会很⾃然的⽤ `spw1` 作为初始化参数（即⽤ `std::shared_ptr` 拷⻉构造），那就没什么问题了：

```cpp
std::shared_ptr<Widget> spw2(spw1); // spw2使⽤spw1⼀样的控制块
```

⼀个尤其令⼈意外的地⽅是使⽤ `this` 原始指针作为 `std::shared_ptr` 构造函数实参的时候可能导致创建多个控制块。假设我们的程序使⽤ `std::shared_ptr` 管理 `Widget` 对象，我们有⼀个数据结构⽤于跟踪已经处理过的 `Widget` 对象：

```cpp
std::vector<std::shared_ptr<Widget>> processedWidgets;
```

继续，假设 `Widget` 有⼀个⽤于处理的成员函数：

```cpp
class Widget {
public:
    …
    void process();
    …
};
```

对于 `Widget::process` 看起来合理的代码如下：

```cpp
void Widget::process()
{
    … // 处理Widget
    processedWidgets.emplace_back(this); // 然后将他加到已处理过的Widget的列表中
                                        // 这是错的
}
```

这是错的——或者⾄少⼤部分是错的。（错误的部分是传递 `this`，而不是使⽤了 `emplace_back`。如果你不熟悉 `emplace_back`，参⻅ Item42）。上⾯的代码可以通过编译，但是向容器传递⼀个原始指针（`this`），`std::shared_ptr` 会由此为指向的对象（`*this`）创建⼀个控制块。那看起来没什么问题，直到你意识到如果成员函数外⾯早已存在指向Widget对象的指针，它是未定义⾏为的.

`std::shared_ptr` API已有处理这种情况的设施。它的名字可能是 C++ 标准库中最奇怪的⼀个：`std::enable_shared_from_this`。它是⼀个⽤做基类的模板类，模板类型参数是某个想被 `std::shared_ptr` 管理且能从该类型的this对象上安全创建 `std::shared_ptr` 指针的存在。在我们的例⼦中，`Widget`将会继承⾃ `std::enable_shared_from_this` ：

```cpp
class Widget: public std::enable_shared_from_this<Widget> {
public:
    …
    void process();
    …
};
```

正如我所说，`std::enable_shared_from_this` 是⼀个⽤作基类的模板类。它的模板参数总是某个继承⾃它的类，所以Widget继承⾃ `std::enable_shared_from_this<Widget>`。如果某类型继承⾃⼀个由该类型（译注：作为模板类型参数）进⾏模板化得到的基类这个东西让你⼼脏有点遭不住，别去想它就好了。代码完全合法，而且它背后的设计模式也是没问题的，并且这种设计模式还有个标准名字，尽管该名字和 `std::enable_shared_from_this` ⼀样怪异。这个标准名字就是奇异递归模板模式(`TheCuriously Recurring Template Pattern(CRTP)`)。如果你想学更多关于它的内容，请搜索引擎⼀展⾝⼿，现在我们要回到 `std::enable_shared_from_this` 上。

`std::enable_shared_from_this` 定义了⼀个成员函数，成员函数会创建指向当前对象的 `std::shared_ptr` 却不创建多余控制块。这个成员函数就是 `shared_from_this`，⽆论在哪当你想使⽤ `std::shared_ptr` 指向 `this` 所指对象时都请使⽤它。这⾥有个 `Widget::process` 的安全实现：

```cpp
void Widget::process()
{
// 和之前⼀样，处理Widget
…
// 把指向当前对象的shared_ptr加⼊processedWidgets
processedWidgets.emplace_back(shared_from_this());
}
```

从内部来说，`shared_from_this` 查找当前对象控制块，然后创建⼀个新的 `std::shared_ptr` 指向这个控制块。设计的依据是当前对象已经存在⼀个关联的控制块。要想符合设计依据的情况，必须已经存在⼀个指向当前对象的 `std::shared_ptr` (即调⽤ `shared_from_this` 的成员函数外⾯已经存在⼀个 `std::shared_ptr`)。如果没有 `std::shared_ptr` 指向当前对象（即当前对象没有关联控制块），⾏为是未定义的，`shared_from_this` 通常抛出⼀个异常。

要想防⽌客⼾端在调⽤ `std::shared_ptr` 前先调⽤ `shared_from_this`，继承⾃ `std::enable_shared_from_this` 的类通常将它们的构造函数声明为 `private`，并且让客⼾端通过⼯⼚⽅法创建 `std::shared_ptr`。以 `Widget` 为例，代码可以是这样：

```cpp
class Widget: public std::enable_shared_from_this<Widget> {
public:
// 完美转发参数的⼯⼚⽅法
template<typename... Ts>
static std::shared_ptr<Widget> create(Ts&&... params);
…
void process(); // 和前⾯⼀样
…
private:
…
};
```

现在，你可能隐约记得我们讨论控制块的动机是想了解 `std::shared_ptr` 关联⼀个控制块的成本。既然我们已经知道了怎么避免创建过多控制块，就让我们回到原来的主题。

控制块通常只占⼏个word⼤小，⾃定义销毁器和分配器可能会让它变⼤⼀点。通常控制块的实现⽐你想的更复杂⼀些。它使⽤继承，甚⾄⾥⾯还有⼀个虚函数（⽤来确保指向的对象被正确销毁）。这意味着使⽤ `std::shared_ptr` 还会招致控制块使⽤虚函数带来的成本。

了解了动态分配控制块，任意⼤小的销毁器和分配器，虚函数机制，原⼦引⽤计数修改，你对于 `std::shared_ptr` 的热情可能有点消退。可以理解，对每个资源管理问题来说都没有最佳的解决⽅案。但就它提供的功能来说，`std::shared_ptr` 的开销是⾮常合理的。在通常情况下，`std::shared_ptr`创建控制块会使⽤默认销毁器和默认分配器，控制块只需三个字节⼤小。它的分配基本上是⽆开销的。（开销被并⼊了指向的对象的分配成本⾥。细节参⻅ Item21）。对 `std::shared_ptr` 解引⽤的开销不会⽐原始指针⾼。执⾏原⼦引⽤计数修改操作需要承担⼀两个原⼦操作开销，这些操作通常都会⼀⼀映射到机器指令上，所以即使对⽐⾮原⼦指令来说，原⼦指令开销较⼤，但是它们仍然只是单个指令。对于每个被 `std::shared_ptr` 指向的对象来说，控制块中的虚函数机制产⽣的开销通常只需要承受⼀次，即对象销毁的时候。

作为这些轻微开销的交换，你得到了动态分配的资源的⽣命周期⾃动管理的好处。⼤多数时候，⽐起⼿动管理，使⽤ `std::shared_ptr` 管理共享性资源都是⾮常合适的。如果你还在犹豫是否能承受 `std::shared_ptr` 带来的开销，那就再想想你是否需要共享资源。如果独占资源可⾏或者可能可⾏，⽤ `std::unique_ptr` 是⼀个更好的选择。它的性能 `profile` 更接近于原始指针，并且从 `std::unique_ptr` 升级到 `std::shared_ptr` 也很容易，因为 `std::shared_ptr` 可以从 `std::unique_ptr` 上创建。

反之不⾏。当你的资源由 `std::shared_ptr` 管理，现在⼜想**修改资源⽣命周期管理⽅式是没有办法的**。即使引⽤计数为⼀，你也不能重新修改资源所有权，改⽤ `std::unique_ptr` 管理它。所有权和 `std::shared_ptr` 指向的资源之前签订的协议是“除⾮死亡否则永不分离”。不能离婚，不能废除，没有特许。

`std::shared_ptr` 不能处理的另⼀个东西是数组。和 `std::unique_ptr` 不同的是，`std::shared_ptr` 的 API 设计之初就是针对单个对象的，没有办法 `std::shared_ptr<T[]>`。⼀次⼜⼀次，“聪明”的程序员踌躇于是否该使⽤ `std::shared_ptr<T>` 指向数组，然后传⼊⾃定义数组销毁器。（即 `delete []`）。这可以通过编译，但是是⼀个糟糕的注意。⼀⽅⾯，`std::shared_ptr` 没有提供 `operator[]` 重载，所以数组索引操作需要借助怪异的指针算术。另⼀⽅⾯， `std::shared_ptr` ⽀持转换为指向基类的指针，这对于单个对象来说有效，但是当⽤于数组类型时相当于在类型系统上开洞。（出于这个原因， `std::unique_ptr` 禁⽌这种转换。）。更重要的是，C++11 已经提供了很多内置数组的候选⽅案（⽐如 `std::array` , `std::vector` , `std::string` ）。声明⼀个指向傻⽠数组的智能指针⼏乎总是标识着糟糕的设计。

总结：
- `std::shared_ptr` 为任意共享所有权的资源⼀种⾃动垃圾回收的便捷⽅式。
- 较之于 `std::unique_ptr`，`std::shared_ptr` 对象通常⼤两倍，控制块会产⽣开销，需要原⼦引⽤计数修改操作。
- 默认资源销毁是通过 `delete`，但是也⽀持⾃定义销毁器。销毁器的类型是什么对于 `std::shared_ptr` 的类型没有影响。
- 避免从原始指针变量上创建 `std::shared_ptr`。