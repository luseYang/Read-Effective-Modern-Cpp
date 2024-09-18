---
title: Item 27 熟悉通⽤引⽤重载的替代⽅法
tags: ["通用引用"]
categories: [Effective Modern C++]
description: '熟悉通⽤引⽤重载的替代⽅法'
date: 2024-08-11 13:46:21
cover: https://images.pexels.com/photos/27520824/pexels-photo-27520824.jpeg
---

# Abandon overloading

在 Item 26 中的第⼀个例⼦中，`logAndAdd` 代表了许多函数，这些函数可以使⽤不同的名字来避免在通⽤引⽤上的重载的弊端。例如两个重载的 `logAndAdd` 函数，可以分别改名为 `logAndAddName` 和 `logAndAddNameIdx`。但是，这种⽅式不能⽤在第⼆个例⼦，`Person` 构造函数中，因为构造函数的名字本类名固定了。此外谁愿意放弃重载呢？

# Pass by const T&

⼀种替代⽅案是退回到 C++98，然后将通⽤引⽤替换为 `const` 的左值引⽤。事实上，这是 Item 26 中⾸先考虑的⽅法。缺点是效率不⾼，会有拷⻉的开销。现在我们知道了通⽤引⽤和重载的组合会导致问题，所以放弃⼀些效率来确保⾏为正确简单可能也是⼀种不错的折中。

# Pass by value

通常在不增加复杂性的情况下提⾼性能的⼀种⽅法是，将按引⽤传递参数替换为按值传递，这是违反直觉的。该设计遵循 Item 41 中给出的建议，即在你知道要拷⻉时就按值传递，因此会参考 Item 41 来详细讨论如何设计与⼯作，效率如何。这⾥，在 `Person` 的例⼦中展⽰：

```cpp
class Person {
public:
    explicit Person(std::string p) // replace T&& ctor; see
    : name(std::move(n)) {} // Item 41 for use of std::move
    explicit Person(int idx)
    : name(nameFromIdx(idx)) {}
    ...
private:
    std::string name;
};
```

因为没有 `std::string` 构造器可以接受整型参数，所有 `int` 或者其他整型变量（⽐如 `std::size_t`、`hort`、`long` 等）都会使⽤ `int` 类型重载的构造函数。相似的，所有 `std::string` 类似的参数（字⾯量等）都会使⽤ `std::string` 类型的重载构造函数。没有意外情况。我想你可能会说有些⼈想要使⽤ 0 或者 `NUL`L 会调⽤ `int` 重载的构造函数，但是这些⼈应该参考 Item 8 反复阅读指导使⽤ 0 或者 `NULL` 作为空指针让他们恶⼼。

# Use Tag dispatch

传递 `const` 左值引⽤参数以及按值传递都不⽀持完美转发。如果使⽤通⽤引⽤的动机是完美转发，我们就只能使⽤通⽤引⽤了，没有其他选择。但是⼜不想放弃重载。所以如果不放弃重载⼜不放弃通⽤引⽤，如何避免咋通⽤引⽤上重载呢？

实际上并不难。通过查看重载的所有参数以及调⽤的传⼊参数，然后选择最优匹配的函数----计算所有参数和变量的组合。通⽤引⽤通常提供了最优匹配，但是如果通⽤引⽤是包含其他⾮通⽤引⽤参数列表的⼀部分，则不是通⽤引⽤的部分会影响整体。这基本就是 tag dispatch ⽅法，下⾯的⽰例会使这段话更容易理解。

我们将 tag dispatch 应⽤于 `logAndAdd` 例⼦，下⾯是原来的代码，以免你找不到 Item 26 的代码位置：

```cpp
std::multiset<std::string> names; // global data structure
template<typename T> // make log entry and add
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_clokc::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}
```

就其本⾝而⾔，功能执⾏没有问题，但是如果引⼊⼀个 int 类型的重载，就会重新陷⼊Item 26中描述的⿇烦。这个Item的⽬标是避免它。不通过重载，我们重新实现 `logAndAdd` 函数分拆为两个函数，⼀个针对整型值，⼀个针对其他。`logAndAdd` 本⾝接受所有的类型。

这两个真正执⾏逻辑的函数命名为 `logAndAddImpl` 使⽤重载。⼀个函数接受通⽤引⽤参数。所以我们同时使⽤了重载和通⽤引⽤。但是每个函数接受第⼆个参数，表征传⼊的参数是否为整型。这第⼆个参数可以帮助我们避免陷⼊到Item 26中提到的⿇烦中，因为我们将其安排为第⼆个参数决定选择哪个重载函数。

```cpp
template<typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(std::forward<T>(name),
    std::is_integral<T>()); // not quite correct
}
```

这个函数转发它的参数给 `logAndAddImpl` 函数，但是多传递了⼀个表⽰是否T为整型的变量。⾄少，这就是应该做的。对于右值的整型参数来说，这也是正确的。但是如同 Item 28 中说明，如果左值参数传递给通⽤引⽤ `name`，类型推断会使左值引⽤。所以如果左值 `int` 被传⼊ `logAndAdd`，T将被推断为 `int&`。这不是⼀个整型类型，因为引⽤不是整型类型。这意味着 `std::is_integral<T>` 对于左值参数返回 `false`，即使确实传⼊了整型值。

意识到这个问题基本相当于解决了它，因为C++标准库有⼀个类型 `trait`（参⻅Item 9），`std::remove_reference`，函数名字就说明做了我们希望的：移除引⽤。所以正确实现的代码应该是这样：

```cpp
template<typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(std::forward<T>(name),
    std::is_instegral<typename std::remove_reference<T>::type>());
}
```

这个代码很巧妙。（在 C++14 中，你可以通过 `std::remove_reference_t<T>` 来简化写法，参看Item9）处理完之后，我们可以将注意⼒转移到名为 `logAndAddImpl` 的函数上了。有两个重载函数，第⼀个仅⽤于⾮整型类型（即 `std::is_instegral<typename std::remove_reference<T>::type>()` 是 `false`）：

```cpp
template<typename T>
void logAndAddImpl(T&& name, std::false_type) // ⾼亮为std::false_type
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}
```

⼀旦你理解了⾼亮参数的含义代码就很直观。概念上，`logAndAdd` 传递⼀个布尔值给 `logAndAddImpl` 表明是否传⼊了⼀个整型类型，但是 `true` 和 `false` 是运⾏时值，我们需要使⽤编译时决策来选择正确的 logAndAddImpl 重载。这意味着我们需要⼀个类型对应 `true`，`false` 同理。这个需要是经常出现的，所以标准库提供了这样两个命名 `std::true_type` and `std::false_type`。 `logAndAdd` 传递给 `logAndAddImpl` 的参数类型取决于T是否整型，如果T是整型，它的类型就继承⾃ s`td::true_type`，反之继承⾃ `std::false_type` 。最终的结果就是，当 `T` 不是整型类型时，这个 `logAndAddImpl` 重载会被调⽤。

第⼆个重载覆盖了相反的场景：当T是整型类型。在这个场景中，`logAndAddImpl` 简单找到下标处的 `name`，然后传递给 `logAndAdd`：

```cpp
std::string nameFromIdx(int idx); // as in item 26
void logAndAddImpl(int idx, std::true_type) // ⾼亮：std::true_type
{
    logAndAdd(nameFromIdx(idx));
}
```

通过下标找到对应的 `name`，然后让 `logAndAddImpl` 传递给 `logAndAdd`，我们避免了将⽇志代码放⼊这个 `logAndAddImpl` 重载中。

在这个设计中，类型 `std::true_type` 和` std::false_type` 是“标签”，其唯⼀⽬的就是强制重载解析按照我们的想法来执⾏。注意到我们甚⾄没有对这些参数进⾏命名。他们在运⾏时毫⽆⽤处，事实上我们希望编译器可以意识到这些 `tag` 参数是⽆⽤的然后在程序执⾏时优化掉它们（⾄少某些时候有些编译器会这样做）。这种在 `logAndAdd` 内部的通过 `tag` 来实现重载实现函数的“分发”，因此这个设计名称为：`tagdispatch`。这是模板元编程的标准构建模块，你对现代 C++ 库中的代码了解越多，你就会越多遇到这种设计。

就我们的⽬的而⾔，`tag dispatch`的重要之处在于它可以允许我们组合重载和通⽤引⽤使⽤，而没有Item 26中提到的问题。分发函数--- `logAndAdd` ----接受⼀个没有约束的通⽤引⽤参数，但是这个函数没有重载。实现函数--- `logAndAddImpl` ----是重载的，⼀个接受通⽤引⽤参数，但是重载规则不仅依赖通⽤引⽤参数，还依赖新引⼊的 `tag` 参数。结果是 `tag` 来决定采⽤哪个重载函数。通⽤引⽤参数可以⽣成精确匹配的事实在这⾥并不重要。（译者注：这⾥确实⽐较啰嗦，如果理解了上⾯的内容，这段完全可以没有。）

# Constraining templates that take universal references（约束使⽤通⽤引⽤的模板）

`tag dispatch` 的关键是存在单独⼀个函数（没有重载）给客⼾端API。这个单独的函数分发给具体的实现函数。创建⼀个没有重载的分发函数通常是容易的，但是Item 26中所述第⼆个问题案例是 `Person` 类的完美转发构造函数，是个例外。编译器可能会⾃⾏⽣成拷⻉和移动构造函数，所以即使你只写了⼀个构造函数并在其中使⽤ `tag dispatch`，编译器⽣成的构造函数也打破了你的期望。

实际上，真正的问题不是编译器⽣成的函数会绕过tag diapatch设计，而是不总会绕过 `tag dispatch`。你希望类的拷⻉构造总是处理该类型的 non-const 左值构造请求，但是如同Item 26中所述，提供具有通⽤引⽤的构造函数会使通⽤引⽤构造函数被调⽤而不是拷⻉构造函数。还说明了当⼀个基类声明了完美转发构造函数，派⽣类实现⾃⼰的拷⻉和移动构造函数时会发⽣错误的调⽤（调⽤基类的完美转发构造函数而不是基类的拷⻉或者移动构造）

这种情况，采⽤通⽤引⽤的重载函数通常⽐期望的更加贪⼼，但是有不满⾜使⽤ `tag dispatch` 的条件。你需要不同的技术，可以让你确定允许使⽤通⽤引⽤模板的条件。朋友你需要的就是 `std::enable_if`。

`std::enable_if` 可以给你提供⼀种强制编译器执⾏⾏为的⽅法，即使特定模板不存在。这种模板也会被禁⽌。默认情况下，所有模板是启⽤的，但是使⽤ `std::enable_if` 可以使得仅在条件满⾜时模板才启⽤。在这个例⼦中，我们只在传递的参数类型不是 `Person` 使⽤ `Person` 的完美转发构造函数。如果传递的参数是 `Person`，我们要禁⽌完美转发构造函数（即让编译器忽略它），因此就是拷⻉或者移动构造函数处理，这就是我们想要使⽤ `Person` 初始化另⼀个 `Person` 的初衷。

这个主意听起来并不难，但是语法⽐较繁杂，尤其是之前没有接触过的话，让我慢慢引导你。有⼀些使⽤ `std::enbale_if` 的样板，让我们从这⾥开始。下⾯的代码是 `Person` 完美转发构造函数的声明，我仅展⽰声明，因为实现部分跟 Item 26 中没有区别。

```cpp
class Person {
public:
    template<typename T,
    typename = typename std::enable_if<condition>::type> // 本⾏⾼亮
    explicit Person(T&& n);
    ...
};
```

为了理解⾼亮部分发⽣了什么，我很遗憾的表⽰你要⾃⾏查询语法含义，因为详细解释需要花费⼀定空间和时间，而本书并没有⾜够的空间（在你⾃⾏学习过程中，请研究 `"SFINAE"` 以及 `std::enable_if` ，因为 `“SFINAE”` 就是使 `std::enable_if` 起作⽤的技术）。这⾥我想要集中讨论条件的表⽰，该条件表⽰此构造函数是否启⽤。

这⾥我们想表⽰的条件是确认 `T` 不是 `Person` 类型，即模板构造函数应该在T不是 `Person` 类型的时候启⽤。因为 `type trait` 可以确定两个对象类型是否相同（`std::is_same`），看起来我们需要的就是 `!std::is_same<Person, T>::value`（注意语句开始的！，我们想要的是不相同）。这很接近我们想要的了，但是不完全正确，因为如同 Item 28 中所述，对于通⽤引⽤的类型推导，如果是左值的话会推导成左值引⽤，⽐如这个代码:

```cpp
Person p("Nancy");
auto cloneOfP(p); // initialize from lvalue
```

`T` 的类型在通⽤引⽤的构造函数中被推导为 Person& 。 Person 和 Person& 类型是不同的，`std::is_same` 对⽐ `std::is_same<Person, Person&>::value`会是 `false` 。

如果我们更精细考虑仅当T不是 `Person` 类型才启⽤模板构造函数，我们会意识到当我们查看 `T` 时，应该忽略：
- 是否引⽤。对于决定是否通⽤引⽤构造器启⽤的⽬的来说，`Person`, `Person&`, `Person&&` 都是跟 `Person` ⼀样的。
- 是不是 `const` 或者 `volatile`。如上所述，`const Person`, `volatile Person`, `const volatile Person` 也是跟 `Person` ⼀样的。

这意味着我们需要⼀种⽅法消除对于 `T` 的 引⽤，`const`, `volatile` 修饰。再次，标准库提供了这样的功能 `type trait`，就是 `std::decay`。`std::decay<T>::value` 与 `T` 是相同的，只不过会移除 引⽤, `const`, `volatile` 的修饰。（这⾥我没有说出另外的真相，`std::decay` 如同其名⼀样，可以将 `array` 或者 `function` 退化成指针，参考 Item 1，但是在这⾥讨论的问题中，它刚好合适）。我们想要控制构造器是否启⽤的条件可以写成

```cpp
!std::is_same<Person, typename std::decay<T>::type>::value
```

表⽰ `Person` 与 `T` 的类型不同。将其带回整体代码中，`Person` 的完美转发构造函数的声明如下：

```cpp
class Person {
public:
    template<typename T,
        typename = typename std::enable_if<
            !std::is_same<Person, typename std::decay<T>::type>::value>::type> // 本⾏⾼亮
    explicit Person(T&& n);
    ...
};
```

如果你之前从没有看到过这种类型的代码，那你可太幸福了。最后是这种设计是有原因的。当你使⽤其他机制来避免同时使⽤重载和通⽤引⽤时（你总会这样做），确实应该那样做。不过，⼀旦你习惯了使⽤函数语法和尖括号的使⽤，也不坏。此外，这可以提供你⼀直想要的⾏为表现。在上⾯的声明中，使⽤ `Person` 初始化⼀个 `Person` ----⽆论是左值还是右值，`const` 还是 `volatile` 都不会调⽤到通⽤引⽤构造函数。

成功了，对吗？确实！

当然没有。等会再庆祝。Item 26 还有⼀个情景需要解决，我们需要继续探讨下去。

假定从 `Person` 派⽣的类以常规⽅式实现拷⻉和移动操作：

```cpp
class SpecialPerson: public Person {
public:
    SpecialPerson(const SpecialPerson& rhs): Person(rhs)
    {...} // copy ctor; calls base class forwarding ctor!
    SpecialPerson(SpecialPerson&& rhs): Person(std::move(rhs))
    {...} // move ctor; calls base class forwarding ctor!
};
```

这和 Item 26 中的代码是⼀样的，包括注释也是⼀样。当我们拷⻉或者移动⼀个 `SpecialPerson` 对象时，我们希望调⽤基类对应的拷⻉和移动构造函数，但是这⾥，我们将 `SpecialPerson` 传递给基类的构造器，因为 `SpecialPerson` 和 Person 类型不同，所以完美转发构造函数是启⽤的，会实例化为精确匹配的构造函数。⽣成的精确匹配的构造函数之于重载规则⽐基类的拷⻉或者移动构造函数更优，所以这⾥的代码，拷⻉或者移动 `SpecialPerson` 对象就会调⽤ `Person` 类的完美转发构造函数来执⾏基类的部分。跟 Item 26 的困境⼀样。

派⽣类仅仅是按照常规的规则⽣成了⾃⼰的移动和拷⻉构造函数，所以这个问题的解决还要落实在在基类，尤其是控制是否使⽤ Person 通⽤引⽤构造函数启⽤的条件。现在我们意识到不只是禁⽌ `Person` 类型启⽤模板构造器，而是禁⽌ `Person` 以及任何派⽣⾃ `Person` 的类型启⽤模板构造器。讨厌的继承！

你应该不意外在这⾥看到标准库中也有 `type trait` 判断⼀个类型是否继承⾃另⼀个类型，就是 `std::is_base_of`。如果 `std::is_base_of<T1, T2>` 是 `true` 表⽰ `T2` 派⽣⾃ `T1`。类型系统是⾃派⽣的，表⽰ `std::is_base_of<T, T>::value` 总是 `true`。这就很⽅便了，我们想要修正关于我们控制 `Person` 完美转发构造器的启⽤条件，只有当 `T` 在消除 引⽤，`const`, `volatile` 修饰之后，并且既不是 `Person` ⼜不是 `Person` 的派⽣类，才满⾜条件。所以使⽤ `std::is_base_of` 代替 `std::is_same` 就可以了：

```cpp
class Person {
public:
    template<typename T,typename = typename std::enable_if<!std::is_base_if<Person, typename std::decay<T>::type>::value>::type>
    explicit Person(T&& n);
    ...
};
```

现在我们终于完成了最终版本。这是 C++11 版本的代码，如果我们使⽤ C++14，这份代码也可以⼯作，但是有更简洁⼀些的写法如下：

```cpp
class Person { // C++14
public:
    template<typename T, typename = std::enable_if_t<!std::is_base_of<Person,std::decay_t<T>>::value>>explicit Person(T&& n);
    ...
};
```

好了，我承认，我⼜撒谎了。我们还没有完成，但是越发接近最终版本了。⾮常接近，我保证。

我们已经知道如何使⽤ `std::enable_if` 来选择性禁⽌ `Person` 通⽤引⽤构造器来使得⼀些参数确保使⽤到拷⻉或者移动构造器，但是我们还是不知道将其应⽤于区分整型参数和⾮整型参数。毕竟，我们的原始⽬标是解决构造函数模糊性问题。

我们需要的⼯具都介绍过了，我保证都介绍了，
1. 加⼊⼀个 `Person` 构造函数重载来处理整型参数
2. 约束模板构造器使其对于某些参数禁⽌
使⽤这些我们讨论过的技术组合起来，就能解决这个问题了：

```cpp
class Person { // C++14
public:
    template<typename T, typename = std::enable_if_t<!std::is_base_of<Person, std::decay_t<T>::value && !std::is_integral<std::remove_reference_t<T>>::value>>explicit Person(T&& n): name(std::forward<T>(n)){...} // ctor for std::strings and args convertible to strings

    explicit Person(int idx): name(nameFromIdx(idx)){...} // ctor for integral args
    ... // copy and move ctors, etc
private:
    std::string name;
};
```

看！多么优美！好吧，优美之处只是对于那些迷信模板元编程之⼈，但是事实却是提出了不仅能⼯作的⽅法，而且极具技巧。因为使⽤了完美转发，所以具有最⼤效率，因为控制了使⽤通⽤引⽤的范围，可以避免对于⼤多数参数能实例化精确匹配的滥⽤问题。

# Trade-offs （权衡，折中）

本Item提到的前三个技术---`abandoning overloading`, `passing by const T&`, `passing by value`---在函数调⽤中指定每个参数的类型。后两个技术----`tag dispatch` 和 `constraing template eligibility`----使⽤完美转发，因此不需要指定参数类型。这⼀基本决定（是否指定类型）有⼀定后果。

通常，完美转发更有效率，因为它避免了仅处于符合参数类型而创建临时对象。在 `Person` 构造函数的例⼦中，完美转发允许将 `Nancy` 这种字符串字⾯量转发到容器内部的 `std::string` 构造器，不使⽤完美
转发的技术则会创建⼀个临时对象来满⾜传⼊的参数类型。

但是完美转发也有缺点。·即使某些类型的参数可以传递给特定类型的参数的函数，也⽆法完美转发。Item 30 中探索了这⽅⾯的例⼦。

第⼆个问题是当 `client` 传递⽆效参数时错误消息的可理解性。例如假如创建⼀个 `Person` 对象的 `client` 传递了⼀个由 `char16_t` （⼀种C++11引⼊的类型表⽰16位字符）而不是 `char`（`std::string` 包含的）：

```cpp
Person p(u"Konrad Zuse"); // "Konrad Zuse" consists of characters of type const
char16_t
```

使⽤本 Item 中讨论的前三种⽅法，编译器将看到可⽤的采⽤ `int` 或者 `std::string` 的构造函数，并且它们或多或少会产⽣错误消息，表⽰没有可以从 `const char16_t` 转换为 `int` 或者 `std::string` 的⽅法。

但是，基于完美转发的⽅法，`const char16_t` 不受约束地绑定到构造函数的参数。从那⾥将转发到 `Person` 的 `std::string` 的构造函数，在这⾥，调⽤者传⼊的内容(`const char16_t` 数组)与所需内容(`std::string` 构造器可接受的类型)发⽣的不匹配会被发现。由此产⽣的错误消息会让⼈更容易理解，在我使⽤的编译器上，会产⽣超过 160 ⾏错误信息。

在这个例⼦中，通⽤引⽤仅被转发⼀次（从 `Person` 构造器到 `std::string` 构造器），但是更复杂的系统中，在最终通⽤引⽤到达最终判断是否可接受的函数之前会有多层函数调⽤。通⽤引⽤被转发的次数越多，产⽣的错误消息偏差就越⼤。许多开发者发现仅此问题就是在性能优先的接口使⽤通⽤引⽤的障碍。（译者注：最后⼀句话可能翻译有误，待确认）

在 `Person` 这个例⼦中，我们知道转发函数的通⽤引⽤参数要⽀持 `std::string` 的初始化，所以我们可以⽤ `static_assert` 来确认是不是⽀持。` std::is_constructible type trait` 执⾏编译时测试⼀个类型的对象是否可以构造另⼀个不同类型的对象，所以代码可以这样：

```cpp
class Person {
public:
    template<typename T, typename = std::enable_if_t<!std::is_base_of<Person, std::decay_t<T>>::value && !std::is_integral<std::remove_reference_t<T>>::value>>explicit Person(T&& n) :name(std::forward<T>(n)){
    //assert that a std::string can be created from a T object(这⾥到...⾼亮)
        static_assert(
            std::is_constructible<std::string, T>::value, "Parameter n can't be used to construct a std::string"
        );
        ... // the usual ctor work goes here
    }
    ... // remainder of Person class (as before)
};
```

如果 `client` 代码尝试使⽤⽆法构造 `std::strin`g 的类型创建 `Person`，会导致指定的错误消息。不幸的是，在这个例⼦中，`static_assert` 在构造函数体中，但是作为成员初始化列表的部分在检查之前。所以我使⽤的编译器，结果是由 `static_assert` 产⽣的清晰的错误消息在常规错误消息（最多 160 ⾏以上那个）后出现。

# 总结

- 通⽤引⽤和重载的组合替代⽅案包括使⽤不同的函数名，通过 `const` 左值引⽤传参，按值传递参数，使⽤ `tag dispatch`
- 通过 `std::enable_if` 约束模板，允许组合通⽤引⽤和重载使⽤，`std::enable_if` 可以控制编译器哪种条件才使⽤通⽤引⽤的实例
- 通⽤引⽤参数通常具有⾼效率的优势，但是可⽤性就值得斟酌