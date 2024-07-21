---
title: Item 8
tags: ["现代C++", "nullptr"]
categories: [Effective Modern C++]
description: '优先使用nullptr'
date: 2024-07-19 19:25:21
cover: https://s2.loli.net/2024/07/19/QxH9Ms4ml5GWdco.jpg
---

# 条款8：优先考虑nullptr而非0和NULL

很明显的一个问题：字面值 0 是一个 `int` 型的整数，不是一个指针

如果 C++ 发现当前上下文只能使用指针，他才会把 0 解释为指针，但那属于最后的退路，一般来说 C++ 的解析策略是把 0 看作 `int` 而不是一个指针

实际上 `NULL` 也是这样的。但是 `NULL` 的实现细节有些不确定因素，因为实现是**被允许给NULL⼀个除了 `int` 之外的整型类型（⽐如 `long`）**。这不常⻅，但也算不上问题所在。这⾥的问题不是NULL没有⼀个确定的类型，而是 0 和 `NULL` 都不是指针类型。

在 C++98 中，对指针类型和整型进⾏重载意味着可能导致奇怪的事情。如果给下⾯的重载函数传递 0 或 `NULL`，它们绝不会调⽤指针版本的重载函数：

```cpp
void f(int); //三个f的重载函数
void f(bool);
void f(void*);

f(0); //调⽤f(int)而不是f(void*)

f(NULL); //可能不会被编译，⼀般来说调⽤f(int),绝对不会调⽤f(void*)
```

而 `f(NULL)` 的不确定⾏为是由 `NULL` 的实现不同造成的。如果 `NULL` 被定义为 0L （指的是0为long类型），这个调⽤就具有⼆义性，因为从 `long` 到 `int` 的转换或从 `long` 到 `bool `的转换或 0L 到 `void*` 的转换都会被考虑

有趣的是源代码表现出的意思（使⽤ `NULL` 调⽤ `f`）和实际想表达的意思（⽤整型数据调⽤ `f`）是相⽭盾的。这种违反直觉的⾏为导致 C++98 程序员都将避开同时重载指针和整型作为编程准则。在 C++11 中这个编程准则也有效

而 `nullptr` 的有点是它不是整型。其实严格来说也不算一个指针类型，但是可以把它认为是一个通用类型的指针。

`nullptr` 的类型是 `std::nullptr_t` ,带一个完美的循环定义后，`std::nullptr_t `⼜被定义为 `nullptr` 。

`std::nullptr_t` 可以转换为指向任何内置类型的指针，这也是为什么我把它叫做通⽤类型的指针。使⽤ `nullptr` 调⽤f将会调⽤ `void* `版本的重载函数，因为 `nullptr` 不能被视作任何整型：

使⽤nullptr*代替0和NULL可以避开了那些令⼈奇怪的函数重载决议，这不是它的唯⼀优势。它也可以使代码表意明确，尤其是当和auto⼀起使⽤时。举个例⼦，假如你在⼀个代码库中遇到了这样的代码：

```cpp
auto result = findRecord( /* arguments */ );
if (result == 0) {
    …
}
```

如果你不知道 `findRecord` 返回了什么，那么你就不太清楚到底 `result` 是⼀个指针类型还是⼀个整型。毕竟，0 也可以像我们之前讨论的那样被解析。但是换⼀种假设如果你看到这样的代码：

```cpp
auto result = findRecord( /* arguments */ );
if (result == nullptr) {
    …
}
```

这就没有任何歧义：`result` 的结果 **⼀定是指针类型**。当模板出现时 `nullptr` 就更有⽤了。假如你有⼀些函数只能被合适的已锁互斥量调⽤。每个函数都有⼀个不同类型的指针：

```cpp
int f1(std::shared_ptr<Widget> spw); // 只能被合适的已锁互斥量调⽤
double f2(std::unique_ptr<Widget> upw);
bool f3(Widget* pw);
```

如果这样传递空指针：

```cpp
std::mutex f1m, f2m, f3m; // 互斥量f1m，f2m，f3m，各种⽤于f1，f2，f3函数
using MuxGuard = std::lock_guard<std::mutex>;
    …
{
    MuxGuard g(f1m); // 为f1m上锁向f1传递控制空指针解锁
    auto result = f1(0);  
} 
…
{
    MuxGuard g(f2m); // 为f2m上锁向f2传递控制空指针解锁
    auto result = f2(NULL); 
}  
…
{
    MuxGuard g(f3m); // 为f3m上锁向f3传递控制空指针解锁
    auto result = f3(nullptr);  
}  
```

前两个调⽤没有使⽤ `nullptr`，但是代码可以正常运⾏，这也许对⼀些东西有⽤。但是重复的调⽤代码——为互斥量上锁，调⽤函数，解锁互斥量——更令⼈遗憾。它让⼈很烦。**模板就是被设计于减少重复代码**，所以让我们模板化这个调⽤流程：

```cpp
template<typename FuncType, typename MuxType, typename PtrType>
auto lockAndCall(FuncType func, 
                MuxType& mutex, PtrType ptr) -> decltype(func(ptr)) {
    MuxGuard g(mutex);
    return func(ptr);
}
```

可以写这样的代码调⽤lockAndCall模板：

```cpp
auto result1 = lockAndCall(f1, f1m, 0);         // 错误！
…
auto result2 = lockAndCall(f2, f2m, NULL);      // 错误！
…
auto result3 = lockAndCall(f3, f3m, nullptr);    // 没问题`
```

代码虽然可以这样写，但是就像注释中说的，前两个情况不能通过编译。在第⼀个调⽤中存在的问题是当 0 被传递给 `lockAndCall` 模板，**模板类型推导会尝试去推导实参类型，0 的类型总是 `int`，所以 `int` 版本的实例化中的 `func` 会被 `int` 类型的实参调⽤**。这与 `f1` 期待的参数 `std::shared_ptr` 不符。**传递 0 本来想表⽰空指针，结果 `f1` 得到的是和它相差⼗万⼋千⾥的 `int` 。** 把 `int` 类型看做 `std::shared_ptr` 类型⾃然是⼀个类型错误。在模板 `lockAndCall `中使⽤ 0 之所以失败是因为得到的是 `int` 但实际上模板期待的是⼀个 `std::shared_ptr`。

第⼆个使⽤ `NULL` 调⽤的分析也是⼀样的。当 `NULL` 被传递给 `lockAndCall`，形参 `ptr` 被推导为整型，然后当 `ptr` ————⼀个 `int` 或者类似 `int` 的类型—————传递给 `f2` 的时候就会出现类型错误。当 `ptr` 被传递给 `f3` 的时候，隐式转换使 `std::nullptr_t` 转换为 `Widget*`  ，因为 `std::nullptr_t` 可以隐式转换为任何指针类型。

模板类型推导将 0 和 `NULL` 推导为⼀个错误的类型，这就导致它们的替代品 `nullptr` 很吸引⼈。使⽤ `nullptr`，模板不会有什么特殊的转换。另外，使⽤ `nullptr` 不会让你受到同重载决议特殊对待 0 和 `NULL` ⼀样的待遇。当你想⽤⼀个空指针，使⽤ `nullptr`，不⽤ 0 或者 `NULL`。

总结：
- 优先考虑 `nullptr` 而⾮ 0 和 `NULL`
- 避免重载指针和整型