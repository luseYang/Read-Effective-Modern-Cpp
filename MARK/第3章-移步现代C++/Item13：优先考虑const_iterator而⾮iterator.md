---
title: Item 13 优先考虑const_iterator而⾮iterator
tags: ["现代C++", "iterator"]
categories: [Effective Modern C++]
description: '优先考虑const_iterator而⾮iterator'
date: 2024-07-22 14:32:21
cover: https://p5.img.cctvpic.com/photoworkspace/contentimg/2023/03/30/2023033011303020756.jpg
---

# 条款13：优先考虑 const_iterator 而⾮ iterator

"STL `const_iterator` 等价于指向常量的指针。它们都指向不能被修改的值。**标准实践是能加上 `const` 就加上**，这也指⽰我们对待 `const_iterator` 应该如出⼀辙。"

上面的说法对于 C++98 和 C++11 都是正确的，但是在 C++98 中，标准库对于 `const_iterator` 的支持不是很完整。

> 首先不容易创建它们，其次就算有了它，它的使用也是受限的。

假如你想在 `std::vector<int>` 中查找第⼀次出现 1983 (这是 C++ 代替 C with classes 的那⼀年)的位置，然后插⼊1998(这是第⼀个 ISO C++ 标准被接纳的那⼀年)。如果 `vector` 中没有 1983，那么就在 `vector` 尾部插⼊。在 C++98 中使⽤ `iterator` 可以很容易做到:

```cpp
std::vector<int> values;
...
std::vector<int>::iterator it = std::find(values.begin(), values.end(), 1983);
values.insert(it, 1998);
```

但是这⾥ `iterator` 真的不是⼀个好的选择，因为这段代码**不修改 `iterator` 指向的内容**。⽤ `const_iterator` 重写这段代码是很平常的，但是在 C++98 中就不是了。

```cpp
typedef std::vector<int>::iterator IterT; // typetypedef
std::vector<int>::const_iterator ConstIterT; // defs
std::vector<int> values;
…
ConstIterT ci = std::find(static_cast<ConstIterT>(values.begin()), static_cast<ConstIterT>(values.end()), 1983);
values.insert(static_cast<IterT>(ci), 1998); // 可能⽆法通过编译，原因⻅下
```

`typedef` 不是强制的，但是可以让类型转换更好写。（你可能想知道为什么我使⽤ `typedef` 而不是 Item 9 提到的别名声明，因为这段代码在演⽰ C++98 做法，别名声明是 C++11 加⼊的特性）

之所以 `std::find` 的调⽤会出现类型转换是因为在 C++98 中 `values` 是⾮常量容器，没办法简简单单的从⾮常量容器中获取 `const_iterator` 。严格来说类型转换不是必须的，因为⽤其他⽅法获取 `const_iterator` 也是可以的

当你费劲地获得了 `const_iterator`，事情可能会变得更糟，因为 C++98 中，插⼊操作的位置只能由 `iterator` 指定，`const_iterator` 是不被接受的。这也是我在上⾯的代码中， 将 `const_iterator` 转换为 `iterat` 的原因，因为向 `insert` 传⼊ `const_iterator` 不能通过编译。

⽼实说，上⾯的代码也可能⽆法编译，因为**没有⼀个可移植的从 `const_iterator` 到 `iterator` 的⽅法**，即使使⽤ `static_cast` 也不⾏。甚⾄传说中的⽜⼑ `reinterpret_cast` 也杀不了这条鸡。（它 C++98 的限制，也不是 C++11 的限制，只是 `const_iterator` 就是不能转换为 `iterator`，不管看起来对它们施以转换是有多么合理。）不过有办法⽣成⼀个 `iterator`，使其指向和 `const_iterator` 指向相同，但是看起来不明显，也没有⼴泛应⽤，在这本书也不值得讨论。除此之外，我希望⽬前我陈述的观点是清晰的：`const_iterator`在 C++98 中会有很多问题。这⼀天结束时，开发者们不再相信能加 `const` 就加它的教条，而是只在实⽤的地⽅加它，C++98 的 `const_iterator` 不是那么实⽤。

所有的这些都在 C++11 中改变了，现在 `const_iterator` 即容易获取⼜容易使⽤。容器的成员函数 `cbegin` 和 `cend` 产出 `const_iterator`，甚⾄对于⾮常量容器，那些之前只使⽤ `iterator` 指⽰位置的 STL 成员函数也可以使⽤ `const_iterator` 了。使⽤ C++11 `const_iterator` 重写 C++98 使⽤ `iterator` 的代码也稀松平常：

```cpp
std::vector<int> values;    // 和之前一样
...
auto it = std::find(values.cbegin(), values.cend(), 1983);
values.insert(it, 1998);
```

现在使用 const_iterator 的代码就很实用了

唯⼀⼀个 C++11 对于 `const_iterator` ⽀持不⾜（译注：C++14 ⽀持但是 C++11 的时候还没）的情况是：当你想写最⼤程度通⽤的库，并且这些库代码为⼀些容器和类似容器的数据结构提供⾮成员函数 `begin、end(以及cbegin，cend，rbegin，rend)` 而不是成员函数（其中⼀种情况就是原⽣数组）。最⼤程度通⽤的库会考虑使⽤⾮成员函数而不是假设成员函数版本存在。

举个例⼦，我们可以泛化下⾯的 `findAndInsert` ：

```cpp
template<typename C, typename V>
void findAndInsert(C& container, const V& targetVal, const V& insertVal){
    using std::cbegin;
    using std::cend;
    auto it = std::find(cbegin(container), cend(container), targetVal);
    container.insert(it, insertVal);
}
```

它可以在 C++14 ⼯作良好，但是很遗憾，C++11 不在良好之列。由于标准化的疏漏，C++11 只添加了⾮成员函数 `begin和end`，但是没有添加 `cbegin，cend，rbegin，rend，crbegin，crend`。

如果使用的是 C++11 并且想写一个最大程度的通用代码，而你使⽤的 STL 没有提供缺失的⾮成员函数 `cbegin` 和它的朋友们，你可以简单的抛出你⾃⼰的实现。

```cpp
template <class C>
auto cbegin(const C& container)->decltype(std::begin(container))
{
    return std::begin(container); // 解释⻅下
}
```

你可能很惊讶⾮成员函数 `cbegin` 没有调⽤成员函数 `cbegin` 吧？但是请跟逻辑走。这个 `cbegin` 模板接受任何容器或者类似容器的数据结构 C ，并且通过 `const` 引⽤访问第⼀个实参 `container`。如果 C 是⼀个普通的容器类型（如 `std::vector<int>` ），`container` 将会引⽤⼀个常量版本的容器（即 `conststd::vector<int>&` ）。对 `const` 容器调⽤⾮成员函数 `begin` (由 C++11 提供)将产出 `const_iterator`，这个迭代器也是模板要返回的。⽤这种⽅法实现的好处是就算容器只提供 `begin` 不提供 `cbegin` 也没问题。那么现在你可以将这个⾮成员函数 `cbegin` 施于只⽀持 `begin` 的容器。

如果 C 是原⽣数组，这个模板也能⼯作。这时，`container` 成为⼀个 `const` 数组。C++11 为数组提供特化版本的⾮成员函数 `begin`，它返回指向数组第⼀个元素的指针。⼀个 `const` 数组的元素也是 `const`，所以对于 `const` 数组，⾮成员函数 `begin` 返回指向 `const` 的指针。在数组的上下⽂中，所谓指向 `const` 的指针，也就是 `const_iterator` 了。

回到最开始，本条款的中⼼是⿎励你只要能就使⽤ `const_iterator`。最原始的动机是——只要它有意义就加上 `const——C++98 `就有的思想。但是在 ` C++98，它只是⼀般有⽤，到了 C++11,它就是极其有⽤了，C++14 在其基础上做了些修补⼯作。

总结：
- 优先考虑 `const_iterator` 而⾮ `iterator`
- 在最⼤程度通⽤的代码中，优先考虑⾮成员函数版本的 `begin，end，rbegin`等，而⾮同名成员函数