---
title: Item 10 Item10：优先考虑限域枚举而非未限域枚举
tags: ["现代C++", "枚举", "作用域"]
categories: [Effective Modern C++]
description: '优先考虑限域枚举而非未限域枚举'
date: 2024-07-20 20:19:21
cover: https://s2.loli.net/2024/07/19/QxH9Ms4ml5GWdco.jpg
---

# 条款10：限域枚举和非限域枚举

通常来说，在花括号中声明⼀个名字会限制它的作⽤域在花括号之内。但这对于 C++98 ⻛格的 `enum` 中声明的枚举名是不成⽴的。这些在 `enum` 作⽤域中声明的枚举名所在的作⽤域也包括 `enum` 本⾝，也就是说这些枚举名和 `enum` 所在的作⽤域中声明的相同名字没有什么不同

```cpp
enum Color { black, white, red };   // black, white, red 和
                                    // Color⼀样都在相同作⽤域
auto white = false;                 // 错误! white早已在这个作⽤
                                    // 域中存在
```

事实上这些枚举名泄漏进和它们所被定义的 `enum` 域⼀样的作⽤域。有⼀个官⽅的术语：**未限域枚举(unscoped enum)**在 C++11 中它们有⼀个相似物，**限域枚举(scoped enum)，它不会导致枚举名泄漏**：

```cpp
enum class Color { black, white, red };// black, white, red被限制在Color域内

auto white = false; // 没问题，同样域内没有这个名字

Color c = white;    //错误，这个域中没有white

Color c = Color::white;     // 没问题
auto c = Color::white;      // 也没问题（也符合条款5的建议）
```

因为限域枚举是通过 `enum class` 声明，所以它们有时候也被称为**枚举类(enum classes)**。

使⽤限域枚举减少命名空间污染是⼀个⾜够合理使⽤它的理由，其实限域枚举还有第⼆个吸引⼈的优点：在它的作⽤域中,枚举名是**强类型**的。**未限域枚举中的枚举名会隐式转换为整型（现在，也可以转换为浮点类型）**。因此下⾯这种歪曲语义的做法也是完全有效的：

```cpp
enum Color { black, white, red }; // 未限域枚举
std::vector<std::size_t> // func返回x的质因⼦
primeFactors(std::size_t x);
Color c = red;
…
if (c < 14.5) { // Color与double⽐较 (!)
    auto factors = primeFactors(c);   // 计算⼀个Color的质因⼦(!)
…
}
```

在 `enum` 后⾯写⼀个 `class` 就可以将⾮限域枚举转换为限域枚举，接下来就是完全不同的故事展开了。现在不存在任何隐式转换可以将限域枚举中的枚举名转化为任何其他类型。

```cpp
enum class Color { black, white, red };         // Color现在是限域枚举
Color c = Color::red;                           // 和之前⼀样，只是
…                                               // 多了⼀个域修饰符

if (c < 14.5) {                                 // 错误！不能⽐较
                                                // Color和double
    auto factors =                              // 错误! 不能向参数为std::size_t的函数
    primeFactors(c);                            // 传递Color参数
…
}
```

如果你真的很想执⾏ `Color` 到其他类型的转换，和平常⼀样，使⽤正确的类型转换运算符扭曲类型系统：

```cpp
if (static_cast<double>(c) < 14.5) { // 奇怪的代码，但是有效
    auto factors = // suspect, but
        primeFactors(static_cast<std::size_t>(c)); // 能通过编译
    …
}
```

似乎⽐起⾮限域枚举而⾔限域枚举有第三个好处，因为限域枚举可以前置声明。⽐如，它们可以不指定枚举名直接前向声明：

```cpp
enum Color;         // 错误
enum class Color;   // 正确
```

其实这是⼀个误导。在 C++11 中，⾮限域枚举也可以被前置声明，但是只有在做⼀些其他⼯作后才能实现。这些⼯作来源于⼀个事实：在 C++ 中所有的枚举都有⼀个由编译器决定的整型的基础类型。对于⾮限域枚举⽐如 `Color`，

```cpp
enum Color { black, white, red };
```

编译器可能选择 `char` 作为基础类型，因为这⾥只需要表⽰三个值。然而，有些枚举中的枚举值范围可能会⼤些，⽐如：

```cpp
enum Status{
    good = 0,
    failed = 1,
    incomplete = 100,
    corrupt = 200,
    indeterminate = 0xFFFFFFFF
};
```

这⾥值的范围从 0 到 0xFFFFFFFF。除了在不寻常的机器上（⽐如⼀个 `char` ⾄少有 32bits 的那种），编译器都会选择⼀个⽐ `char` ⼤的整型类型来表⽰ `Status` 。

为了⾼效使⽤内存，**编译器通常在确保能包含所有枚举值的前提下为枚举选择⼀个最小的基础类型**。在⼀些情况下，编译器将会优化速度，舍弃⼤小，这种情况下它可能不会选择最小的基础类型，而是选择对优化⼤小有帮助的类型。**为此，C++98 只⽀持枚举定义（所有枚举名全部列出来）；**枚举声明是不被允许的。这使得编译器能为之前使⽤的每⼀个枚举选择⼀个基础类型。

但是不能前置声明枚举也是有缺点的。最⼤的缺点莫过于它可能增加编译依赖。再次考虑 `Status` 枚举：

```cpp
enum Status { good = 0,
    failed = 1,
    incomplete = 100,
    corrupt = 200,
    indeterminate = 0xFFFFFFFF
};
```

这种 `enum` 很有可能⽤于整个系统，因此系统中每个包含这个头⽂件的组件都会依赖它。如果引⼊⼀个新状态值，

```cpp
enum Status { good = 0,
    failed = 1,
    incomplete = 100,
    corrupt = 200,
    audited = 500,
    indeterminate = 0xFFFFFFFF
};
```

那么可能整个系统都得重新编译，即使只有⼀个⼦系统——或者⼀个函数使⽤了新添加的枚举名。这是⼤家都不希望看到的。C++11 中的前置声明可以解决这个问题。⽐如这⾥有⼀个完全有效的限域枚举声明和⼀个以该限域枚举作为形参的函数声明：

```cpp
enum class Status; // forward declaration
void continueProcessing(Status s); // use of fwd-declared enum
```

即使 `Status` 的定义发⽣改变，包含这些声明的头⽂件也不会重新编译。而且如果 `Status` 添加⼀个枚举名（⽐如添加⼀个 `audited` ），`continueProcessing` 的⾏为不受影响（因为 `continueProcessing` 没有使⽤这个新添加的 `audited`），`continueProcessing` 也不需要重新编译。但是如果编译器在使⽤它之前需要知晓该枚举的⼤小，该怎么声明才能让 C++11 做到 C++98 不能做到的事情呢？

限域枚举的基础类型总是已知的，而对于⾮限域枚举，你可以指定它。默认情况下，限域
举的基础类型是 `int`：

```cpp
enum class Status; // 基础类型是int
```

如果默认的 `int` 不适⽤，你可以重写它：

```cpp
enum class Status: std::uint32_t; // Status的基础类型
// 是std::uint32_t
// (需要包含 <cstdint>)
```

不管怎样，编译器都知道限域枚举中的枚举名占⽤多少字节。要为⾮限域枚举指定基础类型，你可以同上，然后前向声明⼀下：

```cpp
enum Color: std::uint8_t; // 为⾮限域枚举Color指定
// 基础为
// std::uint8_t
```

基础类型说明也可以放到枚举定义处：

```cpp
enum class Status: std::uint32_t { good = 0,
    failed = 1,
    incomplete = 100,
    corrupt = 200,
    audited = 500,
    indeterminate = 0xFFFFFFFF
};
```

限域枚举避免命名空间污染而且不接受隐式类型转换，但是有⼀种情况下⾮限域枚举是很有⽤的。那就是获取 C++11 tuples 中的字段的时候。⽐如在社交⽹站中，假设我们有⼀个 `tuple` 保存了⽤⼾的名字，`email` 地址，声望点：

```cpp
using UserInfo = std::tuple<std::string, std::string, std::size_t> ;              // 名字   email地址  声望
```

虽然注释说明了 `tuple` 各个字段对应的意思，但当你在另⽂件遇到下⾯的代码那之前的注释就不是那么有⽤了：

```cpp
UserInfo uInfo; // tuple对象
…
auto val = std::get<1>(uInfo); // 获取第⼀个字段
```

可以使⽤⾮限域枚举将名字和字段编号关联起来以避免上述需求：

```cpp
enum UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo;
…
auto val = std::get<uiEmail>(uInfo); // ，获取⽤⼾email
```

之所以它能正常⼯作是因为 `UserInfoFields` 中的枚举名隐式转换成 `std::size_t` 了,其中 `std::size_t` 是 `std::get` 模板实参所需的。

对应的限域枚举版本就很啰嗦了：

```cpp
enum class UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo; // as before
…
auto val =std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>(uInfo);
```

总结
- C++98的枚举即⾮限域枚举
- 限域枚举的枚举名仅在enum内可⻅。要转换为其它类型只能使⽤cast。
- ⾮限域/限域枚举都⽀持基础类型说明语法，限域枚举基础类型默认是 int 。⾮限域枚举没有默认基础类型。
- 限域枚举总是可以前置声明。⾮限域枚举仅当指定它们的基础类型时才能前置。
