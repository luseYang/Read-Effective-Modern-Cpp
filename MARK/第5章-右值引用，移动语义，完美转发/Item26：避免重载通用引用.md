---
title: Item 26 避免在通⽤引⽤上重载 std::forward
tags: ["通用引用"]
categories: [Effective Modern C++]
description: '避免在通⽤引⽤上重载'
date: 2024-08-10 13:52:21
cover: https://images.pexels.com/photos/27520824/pexels-photo-27520824.jpeg
---

假定你需要写⼀个函数，它使⽤ `name` 这样⼀个参数，打印当前⽇期和具体时间到⽇志中，然后将 `name` 加⼊到⼀个全局数据结构中。你可能写出来这样的代码：

```cpp
std::multiset<std::string> names; // global data structure
void logAndAdd(const std::string& name)
{
    auto now = std::chrono::system_lock::now(); // get current time
    log(now, "logAndAdd"); // make log entry
    names.emplace(name); // add name to global data structure; see Item 42 for infoon emplace
}
```

这份代码没有问题，但是效率方面不足。考虑这三个调⽤：

```cpp
std::string petName("Darla");
logAndAdd(petName); // pass lvalue std::string
logAndAdd(std::string("Persephone")); // pass rvalue std::string
logAndAdd("Patty Dog"); // pass string literal
```

在第⼀个调⽤中，`logAndAdd` 使⽤变量作为参数。在 `logAndAdd` 中 `name` 最终也是通过 `emplace` 传递给 `names`。因为 `name` 是左值，会拷⻉到 `names` 中。没有⽅法避免拷⻉，因为是左值传递的。

在第三个调⽤中，参数 `name` 绑定⼀个右值，**但是这次是通过 `"Patty Dog"` 隐式创建的临时 `std::string` 变量**。在第⼆个调⽤总，`name` 被拷⻉到 `names`，但是这⾥，传递的是⼀个字符串字⾯量。直接将字符串字⾯量传递给 `emplace`，不会创建 `std::string` 的临时变量，而是直接在 `std::multiset` 中通过字⾯量构建 `std::string`。在第三个调⽤中，我们会消耗 `std::string` 的拷⻉开销，但是连移动开销都不想有，更别说拷⻉的。

我们可以通过使⽤通⽤引⽤（参⻅ Item 24）重写第⼆个和第三个调⽤来使效率提升，按照 Item 25 的说法，`std::forward` 转发引⽤到 `emplace`。代码如下：

```cpp
template<typename T>
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_lock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}

std::string petName("Darla"); // as before
logAndAdd(petName); // as before , copy
logAndAdd(std::string("Persephone")); // move rvalue instead of copying it
logAndAdd("Patty Dog"); // create std::string in multiset instead of copying atemporary std::string
```

`client` 不总是有访问 `logAndAdd` 要求的 `names` 的权限。有些 `clients` 只有 `names` 的下标。为了⽀持这种 `client`，`logAndAdd` 需要重载为：

```cpp
std::string nameFromIdx(int idx); // return name corresponding to idx
void logAndAdd(int idx)
{
    auto now = std::chrono::system_lock::now();
    log(now, "logAndAdd");
    names.emplace(nameFromIdx(idx));
}
```

之后的两个调用按照预期工作：

```cpp
std::string petName("Darla");
logAndAdd(petName);
logAndAdd(std::string("Persephone"));
logAndAdd("Patty Dog"); // these calls all invoke the T&& overload
logAndAdd(22); // calls int overload
```

事实上，这只能基本按照预期⼯作，假定⼀个 `client` 将 `short` 类型当做下标传递给 `logAndAdd`:

```cpp
short nameIdx;
...
logAndAdd(nameIdx); // error!
```

之后⼀⾏的 `error` 说明并不清楚，下⾯让我来说明发⽣了什么:有两个重载的 `logAndAdd`。⼀个使⽤通⽤应⽤推导出T的类型是 `short`，因此可以精确匹配。对于 `int` 参数类型的重载 `logAndAdd` 也可以 `short` 类型提升后匹配成功。根据正常的重载解决规则，精确匹配优先于类型提升的匹配，所以被调⽤的是通⽤引⽤的重载。

在通⽤引⽤中的实现中，将 `short` 类型 `emplace` 到 `std::string` 的容器中，发⽣了错误。所有这⼀切的原因就是对于 `short` 类型通⽤引⽤重载优先于 `int` 类型的重载。

使⽤通⽤引⽤类型的函数在 C++ 中是贪婪函数。他们机会可以精确匹配任何类型的参数（极少不适⽤的类型在 Item 30 中介绍）。这也是组合重载和通⽤引⽤使⽤是糟糕主意的原因：通⽤引⽤的实现会匹配⽐开发者预期要多得多的参数类型。

⼀个更容易调⼊这种陷阱的例⼦是完美转发构造函数。简单对 `logAndAdd` 例⼦进⾏改造就可以说明这个问题。将使⽤ `std::string` 类型改为⾃定义 P`erson 类型即可：

```cpp
class Person
{
public:
    template<typename T>
    explicit Person(T&& n) :name(std::forward<T>(n)) {} // perfect forwarding ctor;initializes data member
    explicit Person(int idx): name(nameFromIdx(idx)) {}
    ...
private:
    std::string name;
};
```

在 `logAndAdd` 的例⼦中，传递⼀个不是 `int` 的整型变量（⽐如 `std::size_t`, `short`, `long` 等）会调⽤通⽤引⽤的构造函数而不是int的构造函数，这会导致编译错误。这⾥这个问题甚⾄更糟糕，因为 `Person` 中存在的重载⽐⾁眼看到的更多。在 Item 17 中说明，在适当的条件下，C++ 会⽣成拷⻉和移动构造函数，即使类包含了模板构造也在合适的条件范围内。如果拷⻉和移动构造被⽣成，`Person` 类看起来就像这样：

```cpp
class Person
{
public:
    template<typename T>
    explicit Person(T&& n) :name(std::forward<T>(n)) {} // perfect forwarding ctor
    explicit Person(int idx); // int ctor
    Person(const Person& rhs); // copy ctor(complier-generated)
    Person(Person&& rhs); // move ctor (compiler-generated)
    ...
};
```

只有你在花了很多时间在编译器领域时，下⾯的⾏为才变得直观（译者注：这⾥意思就是这种实现会导致不符合⼈类直觉的结果，下⾯就解释了这种现象的原因）

```cpp
Person p("Nancy");
auto cloneOfP(p); // create new Person from p; this won't compile!
```

这⾥我们视图通过⼀个 `Person` 实例创建另⼀个 `Person` ，显然应该调⽤拷⻉构造即可（`p` 是左值，我们可以思考通过移动操作来消除拷⻉的开销）。但是这份代码不是调⽤拷⻉构造，而是调⽤完美转发构造。然后，该函数将尝试使⽤ `Person` 对象 `p` 初始化 `Person` 的 `std::string` 的数据成员，编译器就会报错。“为什么？”你可能会疑问，“为什么拷⻉构造会被完美转发构造替代？我们显然想拷⻉ `Person` 到另⼀个 `Person`”。确实我们是这样想的，但是编译器严格遵循 C++ 的规则，这⾥的相关规则就是控制对重载函数调⽤的解析规则。实例化之后，`Person` 类看起来是这样的：

```cpp
class Person {
public:
    explicit Person(Person& n) // instantiated from
    : name(std::forward<Person&>(n)) {} // perfect-forwarding
    // template
    explicit Person(int idx); // as before
    Person(const Person& rhs); // copy ctor (compiler-generated)
    ...
};
```

在 `auto cloneOfP(p)`; 语句中，`p` 被传递给拷⻉构造或者完美转发构造。调⽤拷⻉构造要求在 `p` 前加上 `const` 的约束，而调⽤完美转发构造不需要任何条件，所以编译器按照规则：采⽤最佳匹配，这⾥调⽤了完美转发的实例化的构造函数。如果我们将本例中的传递的参数改为 `const` 的，会得到完全不同的结果：

```cpp
const Person cp("Nancy");
auto cloneOfP(cp); // call copy constructor!
```

因为被拷⻉的对象是 `const` ，是拷⻉构造函数的精确匹配。虽然模板参数可以实例化为完全⼀样的函数签名：

```cpp
class Person {
public:
    explicit Person(const Person& n); // instantiated from template
    Person(const Person& rhs); // copy ctor(compiler-generated)
    ...
};
```

但是⽆所谓，因为重载规则规定当模板实例化函数和⾮模板函数（或者称为“正常”函数）匹配优先级相当时，优先使⽤“正常”函数。拷⻉构造函数（正常函数）因此胜过具有相同签名的模板实例化函数。

（如果你想知道为什么编译器在⽣成⼀个拷⻉构造函数时还会模板实例化⼀个相同签名的函数，参考 Item17）

当继承纳⼊考虑范围时，完美转发的构造函数与编译器⽣成的拷⻉、移动操作之间的交互会更加复杂。尤其是，派⽣类的拷⻉和移动操作会表现的⾮常奇怪。来看⼀下：

```cpp
class SpecialPerson: public Person {
public:
    SpecialPerson(const SpecialPerson& rhs) :Person(rhs)
    {...} // copy ctor; calls base class forwarding ctor!
    SpecialPerson(SpecialPerson&& rhs): Person(std::move(rhs))
    {...} // move ctor; calls base class forwarding ctor!
};
```

如同注释表⽰的，派⽣类的拷⻉和移动构造函数没有调⽤基类的拷⻉和移动构造函数，而是调⽤了基类的完美转发构造函数！为了理解原因，要知道派⽣类使⽤ `SpecialPerson` 作为参数传递给其基类，然后通过模板实例化和重载解析规则作⽤于基类。最终，代码⽆法编译，因为 `std::string` 没有 `SpecialPerson` 的构造函数。

我希望到⽬前为⽌，已经说服了你，如果可能的话，避免对通⽤引⽤的函数进⾏重载。但是，如果在通⽤引⽤上重载是糟糕的主意，那么如果需要可转发⼤多数类型的参数，但是对于某些类型⼜要特殊处理应该怎么办？存在多种办法。实际上，下⼀个Item，Item27专⻔来讨论这个问题，敬请阅读。

# 总结
- 对通⽤引⽤参数的函数进⾏重载，调⽤机会会⽐你期望的多得多
- 完美转发构造函数是糟糕的实现，因为对于 `non-const` 左值不会调⽤拷⻉构造而是完美转发构造，而且会劫持派⽣类对于基类的拷⻉和移动构造