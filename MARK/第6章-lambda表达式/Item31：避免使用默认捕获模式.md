---
title: Item 31 避免使用默认捕获模式
tags: ["lambda表达式"]
categories: [Effective Modern C++]
description: '避免使用默认捕获模式'
date: 2024-08-18 18:54:40
cover: https://images.pexels.com/photos/27308308/pexels-photo-27308308.jpeg
---
 
# lambda

Lambda 表达式是 C++ 编程中的游戏规则改变者。这有点令⼈惊讶，因为它没有给语⾔带来新的表达能⼒。Lambda 可以做的所有事情都可以通过其他⽅式完成。但是 lambda 是创建函数对象相当便捷的⼀种⽅法，对于⽇常的 C++ 开发影响是巨⼤的。没有 lambda 时，标准库中的 `_if` 算法（⽐如，`std::find_if`, `std::remove_if`, `std::count_if` 等）通常需要繁琐的谓词，但是当有 lambda 可⽤时，这些算法使⽤起来就变得相当⽅便。⽐较函数（⽐如，`std::sort`, `std::nth_element`,`std::lower_bound`等）与算法函数也是相同的。在标准库外，lambda 可以快速创建 `std::unique_ptr` 和 `std::shared_ptr` 的⾃定义 `deleter`，并且使线程 `API` 中条件变量的条件设置变得同样简单（参⻅ Item 39）。除了标准库，lambda 有利于即时的回调函数，接口适配函数和特定上下⽂中的⼀次性函数。Lambda 确实使 C++ 成为更令⼈愉快的编程语⾔。

与 Lambda 相关的词汇可能会令⼈疑惑，这⾥做⼀下简单的回顾：

- *lambda 表达式就是⼀个表达式*。在代码的⾼亮部分就是lambda

```cpp
std::find_if(container.begin(), container.end(),
    [](int val){ return 0 < val && val < 10; }); // 本⾏⾼亮
```

- *闭包是lambda创建的运⾏时对象*。依赖捕获模式，闭包持有捕获数据的副本或者引⽤。在上⾯的 `std::find_if` 调⽤中，闭包是运⾏时传递给 `std::find_if` 第三个参数。
- *闭包类是从中实例化闭包的类*。每个 lambda 都会使编译器⽣成唯⼀的闭包类。Lambda 中的语句成为其闭包类的成员函数中的可执⾏指令。

Lambda 通常被⽤来创建闭包，该闭包仅⽤作函数的参数。上⾯对 `std::find_if` 的调⽤就是这种情况。然而，闭包通常可以拷⻉，所以可能有多个闭包对应于⼀个 lambda。⽐如下⾯的代码：

```cpp
{
    int x; // x is local variable
    ...
    auto c1 = [x](int y) { return x * y > 55; }; // c1 is copy of the closure produced by the lambda
    auto c2 = c1; // c2 is copy of c1
    auto c3 = c2; // c3 is copy of c2
    ...
}
```

c1， c2，c3都是 lambda 产⽣的闭包的副本。

⾮正式的讲，模糊 lambda，闭包和闭包类之间的界限是可以接受的。但是，在随后 的Item 中，区分编译期还是运⾏时以及它们之间的相互关系是重要的。

# 避免使用默认捕获模式

C++11 中有两种默认的捕获模式：按引⽤捕获和按值捕获。但按引⽤捕获可能会带来悬空引⽤的问题，而按值引⽤可能会诱骗你让你以为能解决悬空引⽤的问题（实际上并没有），还会让你以为你的闭包是独⽴的（事实上也不是独⽴的）。

按引⽤捕获会导致闭包中包含了对局部变量或者某个形参（位于定义 lambda 的作⽤域）的引⽤，如果该 lambda 创建的闭包⽣命周期超过了局部变量或者参数的⽣命周期，那么闭包中的引⽤将会变成悬空引⽤。举个例⼦，假如我们有⼀个元素是过滤函数的容器，该函数接受⼀个 `int` 作为参数，并返回⼀个布尔值，该布尔值的结果表⽰传⼊的值是否满⾜过滤条件。

```cpp
using FilterContainer = // see Item 9 for
std::vector<std::function<bool(int)>>; // "using", Item 2
// for std::function
FilterContainer filters; // filtering funcs
```

我们可以添加⼀个过滤器，⽤来过滤掉 5 的倍数。

```cpp
filters.emplace_back( // see Item 42 for
    [](int value) { return value % 5 == 0; } // info on
);
````

然而我们可能需要的是能够在运⾏期获得被除数，而不是将 5 硬编码到 lambda 中。因此添加的过滤器逻辑将会是如下这样：

```cpp
void addDivisorFilter()
{
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();
    auto divisor = computeDivisor(calc1, calc2);
    filters.emplace_back( // danger!
        [&](int value) { return value % divisor == 0; } // ref to
    ); //
    divisor
}
// will
// dangle!
```

这个代码实现是⼀个定时炸弹。lambda 对局部变量 `divisor` 进⾏了引⽤，但该变量的⽣命周期会在 `addDivisorFilter` 返回时结束，刚好就是在语句 `filters`. `emplace_back` 返回之后，因此该函数的本质就是容器添加完，该函数就死亡了。使⽤这个 `filter` 会导致未定义⾏为，这是由它被创建那⼀刻起就决定了的。

现在，同样的问题也会出现在 `divisor` 的显式按引⽤捕获。

```cpp
filters.emplace_back(
    [&divisor](int value) // danger! ref to
    { return value % divisor == 0; } // divisor will
);
```

但通过显式的捕获，能更容易看到 lambda 的可⾏性依赖于变量 `divisor` 的⽣命周期。另外，写成这种形式能够提醒我们要注意确保 `divisor` 的⽣命周期⾄少跟 lambda 闭包⼀样⻓。⽐起 `"[&]"` 传达的意思，显式捕获能让⼈更容易想起“确保没有悬空变量”。

如果你知道⼀个闭包将会被⻢上使⽤（例如被传⼊到⼀个stl算法中）并且不会被拷⻉，那么在lambda环
境中使⽤引⽤捕获将不会有⻛险。在这种情况下，你可能会争论说，没有悬空引⽤的危险，就不需要避
免使⽤默认的引⽤捕获模式。例如，我们的过滤lambda只会⽤做C++11中std::all_of的⼀个参数，返回
满⾜条件的所有元素：

```cpp
template<typename C>
void workWithContainer(const C& container)
{
    auto calc1 = computeSomeValue1(); // as above
    auto calc2 = computeSomeValue2(); // as above
    auto divisor = computeDivisor(calc1, calc2); // as above
    using ContElemT = typename C::value_type; // type of
                                            // elements in
                                            // container
    using std::begin; // for
    using std::end; // genericity;
    // see Item 13
    if (std::all_of( // if all values
        begin(container), end(container), // in container
        [&](const ContElemT& value) // are multiples
        { return value % divisor == 0; }) // of divisor...
    ) {
        … // they are...
    } 
    else {
        … // at least one
    }   // isn't...
}
```

的确如此，这是安全的做法，但这种安全是不确定的。如果发现 lambda 在其它上下⽂中很有⽤（例如作为⼀个函数被添加在 `filters` 容器中），然后拷⻉粘贴到⼀个 `divisor` 变量已经死亡的，但闭包⽣命周期还没结束的上下⽂中，你⼜回到了悬空的使⽤上了。同时，在该捕获语句中，也没有特别提醒了你注意分析 `divisor` 的⽣命周期。

从⻓期来看，使⽤显式的局部变量和参数引⽤捕获⽅式，是更加符合软件⼯程规范的做法。

额外提⼀下，C++14 ⽀持了在 lambda 中使⽤ `auto` 来声明变量，上⾯的代码在 C++14 中可以进⼀步简化，`ContElemT` 的别名可以去掉，`if` 条件可以修改为：

```cpp
if (std::all_of(begin(container), end(container),
    [&](const auto& value) // C++14
    { return value % divisor == 0; }))
```

⼀个解决问题的⽅法是，`divisor` 按值捕获进去，也就是说可以按照以下⽅式来添加 lambda：

```cpp
filters.emplace_back( // now
[=](int value) { return value % divisor == 0; } // divisor
); // can't
// dangle
```

这⾜以满⾜本实例的要求，但在通常情况下，按值捕获并不能完全解决悬空引⽤的问题。这⾥的问题是如果你按值捕获的是⼀个指针，你将该指针拷⻉到 `lambda` 对应的闭包⾥，但这样并不能避免 `lambda` 外删除指针的⾏为，从而导致你的指针变成悬空指针。

也许你要抗议说：“这不可能发⽣。看过了第四章，我对智能指针的使⽤⾮常热衷。只有那些失败的 C++98 的程序员才会⽤裸指针和 `delete` 语句。”这也许是正确的，但却是不相关的，因为事实上你的确会使⽤裸指针，也的确存在被你删除的可能性。只不过在现代的 C++ 编程⻛格中，不容易在源代码中显露出来而已。

假设在⼀个 `Widget` 类，可以实现向过滤容器添加条⽬：

```cpp
class Widget {
public:
    … // ctors, etc.
    void addFilter() const; // add an entry to filters
private:
    int divisor; // used in Widget's filter
};
```

这是 `Widget::addFilter` 的定义：

```cpp
void Widget::addFilter() const
{
    filters.emplace_back(
        [=](int value) { return value % divisor == 0; }
    );
}
```

这个做法看起来是安全的代码，lambda 依赖于变量 `divisor`，但默认的按值捕获被拷⻉进了 lambda 对应的所有⽐保重，这真的正确吗？

错误，完全错误。

闭包只会对 lambda 被创建时所在作⽤域⾥的⾮静态局部变量⽣效。在 `Widget::addFilter()` 的视线⾥，`divisor` 并不是⼀个局部变量，而是 `Widget` 类的⼀个成员变量。它不能被捕获。如果默认捕获模式被删除，代码就不能编译了：

```cpp
void Widget::addFilter() const
{
    filters.emplace_back( // error!
        [](int value) { return value % divisor == 0; } // divisor
    ); // not
} // available
```

另外，如果尝试去显式地按引⽤或者按值捕获 `divisor` 变量，也⼀样会编译失败，因为 `divisor` 不是这⾥的⼀个局部变量或者参数。

```cpp
void Widget::addFilter() const
{
    filters.emplace_back(
        [divisor](int value) // error! no local
        { return value % divisor == 0; } // divisor to capture
    );
}
```

因此这⾥的默认按值捕获并不是不会变量 `divisor`，但它的确能够编译通过，这是怎么⼀回事呢？

解释就是这⾥隐式捕获了 `this` 指针。每⼀个⾮静态成员函数都有⼀个 `this` 指针，每次你使⽤⼀个类内的成员时都会使⽤到这个指针。例如，编译器会在内部将 `divisor` 替换成 `this->divisor`。这⾥ `Widget::addFilter()` 的版本就是按值捕获了 `this`。

```cpp
void Widget::addFilter() const
{
    filters.emplace_back(
        [=](int value) { return value % divisor == 0; }
    );
}
```

真正被捕获的是 `Widget` 的 `this` 指针。编译器会将上⾯的代码看成以下的写法：

```cpp
void Widget::addFilter() const
{
    auto currentObjectPtr = this;
    filters.emplace_back(
        [currentObjectPtr](int value)
        { return value % currentObject->divisor == 0; }
    );
}
```

明⽩了这个就相当于明⽩了 lambda 闭包的⽣命周期与 `Widget` 对象的关系，闭包内含有 `Widget` 的 `this` 指针的拷⻉。特别是考虑以下的代码，再参考⼀下第四章的内容，只使⽤智能指针：

```cpp
using FilterContainer = // as before
    std::vector<std::function<bool(int)>>;
FilterContainer filters; // as before
void doSomeWork()
{
    auto pw =                       // create Widget; see
        std::make_unique<Widget>(); // Item 21 for
                                    // std::make_unique
    pw->addFilter();    // add filter that uses
                        // Widget::divisor
    …
}                       // destroy Widget; filters
                        // now holds dangling pointer!
```

当调⽤ `doSomeWork` 时，就会创建⼀个过滤器，其⽣命周期依赖于由 `std::make_unique` 管理的 `Widget` 对象。即⼀个含有 `Widget this` 指针的过滤器。这个过滤器被添加到 `filters` 中，但当 `doSomeWork` 结束时，`Widget` 会由 `std::unique_ptr` 去结束其⽣命。从这时起，`filter` 会含有⼀个悬空指针。

这个特定的问题可以通过做⼀个局部拷⻉去解决：

```cpp
void Widget::addFilter() const
{
    auto divisorCopy = divisor; // copy data member
    filters.emplace_back(
        [divisorCopy](int value) // capture the copy
        { return value % divisorCopy == 0; } // use the copy
    );
}
```

事实上如果采⽤这种⽅法，默认的按值捕获也是可⾏的。

```cpp
void Widget::addFilter() const
{
    auto divisorCopy = divisor; // copy data member
    filters.emplace_back(
        [=](int value) // capture the copy
        { return value % divisorCopy == 0; } // use the copy
    );
}
```

但为什么要冒险呢？当你⼀开始捕获 `divisor` 的时候，默认的捕获模式就会⾃动将 `this` 指针捕获进来了。

在 C++14 中，⼀个更好的捕获成员变量的⽅式时使⽤通⽤的 lambda 捕获：

```cpp
void Widget::addFilter() const
{
    filters.emplace_back( // C++14:
        [divisor = divisor](int value) // copy divisor to closure
        { return value % divisor == 0; } // use the copy
    );
}
```

这种通⽤的 lambda 捕获并没有默认的捕获模式，因此在 C++14 中，避免使⽤默认捕获模式的建议仍然时成⽴的。

使⽤默认的按值捕获还有另外的⼀个缺点，它们预⽰了相关的闭包是独⽴的并且不受外部数据变化的影响。⼀般来说，这是不对的。lambda 并不会独⽴于局部变量和参数，但也没有不受静态存储⽣命周期的影响。⼀个定义在全局空间或者指定命名空间的全局变量，或者是⼀个声明为 `static` 的类内或⽂件内的成员。这些对象也能在 lambda ⾥使⽤，但它们不能被捕获。但按值引⽤可能会因此误导你，让你以为捕获了这些变量。参考下⾯版本的 `addDivisorFilter()` 函数：

```cpp
void addDivisorFilter()
{
    static auto calc1 = computeSomeValue1(); // now static
    static auto calc2 = computeSomeValue2(); // now static
    static auto divisor = // now static
    computeDivisor(calc1, calc2);
    filters.emplace_back(
        [=](int value) // captures nothing!
        { return value % divisor == 0; } // refers to above static
    );
    ++divisor; // modify divisor
}
```

随意地看了这份代码的读者可能看到 `"[=]"`，就会认为“好的，lambda 拷⻉了所有使⽤的对象，因此这是独⽴的”。但上⾯的例⼦就表现了不独⽴闭包的⼀种情况。它没有使⽤任何的⾮ `static` 局部变量和形参，所以它没有捕获任何东西。然而 lambda 的代码引⽤了静态变量 `divisor`，任何 lambda 被添加到 `filters` 之
后，`divisor` 都会递增。通过这个函数，会把许多 lambda 都添加到 `filiters` ⾥，但每⼀个 lambda 的⾏为都是新的（分别对应新的 `divisor`值）。这个 lambda 是通过引⽤捕获 `divisor`，这和默认的按值捕获表⽰的含义有着直接的⽭盾。如果你⼀开始就避免使⽤默认的按值捕获模式，你就能解除代码的⻛险。

# 总结

- 默认的按引⽤捕获可能会导致悬空引⽤；
- 默认的按值引⽤对于悬空指针很敏感（尤其是 `this` 指针），并且它会误导⼈产⽣ lambda 是独⽴的想法；