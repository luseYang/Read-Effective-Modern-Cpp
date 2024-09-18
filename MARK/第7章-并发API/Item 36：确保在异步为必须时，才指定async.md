---
title: Item 35 优先基于任务编程而不是基于线程
tags: ["并发编程", "多线程"]
categories: [Effective Modern C++]
description: '优先基于任务编程而不是基于线程'
date: 2024-08-26 20:18:21
cover: https://images.pexels.com/photos/26733349/pexels-photo-26733349.jpeg
---

# 前言

当你调⽤ `std::async` 执⾏函数时（或者其他可调⽤对象），你通常希望异步执⾏函数。但是这并不⼀定是你想要 `std::async` 执⾏的操作。你确实通过 `std::async` launch policy（译者注：这⾥没有翻译）要求执⾏函数，有两种标准policy，都通过 `std::launch` 域的枚举类型表⽰（参⻅ Item10 关于枚举的更多细节）。假定⼀个函数 `f` 传给 `std::async` 来执⾏：

- `std::launch::async` 的 launch policy 意味着f必须异步执⾏，即在不同的线程
- `std::launch::deferred` 的 launch policy 意味着f仅仅在当调⽤ `get` 或者 `wait` 要求 `std::async` 的返回值时才执⾏。这表⽰f推迟到被求值才延迟执⾏（译者注：异步与并发是两个不同概念，这⾥侧重于惰性求值）。当 `get` 或 `wait` 被调⽤，`f` 会同步执⾏，即调⽤⽅停⽌直到f运⾏结束。如果 `get` 和 `wait` 都没有被调⽤，`f` 将不会被执⾏

有趣的是，`std::async` 的默认 launch policy 是以上两种都不是。相反，是求或在⼀起的。下⾯的两种调⽤含义相同

```cpp
auto fut1 = std::async(f); // run f using default launch policy
auto fut2 = std::async(std::launch::async | std::launch::deferred, f); // run f either async or defered
```

因此默认策略允许 `f` 异步或者同步执⾏。如同 Item35 中指出，这种灵活性允许 `std::async` 和标准库的线程管理组件（负责线程的创建或销毁）避免超载。这就是使⽤ `std::async` 并发编程如此⽅便的原因。

但是，使⽤默认启动策略的 `std::async` 也有⼀些有趣的影响。给定⼀个线程 `t` 执⾏此语句：

```cpp
auto fut = std::async(f); // run f using default launch policy
```

- ⽆法预测 `f` 是否会与 `t` 同时运⾏，因为 `f` 可能被安排延迟运⾏

- ⽆法预测 `f` 是否会在调⽤ `get` 或 `wait` 的线程上执⾏。如果那个线程是 `t`，含义就是⽆法预测 `f` 是否也在线程 `t` 上执⾏

- ⽆法预测 `f` 是否执⾏，因为不能确保 `get` 或者 `wait` 会被调⽤

默认启动策略的调度灵活性导致使⽤线程本地变量⽐较⿇烦，因为这意味着如果 `f` 读写了线程本地存储（thread-local storage, TLS），不可能预测到哪个线程的本地变量被访问：

```cpp
auto fut = std::async(f); // TLS for f possibly for independent thread, but possibly for thread invoking get or wait on fut
```

还会影响到基于超时机制的wait循环，因为在 `task` 的 `wait_for` 或者 `wait_until` 调⽤中（参⻅ Item35）会产⽣延迟求值（`std::launch::deferred`）。意味着，以下循环看似应该终⽌，但是实际上永远运⾏：

```cpp
using namespace std::literals; // for C++14 duration suffixes; see Item 34
void f()
{
std::this_thread::sleep_for(1s);
}
auto fut = std::async(f);
while (fut.wait_for(100ms) != std::future_status::ready)
{ // loop until f has finished running... which may never happen!
...
}
```

如果 `f` 与调⽤ `std::async` 的线程同时运⾏（即，如果为 `f` 选择的启动策略是 `std::launch::async`），这⾥没有问题（假定f最终执⾏完毕），但是如果f是延迟执⾏，`fut.wait_for` 将总是返回 `std::future_status::deferred`。这表⽰循环会永远执⾏下去。

这种错误很容易在开发和单元测试中忽略，因为它可能在负载过⾼时才能显现出来。当机器负载过重时，任务推迟执⾏才最有可能发⽣。毕竟，如果硬件没有超载，没有理由不安排任务并发执⾏。

修复也是很简单的：只需要检查与 `std::async` 的 `future` 是否被延迟执⾏即可，那样就会避免进⼊⽆限循环。不幸的是，没有直接的⽅法来查看 `future` 是否被延迟执⾏。相反，你必须调⽤⼀个超时函数----⽐如 `wait_for` 这种函数。在这个逻辑中，你不想等待任何事，只想查看返回值是否

`std::future_status::deferred`，如果是就使⽤ 0 调⽤ `wait_for` 来终⽌循环。

```cpp
auto fut = std::async(f);
if (fut.wait_for(0s) == std::future_status::deferred) { // if task is deferred
    ... // use wait or get on fut to call f synchronously
}

else { // task isn't deferred
    while(fut.wait_for(100ms) != std::future_status::ready) { // infinite loop not possible(assuming f finished)
    ... // task is neither deferred nor ready, so do concurrent word until it's ready
    }
}
```

这些各种考虑的结果就是，只要满⾜以下条件，`std::async` 的默认启动策略就可以使⽤：
- `task` 不需要和执⾏ `get` or `wait` 的线程并⾏执⾏
- 不会读写线程的线程本地变量
- 可以保证在 `std::async` 返回的将来会调⽤ `get` or `wait` ，或者该任务可能永远不会执⾏是可以接受的
- 使⽤ `wait_for` or `wait_until` 编码时考虑 `deferred` 状态

如果上述条件任何⼀个都满⾜不了，你可能想要保证 `std::async` 的任务真正的异步执⾏。进⾏此操作的⽅法是调⽤时，将 `std::launch::async` 作为第⼀个参数传递：

```cpp
auto fut = std::async(std::launch::async, f); // launch f asynchronously
```

事实上，具有类似 `std::async` ⾏为的函数，但是会⾃动使⽤ `std::launch::async` 作为启动策略的⼯具也是很容易编写的，C++11 版本如下：

```cpp
template<typename F, typename... Ts>
inline std::future<typename std::result_of<F(Ts...)>::type>
reallyAsync(F&& f, Ts&&... params)
{
    return std::async(std::launch::async, std::forward<F>(f), std::forward<Ts>(params)...);
}
```

这个函数接受⼀个可调⽤对象和 0 或多个参数 `params` 然后完美转发（参⻅Item25）给 `std::async`，使⽤ `std::launch::async` 作为启动参数。就像 `std::async` ⼀样，返回 `std::future` 类型。确定结果的类型很容易，因为类型特征 `std::result_of` 可以提供（参⻅ Item9 关于类型特征的详细表述）。

`reallyAsync` 就像 `std::async` ⼀样使⽤：

```cpp
auto fut = reallyAsync(f);
```

在 C++14 中，返回类型的推导能⼒可以简化函数的定义：

```cpp
template<typename f, typename... Ts>
inline auto reallyAsync(F&& f, Ts&&... params)
{
    return std::async(std::launch::async, std::forward<T>(f), std::forward<Ts>(params)...);
}
```

这个版本清楚表明，`reallyAsync` 除了使⽤ `std::launch::async` 启动策略之外什么也没有做。

# 总结

- `std::async` 的默认启动策略是异步或者同步的
- 灵活性导致访问 `thread_locals` 的不确定性，隐含了 `task` 可能不会被执⾏的意思，会影响程序基于 `wait` 的超时逻辑
- 只有确实异步时才指定 `std::launch::async`

