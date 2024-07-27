---
title: Item 18 对于独占资源使⽤std::unique_ptr
tags: ["智能指针", "std::unique_ptr"]
categories: [Effective Modern C++]
description: 'std::unique_ptr'
date: 2024-07-25 15:35:21
cover: https://images.pexels.com/photos/20623114/pexels-photo-20623114.jpeg
---

# 前言
原始指针的一些问题：
1. 它的声明不能指⽰所指到底是**单个对象**还是**数组**。
2. 它的声明没有告诉你⽤完后是否应该销毁它，即**指针是否拥有所指之物。**
3. 如果你决定你应该销毁对象所指，没⼈告诉你该⽤ `delete` 还是其他析构机制（⽐如将指针传给专⻔的销毁函数）。
4. 如果你发现该⽤ `delete`。 原因 1 说了不知道是 `delete` 单个对象还是 `delete` 数组。如果⽤错了结果是未定义的。
5. 假设你确定了指针所指，知道销毁机制，也很难确定你在所有执⾏路径上都执⾏了销毁操作（包括异常产⽣后的路径）。少⼀条路径就会产⽣资源泄漏，销毁多次还会导致未定义⾏为。
6. ⼀般来说没有办法告诉你指针是否变成了悬空指针，即内存中不再存在指针所指之物。悬空指针会在对象销毁后仍然指向它们。

# 条款18：对于独占资源使⽤std::unique_ptr

当你需要⼀个智能指针时，`std::unique_ptr` 通常是最合适的。可以合理假设，默认情况下，`std::unique_ptr` 等同于原始指针，而且对于⼤多数操作（包括取消引⽤），他们执⾏的指令完全相同。这意味着你甚⾄可以在内存和时间都⽐较紧张的情况下使⽤它。如果原始指针够小够快，那么 `std::unique_ptr` ⼀样可以。

`std::unique_ptr` 体现了**专有所有权语义**。⼀个 `non-null std::unique_ptr` 始终有其指向的内容。移动操作将所有权从源指针转移到⽬的指针，拷⻉操作是不允许的，因为**如果你能拷⻉⼀个 `std::unique_ptr`，你会得到指向相同内容的两个 `std::unique_ptr`，每个都认为⾃⼰拥有资源，销毁时就会出现重复销毁。**因此，`std::unique_ptr` 只⽀持移动操作。当 `std::unique_ptr` 被销毁时，其指向的资源也执⾏析构函数。而原始指针需要显⽰调⽤ `delete` 来销毁指针指向的资源。

`std::unique_ptr` 的常⻅⽤法是作为继承层次结构中对象的⼯⼚函数（工厂函数是设计模式中的一种，用于创建对象而不必指定具体的类。这种方法的主要目的是将对象的创建过程封装起来）返回类型。假设我们有⼀个基类 `Investment`（⽐如 `stocks, bonds, real estate` 等）的继承结构。

```cpp
class Investment { ... };
class Sock: public Investment {...};
class Bond: public Investment {...};
class RealEstate: public Investment {...};
```

这种继承关系的⼯⼚函数在堆上分配⼀个对象然后返回指针，调⽤⽅在不需要的时候，销毁对象。这使⽤场景完美匹配 `std::unique_ptr`，因为调⽤者对⼯⼚返回的资源负责（即对该资源的专有所有权），并且 `std::unique_ptr` 会⾃动销毁指向的内容。可以这样声明：

```cpp
template<typename... Ts>
std::unique_ptr<Investment> makeInvestment(Ts&&... params);
```

调⽤者应该在单独的作⽤域中使⽤返回的 `std::unique_ptr` 智能指针：

```cpp
{
    ...
    auto pInvestment = makeInvestment(arguments);
    ...
} //destroy *pInvestment
```

但是也可以在所有权转移的场景中使⽤它，⽐如将⼯⼚返回的 `std::unique_ptr` 移⼊容器中，然后将容器元素移⼊对象的数据成员中，然后对象随即被销毁。

发⽣这种情况时，并且销毁该对象将导致销毁从⼯⼚返回的资源，对象 `std::unique_ptr` 的数据成员也被销毁。如果所有权链由于异常或者其他⾮典型控制流出现中断（⽐如提前 `return` 函数或者循环中的 `break`），则拥有托管资源的 `std::unique_ptr` 将保证指向内容的析构函数被调⽤，销毁对应资源。

默认情况下，销毁将通过 `delete` 进⾏，但是在构造过程中，可以⾃定义 `std::unique_ptr` 指向对象的析构函数：任意函数（或者函数对象，包括lambda）。如果通过 `makeInvestment` 创建的对象不能直接被删除，应该⾸先写⼀条⽇志，可以实现如下：

```cpp
auto delInvmt = [](Investment* pInvestment)
{
    makeLogEntry(pInvestment);
    delete pInvestment;
};
template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)>makeInvestment(Ts&& params)
{
    std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt);
    if (/*a Stock object should be created*/)
    {
        pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if ( /* a Bond object should be created */ )
    {
        pInv.reset(new Bond(std::forward<Ts>(params)...));
    }
    else if ( /* a RealEstate object should be created */ )
    {
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));
    }
    return pInv;
}
```

假设你把 `makeInvestment` 的调⽤结果存储在 `auto` 变量中，那么你可能会忽略在删除过程中需要特殊处理的事实，当然，因为使⽤了 `unique_ptr` 意味着你不需要考虑在资源释放时的路径，以及确保只释放⼀次，`std::unique_ptr` ⾃动解决了这些问题。从使⽤者⻆度，`makeInvestment` 接口很棒。

这个实现确实相当棒，如果你理解了：
- `delInvmt` 是⾃定义的从 `makeInvestment` 返回的析构函数。所有的⾃定义的析构⾏为接受要销毁对象的原始指针，然后执⾏销毁操作。如上例⼦。使⽤ lambda 创建 `delInvmt` 是⽅便的，而且，正如稍后看到的，⽐编写常规的函数更有效
- 当使⽤⾃定义删除器时，必须将其作为第⼆个参数传给 `std::unique_ptr`。对于 `decltype`，更多信息查看 Item3
- `makeInvestment` 的基本策略是创建⼀个空的 `std::unique_ptr`，然后指向⼀个合适类型的对象，然后返回。为了与 pInv 关联⾃定义删除器，作为构造函数的第⼆个参数
- 尝试将原始指针（⽐如 `new` 创建）赋值给 `std::unique_ptr` 通不过编译，因为不存在从原始指针到智能指针的隐式转换。这种隐式转换会出问题，所以禁⽌。这就是为什么通过 `reset` 来传递 `new` 指针的原因
- 使⽤ `new` 时，要使⽤ `std::forward` 作为参数来完美转发给 `makeInvestment` （查看Item 25）。这使调⽤者提供的所有信息可⽤于正在创建的对象的构造函数
- ⾃定义删除器的参数类型是 `Investment*`，尽管真实的对象类型是在 `makeInvestment` 内部创建的，它最终通过在 lambda 表达式中，作为 `Investment*` 对象被删除。这意味着我们通过基类指针删除派⽣类实例，为此，基类必须是虚函数析构：

```cpp
class Investment {
public:
    ...
    virtual ~Investment();
    ...
};
```

在 C++14 中，函数的返回类型推导存在（参阅Item 3），意味着 `makeInvestment` 可以更简单，封装的⽅式实现：

```cpp
template<typename... Ts>
makeInvestment(Ts&& params)
{
    auto delInvmt = [](Investment* pInvestment)
    {
        makeLogEntry(pInvestment);
        delete pInvestment;
    };
    std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt);
    if (/*a Stock object should be created*/)
    {
        pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if ( /* a Bond object should be created */ )
    {
        pInv.reset(new Bond(std::forward<Ts>(params)...));
    }
    else if ( /* a RealEstate object should be created */ )
    {
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));
    }
    return pInv;
}
```

当使⽤默认删除器时，你可以合理假设 `std::unique_ptr` 和原始指针⼤小相同。当⾃定义删除器时，情况可能不再如此。删除器是个函数指针，通常会使 `std::unique_ptr` 的字节从⼀个增加到两个。对于删除器的函数对象来说，⼤小取决于函数对象中存储的状态多少，⽆状态函数对象（⽐如没有捕获的 lambda 表达式）对⼤小没有影响，这意味当⾃定义删除器可以被lambda实现时，尽量使⽤ lambda

```cpp
auto delInvmt = [](Investment* pInvestment)
{
    makeLogEntry(pInvestment);
    delete pInvestment;
};
template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)>makeInvestment(Ts&& params); //返回Investment*的⼤小

void delInvmt2(Investment* pInvestment)
{
    makeLogEntry(pInvestment);
    delete pInvestment;
}

template<typename... Ts>
std::unique_ptr<Investment, void(*)(Investment*)>
makeInvestment(Ts&&... params); //返回Investment*的指针加⾄少⼀个函数指针的⼤小
```

具有很多状态的⾃定义删除器会产⽣⼤尺⼨ `std::unique_ptr` 对象。如果你发现⾃定义删除器使得你的 `std::unique_ptr` 变得过⼤，你需要审视修改你的设计。

`std::unique_ptr` 有两种形式，⼀种⽤于单个对象（ `std::unique_ptr<T>` ），⼀种⽤于数组（ `std::unique_ptr<T[]>` ）。结果就是，指向哪种形式没有歧义。`std::unique_ptr` 的API设计会⾃动匹配你的⽤法，⽐如 [] 操作符就是数组对象，* 和 -> 就是单个对象专有。

数组的 `std::unique_ptr` 的存在应该不被使⽤，因为 `std::array`, `std::vector`, `std::string` 这些更好⽤的数据容器应该取代原始数组。原始数组的使⽤唯⼀情况是你使⽤类似 C 的 API 返回⼀个指向堆数组的原始指针。

`std::unique_ptr` 是C++11中表⽰专有所有权的⽅法，但是其最吸引⼈的功能之⼀是它可以轻松⾼效的转换为 `std::shared_ptr` ：

```cpp
std::shared_ptr<Investment> sp = makeInvestment(arguments);
```

这就是为什么 `std::unique_ptr` ⾮常适合⽤作⼯⼚函数返回类型的关键部分。 ⼯⼚函数⽆法知道调⽤者是否要对它们返回的对象使⽤专有所有权语义，或者共享所有权（即 `std::shared_ptr`）是否更合适。 通过返回 `std::unique_ptr`，⼯⼚为调⽤者提供了最有效的智能指针，但它们并不妨碍调⽤者⽤其更灵活的兄弟替换它。

总结：
- `std::unique_ptr` 是轻量级、快速的、只能 move 的管理专有所有权语义资源的智能指针
- 默认情况，资源销毁通过 `delete`，但是⽀持⾃定义 `delete` 函数。有状态的删除器和函数指针会增加 `std::unique_ptr` 的⼤小
- 将 `std::unique_ptr` 转化为 `std::shared_ptr` 是简单的

