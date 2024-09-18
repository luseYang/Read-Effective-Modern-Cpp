---
title: Item 32 使⽤初始化捕获来移动对象到闭包中
tags: ["lambda"]
categories: [Effective Modern C++]
description: '使⽤初始化捕获来移动对象到闭包中'
date: 2024-08-24 14:32:21
cover: https://images.pexels.com/photos/26599586/pexels-photo-26599586.jpeg
---

# 前言

在某些场景下，按值捕获和按引用捕获都不是你所想要的。如果有一个只能被移动的对象（例如：`std::unique_ptr` 或 `std::future`）要进入到闭包里，使用 C++11 是无法实现的。如果你要复制的对象复制开销非常高，但移动的成本却不高（例如标准库中的⼤多数容器），并且你希望的是宁愿移动该对象到闭包而不是复制它。然而 C++11 却无法实现这一目标。

但如果你的编译器⽀持 C++14，那⼜是另⼀回事了，它能⽀持将对象移动道闭包中。如果你的兼容⽀持 C++14，那么请愉快地阅读下去。如果你仍然在使⽤仅⽀持 C++11 的编译器，也请愉快阅读，因为在 C++11 中有很多⽅法可以实现近似的移动捕获。

**缺少移动捕获被认为是 C++11 的⼀个缺点**，直接的补救措施是将该特性添加到 C++14 中，但标准化委员会选择了另⼀种⽅法。他们引⼊了⼀种新的捕获机制，该机制⾮常灵活，移动捕获是它执⾏的技术之⼀。新功能被称作初始化捕获，它⼏乎可以完成 C++11 捕获形式的所有⼯作，甚⾄能完成更多功能。默认的捕获模式使得你⽆法使⽤初始化捕获表⽰，但第31项说明提醒了你⽆论如何都应该远离这些捕获模式。（在 C++11 捕获模式所能覆盖的场景⾥，初始化捕获的语法有点不⼤⽅便。因此在 C++11 的捕获模式能完成所需功能的情况下，使⽤它是完全合理的）。

# 初始化捕获

使⽤初始化捕获可以让你指定：

1. 从 lambda ⽣成的闭包类中的数据成员名称；
2. 初始化该成员的表达式；

这是使⽤初始化捕获将 `std::unique_ptr` 移动到闭包中的⽅法：

```cpp
class Widget { // some useful type
public:
    ...
    bool isValidated() const;
    bool isProcessed() const;
    bool isArchived() const;
    private: ...
};

auto pw = std::make_unique<Widget>(); // create Widget; see Item 21 for info on
std::make_unique configure *pw

auto func = [pw = std::move(pw)] 
{ return pw->isValidated() && pw->isArchived(); };
```

上⾯的⽂本包含了初始化捕获的使⽤，`"="` 的左侧是指定的闭包类中数据成员的名称，右侧则是初始化表达式。有趣的是，`"="` 左侧的作⽤范围不同于右侧的作⽤范围。在上⾯的⽰例中，`'='` 左侧的名称 `pw` 表⽰闭包类中的数据成员，而右侧的名称 `pw` 表⽰在lambda上⽅声明的对象，即由调⽤初始化的变量到调⽤ `std::make_unique`。因此，`pw = std :: move(pw)` 的意思是“在闭包中创建⼀个数据成员 `pw`，并通过将 `std::move` 应⽤于局部变量 `pw` 的⽅法来初始化该数据成员。

⼀般中，lambda 主体中的代码在闭包类的作⽤范围内，因此 `pw` 的使⽤指的是闭包类的数据成员。

在此⽰例中，注释 `configure * pw` 表⽰在由 `std::make_unique` 创建窗口小部件之后，再由lambda捕获到该窗口小部件的 `std::unique_ptr` 之前，该窗口小部件即 `pw` 对象以某种⽅式进⾏了修改。如果不需要这样的配置，即如果 `std::make_unique` 创建的 `Widget` 处于适合被 lambda 捕获的状态，则不需要局部变量 `pw`，因为闭包类的数据成员可以通过直接初始化 `std::make_unique` 来实现：

```cpp
auto func = [pw = std::make_unique<Widget>()] 
{ return pw->isValidated() // in closure w/
&& pw->isArchived(); }; // result of call // to make_unique
```

这清楚地表明了，这个 C++14 的捕获概念是从 C++11 发展出来的的，在 C++11 中，⽆法捕获表达式的结果。 因此，初始化捕获的另⼀个名称是⼴义 lambda 捕获。

但是，如果您使⽤的⼀个或多个编译器不⽀持 C++14 的初始捕获怎么办？ 如何使⽤不⽀持移动捕获的语⾔完成移动捕获？

请记住，lambda 表达式只是⽣成类和创建该类型对象的⼀种⽅式而已。如果对于 lambda，你觉得⽆能为⼒。 那么我们刚刚看到的 C++14 的⽰例代码可以⽤ C++11 重新编写，如下所⽰：

```cpp
class IsValAndArch {
public:
    using DataType = std::unique_ptr<Widget>; // "is validated and archived"
    explicit IsValAndArch(DataType&& ptr) // Item 25 explains
    : pw(std::move(ptr)) {} // use of std::move
    bool operator()() const
    { return pw->isValidated() && pw->isArchived(); }
private:
    DataType pw;
};

auto func = IsValAndArch(std::make_unique<Widget>());
```

这个代码量⽐ lambda 表达式要多，但这并不难改变这样⼀个事实，即如果你希望使⽤⼀个 C++11 的类来⽀持其数据成员的移动初始化，那么你唯⼀要做的就是在键盘上多花点时间。

如果你坚持要使⽤ lambda（并且考虑到它们的便利性，你可能会这样做），可以在 C++11 中这样使⽤

1. 将要捕获的对象移动到由 `std::bind`；
2. 将被捕获的对象赋予⼀个引⽤给 lambda；

如果你熟悉 `std::bind`，那么代码其实⾮常简单。如果你不熟悉std::bind，那可能需要花费⼀些时间来习惯改代码，但这⽆疑是值得的。

假设你要创建⼀个本地的 `std::vector`，在其中放⼊⼀组适当的值，然后将其移动到闭包中。在 C++14 中，这很容易实现：

```cpp
std::vector<double> data; // object to be moved
// into closure
// populate data
auto func = [data = std::move(data)] { /* uses of data */ }; // C++14 init capture
```

我已经对该代码的关键部分进⾏了⾼亮：要移动的对象的类型（`std::vector\<double>`），该对象的名称（数据）以及⽤于初始化捕获的初始化表达式（ `std::move(data)`）。C++11 的等效代码如下，其中我强调了相同的关键事项：

```cpp
std::vector<double> data; // as above
auto func =
std::bind(
// C++11 emulation
[](const std::vector<double>& data) { /* uses of data */ }, // of init
capture
std::move(data)
);
```

如 lambda 表达式⼀样，`std::bind` ⽣产了函数对象。我将它称呼为由 `std::bind` 所绑定对象返回的函数对象。`std::bind` 的第⼀个参数是可调⽤对象，后续参数表⽰要传递给该对象的值。⼀个绑定的对象包含了传递给 `std::bind` 的所有参数副本。对于每个左值参数，绑定对象中的对应对象都是复制构造的。对于每个右值，它都是移动构造的。在此⽰例中，第⼆个参数是⼀个右值（`std::move` 的结果，请参⻅第23项），因此将数据移动到绑定对象中。这种移动构造是模仿移动捕获的关键，因为将右值移动到绑定对象是我们解决⽆法将右值移动到 C++11 闭包中的⽅法。

当“调⽤”绑定对象（即调⽤其函数调⽤运算符）时，其存储的参数将传递到最初传递给 `std::bind` 的可调⽤对象。在此⽰例中，这意味着当调⽤ `func`（绑定对象）时，`func` 中所移动构造的数据副本将作为参数传递给传递给 `std::bind` 中的 lambda。

该 lambda 与我们在 C++14 中使⽤的 lambda 相同，只是添加了⼀个参数data来对应我们的伪移动捕获对象。此参数是对绑定对象中数据副本的左值引⽤。（这不是右值引⽤，因尽管⽤于初始化数据副本的表达式（ `std::move(data)·）为右值，但数据副本本⾝为左值。）因此，lambda将对绑定在对象内部的移动构造数据副本进⾏操作。

默认情况下，从 lambda ⽣成的闭包类中的 `operator()` 成员函数为 `const` 的。这具有在 lambda 主体内呈现闭包中的所有数据成员为 `const` 的效果。但是，绑定对象内部的移动构造数据副本不⼀定是 `const` 的，因此，为了防⽌在 lambda 内修改该数据副本，lambda 的参数应声明为 `const` 引⽤。 如果将 lambda 声明为可变的，则不会在其闭包类中将 `operator()` 声明为 `const`，并且在 lambda 的参数声明中省略 `const` 也是合适的：

```cpp
auto func =
    std::bind( // C++11 emulation
    [](std::vector<double>& data) mutable // of init capture
    { /* uses of data */ }, // for
mutable lambda std::move(data)
);
```

因为该绑定对象存储着传递给 `std::bind` 的所有参数副本，所以在我们的⽰例中，绑定对象包含由 lambda ⽣成的闭包副本，这是它的第⼀个参数。 因此闭包的⽣命周期与绑定对象的⽣命周期相同。 这很重要，因为这意味着只要存在闭包，包含伪移动捕获对象的绑定对象也将存在。

如果这是您第⼀次接触 `std::bind`，则可能需要先阅读您最喜欢的 C++11 参考资料，然后再进⾏讨论所有详细信息。 即使是这样，这些基本要点也应该清楚：

- ⽆法将移动构造⼀个对象到 C++11 闭包，但是可以将对象移动构造为C++11的绑定对象。
- 在 C++11 中模拟移动捕获包括将对象移动构造为绑定对象，然后通过引⽤将对象移动构造传递给 lambda。
- 由于绑定对象的⽣命周期与闭包对象的⽣命周期相同，因此可以将绑定对象中的对象视为闭包中的对象。

作为使⽤ `std::bind` 模仿移动捕获的第⼆个⽰例，这是我们之前看到的在闭包中创建 `std::unique_ptr` 的 C++14 代码：

```cpp
auto func = [pw = std::make_unique<Widget>()] // as before,
{ return pw->isValidated() // create pw
&& pw->isArchived(); }; // inclosure
```

这是C++11的模拟实现：

```cpp
auto func = std::bind(
    [](const std::unique_ptr<Widget>& pw)
    { return pw->isValidated()
    && pw->isArchived(); },
    std::make_unique<Widget>()
);
```

具备讽刺意味的是，这⾥我展⽰了如何使⽤ `std::bind` 解决C++11 lambda中的限制，但在条款34中，我却主张在 `std::bind` 上使⽤ lambda。

但是，该条⽬解释的是在 C++11 中有些情况下 `std::bind` 可能有⽤，这就是其中⼀种。 （在 C++14 中，初始化捕获和⾃动参数等功能使得这些情况不再存在。）

# 总结

- 使⽤ C++14 的初始化捕获将对象移动到闭包中。
- 在 C++11 中，通过⼿写类或 `std::bind` 的⽅式来模拟初始化捕获。

