---
title: Item 9
tags: ["现代C++", "using", "typedef"]
categories: [Effective Modern C++]
description: '优先考虑 using 而不是 typedef'
date: 2024-07-19 20:19:21
cover: https://s2.loli.net/2024/07/18/3feQwLTjdDBMKVr.png
---

# 条款9：优先考虑别名声明而⾮typedef

有时候一些很复杂又很多地方能用到的类型或者声明，比如 `std::unique_ptr<std::unordered_map<std::string,std::string>>` 这样的类型，每个地方都要写的话太恐怖了，我们可以引入 `typedef`:

```cpp
typedef std::unique_ptr<std::unordered_map<std::string, std::string>>
UPtrMapSS;
```

但是呢 `typedef` 毕竟是 C++98 的东西了，或多或少有点过时，虽然他可以在 C++11 中工作，但是 C++11 也提供了一个别名声明：

```cpp
using UPtrMapSS = std::unique_ptr<std::unordered_map<std::string, std::string>>;
```

由于这⾥给出的 `typedef` 和别名声明做的都是完全⼀样的事情，我们就有理由想知道会不会出于⼀些技术上的原因所以两者中有⼀个更好？

这⾥就不得不提一下代码的可读性了，很多⼈都发现当声明⼀个函数指针时别名声明更容易理解：

```cpp
// FP是⼀个指向函数的指针的同义词，它指向的函数带有int和const std::string&形参，不返回任何东西
typedef void (*FP)(int, const std::string&); // typedef

//同上
using FP = void (*)(int, const std::string&); // 别名声明
```

当然，两个结构都不是⾮常让⼈满意，没有⼈喜欢花⼤量的时间处理函数指针类型的别名

不过有⼀个地⽅使⽤别名声明吸引⼈的理由是存在的：模板。特别的，别名声明可以被模板化但是 `typedef` 不能。这使得 C++11 程序员可以很直接的表达⼀些 C++98 程序员只能把 `typedef` 嵌套进模板化的 `struct` 才能表达的东西，**考虑⼀个链表的别名，链表使⽤⾃定义的内存分配器**，MyAlloc。

```cpp
template<typename T>
using MyAllocList = std::list<T,MyAlloc<T>>;

MyAllocList<Widget> lw;
```

使用 `typedef` 的话：

```cpp
template<typename T>
struct MyAllocList {
    typedef std::list<T, MyAlloc<T>> type;
};
MyAllocList<Widget>::type lw;
```

更糟糕的是，如果你想使⽤在⼀个模板内使⽤ `typedef` 声明⼀个持有链表的对象，而这个对象⼜使⽤了模板参数，你就不得不在在 `typedef` 前⾯加上 `typename` 消除歧义。

```cpp
template<typename T>
class Widget {
private:
    typename MyAllocList<T>::type list;
    …
};
```

这⾥ `MyAllocList::type` 使⽤了⼀个类型，这个类型依赖于模板参数 `T`。因此 `MyAllocList::type` 是⼀个**依赖类型**，在 C++ 很多讨⼈喜欢的规则中的⼀个提到必须要在依赖类型名前加上 `typename`。如果使⽤别名声明定义⼀个 `MyAllocList`，就不需要使⽤ `typename`。

```cpp
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>; // as before
template<typename T>
class Widget {
private:
    MyAllocList<T> list;
    …
};
```

当编译器处理 `Widget` 模板时遇到 `MyAllocList`（使⽤模板别名声明的版本），它们知道 `MyAllocList` 是⼀个类型名，因为 `MyAllocList` 是⼀个别名模板。它⼀定是⼀个类型名。因此 `MyAllocList` 就是⼀个⾮依赖类型，就不要求必须使⽤ `typename` 。

当编译器在 `Widget` 的模板中看到 `MyAllocList::type` （使⽤ `typedef` 的版本），它不能确定那是⼀个类型的名称。因为可能存在 `MyAllocList` 的⼀个特化版本没有 `MyAllocList::type`。那听起来很不可思议，但不要责备编译器穷尽考虑所有可能。举个例⼦，⼀个误⼊歧途的⼈可能写出这样的代码：

```cpp
class Wine { … };
template<> // 当T是Wine
class MyAllocList<Wine> {   // 特化MyAllocList
private:
    enum class WineType     // 参⻅Item10了解
    { 
        White, 
        Red, 
        Rose 
    };                      // "enum class"
    WineType type;          // 在这个类中，type是
    …                       // ⼀个数据成员！
};
```

就像你看到的，`MyAllocList::type` 不是⼀个类型。如果 `Widget` 使⽤ `Wine` 实例化，在 `Widget` 模板中的 `MyAllocList::type` 将会是⼀个数据成员，不是⼀个类型。在 `Widget `模板内，如果 `MyAllocList::type `表⽰的类型依赖于 `T`，编译器就会坚持要求你在前⾯加上 `typename`。

---

如果你尝试过模板元编程， 你⼀定会碰到**取模板类型参数然后基于它创建另⼀种类型的情况。**

举个例⼦，给⼀个类型 `T`，如果你想去掉 `T` 的常量修饰和引⽤修饰，⽐如你想把 `const std::string&` 变成 `const std::string`。⼜或者你想给⼀个类型加上 `const` 或左值引⽤，⽐如把 `Widget` 变成 `const Widget` 或 `Widget&`。

C++11 在 `type traits` 中给了你⼀系列⼯具去实现类型转换，如果要使⽤这些模板请包含头⽂件 `<type_traits>` 。

```cpp
std::remove_const<T>::type          // 从const T中产出T
std::remove_reference<T>::type      // 从T&和T&&中产出T
std::add_lvalue_reference<T>::type  // 从T中产出T&
```

注意类型转换尾部的 `::type`。如果你在⼀个模板内部使⽤类型参数，你也需要在它们前⾯加上 `typename`。⾄于为什么要这么做是因为这些 `type traits` 是通过在 `struct` 内嵌套 `typedef` 来实现的。

C++14 它们才提供了使⽤别名声明的版本。这些别名声明有⼀个通⽤形式：对于 C++11 的类型转换 `std::transformation::type` 在 C++14 中变成了 `std::transformation_t`。

```cpp
std::remove_const<T>::type // C++11: const T → T
std::remove_const_t<T> // C++14 等价形式

std::remove_reference<T>::type // C++11: T&/T&& → T
std::remove_reference_t<T> // C++14 等价形式

std::add_lvalue_reference<T>::type // C++11: T → T&
std::add_lvalue_reference_t<T> // C++14 等价形式
```

总结：
- `typedef` 不⽀持模板化，但是别名声明⽀持。
- 别名模板避免了使⽤ `::type` 后缀，而且在模板中使⽤ `typedef` 还需要在前⾯加上 `typename`
- C++14 提供了 C++11 所有类型转换的别名声明版本