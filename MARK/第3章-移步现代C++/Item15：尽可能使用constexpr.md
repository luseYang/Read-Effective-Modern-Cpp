---
title: Item 15 尽可能使用 constexpr
tags: ["constexpr", "现代C++", "const"]
categories: [Effective Modern C++]
description: '尽可能使用constexpr'
date: 2024-07-23 11:55:21
cover: https://images.pexels.com/photos/20554662/pexels-photo-20554662.jpeg
---

# 条款15：尽可能的使⽤constexpr

`constexpr` 当⽤于对象上⾯，它本质上就是 `const` 的加强形式，但是当它⽤于函数上，意思就⼤不相同了。

从概念上来说，`constexpr` 表明⼀个值不仅仅是常量，还是编译期可知的。这个表述并不全⾯，因为当 `constexpr` 被⽤于函数的时候，事情就有⼀些细微差别了。

我现在只想说，你不能假设 `constexpr` 函数是 `const`，也不能保证它们的（译注：返回）值是在编译期可知的。最有意思的是，这些是特性。关于 `constexpr` 函数返回的结果不需要是 `const`，也不需要编译期可知这⼀点是良好的⾏为。

编译期可知的值 “享有特权”，**它们可能被存放到只读存储空间中。对于那些嵌⼊式系统的开发者，这个特性是相当重要的。**更⼴泛的应⽤是 “其值编译期可知” 的常量整数会出现在需要“整型常量表达式的 `context` 中，这类 `context` 包括数组⼤小，整数模板参数（包括 `std::array` 对象的⻓度），枚举量，对⻬修饰符（译注： `alignas(val)` ），等等。如果你想在这些 `context` 中使⽤变量，你⼀定会希望将它们声明为 `constexpr`，因为编译器会确保它们是编译期可知的：

```cpp
int sz; // ⾮constexpr变量
…
constexpr auto arraySize1 = sz;     // 错误! sz的值在
                                    // 编译期不可知
std::array<int, sz> data1;          // 错误!⼀样的问题
constexpr auto arraySize2 = 10;     // 没问题，10是编译
                                    // 期可知常量
std::array<int, arraySize2> data2;  // 没问题, arraySize2是constexpr
```

注意 `const` 不提供 `constexpr` 所能保证之事，因为const对象不需要在编译期初始化它的值。

```cpp
int sz; // 和之前⼀样
const auto arraySize = sz; // 没问题，arraySize是sz的常量复制
std::array<int, arraySize> data; // 错误，arraySize值在编译期不可知
```

换句话说：所有 `constexpr` 的对象都是 `const`，但不是所有 `const`都是 `constexpr`。**如果你想编译器保证⼀个变量有⼀个可以放到那些需要编译期常量的上下⽂的值，你需要的⼯具是constexpr而不是const。**

如果使⽤场景涉及函数，那 `constexpr` 就更有趣了。如果实参是编译期常量，它们将产出编译期值；如果是运⾏时值，它们就将产出运⾏时值。

- `constexpr` 函数可以⽤于需求编译期常量的上下⽂。如果你传给 `constexpr` 函数的实参在编译期可知，那么结果将在编译期计算。如果实参的值在编译期不知道，你的代码就会被拒绝。
- 当⼀个 `constexpr` 函数被⼀个或者多个编译期不可知值调⽤时，它就像普通函数⼀样，运⾏时计算它的结果。这意味着你不需要两个函数，⼀个⽤于编译期计算，⼀个⽤于运⾏时计算。`constexpr` 全做了。

假设我们需要⼀个数据结构来存储⼀个实验的结果，而这个实验可能以各种⽅式进⾏。实验期间⻛扇转速，温度等等都可能导致亮度值改变，亮度值可以是⾼，低，或者⽆。如果有 n 个实验相关的环境条件。它们每⼀个都有三个状态，最终可以得到的组合 3^n 个。储存所有实验结果的所有组合需要这个数据结构⾜够⼤。假设每个结果都是int并且n是编译期已知的（或者可以被计算出的），⼀个 `std::array` 是⼀个合理的选择。我们需要⼀个⽅法在编译期计算 3^n 。C++ 标准库提供了 `std::pow`，它的数学意义正是我们所需要的，但是，对我们来说，这⾥还有两个问题。第⼀， `std::pow` 是为浮点类型设计的 我们需要整型结果。第⼆， `std::pow` 不是 `constexpr`（即，使⽤编译期可知值调⽤得到的可能不是编译期可知的结果），所以我们不能⽤它作为 `std::array` 的⼤小。

幸运的是，我们可以应需写个 pow 。

```cpp
constexpr int pow(int base, int exp) noexcept{ // pow是constexpr函数, 绝不抛异常
    … // 实现在这⾥
}
constexpr auto numConds = 5; //条件个数

std::array<int, pow(3, numConds)> results; // 结果有3^numConds个元素
```

`pow` 前⾯的 `constexpr` 没有告诉我们 `pow` 返回⼀个 `const` 值，它只说了如果 `base` 和 `exp` 是编译期常量， `pow` 返回值可能是编译期常量。如果`base` 和(或) `exp`不是编译期常量， `pow` 结果将会在运⾏时计算。这意味着 `pow` 不仅可以⽤于像 `std::array` 的⼤小这种需要编译期常量的地⽅，它也可以⽤于运⾏时环境：

```cpp
auto base = readFromDB("base"); // 运⾏时获取三个值
auto exp = readFromDB("exponent");
auto baseToExp = pow(base, exp); // 运⾏时调⽤pow
```

因为 `constexpr` 函数必须能在编译期值调⽤的时候返回编译器结果，就必须对它的实现施加⼀些限制。这些限制在 C++11 和 C++14 标准间有所出⼊。

C++11 中，`constexpr` 函数的代码不超过⼀⾏语句：⼀个 `return`。听起来很受限，但实际上有两个技巧可以扩展 `constexpr` 函数的表达能⼒。第⼀，使⽤三元运算符 “?:” 来代替 `if-else` 语句，第⼆，使⽤递归代替循环。因此 `pow` 可以像这样实现：

```cpp
constexpr int pow(int base, int exp) noexcept
{
return (exp == 0 ? 1 : base * pow(base, exp - 1));
}
```

这样没问题，但是很难想象除了使⽤函数式语⾔的程序员外会觉得这样硬核的编程⽅式更好。在 C++14 中，`constexpr`函数的限制变得⾮常宽松了，所以下⾯的函数实现成为了可能；

```cpp
constexpr int pow(int base, int exp) noexcept // C++14
{
auto result = 1;
for (int i = 0; i < exp; ++i) result *= base;
return result;
}
```

`constexpr` 函数限制为只能获取和返回字⾯值类型，这基本上意味着具有那些类型的值能在编译期决定。在 C++11 中，除了 `void` 外的所有内置类型外还包括⼀些⽤⼾定义的字⾯值，因为构造函数和其他成员函数可以是 `constexpr`：

```cpp
class Point {
public:
    constexpr Point(double xVal = 0, double yVal = 0) noexcept : x(xVal), y(yVal){}
    constexpr double xValue() const noexcept { return x; }
    constexpr double yValue() const noexcept { return y; }

    void setX(double newX) noexcept { x = newX; }
    void setY(double newY) noexcept { y = newY; }
private:
    double x, y;
};
```

`Point` 的构造函数被声明为 `constexpr`，因为如果传⼊的参数在编译期可知，`Point` 的数据成员也能在编译器可知。因此 `Point` 就能被初始化为 `constexpr`：

```cpp
constexpr Point p1(9.4, 27.7); // 没问题，构造函数会在编译期“运⾏”
constexpr Point p2(28.8, 5.3); // 也没问题
```

类似的，`xValue` 和 `yValue` 的 `getter` 函数也能是 `constexpr`，因为如果对⼀个编译期已知的 `Point` 对象调⽤ `getter`，数据成员 `x` 和 `y` 的值也能在编译期知道。这使得我们可以写⼀个 `constexpr` 函数⾥⾯调⽤ `Point` 的 `getter` 并初始化 `constexpr` 的对象：

```cpp
constexpr
Point midpoint(const Point& p1, const Point& p2) noexcept
{
    return { 
        (p1.xValue() + p2.xValue()) / 2,
        (p1.yValue() + p2.yValue()) / 2 
    };
}
constexpr auto mid = midpoint(p1, p2);
```

它意味着 `mid` 对象通过调⽤构造函数，`getter` 和成员函数就能在只读内存中创建！它也意味着你可以在模板或者需要枚举量的表达式⾥⾯使⽤像 `mid.xValue()*10` 的表达式！它也意味着以前相对严格的某⼀⾏代码只能⽤于编译期，某⼀⾏代码只能⽤于运⾏时的界限变得模糊，⼀些运⾏时的普通计算能并⼊编译时。越多这样的代码并⼊，你的程序就越快。（当然，编译会花费更⻓时间）

在 C++11 中，有两个限制使得 `Point` 的成员函数 `setX` 和 `setY` 不能声明为 `constexpr`。第⼀，它们修改它们操作的对象的状态， 并且在 C++11 中，`constexpr` 成员函数是隐式的 `const`。第⼆，它们只能有 `void` 返回类型，`void` 类型不是 C++11 中的字⾯值类型。这两个限制在 C++14 中放开了，所以 C++14 中 `Point` 的 `setter`也能声明为 `constexpr`：

```cpp
class Point {
public:
    ...
    constexpr void setX(double newX) noexcept { x = newX; }
    constexpr void setY(double newY) noexcept { y = newY; }
    ...
};
```

现在也能写这样的函数：

```cpp
constexpr Point reflection(const Point& p) noexcept
{
    Point result;
    result.setX(-p.xValue());
    result.setY(-p.yValue());
    return result;
}
```

客⼾端代码可以这样写：

```cpp
constexpr Point p1(9.4, 27.7);
constexpr Point p2(28.8, 5.3);
constexpr auto mid = midpoint(p1, p2);
constexpr auto reflectedMid = // reflectedMid的值
    reflection(mid); // 在编译期可知
```

`constexpr` 是对象和函数接口的⼀部分。加上 `constexpr` 相当于宣称“我能在C++ 要求常量表达式的地⽅使⽤它”。如果你声明⼀个对象或者函数是 `constexpr`，客⼾端程序员就会在那些场景中使⽤它。如果你后⾯认为使⽤ `constexpr` 是⼀个错误并想移除它，你可能造成⼤量客⼾端代码不能编译。尽可能的使⽤ `constexpr` 表⽰你需要⻓期坚持对某个对象或者函数施加这种限制。

总结：
- `constexpr` 对象是 `cosnt`，它的值在编译期可知
- 当传递编译期可知的值时，`cosntexpr` 函数可以产出编译期可知的结果

