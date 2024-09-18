---
title: Item 23 理解std::move和std::forward
tags: ["std::forward", "std::move"]
categories: [Effective Modern C++]
description: '理解std::move和std::forward'
date: 2024-08-07 13:27:21
cover: https://images.pexels.com/photos/8850666/pexels-photo-8850666.jpeg
---

# 前言

当第一次了解到移动语义和完美转发的时候，他们看起来很直观：
- **移动语义使编译器有可能⽤廉价的移动操作来代替昂贵的复制操作**。正如复制构造函数和复制赋值操作符给了你赋值对象的权利⼀样，移动构造函数和移动赋值操作符也给了控制移动语义的权利。移动语义也允许创建只可移动(move-only)的类型，例如 `std::unique_ptr`, `std::future` 和 `std::thread`。
- 完美转发使接受任意数量参数的函数模版成为可能，它可以将参数转发到其他函数，使目标函数接受到的参数与被传递给转发函数的参数保持一致。

**右值引⽤**是连接这两个截然不同的概念的胶合剂。它隐藏在语⾔机制之下，使移动语义和完美转发变得可能。

但是移动语义、完美转发和右值引⽤的世界⽐它所呈现的更加微妙。举个例⼦,`std::move` 并不移动任何东西，完美转发也并不完美。**移动操作并不永远⽐复制操作更廉价**；即便如此，它也并不总是像你期望的那么廉价。而且，它也并不总是被调⽤，即使在当移动操作可⽤的时候。构造 `type&&` 也并⾮总是代表⼀个右值引⽤。

⽆论你挖掘这些特性有多深，它们看起来总是还有更多隐藏起来的部分。幸运的是，它们的深度总是有限的。本章将会带你到最基础的部分。⼀旦到达，C++11 的这部分特性将会具有⾮常⼤的意义。⽐如，你会掌握 `std::move` 和 `std::forward` 的惯⽤法。你能够对 `type&&` 的歧义性质感到舒服。你会理解移动操作的令⼈惊奇的不同代价的背后真相。这些⽚段都会豁然开朗。在这⼀点上，你会重新回到⼀开始的状态，因为移动语义、完美转发和右值引⽤都会⼜⼀次显得直截了当。但是这⼀次，它们不再使⼈困惑。

在本章的这些小节中，⾮常重要的⼀点是要牢记**参数永远是左值(lValue)**，即使它的类型是⼀个右值引⽤。⽐如，假设

```cpp
void f(widget&& w);
```

参数 w 是一个左值，即使它的类型是一个 `widget` 的右值引用。

# 理解 std::move 和 std::forward

为了了解 `std::move` 和 `std::forward`，⼀种有⽤的⽅式是从它们不做什么这个⻆度来了解它们。`std::move` 不移动(move)任何东西，`std::forward` 也不转发(forward)任何东西。在运⾏期间(runtime)，它们不做任何事情。它们不产⽣任何可执⾏代码，⼀字节也没有。

`std::move` 和 `std::forward` 仅仅是执⾏转换的函数（事实上是函数模板）。`std::move` ⽆条件的将它的参数转换为右值，而 `std::forward` 只在特定情况满⾜时下进⾏转换。

这⾥是⼀个C++11的 std::move 的⽰例实现。它并不完全满⾜标准细则，但是它已经⾮常接近了。

```cpp
template <typename T> //in namespace std
typename remove_reference<T>::type&& move(T&& param)
{
    using ReturnType = // alias declaration;
    typename remove_reference<T>::type&&; // ⻅ Item 9

    return static_cast<ReturnType>(param);
}
```

`std::move` 接受⼀个对象的引⽤(准确的说，⼀个通⽤引⽤，后⻅Item 24)，返回⼀个指向同对象的引⽤。

该函数返回类型的 `&&` 部分表明 `std::move` 函数返回的是⼀个右值引⽤，但是，正如 Item 28 所解释的那样，如果类型 `T` 恰好是⼀个左值引⽤，那么 `T&&` 将会成为⼀个左值引⽤。为了避免如此，类型萃取器(type trait，⻅Item 9) `std::remove_reference` 应⽤到了类型 `T` 上，因此确保了 `&&` 被正确的应⽤到了⼀个不是引⽤的类型上。这保证了 `std::move` 返回的真的是右值引⽤，这很重要，因为函数返回的右值引⽤是右值(rvalues)。因此，`std::move` 将它的参数转换为⼀个右值，这就是它的全部作⽤。

此外，`std::move` 在C++14中可以被更简单地实现。多亏了函数返回值类型推导(⻅Item 3)和标准库的模板别名 `std::remove_reference_t` (⻅Item 9)，`std::move` 可以这样写：

```cpp
template <typename T>
decltype(auto) move(T&& param) //C++14;still in namesapce std
{
using ReturnType = remove_referece_t<T>&&;
return static_cast<ReturnType>(param);
}
```

因为 `std::move` 除了转换它的参数到右值以外什么也不做，有⼀些提议说它的名字叫 `rvalue_cast` 可能会更好。虽然可能确实是这样，但是它的名字已经是 `std::move`，所以记住 `std::move` 做什么和不做什么很重要。它其实并不移动任何东西。

事实上，右值只不过经常是移动操作的候选者。假设你有⼀个类，它⽤来表⽰⼀段注解。这个类的构造函数接受⼀个包含有注解的 `std::string` 作为参数，然后它复制该参数到类的数据成员。假设你了解Item 41,你声明⼀个值传递的参数:

```cpp
class Annotation {
public:
    explicit Annotation(std::string text); //将会被复制的参数
    ... //如同 Item 41,
};
```

但是 `Annotation` 类的构造函数仅仅是需要读取参数 `text` 的值，它并不需要修改它。为了和历史悠久的传统：能使⽤ `const` 就使⽤ `const` 保持⼀致，你修订了你的声明以使 `text` 变成 `const`，

```cpp
class Annotation {
public:
    explicit Annotation(const std::string text);
    ...
};
```

当复制参数 `text` 到⼀个数据成员的时候，为了避免⼀次复制操作的代价，你仍然记得来⾃ Item 41 的建议，把 `std::move` 应⽤到参数 `text` 上，因此产⽣⼀个右值，

```cpp
class Annotation {
public:
    explicit Annotation(const std::string text)
    ：value(std::move(text)) //"move" text到value上；这段代码执⾏起来
    //并不如看起来那样
    {...}
    ...
private:
    std::string value;
};
```

这段代码可以编译，可以链接，可以运⾏。这段代码将数据成员 `value` 设置为 `text` 的值。这段代码与你期望中的完美实现的唯⼀区别，是 `text` 并不是被移动到 `value` ，而是被复制。诚然， `text` 通过 `std::move` 被转换到右值，但是 `text` 被声明为 `const std::string` ，所以在转换之前，`text` 是⼀个左值的` const std::string`，而转换的结果是⼀个右值的 `const std::string`，但是纵观全程，`const` 属性⼀直保留。

当编译器决定哪⼀个 `std::string` 的构造函数被构造时，考虑它的作⽤，将会有两种可能性。

```cpp
class string { //std::string事实上是
public: //std::basic_string<char>的类型别名
    ...
    string(const string& rhs); //复制构造函数
    string(string&& rhs); //移动构造函数
}
```

在类 `Annotation` 的构造函数的成员初始化列表中，`std::move(text)` 的结构是⼀个 `const std::string` 的右值。这个右值不能被传递给 `std::string` 的移动构造函数，因为移动构造函数只接受⼀个指向⾮常量`std::string` 的右值引⽤。然而，该右值却可以被传递给 `std::string` 的复制构造函数，因为指向常量的左值引⽤允许被绑定到⼀个常量右值上。因此，`std::string` 在成员初始化的过程中调⽤了复制构造函数，即使 `text` 已经被转换成了右值。这样是为了确保维持常量属性的正确性。从⼀个对象中移动出某个值通常代表着修改该对象，所以语⾔不允许常量对象被传递给可以修改他们的函数（例如移动构造函数）。

从这个例⼦中，可以总结出两点。第⼀，不要在你希望能移动对象的时候，声明他们为常量。对常量对象的移动请求会悄⽆声息的被转化为复制操作。第⼆点，`std::move` 不仅不移动任何东西，而且它也不保证它执⾏转换的对象可以被移动。关于 `std::move`，你能确保的唯⼀⼀件事就是将它应⽤到⼀个对象上，你能够得到⼀个右值。关于 `std::forward` 的故事与 `std::move` 是相似的，但是与 `std::move` 总是⽆条件的将它的参数转换为右值不同，`std::forward` 只有在满⾜⼀定条件的情况下才执⾏转换。`std::forward` 是有条件的转换。要明⽩什么时候它执⾏转换，什么时候不，想想 `std::forward` 的典型⽤法。最常⻅的情景是⼀个模板函数，接收⼀个通⽤引⽤参数，并将它传递给另外的函数：

```cpp
void process(const Widget& lvalArg); //左值处理
void process(Widget&& rvalArg); //右值处理

template <typename T> //⽤以转发参数到process的模板
void logAndProcess(T&& param)
{
    auto now = //获取现在时间
    std::chrono::system_clock::now();
    makeLogEntry("calling 'process'",now);
    process(std::forward<T>(param));
}
```

考虑两次对 `logAndProcess` 的调⽤，⼀次左值为参数，⼀次右值为参数，

```cpp
widget w;
logAndProcess(w);               //call with lvalue
logAndProcess(std::move(w));    //call with rvalue
```

在 `logAndProcess` 函数的内部，参数 `param` 被传递给函数 `process` 。函数 `process` 分别对左值和右值参数做了重载。当我们使⽤左值来调⽤ `logAndProcess` 时，⾃然我们期望该左值被当作左值转发给 `process` 函数，而当我们使⽤右值来调⽤ `logAndProcess` 函数时，我们期望 `process` 函数的右值重载版本被调⽤。

但是参数 `param` ，正如所有的其他函数参数⼀样，是⼀个左值。每次在函数 `logAndProcess` 内部对函数 `process` 的调⽤，都会因此调⽤函数 `process` 的左值重载版本。为防如此，我们需要⼀种机制: 当且仅当传递给函数 `logAndProcess` 的⽤以初始化参数 `param` 的值是⼀个右值时，参数 `param` 会被转换为有⼀个右值。这就是为什么 `std::forward` 是⼀个有条件的转换：它只把由右值初始化的参数，转换为右值。

你也许会想知道 `std::forward` 是怎么知道它的参数是否是被⼀个右值初始化的。举个例⼦，在上述代码中， `std::forward` 是怎么分辨参数 `param` 是被⼀个左值还是右值初始化的？ 简短的说，该信息藏在函数 `logAndProcess` 的模板参数 T 中。该参数被传递给了函数 `std::forward `，它解开了含在其中的信息。该机制⼯作的细节可以查询 Item 28.

考虑到 `std::move` 和 `std::forward` 都可以归结于转换，他们唯⼀的区别就是 `std::move` 总是执⾏转换，而 `std::forward` 偶尔为之。你可能会问是否我们可以免于使⽤ `std::move` 而在任何地⽅只使⽤ `std::forward`。 从纯技术的⻆度，答案是yes: `std::forward` 是可以完全胜任， std::move 并⾮必须。当然，其实两者中没有哪⼀个函数是真的必须的，因为我们可以到处直接写转换代码，但是我希望我们能同意：这将相当的，嗯，让⼈恶⼼。

`std::move` 的吸引⼒在于它的便利性： 减少了出错的可能性，增加了代码的清晰程度。考虑⼀个类，我们希望统计有多少次移动构造函数被调⽤了。我们只需要⼀个静态的计数器 `static counter`，它会在移动构造的时候⾃增。假设在这个类中，唯⼀⼀个⾮静态的数据成员是 std::string ，⼀种经典的移动构造函数(例如，使⽤ `std::move`)可以被实现如下:

```cpp
class Widget{
public:
    Widget(Widget&& rhs)
    : s(std::move(rhs.s))
    {
        ++moveCtorCalls;
    }
private:
    static std::size_t moveCtorCalls;
    std::string s;
}
```

如果要⽤ `std::forward` 来达成同样的效果，代码可能会看起来像

```cpp
class Widget{
public:
    Widget(Widget&& rhs) //不⾃然，不合理的实现
    : s(std::forward<std::string>(rhs.s))
    {
        ++moveCtorCalls;
    }
    ...
}
```

注意，第⼀，`std::move` 只需要⼀个函数参数(`rhs.s`)，而 `std::forward `不但需要⼀个函数参数，还需要⼀个模板类型参数 `std::string`。其次，我们转发给 `std::forward` 的参数类型应当是⼀个⾮引⽤，因为传递的参数应该是⼀个右值(⻅ Item 28)。 同样，这意味着`std::move` ⽐起 `std::forward` 来说需要打更少的字，并且免去了传递⼀个表⽰我们正在传递⼀个右值的类型参数。同样，它根绝了我们传递错误类型的可能性，（例如，`std::string&` 可能导致数据成员s 被复制而不是被移动构造）。

更重要的是，`std::move` 的使⽤代表着⽆条件向右值的转换，而使⽤ `std::forward` 只对绑定了右值的引⽤进⾏到右值转换。这是两种完全不同的动作。前者是典型地为了移动操作，而后者只是传递（亦作转发）⼀个对象到另外⼀个函数，保留它原有的左值属性或右值属性。因为这些动作实在是差异太⼤，所以我们拥有两个不同的函数（以及函数名）来区分这些动作。

# 总结
- `std::move` 执⾏到右值的⽆条件的转换，但就 ⾃⾝而⾔，它不移动任何东西。
- `std::forward` 只有当它的参数被绑定到⼀个右值时，才将参数转换为右值。
- `std::move` 和 `std::forward` 在运⾏期什么也不做。