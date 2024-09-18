---
title: Item 28 理解引用折叠
tags: ["引用折叠"]
categories: [Effective Modern C++]
description: '理解引用折叠'
date: 2024-08-13 13:54:21
cover: https://images.pexels.com/photos/27350220/pexels-photo-27350220.jpeg
---

Item23 中指出，当参数传递给模板函数时，模板参数的类型是左值还是右值被推导出来。但是并没有提到只有当参数被声明为通⽤引⽤时，上述推导才会发⽣，但是有充分的理由忽略这⼀点：因为通⽤引⽤是 Item24 中才提到。回过头来看，通⽤引⽤和左值/右值编码意味着：

```cpp
template<typename T>
void func(T&& param);
```

被推导的模板参数T将根据被传⼊参数类型被编码为左值或者右值。

编码机制是简单的。当左值被传⼊时，T被推导为左值。当右值被传⼊时，T被推导为⾮引⽤（请注意不对称性：左值被编码为左值引⽤，右值被编码为⾮引⽤），因此：

```cpp
Widget widgetFactory(); // function returning rvalue
Widget w; // a variable(an lvalue)
func(w); // call func with lvalue; T deduced to be Widget&
func(widgetFactory()); // call func with rvalue; T deduced to be Widget
```

上⾯的两种调⽤中，`Widge` t被传⼊，因为⼀个是左值，⼀个是右值，模板参数T被推导为不同的类型。正如我们很快看到的，这决定了通⽤引⽤成为左值还是右值，也是 `std::forward` 的⼯作基础。

在我们更加深⼊ `std::forward` 和通⽤引⽤之前，必须明确在 C++ 中引⽤的引⽤是⾮法的。不知道你是否尝试过下⾯的写法，编译器会报错：

```cpp
int x;
...
auto& & rx = x; //error! can't declare reference to reference
```

考虑下，如果⼀个左值传给模板函数的通⽤引⽤会发⽣什么：

```cpp
template<typename T>
void func(T&& param);

func(w); // invoke func with lvalue; T deduced as Widget&
```

如果我们把推导出来的类型带⼊回代码中看起来就像是这样：

```cpp
void func(Widget& && param);
```

引⽤的引⽤！但是编译器没有报错。我们从 Item24 中了解到因为通⽤引⽤ `param` 被传⼊⼀个左值，所以 `param` 的类型被推导为左值引⽤，但是编译器如何采⽤T的推导类型的结果，这是最终的函数签名？

```cpp
void func(widget& param);
```

答案是引⽤折叠。是的，禁⽌你声明引⽤的引⽤，但是编译器会在特定的上下⽂中使⽤，包括模板实例的例⼦。当编译器⽣成引⽤的引⽤时，引⽤折叠指导下⼀步发⽣什么。

存在两种类型的引⽤（左值和右值），所以有四种可能的引⽤组合（左值的左值，左值的右值，右值的右值，右值的左值）。如果⼀个上下⽂中允许引⽤的引⽤存在（⽐如，模板函数的实例化），引⽤根据规则折叠为单个引⽤：

> 如果任⼀引⽤为左值引⽤，则结果为左值引⽤。否则（即，如果引⽤都是右值引⽤），结果为右值引⽤

在我们上⾯的例⼦中，将推导类型 `Widget&` 替换模板 `func` 会产⽣对左值引⽤的右值引⽤，然后引⽤折叠规则告诉我们结果就是左值引⽤。

引⽤折叠是 `std::forward` ⼯作的⼀种关键机制。就像 Item25 中解释的⼀样，`std::forward` 应⽤在通⽤引⽤参数上，所以经常能看到这样使⽤：

```cpp
template<typename T>
void f(T&& fParam)
{
    ... // do some work
    someFunc(std::forward<T>(fParam)); // forward fParam to someFunc
}
```

因为 `fParam` 是通⽤引⽤，我们知道参数T的类型将在传⼊具体参数时被编码。`std::forward` 的作⽤是当传⼊参数为右值时，即T为⾮引⽤类型，才将 `fParam` （左值）转化为⼀个右值。

`std::forward` 可以这样实现：

```cpp
template<typename T>
T&& forward(typename remove_reference<T>::type& param)
{
    return static_cast<T&&>(param);
}
```

这不是标准库版本的实现（忽略了⼀些接口描述），但是为了理解 `std::forward` 的⾏为，这些差异⽆
关紧要。

假设传⼊到 `f` 的 `Widget` 的左值类型。`T` 被推导为 `Widget&`，然后调⽤ `std::forward` 将初始化为 `std::forward<Widget&>`。带⼊到上⾯的 `std::forward` 的实现中：

```cpp
Widget& && forward(typename remove_reference<Widget&>::type& param)
{
    return static_cast<Widget& &&>(param);
}
``` 

`std::remove_reference<Widget&>::type` 表⽰ `Widget`（查看Item9），所以 `std::forward` 成为：

```cpp
Widget& && forward(Widget& param)
{
    return static_cast<Widget& &&>(param);
}
```

根据引⽤折叠规则，返回值和 `static_cast` 可以化简，最终版本的 `std::forward` 就是

```cpp
Widget& forward(Widget& param)
{
    return static_cast<Widget&>(param);
}
```

正如你所看到的，当左值被传⼊到函数模板f时，`std::forward` 转发和返回的都是左值引⽤。内部的转换不做任何事，因为 `param` 的类型已经是 `Widget&`，所以转换没有影响。左值传⼊会返回左值引⽤。通过定义，左值引⽤就是左值，因此将左值传递给 `std::forward` 会返回左值，就像说的那样，完美转发。

现在假设⼀下，传递给f的是⼀个 `Widget` 的右值。在这个例⼦中，`T` 的类型推导就是 `Widget`。内部的 `std::forward` 因此转发 `std::forward<Widget>`，带⼊回 `std::forward` 实现中：

```cpp
Widget&& forward(typename remove_reference<Widget>::type& param)
{
    return static_cast<Widget&&>(param);
}
```

将 `remove_reference` 引⽤到⾮引⽤的类型上还是相同的类型，所以化简如下

```cpp
Widget&& forward(Widget& param)
{
    return static_cast<Widget&&>(param);
}
```

这⾥没有引⽤的引⽤，所以不需要引⽤折叠，这就是最终版本。

从函数返回的右值引⽤被定义为右值，因此在这种情况下，`std::forward` 会将 `f` 的参数 `fParam`（左值）转换为右值。最终结果是，传递给 `f` 的右值参数将作为右值转发给 `someFunc`，完美转发。

在 C++14 中，`std::remove_reference_t` 的存在使得实现变得更简单：

```cpp
template<typename T> // C++ 14; still in namepsace std
T&& forward(remove_reference_t<T>& param)
{
    return static_cast<T&&>(param);
}
```

引⽤折叠发⽣在四种情况下。第⼀，也是最常⻅的就是模板实例化。第⼆，是 `auto` 变量的类型⽣成，具体细节类似模板实例化的分析，因为类型推导基本与模板实例化雷同（参⻅Item2）。考虑下⾯的例⼦：

```cpp
template<typename T>
void func(T&& param);
Widget widgetFactory(); // function returning rvalue
Widget w; // a variable(an lvalue)
func(w); // call func with lvalue; T deduced to be Widget&
func(widgetFactory()); // call func with rvalue; T deduced to be Widget
```

在 `auto` 的写法中，规则是类似的：`auto&& w1 = w;` 初始化 `w1` 为⼀个左值，因此为 `auto` 推导出类型 `Widget&`。带回去就是 `Widget& && w1 = w`，应⽤引⽤折叠规则，就是 `Widget& w1 = w`,结果就是 `w1` 是⼀个左值引⽤。

另⼀⽅⾯，`auto&& w2 = widgetFactory();` 使⽤右值初始化 `w2`，⾮引⽤带回 `Widget&& w2 = widgetFactory()`。没有引⽤的引⽤，这就是最终结果。

现在我们真正理解了 Item24 中引⼊的通⽤引⽤。通⽤引⽤不是⼀种新的引⽤，它实际上是满⾜两个条件下的右值引⽤：

- **通过类型推导将左值和右值区分**。`T` 类型的左值被推导为 `&` 类型，`T`类型的右值被推导为 `T`
- **引⽤折叠的发⽣**

通⽤引⽤的概念是有⽤的，因为它使你不必⼀定意识到引⽤折叠的存在，从直觉上判断左值和右值的推导即可。

我说了有四种情况会发⽣引⽤折叠，但是只讨论了两种：模板实例化和auto的类型⽣成。第三，是使⽤ `typedef` 和别名声明（参⻅Item9），如果，在创建或者定义 `typedef` 过程中出现了引⽤的引⽤，则引⽤折叠就会起作

```cpp
template<typename T>
class Widget {
public:
    typedef T&& RvalueRefToT;
    ...
};
```

假设我们使⽤左值引⽤实例化 `Widget`：

```cpp
widget<int&> w;
```

就会出现

```cpp
typedef int& && RvalueRefToT;
```

引⽤折叠就会发挥作⽤：

```cpp
typedef int& RvalueRefToT;
```

这清楚表明我们为 `typedef` 选择的 `name` 可能不是我们希望的那样：`RvalueRefToT` 是左值引⽤的 `typedef`，当使⽤ `Widget` 被左值引⽤实例化时。

最后，也是第四种情况是，`decltype` 使⽤的情况，如果在分析 `decltype` 期间，出现了引⽤的引⽤，引⽤折叠规则就会起作⽤（关于 `decltype`，参⻅Item3）

# 总结
- 引⽤折叠发⽣在四种情况：模板实例化；`auto`类型推导；`typedef`的创建和别名声明；`decltype`
- 当编译器⽣成了引⽤的引⽤时，结果通过引⽤折叠就是单个引⽤。有左值引⽤就是左值引⽤，否则就是右值引⽤
- 通⽤引⽤就是通过类型推导区分左值还是右值，并且引⽤折叠出现的右值引⽤