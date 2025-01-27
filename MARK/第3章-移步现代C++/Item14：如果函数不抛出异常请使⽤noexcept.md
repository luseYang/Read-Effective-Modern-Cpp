---
title: Item 14 如果函数不抛出异常请使⽤noexcept
tags: ["noexcept", "现代C++"]
categories: [Effective Modern C++]
description: '不抛出异常请使⽤noexcept'
date: 2024-07-22 17:08:21
cover: https://images.pexels.com/photos/4400679/pexels-photo-4400679.jpeg
---

# 条款 14:如果函数不抛出异常请使⽤noexcept

在 C++98 中，你不得不写出函数可能抛出的异常类型，若果函数实现有所改变，异常说明也可能需要修改。 改变异常说明会影响客户端代码，因为
调⽤者可能依赖原版本的异常说明。编译器不会为函数实现，异常说明和客⼾端代码中提供⼀致性保障。**⼤多数程序员最终都认为不值得为C++98的异常说明如此⿇烦。**

在C++11标准化过程中，⼤家⼀致认为异常说明真正有⽤的信息是⼀个函数是否会抛出异常。⾮⿊即⽩，⼀个函数可能抛异常，或者不会。这种 **"可能-绝不"** 的⼆元论构成了 C++11 异常说的基础，从根本上改变了 C++98 的异常说明。在 C++11 中，⽆条件的 `noexcept` 保证函数不会抛出任何异常。

关于⼀个函数是否已经声明为 `noexcept` 是接口设计的事。函数的异常抛出⾏为是客⼾端代码最关⼼的。调⽤者可以查看函数是否声明为 `noexcept`，这个可以影响到调⽤代码的异常安全性和效率。

就其本⾝而⾔，函数是否为 `noexcept` 和成员函数是否 `const` ⼀样重要。如果知道这个函数不会抛异常就加上 `noexcept` 是简单天真的接口说明。

这里还有给不抛异常的函数加上 `noexcept` 的动机：**它允许编译器⽣成更好的⽬标代码**。要想知道为什么，我们需要了解 C++98 和 C++11 指明⼀个函数不抛异常的⽅式。考虑⼀个函数 `f`，它允许调⽤者永远不会受到⼀个异常。两种表达⽅式如下：

```cpp
int f(int) throw();   // C++98
int f(int) noexcept;    // C++11
```

如果在运⾏时，`f` 出现⼀个异常，那么就和 `f` 的异常说明冲突了。在 C++98 的异常说明中，调⽤栈会展开⾄ `f` 的调⽤者，⼀些不合适的动作⽐如程序终⽌也会发⽣。C++11 异常说明的运⾏时⾏为明显不同：调⽤栈只是可能在程序终⽌前展开。

展开调⽤栈和可能展开调⽤栈两者对于代码⽣成（code generation）有⾮常⼤的影响。在⼀个 `noexcept` 函数中，当异常传播到函数外，优化器不需要保证运⾏时栈的可展开状态，也不需要保证 `noexcept` 函数中的对象按照构造的反序析构。而`"throw()"`标注的异常声明缺少这样的优化灵活性，它和没加⼀样。可以总结⼀下：

```cpp
RetType function(params) noexcept; // 极尽所能优化
RetType function(params) throw(); // 较少优化
RetType function(params); // 较少优化
```

这是⼀个充分的理由使得你当知道它不抛异常时加上 `noexcept`。

还有⼀些函数让这个案例更充分。移动操作是绝佳的例⼦。假如你有⼀份C++98代码，⾥⾯⽤到了 `std::vector<Widget>`。`Widget` 通过 `push_back` ⼀次⼜⼀次的添加进 `std::vector`：

```cpp
std::vector<Widget> vw;
…
Widget w;
… // work with w
vw.push_back(w); // add w to vw
```

假设这个代码能正常⼯作，你也⽆意修改为 C++11 ⻛格。但是你确实想要 C++11 移动语义带来的性能优势，毕竟这⾥的类型是可以移动的(move-enabled types)。因此你需要确保 `Widget` 有移动操作，可以⼿写代码也可以让编译器⾃动⽣成，当然前提是⾃动⽣成的条件能满⾜（参⻅Item 17）。

当新元素添加到 `std::vector` ， `std::vector` 可能没地⽅放它，换句话说，`std::vector` 的⼤小(size)等于它的容量(capacity)。这时候，`std::vector` 会分配⼀⽚的新的⼤块内存⽤于存放，然后将元素从已经存在的内存移动到新内存。在 C++98 中，移动是通过复制⽼内存区的每⼀个元素到新内存区完成的，然后⽼内存区的每个元素发⽣析构。

这种方法使得 `push_back` 可以提供很强的异常安全保证：**如果复制元素期间抛出异常。`std::vector` 的状态保持不变，因为老内存元素析构必须建立在他们已经成功复制到新内存的前提下。**

在 C++11 中，⼀个很⾃然的优化就是将上述复制操作替换为移动操作。但不幸运的是，这会破坏 `push_back` 的异常安全。**如果 n 个元素已经从⽼内存移动到了新内存区，但异常在移动第 n+1 个元素时抛出，那么 `push_back` 操作就不能完成。但是原始的 `std::vector` 已经被修改：有 n 个元素已经移动走了。**恢复 `std::vector` ⾄原始状态也不太可能，因为从新内存移动到⽼内存本⾝⼜可能引发异常。

这是个很严重的问题，因为⽼代码可能依赖于 `push_back` 提供的强烈的异常安全保证。因此，C++11 版的实现不能简单的将 `push_back` ⾥⾯的复制操作替换为移动操作，除⾮知晓移动操作绝不抛异常，这时复制替换为移动就是安全的，唯⼀的副作⽤就是性能得到提升。

`std::vector::push_back` 受益于 **"如果可以就移动，如果必要则复制"策略**，并且它不是标准库中唯⼀采取该策略的函数。C++98 中还有⼀些函数如 `std::vector::reverse` , `std::deque::insert` 等也受益于这种强异常保证。对于这个函数**只有在知晓移动不抛异常的情况下⽤ C++11 的 `move`替换 C++98 的 copy 才是安全的**。

但是如何知道⼀个函数中的移动操作是否产⽣异常？答案很明显：它检查是否声明 `noexcept`。

`swap` 函数是 `noexcept` 的绝佳⽤地。`swap` 是 STL 算法实现的⼀个关键组件，它也常⽤于拷⻉运算符重载中。它的⼴泛使⽤意味着对其施加不抛异常的优化是⾮常有价值的。有趣的是，标准库的 `swap` 是否 `noexcept` 有时依赖于⽤⼾定义的 `swap` 是否 `noexcept`。⽐如，数组和 `std::pair` 的 `swap` 声明如下：

```cpp
template<calss T, size_t N>
void swap(T(&a)[N], T(&b)[N]) noexcept(noexpect(swap(*a, *b)));

template<class T1, class T2>
struct pair{
    ...
    void swap(pair& p) noexcept(noexcept(swap(first, p.first)) && noexcept(swap(second, p.second)));
    ...
};
```

这些函数视情况 `noexcept` ：它们是否 `noexcept` 依赖于 `noexcept` 声明中的表达式是否 `noexcept`。假设有两个 `Widget` 数组，不抛异常的交换数组前提是数组中的元素交换不抛异常。对于 `Widget` 的交换是否 `noexcept` 决定了对于 `Widget` 数组的交换是否 `noexcept`，类似的，交换两个存放 `Widget` 的 `std::pair` 是否 `noexcept` 依赖于 `Widget` 的交换是否 `noexcept`。事实上交换⾼层次数据结构是否 `noexcept` 取决于它的构成部分的那些低层次数据结构是否异常，这激励你只要可以就提供`noexcept swap`函数,因为如果你的函数不提供 `noexcept` 保证，其它依赖你的⾼层次 `swap` 就不能保证 `noexcept`

现在，我希望你能为 `noexcept` 提供的优化机会感到⾼兴，同时我还得让你缓⼀缓别太⾼兴了。优化很重要，但是正确性更重要。我在这个条款的开头提到 `noexcept` 是函数接口的⼀部分，所以仅当你保证⼀个函数实现在⻓时间内不会抛出异常时才声明 `noexcept`。如果你声明⼀个函数为noexcept，但随即⼜后悔了，你没有选择。你只能从函数声明中移除 `noexcept`（即改变它的接口），这理所当然会影响客⼾端代码。你可以改变实现使得这个异常可以避免，再保留原版本（不正确的）异常说明。如果你这么做，程序将会在异常离开这个函数时终⽌。或者你可以重新设计既有实现，改变实现后再考虑你希望它是什么样⼦。这些选择都不尽⼈意。

这个问题的本质是实际上⼤多数函数都是异常中⽴（exception neutral）的。简单来说就是这些函数自身不会抛出异常，但是它们内部的调用可能抛出异常。此时，异常中⽴函数允许那些抛出异常的函数在调⽤链上更进⼀步直到遇到异常处理程序，而不是就地终⽌。异常中⽴函数决不应该声明为 `noexcept`，因为它们可能抛出那种"让它们过吧"的异常（也就是说在当前这个函数内不处理异常，但是⼜不⽴即终⽌程序，而是让调⽤这个函数的函数处理），因此大多数函数都不应该被指定为 `noexcept`。

对于⼀些函数，使其成为 `noexcept` 是很重要的，它们应当默认如是。在 C++98 构造函数和析构函数抛出异常是糟糕的代码设计——不管是⽤⼾定义的还是编译器⽣成的构造析构都是 `noexcept`。因此它们不需要声明 `noexcept`。析构函数⾮隐式 `noexcept` 的情况仅当类的数据成员明确声明它的析构函数可能抛出异常（即，声明 `noexcept(false)` ）。这种析构函数不常⻅，标准库⾥⾯没有。**如果⼀个对象的析构函数可能被标准库使⽤，析构函数⼜可能抛异常，那么程序的⾏为是未定义的。**

⼀些库接口设计者会区分有宽泛契约(wild contracts)和严格契约(narrow contracts)的函数。有宽泛契约的函数没有前置条件。这种函数不管程序状态如何都能调⽤，它对调⽤者传来的实参不设约束。宽泛契约的函数决不表现出未定义⾏为。反之，没有宽泛契约的函数就有严格契约。对于这些函数，如果违反前置条件，结果将会是未定义的。

如果你写了⼀个有宽泛契约的函数并且你知道它不会抛异常，那么遵循这个条款给它声明⼀个 `noexcept` 是很容易的。

对于严格契约的函数，情况就有点微妙了。举个例⼦，假如你在写⼀个参数为 `std::string` 的函数 `f`，并且这个函数 `f` 很⾃然的决不引发异常。这就在建议我们f应该被声明为 `noexcept`。

现在假如 `f` 有⼀个前置条件：类型为 `std::string` 的参数的⻓度不能超过32个字符。如果现在调⽤ `f` 并传给它⼀个⼤于 32 字符的参数，函数⾏为将是未定义的，因为违反了 （口头/⽂档）定义的前置条件，导致了未定义⾏为。

`f` 没有义务去检查前置条件，它假设这些前置条件都是满⾜的。

即使有前置条件，将f声明为noexcept似乎也是合适的：

```cpp
void f(const std::string& s) noexcept; // 前置条件：s.length() <= 32
```

`f` 的实现者决定在函数⾥⾯检查前置条件冲突。虽然检查是没有必要的，但是也没禁⽌这么做。另外在系统测试时，检查前置条件可能就是有⽤的了。debug ⼀个抛出的异常⼀般都⽐跟踪未定义⾏为起因更容易。那么怎么报告前置条件冲突使得测试⼯具或客⼾端错误处理程序能检测到它呢？简单直接的做法是抛出 "precondition was
violated" 异常，但是如果 `f` 声明了 `noexcept`，这就⾏不通了；抛出⼀个异常会导致程序终⽌。

因为这个原因，区分严格/宽泛契约库设计者⼀般会将 `noexcept` 留给宽泛契约函数。

考虑下⾯的代码，它是完全正确的：

```cpp
void setup(); // 函数定义另在⼀处
void cleanup();
void doWork() noexcept
{
    setup(); // 前置设置
    … // 真实⼯作
    cleanup(); // 执⾏后置清理
}
```

这⾥，`doWork` 声明为 `noexcept`，即使它调⽤了⾮ `noexcept` 函数 `setup` 和 `cleanup`。看起来有点⽭盾，其实可以猜想 `setup` 和 `cleanup` 在⽂档上写明了它们决不抛出异常，即使它们没有写上 `noexcept`。⾄于为什么明明不抛异常却不写 `noexcept` 也是有合理原因的。⽐如，它们可能是⽤ C 写的库函数的⼀部分。（即使⼀些函数从 C 标准库移动到了 std 命名空间，也可能缺少异常规范，`std::strlen` 就是⼀个例⼦，它没有声明 `noexcept`）。或者它们可能是 C++98 库的⼀部分，它们不使⽤ C++98 异常规范的函数的⼀部分，到了 C++11 还没有修订。因为有很多合理原因解释为什么 `noexcept` 依赖于缺少 `noexcept` 保证的函数，所以 C++ 允许这些代码，编译器⼀般也不会给出 warnigns。

总结：
- `noexcept` 是函数接口的⼀部分，这意味着调⽤者会依赖它、
- `noexcept` 函数较之于⾮ `noexcept` 函数更容易优化
- `noexcept` 对于移动语义，`swap`，内存释放函数和析构函数⾮常有⽤
- ⼤多数函数是异常中⽴的（译注：可能抛也可能不抛异常）而不是 `noexcept`


