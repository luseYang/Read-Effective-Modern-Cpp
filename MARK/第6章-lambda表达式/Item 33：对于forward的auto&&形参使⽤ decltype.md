---
title: Item 33 对于 std::forward 的 auto&& 形参使⽤ decltype
tags: ["lambda"]
categories: [Effective Modern C++]
description: '使⽤初始化捕获来移动对象到闭包中'
date: 2024-08-24 14:32:21
cover: https://images.pexels.com/photos/26653530/pexels-photo-26653530.jpeg
---

泛型 lambda(generic lambdas) 是 C++14 中最值得期待的特性之⼀—，因为在 lambda 的参数中可以使⽤ `auto` 关键字。这个特性的实现是⾮常直截了当的：即在闭包类中的 `operator()` 函数是⼀个函数模版。例
如存在这么⼀个 lambda：

```cpp
auto f = [](auto x){ return func(normalize(x)); };
```

对应的闭包类中的函数调用操作符看来就变成这样：

```cpp
class SomeCompilerGeneratedClassName { public:
template<typename T>
auto operator()(T x) const
{ return func(normalize(x)); }
...
};
```

在这个样例中，lambda 对变量 `x` 做的唯⼀⼀件事就是把它转发给函数 `normalize`。如果函数 `normalize` 对待左值右值的⽅式不⼀样，这个lambda的实现⽅式就不⼤合适了，因为即使传递到 lambda 的实参是⼀个右值，lambda传递进去的形参总是⼀个左值。

实现这个 lambda 的正确⽅式是把 `x` 完美转发给函数 `normalize`。这样做需要对代码做两处修改。⾸先，`x` 需要改成通⽤引⽤，其次，需要使⽤ `std::forward` 将 x 转发到函数 `normalize`。实际上的修改如下：

```cpp
auto f = [](auto&& x)
{ return func(normalize(std::forward<???>(x))); };
```

在理论和实际之间存在⼀个问题：你传递给 `std::forward` 的参数是什么类型，就决定了上⾯的 `???` 该怎么修改。

⼀般来说，当你在使⽤完美转发时，你是在⼀个接受类型参数为 `T` 的模版函数⾥，所以你可以写 `std::forward<T>`。但在泛型 lambda 中，没有可⽤的类型参数 `T`。在 lambda ⽣成的闭包⾥，模版化的 `operator()` 函数中的确有⼀个 `T`，但在 lambda ⾥却⽆法直接使⽤它。

前⾯ item28 解释过在传递给通⽤引⽤的是⼀个左值，那么它会变成左值引⽤。传递的是右值就会变成右值引⽤。这意味着在这个 lambda 中，可以通过检查 `x` 的类型来检查传递进来的实参是⼀个左值还是右值，`decltype` 就可以实现这样的效果。传递给 lambda 的是⼀个左值，`decltype(x)` 就能产⽣⼀个左值引⽤；如果传递的是⼀个右值，`decltype(x)` 就会产⽣右值引⽤。

Item28 也解释过在调⽤ `std::forward`，传递给它的类型类型参数是⼀个左值引⽤时会返回⼀个左值；传递的是⼀个⾮引⽤类型时，返回的是⼀个右值引⽤，而不是常规的⾮引⽤。在前⾯的 lambda 中，如果 `x` 绑定的是⼀个左值引⽤，`decltype(x)` 就能产⽣⼀个左值引⽤；如果绑定的是⼀个右值，`decltype(x)` 就会产⽣右值引⽤，而不是常规的⾮引⽤。

在看⼀下 Item28 中关于 `std::forward` 的 C++14 实现：

```cpp
template<typename T> // in namespace
T&& forward(remove_reference_t<T>& param) // std
{
    return static_cast<T&&>(param);
}
```

如果⽤⼾想要完美转发⼀个 `Widget` 类型的右值时，它会使⽤ `Widget` 类型（⾮引⽤类型）来⽰例化 `std::forward`，然后产⽣以下的函数：

```cpp
Widget&& forward(Widget& param)
{           //    instantiation of
    return static_cast<Widget&&>(param); // std::forward when
}           // T is Widget
```

思考⼀下如果⽤⼾代码想要完美转发⼀个 `Widget` 类型的右值，但没有遵守规则将T指定为⾮引⽤类型，而是将 `T` 指定为右值引⽤，这回发⽣什么？思考将T换成 `Widget` 如何，在 `std::forward` 实例化、应⽤了 `remove_reference_t` 后，⾳乐折叠之前，这是产⽣的代码：

```cpp
Widget&& && forward(Widget& param) // instantiation of
{           // std::forward when
    return static_cast<Widget&& &&>(param); // T is Widget&&
}           // (before reference-collapsing)
```

应用了引用折叠后，代码会变成：

```cpp
Widget&& && forward(Widget& param) // instantiation of
{           // std::forward when
    return static_cast<Widget&& &&>(param); // T is Widget&&
}           // (before reference-collapsing)
```

对⽐发现，⽤⼀个右值引⽤去实例化 `std::forward` 和⽤⾮引⽤类型去实例化产⽣的结果是⼀样的。

那是⼀个很好的消息，引⽤当传递给lambda形参x的是⼀个右值实参时，`decltype(x)` 可以产⽣⼀个右值引⽤。前⾯已经确认过，把⼀个左值传给l ambda 时，`decltype(x)` 会产⽣⼀个可以传给 `std::forward` 的常规类型。而现在也验证了对于右值，把 `decltype(x)` 产⽣的类型传递给 `std::forward` 的类型参数是⾮传统的，不过它产⽣的实例化结果与传统类型相同。所以⽆论是左值还
是右值，把 `decltype(x)` 传递给 `std::forward` 都能得到我们想要的结果，因此 lambda 的完美转发可以写成：

```cpp
auto f =
    [](auto&& param)
    {
        return
        func(normalize(std::forward<decltype(pram)>(param)));
    };
```

再加上6个点，就可以让我们的 lambda 完美转发接受多个参数了，因为 C++14 中的 lambda 参数是可变的：

```cpp
auto f =
    [](auto&&... params)
    {
        return
        func(normalized(std::forward<decltype(params)>(params)...));
    };
```

# 总结

- 对 `auto&&` 参数使⽤ `decltype` 来（`std::forward`）转发参数；