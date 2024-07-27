---
title: Item 21 优先考虑使⽤ std::make_unique 和 std::make_shared 而⾮ new
tags: ["智能指针"]
categories: [Effective Modern C++]
description: 'std::make_unique 和 std::make_shared'
date: 2024-07-27 14:12:21
cover: https://images.pexels.com/photos/21533296/pexels-photo-21533296.jpeg
---

# 条款21：优先考虑使⽤ std::make_unique 和 std::make_shared 而⾮ new

让我们先对 `std::make_unique` 和 `std::make_shared` 做个铺垫。 `std::make_shared` 是 C++11 标准的⼀部分，但很可惜的是， `std::make_unique` 不是。它从 C++14 开始加⼊标准库。如果你在使⽤ C++11，不⽤担⼼，⼀个基础版本的 `std::make_unique` 是很容易⾃⼰写出的，如下：

```cpp
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params)
{
    return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
```

正如你看到的，`make_unique` 只是将它的参数完美转发到所要创建的对象的构造函数，从新产⽣的原始指针⾥⾯构造出 `std::unique_ptr`，并返回这个 `std::unique_ptr`。这种形式的函数不⽀持数组和⾃定义析构，但它给出了⼀个⽰范：只需⼀点努⼒就能写出你想要的 `make_uniqe` 函数。需要记住的是，不要把它放到 `std` 命名空间中，因为你可能并不希望在升级⼚家编译器到符合 C++14 标准的时候产⽣冲突。

`std::make_unique` 和 `std::make_shared` 有三个 `make functions`中的两个：接收抽象参数，完美转发到构造函数去动态分配⼀个对象，然后返回这个指向这个对象的指针。第三个 `make function` 是 `std::allocate_shared`.它和 `std::make_shared` ⼀样，除了第⼀个参数是⽤来动态分配内存的对象。

即使是对使⽤和不使⽤ make 函数创建智能指针的最简单⽐较，也揭⽰了为什么最好使⽤这些函数的第⼀个原因。例如：

```cpp
auto upw1(std::make_unique<Widget>()); // with make func
std::unique_ptr<Widget> upw2(new Widget); // without make func
auto spw1(std::make_shared<Widget>()); // with make func
std::shared_ptr<Widget> spw2(new Widget); // without make func
```

使⽤new的版本重复了类型，但是 `make function` 的版本没有。(译者注：这⾥⾼亮的是 `Widget`，⽤ new 的声明语句需要写2遍 `Widget`，`make function`只需要写⼀次) 重复写类型和软件⼯程⾥⾯⼀个关键原则相冲突：应该避免重复代码。源代码中的重复增加了编译的时间，会导致⽬标代码冗余，并且通常会让代码库使⽤更加困难。它经常演变成不⼀致的代码，而代码库中的不⼀致常常导致 bug。此外，打两次字⽐⼀次更费⼒，而且谁不喜欢减少打字负担？

第⼆个使⽤ `make function` 的原因和异常安全有关。假设我们有个函数按照某种优先级处理 `Widget`：

```cpp
void processWidget(std::shared_ptr<Widget> spw, int priority);
```

根据值传递 `std::shared_ptr` 可能看起来很可疑，但是 Item 41 解释了，如果 `processWidget` 总是复制 `std::shared_ptr` (例如，通过将其存储在已处理的 Widget 的数据结构中)，那么这可能是⼀个可复⽤的设计选择。

现在假设我们有⼀个函数来计算相关的优先级 `int computePriority();`

并且我们在调⽤ `processWidget` 时使⽤了 `new` 而不是 `std::make_shared`

```cpp
processWidget(std::shared_ptr<Widget>(new Widget), computePriority());      //potential resource leak!
```

**如注释所说，这段代码可能在 `new Widget` 时发⽣泄露**。为何？调⽤的代码和被调⽤的函数都⽤ `std::shared_ptr s`,且 `std::shared_ptr s` 就是设计出来防⽌泄露的。它们会在最后⼀个 `std::shared_ptr` 销毁时⾃动释放所指向的内存。如果每个⼈在每个地⽅都⽤ `std::shared_ptr_s`, 这段代码怎么会泄露呢？

答案和编译器将源码转换为⽬标代码有关。在运⾏时，⼀个函数的参数必须先被计算，才能被调⽤，所以在调⽤ `processWidget` 之前，必须执⾏以下操作，`processWidget` 才开始执⾏：
- 表达式 `new Widget` 必须计算，例如,⼀个 `Widget` 对象必须在堆上被创建
- 负责管理new出来指针的 `std::shared_ptr<Widget>` 构造函数必须被执⾏
- `computePriority()` 必须运⾏

编译器不需要按照执⾏顺序⽣成代码。“new Widget"必须在 `std::shared_ptr` 的构造函数被调⽤前执⾏，因为 `new` 出来的结果作为构造函数的参数，但 `compute Priority` 可能在这之前，之后，或者之间执⾏。也就是说，编译器可能按照这个执⾏顺序⽣成代码：
1. 执⾏ `new Widget`
2. 执⾏ `computePriority`
3. 运⾏ `std::shared_ptr` 构造函数\

如果按照这样⽣成代码，并且在运⾏是 `computePriority` 产⽣了异常，那么第⼀步动态分配的 `Widget` 就会泄露。因为它永远都不会被第三步的 `std::shared_ptr` 所管理了。

使⽤ std::make_shared 可以防⽌这种问题。调⽤代码看起来像是这样:

```cpp
processWidget(std::make_shared<Widget>(), computePriority());
```

在运⾏时，`std::make_shared` 和 `computePriority` 会先被调⽤。如果是 `std::make_shared`，在 `computePriority` 调⽤前，动态分配 `Widget` 的原始指针会安全的保存在作为返回值的 `std::shared_ptr` 中。如果 `compu tePriority` ⽣成⼀个异常，那么 `std::shared_ptr` 析构函数将确保管理的 `Widget` 被销毁。如果⾸先调⽤ `computePriority `并产⽣⼀个异常，那么 `std::make_shared` 将不会被调⽤，因此也就不需要担⼼ `new Widget` (会泄露)。

如果我们将 `std::shared_ptr`, `std::make_shared` 替换成 `std::unique_ptr`, `std::make_unique`, 同样的道理也适⽤。因此，在编写异常安全代码时，使⽤ `std::make_unique` 而不是new与使⽤ `std::make_shared` 同样重要。

`std::make_shared` 的⼀个特性(与直接使⽤ `new` 相⽐)得到了效率提升。使⽤ `std::make_shared` 允许编译器⽣成更小，更快的代码，并使⽤更简洁的数据结构。考虑以下对 `new` 的直接使⽤：

```cpp
std::shared_ptr<Widget> spw(new Widget);
```

显然，**这段代码需要进⾏内存分配，但它实际上执⾏了两次。**Item19 解释了每个 `std::shared_ptr` 指向⼀个控制块，其中包含被指向对象的引⽤计数。这个控制块的内存在 `std::shared_ptr` 构造函数中分配。因此，**直接使⽤ `new` 需要为 `Widget` 分配⼀次内存，为控制块分配再分配⼀次内存。**

如果使⽤ `std::make_shared` 代替：`auto spw = std::make_shared_ptr<Widget>()`; ⼀次分配⾜矣。**这是因为 `std::make_shared` 分配⼀块内存，同时容纳了 `Widget` 对象和控制块。**这种优化减少了程序的静态⼤小，因为代码只包含⼀个内存分配调⽤，并且它提⾼了可执⾏代码的速度，因为内存只分配⼀次。此外，使⽤ `std::make_shared` 避免了对控制块中的某些簿记信息的需要，潜在地减少了程序的总内存占⽤。

对于 `std::make_shared` 的效率分析同样适⽤于 `std::allocate_shared` ，因此 `std::make_shared` 的性能优势也扩展到了该函数。

更倾向于使⽤函数而不是直接使⽤ `new` 的争论⾮常激烈。尽管它们在软件⼯程、异常安全和效率⽅⾯具有优势，但本item的意⻅是，更倾向于使⽤ make函数，而不是完全依赖于它们。这是因为有些情况下它们不能或不应该被使⽤。

例如，没有 make函数允许指定定制的析构(⻅ item18 和 19)，但是 `std::unique_ptr` 和 `std::shared_ptr` 有构造函数这么做。给 `Widget` ⾃定义⼀个析构:

```cpp
auto widgetDeleter = [](Widget*){...};
```

使⽤ `new` 创建智能指针⾮常简单:

```cpp
std::unique_ptr<Widget, decltype(widgetDeleter)> upw(new Widget, widgetDeleter);

std::shared_ptr<Widget> spw(new Widget, widgetDeleter);
```

对于 make函数，没有办法做同样的事情。make函数第⼆个限制来⾃于其单⼀概念的句法细节。Item7 解释了，当构造函数重载，有 `std::initializer_list` 作为参数和不⽤其作为参数时，⽤⼤括号创建对象更倾向于使⽤ `std::initializer_list` 作为参数的构造函数，而⽤圆括号创建对象倾向于不⽤ `std::initializer_list` 作为参数的构造函数。make函数会将它们的参数完美转发给对象构造函数，但是它们是使⽤圆括号还是⼤括号？对某些类型，问题的答案会很不相同。例如，在这些调⽤中，

```cpp
auto upv = std::make_unique<std::vector<int>>(10, 20);
auto spv = std::make_shared<std::vector<int>>(10, 20);
```

⽣成的智能指针是否指向带有10个元素的 std::vector ，每个元素值为20，或指向带有两个元素的 `std::vector`，其中⼀个元素值 10，另⼀个为 20 ?或者结果是不确定的?

好消息是这并⾮不确定：两种调⽤都创建了 10 个元素，每个值为 20.这意味着在make函数中，完美转发使⽤圆括号，而不是⼤括号。坏消息是如果你想⽤⼤括号初始化指向的对象，你必须直接使⽤new。使⽤make函数需要能够完美转发⼤括号初始化，**但是，正如item31所说，⼤括号初始化⽆法完美转发。**但是，item30 介绍了⼀个变通的⽅法：使⽤ auto 类型推导从⼤括号初始化创建 `std::initializer_list` 对象(⻅Item 2)，然后将 `auto` 创建的对象传递给make函数。

```cpp
// create std::initializer_list
auto initList = { 10, 20 };
// create std::vector using std::initializer_list ctor
auto spv = std::make_shared<std::vector<int>>(initList);
```

对于 `std::unique_ptr`,只有这两种情景（定制删除和⼤括号初始化）使⽤ make函数有点问题。对于 `std::shared_ptr` 和它的 make函数，还有⾄少2个问题。都属于边界问题，但是⼀些开发者常碰到，你也可能是其中之⼀。

⼀些类重载了 `operator new` 和 `operator delete`。这些函数的存在意味着对这些类型的对象的全局内存分配和释放是不合常规的。设计这种定制类往往只会精确的分配、释放对象的⼤小。例如，`Widget`类的 `operator new` 和 `operator delete` 只会处理 `sizeof(Widget)` ⼤小的内存块的分配和释放。这种常识不太适⽤于 `std::shared_ptr` 对定制化分配(通过 `std::allocate_shared`)和释放(通过定制化 `deleters`)，因为 `std::allocate_shared` 需要的内存总⼤小不等于动态分配的对象⼤小，还需要再加上控制块⼤小。因此，适⽤ make函数去创建重载了 `operator new` 和 `operator delete`类的对象是个典型的糟糕想法。与直接使⽤new相⽐，`std::make_shared` 在⼤小和速度上的优势源于 `std::shared_ptr` 的控制块与指向的对象放在同⼀块内存中。当对象的引⽤计数降为 0，对象被销毁(析构函数被调⽤).但是，因为控制块和对象被放在同⼀块分配的内存块中，直到控制块的内存也被销毁，它占⽤的内存是不会被释放的。正如我说，控制块除了引⽤计数，还包含簿记信息。引⽤计数追踪有多少 `std::shared_ptr_s` 指向控制块，但控制块还有第⼆个计数，记录多少个 `std::weak_ptrs` 指向控制块。第⼆个引⽤计数就是`wea kcount`。当⼀个 `std::weak_ptr` 检测对象是否过期时(⻅item 19),它会检测指向的控制块中的引⽤计数(而不是 `weak count`)。如果引⽤计数是0(即对象没有 `std::shared_ptr` 再指向它，已经被销毁了)，`std::weak_ptr` 已经过期。否则就没过期。只要 `std::weak_ptrs` 引⽤⼀个控制块(即 `weak count` ⼤于零)，该控制块必须继续存在。只要控制块存在，包含它的内存就必须保持分配。通过 `std::shared_ptr make`函数分配的内存，直到最后⼀个 `std::shared_ptr` 和最后⼀个指向它的 `std::weak_ptr` 已被销毁，才会释放。如果对象类型⾮常⼤，而且销毁最后⼀个 `std::shared_ptr` 和销毁最后⼀个 `std::weak_ptr` 之间的时间很⻓，那么在销毁对象和释放它所占⽤的内存之间可能会出现延迟。

```cpp
class ReallyBigType { … };
// 通过std::make_shared创建⼀个⼤对象
auto pBigObj = std::make_shared<ReallyBigType>();

… // 创建 std::shared_ptrs 和 std::weak_ptrs
// 指向这个对象，使⽤它们

… // 最后⼀个 std::shared_ptr 在这销毁,
// 但 std::weak_ptrs 还在

… // 在这个阶段，原来分配给⼤对象的内存还分配着

… // 最后⼀个std::weak_ptr在这⾥销毁;
// 控制块和对象的内存被释放
```

直接只⽤ `new`，⼀旦最后⼀个 `std::shared_ptr` 被销毁，`ReallyBigType` 对象的内存就会被释放：

```cpp
class ReallyBigType { … };
//通过new创建特⼤对象
std::shared_ptr<ReallyBigType> pBigObj(new ReallyBigType);

… // 像之前⼀样，创建 std::shared_ptrs 和 std::weak_ptrs
// 指向这个对象，使⽤它们

… // 最后⼀个 std::shared_ptr 在这销毁,
// 但 std::weak_ptrs 还在
// memory for object is deallocated

… // 在这阶段，只有控制块的内存仍然保持分配

… // 最后⼀个std::weak_ptr在这⾥销毁;
// 控制块内存被释放
```

如果你发现⾃⼰处于不可能或不合适使⽤ `std::make_shared` 的情况下，你将想要保证⾃⼰不受我们之前看到的异常安全问题的影响。最好的⽅法是确保在直接使⽤new时，在⼀个不做其他事情的语句中，⽴即将结果传递到智能指针构造函数。这可以防⽌编译器⽣成的代码在使⽤new和调⽤管理新对象的智能指针的构造函数之间发⽣异常。

例如，考虑我们前⾯讨论过的 `processWidget` 函数，对其⾮异常安全调⽤的⼀个小修改。这⼀次，我们将指定⼀个⾃定义删除器:

```cpp
void processWidget(std::shared_ptr<Widget> spw, int priority);
void cusDel(Widget *ptr); // ⾃定义删除器
```

这是⾮异常安全调⽤:

```cpp
//和之前⼀样，潜在的内存泄露
processWidget(
    std::shared_ptr<Widget>(new Widget, cusDel),
    computePriority()
);
```
  
回想⼀下:如果 `computePriority` 在 `new Widget` 之后，而在 `std::shared_ptr` 构造函数之前调⽤，并且如果 `computePriority` 产⽣⼀个异常，那么动态分配的 `Widget` 将会泄漏。

这⾥使⽤⾃定义删除排除了对 `std::make_shared` 的使⽤，因此避免这个问题的⽅法是将 `Widget` 的分配和` std::shared_ptr` 的构造放⼊它们⾃⼰的语句中，然后使⽤得到的 `std::shared_ptr` 调⽤ `processWidget`。这是该技术的本质，不过，正如我们稍后将看到的，我们可以对其进⾏调整以提⾼其性能：
```cpp
std::shared_ptr<Widget> spw(new Widget, cusDel);
processWidget(spw, computePriority()); // 正确，但是没优化，⻅下
```

这是可⾏的，因为 `std::shared_ptr` 假定了传递给它的构造函数的原始指针的所有权，即使构造函数产⽣了⼀个异常。此例中，如果 `spw `的构造函数抛出异常(即⽆法为控制块动态分配内存)，仍然能够保证 `cusDel` 会在 `new Widget` 产⽣的指针上调⽤。

⼀个小小的性能问题是，在异常不安全调⽤中，我们将⼀个右值传递给 `processWidget`

```cpp
processWidget(
    std::shared_ptr<Widget>(new Widget, cusDel), // arg is rvalue
    computePriority()
);
```

但是在异常安全调⽤中，我们传递了左值

```cpp
processWidget(spw, computePriority()); //spw是左值
```

因为 `processWidget` 的 `std::shared_ptr` 参数是传值，传右值给构造函数只需要 `move` ，而传递左值需要拷⻉。对 `std::shared_ptr` 而⾔，这种区别是有意义的，因为拷⻉ `std::shared_ptr` 需要对引⽤计数原⼦加，`move` 则不需要对引⽤计数有操作。为了使异常安全代码达到异常不安全代码的性能⽔平，我们需要⽤ `std::move` 将 `spw` 转换为右值.

```cpp
processWidget(std::move(spw), computePriority());
```

总结：
- 和直接使⽤ `new` 相⽐，`make` 函数消除了代码重复，提⾼了异常安全性。对于 `std::make_shared` 和 `std::allocate_shared`，⽣成的代码更小更快。
- 不适合使⽤ make函数的情况包括需要指定⾃定义删除器和希望⽤⼤括号初始化
- 对于 `std::shared_ptr s`,  make函数可能不被建议的其他情况包括  
(1)有⾃定义内存管理的类和  
(2)特别关注内存的系统，⾮常⼤的对象，以及 `std::weak_ptr s` ⽐对应的 `std::shared_ptr s` 活得更久