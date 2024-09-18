---
title: Item 37 使 std::threads 在所有线程上不可连接路径
tags: ["并发编程", "多线程"]
categories: [Effective Modern C++]
description: '使 std::threads 在所有线程上不可连接路径'
date: 2024-08-26 20:18:21
cover: https://images.pexels.com/photos/27795729/pexels-photo-27795729.jpeg
---

# 前言

每个 `std::thread` 对象处于两个状态之⼀：`joinable` or `unjoinable`。joinable状态的 `std::thread` 对应于正在运⾏或者可能正在运⾏的异步执⾏线程。⽐如，⼀个blocked或者等待调度的 `std::thread` 是 joinable，已运⾏结束的 `std::thread` 也可以认为是 `joinable`.

`unjoinable` 的 `std::thread` 对象⽐如：

- `Default-constructed std::threads`。这种 `std::thread` 没有函数执⾏，因此⽆法绑定到具体的
线程上
- 已经被 `moved` 的 `std::thread` 对象。`move` 的结果就是将 `std::thread` 对应的线程所有权转移给
另⼀个 `std::thread`
- 已经 joined 的 `std::thread`。在 join 之后，`std::thread` 执⾏结束，不再对应于具体的线程
- 已经 detached 的 `std::thread`。detach 断开了 `std::thread` 与线程之间的连接

（译者注：`std::thread` 可以视作状态保存的对象，保存的状态可能也包括可调⽤对象，有没有具体的线程承载就是有没有连接）

`std::thread` 的可连接性如此重要的原因之⼀就是当连接状态的析构函数被调⽤，执⾏逻辑被终⽌。⽐如，假定有⼀个函数 `doWork`，执⾏过滤函数 `filter`，接收⼀个参数 `maxVal`。`doWork` 检查是否满⾜计算所需的条件，然后通过使⽤0到maxVal之间的所有值过滤计算。如果进⾏过滤⾮常耗时，并且确定 `doWork` 条件是否满⾜也很耗时，则将两件事并发计算是很合理的。

我们希望为此采⽤基于任务的设计（参与 Item 35），但是假设我们希望设置做过滤线程的优先级。Item 35 阐释了需要线程的基本句柄，只能通过 `std::thread` 的API来完成；基于任务的 API（⽐如 futures）做不到。所以最终采⽤基于 `std::thread` 而不是基于任务

代码如下：

```cpp
constexpr auto tenMillion = 10000000; // see Item 15 for constexpr
bool doWork(std::function<bool(int)> filter, int maxVal = tenMillion) // return whether computation was performed; see Item2 for std::function
{
    std::vector<int> goodVals;
    std::thread t([&filter, maxVal, &goodVals]
        {
            for (auto i = 0; i <= maxVal; ++i)
        {
            if (filter(i)) goodVals.push_back(i);
        }
        });
    auto nh = t.native_handle(); // use t's native handle to set t's priority
    ...
    if (conditionsAreStatisfied()) {
        t.join(); // let t finish
        performComputation(goodVals); // computation was performed
        return true;
    }
    return false; // computation was not performed
}
```

在解释这份代码为什么有问题之前，看⼀下 `tenMillion` 的初始化可以在 C++14 中更加易读，通过单引号分隔数字：

```cpp
constexpr auto tenMillion = 10'000'000; // C++14
```

还要指出，在开始运⾏之后设置 `t` 的优先级就像把⻢放出去之后再关上⻢厩⻔⼀样（译者注：太晚了）。更好的设计是在t为挂起状态时设置优先级（这样可以在执⾏任何计算前调整优先级），但是我不想你为这份代码考虑这个而分⼼。如果你感兴趣代码中忽略的部分，可以转到 Item 39，那个 Item 告诉你如何以挂起状态开始线程。

返回 `doWork`。如果 `conditionsAreSatisfied()` 返回真，没什么问题，但是如果返回假或者抛出异常，`std::thread` 类型的 `t` 在 `doWork` 结束时会调⽤ `t` 的析构器。这造成程序执⾏中⽌。

你可能会想，为什么 `std::thread` 析构的⾏为是这样的，那是因为另外两种显而易⻅的⽅式更糟：

- **隐式 join**。这种情况下，`std::thread` 的析构函数将等待其底层的异步执⾏线程完成。这听起来是合理的，但是可能会导致性能异常，而且难以追踪。⽐如，如果 `conditonAreStatisfied()` 已经返回了假，`doWork` 继续等待过滤器应⽤于所有值就很违反直觉。

- **隐式 detach**。这种情况下，`std::thread` 析构函数会分离其底层的线程。线程继续运⾏。听起来⽐ join 的⽅式好，但是可能导致更严重的调试问题。⽐如，在 `doWork` 中，`goodVals` 是通过引⽤捕获的局部变量。可能会被 lambda 修改。假定，lambda 的执⾏时异步的，`conditionsAreStatisfied()` 返回假。这时，`doWork` 返回，同时局部变量 `goodVals` 被销毁。堆栈被弹出，并在 `doWork` 的调⽤点继续执⾏线程

某个调⽤点之后的语句有时会进⾏其他函数调⽤，并且⾄少⼀个这样的调⽤可能会占⽤曾经被 `doWork` 使⽤的堆栈位置。我们称为 `f`，当 `f` 运⾏时，`doWork` 启动的 lambda 仍在继续运⾏。该
lambda 可以在堆栈内存中调⽤ `push_back`，该内存曾是 `goodVals`，位于 `doWork` 曾经的堆栈位置。这意味着对 `f` 来说，内存被修改了，想象⼀下调试的时候痛苦

标准委员会认为，销毁连接中的线程如此可怕以⾄于实际上禁⽌了它（通过指定销毁连接中的线程导致程序终⽌）

这使你有责任确保使⽤ `std::thread` 对象时，在所有的路径上最终都是 `unjoinable` 的。但是覆盖每条路径可能很复杂，可能包括 `return, continue, break, goto or exception` ，有太多可能的路径。

每当你想每条路径的块之外执⾏某种操作，最通⽤的⽅式就是将该操作放⼊本地对象的析构函数中。这些对象称为 RAII 对象，通过RAII类来实例化。（RAII全称为 Resource Acquisition Is Initialization）。RAII 类在标准库中很常⻅。⽐如STL容器，智能指针，`std::fstream` 类等。但是标准库没有 RAII 的 `std::thread` 类，可能是因为标准委员会拒绝将 `join` 和 `detach` 作为默认选项，不知道应该怎么样完成 RAII。

幸运的是，完成⾃⾏实现的类并不难。⽐如，下⾯的类实现允许调⽤者指定析构函数 `join` 或者 `detach`：

```cpp
class ThreadRAII {
public:
    enum class DtorAction{ join, detach }; // see Item 10 for enum class info
    ThreadRAII(std::thread&& t, DtorAction a): action(a), t(std::move(t)) {} // in dtor, take action a on t
    ~ThreadRAII()
    {
        if (t.joinable()) {
            if (action == DtorAction::join) {
                t.join();
            } 
            else {
                t.detach();
            }
        }
    }
    std::thread& get() { return t; } // see below

private:
    DtorAction action;
    std::thread t;
};
```

我希望这段代码是不⾔⾃明的，但是下⾯⼏点说明可能会有所帮助：

- 构造器只接受 `std::thread` 右值，因为我们想要 move `std::thread` 对象给 ThreadRAII （再次强调，`std::thread` 不可以复制）

- 构造器的参数顺序设计的符合调⽤者直觉（⾸先传递 `std::thread`，然后选择析构执⾏的动作），但是成员初始化列表设计的匹配成员声明的顺序。将 `std::thread` 成员放在声明最后。在这个类中，这个顺序没什么特别之处，调整为其他顺序也没有问题，但是通常，可能⼀个成员的初始化依赖于另⼀个，因为 `std::thread` 对象可能会在初始化结束后就⽴即执⾏了，所以在最后声明是⼀个好习惯。这样就能保证⼀旦构造结束，所有数据成员都初始化完毕可以安全的异步绑定线程执⾏

- ThreadRAII 提供了 `get` 函数访问内部的 `std::thread` 对象。这类似于标准智能指针提供的 `get` 函数，可以提供访问原始指针的⼊口。提供 `get` 函数避免了 ThreadRAII 复制完整 `std::thread` 接口的需要，因为着 ThreadRAII 可以在需要 `std::thread` 上下⽂的环境中使⽤

在 ThreadRAII 析构函数调⽤ `std::thread` 对象t的成员函数之前，检查 `t` 是否 `joinable`。这是必须的，因为在 `unjoinbale` 的 `std::thread` 上调⽤ `join` or `detach` 会导致未定义⾏为。客⼾端可能会构造⼀个 `std::thread t`，然后通过 `t` 构造⼀个 ThreadRAII ，使⽤ `get` 获取 `t` ，然后移动 `t`，或者调⽤ `join` or `detach`，每⼀个操作都使得 `t` 变为 `unjoinable`

如果你担⼼下⾯这段代码

```cpp
if (t.joinable()) {
    if (action == DtorAction::join) {
        t.join();
    } 
    else {
        t.detach();
    }
}
```

存在竞争，因为在 `t.joinable()` 和 `t.join` or `t.detach` 执⾏中间，可能有其他线程改变了 `t` 为 `unjoinable`，你的态度很好，但是这个担⼼不必要。`std::thread` 只有⾃⼰可以改变 `joinable` or `unjoinable` 的状态。在 ThreadRAII 的析构函数中被调⽤时，其他线程不可能做成员函数的调⽤。如果同时进⾏调⽤，那肯定是有竞争的，但是不在析构函数中，是在客⼾端代码中试图同时在⼀个对象上调⽤两个成员函数（析构函数和其他函数）。通常，仅当所有都为const成员函数时，在⼀个对象同时调⽤两个成员函数才是安全的。

在 `doWork` 的例⼦上使⽤ ThreadRAII 的代码如下：

```cpp
bool doWork(std::function<bool(int)> filter, int maxVal = tenMillion)
{
    std::vector<int> goodVals;
    ThreadRAII t(std::thread([&filter, maxVal, &goodVals] {
        for (auto i = 0; i <= maxVal; ++i) {
            if (filter(i)) goodVals.push_back(i);
        }
        }),
        ThreadRAII::DtorAction::join
    );

    auto nh = t.get().native_handle();
    ...
    if (conditonsAreStatisfied()) {
        t.get().join();
        performComputation(goodVals);
        return true;
    }
    return false;
}
```

这份代码中，我们选择在 ThreadRAII 的析构函数中异步执⾏ join 的动作，因为我们先前分析中，`detach` 可能导致⾮常难缠的 bug。我们之前也分析了 join 可能会导致性能异常（坦率说，也可能调试困难），但是在未定义⾏为（`detach` 导致），程序终⽌（`std::thread` 默认导致），或者性能异常之间选择⼀个后果，可能性能异常是最好的那个。

哎，Item39 表明了使⽤ ThreadRAII 来保证在 `std::thread` 的析构时执⾏ `join` 有时可能不仅导致程序性能异常，还可能导致程序挂起。“适当”的解决⽅案是此类程序应该和异步执⾏的 lambda 通信，告诉它不需要执⾏了，可以直接返回，但是C++11中不⽀持可中断线程。可以⾃⾏实现，但是这不是本书讨论的主题。（译者注：关于这⼀点，C++ Concurrency in Action 的section 9.2 中有详细讨论，也有中⽂版出版）

Item17 说明因为 ThreadRAII 声明了⼀个析构函数，因此不会有编译器⽣成移动操作，但是没有理由 ThreadRAII 对象不能移动。所以需要我们显式声明来告诉编译器⾃动⽣成：

```cpp
class ThreadRAII {
public:
    enum class DtorAction{ join, detach }; // see Item 10 for enum class info
    ThreadRAII(std::thread&& t, DtorAction a): action(a), t(std::move(t)) {} // in
    dtor, take action a on t
    ~ThreadRAII()
    {
        if (t.joinable()) {
            if (action == DtorAction::join) {
                t.join();
            } 
            else {
                t.detach();
            }
        }
    }
    ThreadRAII(ThreadRAII&&) = default;
    ThreadRAII& operator=(ThreadRAII&&) = default;
    std::thread& get() { return t; } // see below
private:
    DtorAction action;
    std::thread t;
};
```

# 总结

- 在所有路径上保证 `thread` 最终是 `unjoinable`
- 析构时 `join` 会导致难以调试的性能异常问题
- 析构时 `detach` 会导致难以调试的未定义⾏为
- 声明类数据成员时，最后声明 `std::thread` 类型成员