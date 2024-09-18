---
title: Item 24 区分通用引用和右值引用
tags: ["右值引用"]
categories: [Effective Modern C++]
description: '区分通用引用和右值引用'
date: 2024-08-08 15:05:21
cover: https://images.pexels.com/photos/27520824/pexels-photo-27520824.jpeg
---

为了声明一个指向某个类型 `T` 的右值引用，我们会写下 `T&&`。由此一个合理的假设是：当你看到一个 `T&&` 出现在代码中，你看到的是一个右值引用，但往往不是这样的：

```cpp
void f(Widget&& param); //右值引⽤
Widget&& var1 = Widget(); //右值引⽤
auto&& var2 = var1; //不是右值引⽤

template <typename T>
void f(std::vector<T>&& param); //右值引⽤

template <typename T>
void f(T&& param); //不是右值引⽤
```

事实上，`T&&` 有两种不同的意思。第⼀种，当然是右值引⽤。这种引⽤表现得正如你所期待的那样: 它们只绑定到右值上，并且它们主要的存在原因就是为了声明某个对象可以被移动。

`T&&` 的第⼆层意思，是它既可以是⼀个右值引⽤，也可以是⼀个左值引⽤。这种引⽤在源码⾥看起来像右值引⽤(也即 `T&&`),但是它们可以表现得它们像是左值引⽤(也即 `T&`)。**它们的⼆重性使它们既可以绑定到右值上(就像右值引⽤)**，也可以绑定到左值上(就像左值引⽤)。 此外，它们还可以绑定到常量和⾮常量的对象上，也可以绑定到 `volatile` 和 `non-volatile` 的对象上，甚⾄可以绑定到即 `const` ⼜ `volatile` 的对象上。它们可以绑定到⼏乎任何东西。这种空前灵活的引⽤值得拥有⾃⼰的名字。我把它叫做通⽤引⽤。(注: Item 25解释了 `std::forward` ⼏乎总是可以应⽤到通⽤引⽤上，并且在这本书即将出版之际，⼀些 C++ 社区的成员已经开始将这种通⽤引⽤称之为转发引⽤)。

在两种情况下会出现通⽤引⽤。最常⻅的⼀种是函数模板参数，正如在之前的⽰例代码中所出现的例⼦:

```cpp
template <typename T>
void f(T&& param); //param是⼀个通⽤引⽤
```

第⼆种情况是 auto 声明符，包含从以上⽰例中取得的这个例⼦:

```cpp
auto&& var2 = var1; //var2是⼀个通⽤引⽤
```

这两种情况的共同之处就是都存在类型推导。在模板 `f` 的内部，参数 `param` 的类型需要被推导，而在变量 `var2` 的声明中，`var2` 的类型也需要被推导。同以下的例⼦相⽐较(同样来⾃于上⾯的⽰例代码)，下⾯的例⼦不带有类型推导。如果你看⻅ `T&&` 不带有类型推导，那么你看到的就是⼀个右值引⽤。

```cpp
void f(Widget&& param);     //没有类型推导
                            //param是⼀个右值引⽤

Widget&& var1 = Widget();   //没有类型推导
                            //var1是⼀个右值引⽤
```

因为通⽤引⽤是引⽤，所以他们必须被初始化。⼀个通⽤引⽤的初始值决定了它是代表了右值引⽤还是左值引⽤。如果初始值是⼀个右值，那么通⽤引⽤就会是对应的右值引⽤，如果初始值是⼀个左值，那么通⽤引⽤就会是⼀个左值引⽤。对那些是函数参数的通⽤引⽤来说，初始值在调⽤函数的时候被提供：

```cpp
template <typename T>
void f(T&& param); //param是⼀个通⽤引⽤

Widget w;
f(w);               //传递给函数f⼀个左值;参数param的类型
                    //将会是Widget&,也即左值引⽤

f(std::move(w));    //传递给f⼀个右值;参数param的类型会是
                    //Widget&&,即右值引⽤
```

对⼀个通⽤引⽤而⾔，类型推导是必要的,但是它还不够。声明引⽤的格式必须正确，并且这种格式是被限制的。它必须是准确的 `T&&` 。再看看之前我们已经看过的代码⽰例:

```cpp
template <typename T>
void f(std::vector<T>&& param); //param是⼀个右值引⽤
```

当函数 `f` 被调⽤的时候，类型 `T` 会被推导（除⾮调⽤者显式地指定它，这种边缘情况我们不考虑）。但是参数 `param` 的类型声明并不是 `T&&`，而是⼀个 `std::vector<T>&&`。这排除了参数 `param` 是⼀个通⽤引⽤的可能性。`param` 因此是⼀个右值引⽤——当你向函数 `f` 传递⼀个左值时，你的编译器将会开⼼地帮你确认这⼀点:

```cpp
std::vector<int> v;
f(v); //错误！不能将左值绑定到右值引⽤
```

即使是出现⼀个简单的 `const` 修饰符，也⾜以使⼀个引⽤失去成为通⽤引⽤的资格:

```cpp
template <typename T>
void f(const T&& param); //param是⼀个右值引⽤
```

如果你在⼀个模板⾥⾯看⻅了⼀个函数参数类型为 `T&&`, 你也许觉得你可以假定它是⼀个通⽤引⽤。错！这是由于在模板内部并不保证⼀定会发⽣类型推导。考虑如下 `push_back` 成员函数，来⾃ `std::vector`:

```cpp
template <class T,class Allocator = allocator<T>> //来⾃C++标准
class vector
{
public:
    void push_back(T&& x);
    ...
}
```

`push_back` 函数的参数当然有资格成为⼀个通⽤引⽤，然而，在这⾥并没有发⽣类型推导。因为 `push_back` 在⼀个特有的 `vector` 实例化之前不可能存在，而实例化`vector` 时的类型已经决定了 `push_back` 的声明。也就是说，

```cpp
std::vector<Widget> v;
```

将会导致 `std::vector` 模板被实例化为以下代码:

```cpp
class vector<Widget , allocagor<Widget>>
{
public:
    void push_back(Widget&& x); // 右值引⽤
}
```

现在你可以清楚地看到，函数 `push_back` 不包含任何类型推导。`push_back` 对于 `vector<T>` 而⾔(有两个函数——它被重载了)总是声明了⼀个类型为指向 `T` 的右值引⽤的参数。

相反，`std::vector` 内部的概念上相似的成员函数 `emplace_back`，却确实包含类型推导:

```cpp
template <class T,class Allocator = allocator<T>> //依旧来⾃C++标准
class vector
{
public:
    template <class... Args>
    void emplace_back(Args&&... args);
    ...
}
```

这⼉，类型参数 `Args` 是独⽴于 `vector` 的类型参数之外的，所以 `Args` 会在每次 `emplace_back` 被调⽤的时候被推导(Okay, `Args` 实际上是⼀个参数包(parameter pack),而不是⼀个类型参数，但是为了讨论之利，我们可以把它当作是⼀个类型参数)。

虽然函数 `emplace_back` 的类型参数被命名为 `Args`，但是它仍然是⼀个通⽤引⽤，这补充了我之前所说的，通⽤引⽤的格式必须是 `T&&` 。 没有任何规定必须使⽤名字 `T`。举个例⼦，如下模板接受⼀个通⽤引⽤，但是格式(`type&&`)是正确的，并且参数 `param` 的类型将会被推导(重复⼀次，不考虑边缘情况，也
即当调⽤者明确给定参数类型的时候)。

```cpp
template <typename MyTemplateType> //param是通⽤引⽤
void someFunc(MyTemplateType&& param);
```

我之前提到，类型为 `auto` 的变量可以是通⽤引⽤。更准确地说，类型声明为 `auto&&` 的变量是通⽤引⽤，因为会发⽣类型推导，并且它们满⾜正确的格式要求(`T&&`)。`auto` 类型的通⽤引⽤不如模板函数参数中的通⽤引⽤常⻅，但是它们在 C++11 中常常突然出现。而它们在 C++14 中出现地更多，因为 C++14 的匿名函数表达式(lambda expressions)可以声明 `auto&&` 类型的参数。举个例⼦，如果你想写⼀个C++14 标准的匿名函数，来记录任意函数调⽤花费的时间，你可以这样:

```cpp
auto timeFuncInvocation =[](auto&& func, auto&&... params) //C++14标准
{
    start timer;
    std::forward<decltype(func)>(func)( //对参数params调⽤func
    std::forward<delctype(params)>(params)...
    );
    stop timer and record elapsed time;
};
```

如果你对位于匿名函数⾥的 `std::forward<decltype(blah blah blah)>` 反应是 "What the ....!", 这只代表着你可能还没有读 Item 33。别担⼼。在本节，重要的事是匿名函数声明的 `auto&&` 类型的参数。`func` 是⼀个通⽤引⽤，可以被绑定到任何可被调⽤的对象，⽆论左值还是右值。 `args` 是 0 个或者多个通⽤引⽤(也就是说，它是个通⽤引⽤参数包)，它可以绑定到任意数⽬、任意类型的对象上。

多亏了 `auto` 类型的通⽤引⽤，函数 `timeFuncInvocation` 可以对近乎任意函数进⾏计时。

# 总结

- 如果⼀个函数模板参数的类型为 `T&&`，并且 T 需要被推导得知，或者如果⼀个对象被声明为 `auto&&`，这个参数或者对象就是⼀个通⽤引⽤。
- 如果类型声明的形式不是标准的 `type&&`，或者如果类型推导没有发⽣，那么 `type&&` 代表⼀个右值引⽤
- 通⽤引⽤，如果它被右值初始化，就会对应地成为右值引⽤;如果它被左值初始化，就会成为左值引⽤。