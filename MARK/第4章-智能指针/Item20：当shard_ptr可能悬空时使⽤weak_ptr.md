---
title: Item 20 当std::shard_ptr可能悬空时使⽤std::weak_ptr
tags: ["智能指针", "std::weak_ptr"]
categories: [Effective Modern C++]
description: 'std::weak_ptr'
date: 2024-07-26 15:37:21
cover: https://images.pexels.com/photos/18609055/pexels-photo-18609055.jpeg
---

# 条款20：当std::shard_ptr可能悬空时使⽤std::weak_ptr

如果有⼀个像 `std::shared_ptr` 的指针但是不参与资源所有权共享的指针是很⽅便的。换句话说，是⼀个类似 `std::shared_ptr` 但不影响对象引⽤计数的指针。这种类型的智能指针必须要解决⼀个 `std::shared_ptr` 不存在的问题：可能指向已经销毁的对象。一个真正的智能指针应该跟踪所值对象，在悬空时知晓，悬空就是指针指向的对象不再存在。这就是对 `std::weak_ptr` 最精确的描述

你可能想知道什么时候该⽤ `std::weak_ptr`。它什么都好除了不太智能。 `std::weak_ptr` 不能解引⽤，也不能测试是否为空值。因为 `std::weak_ptr` 不是⼀个独⽴的智能指针。它是 `std::shared_ptr` 的增强。

这种关系在它创建之时就建⽴了。`std::weak_ptr` 通常从 `std::shared_ptr` 上创建。当从 `std::shared_ptr` 上创建 `std::weak_ptr` 时两者指向相同的对象，但是 `std::weak_ptr` 不会影响所指对象的引⽤计数：

```cpp
auto spw = // after spw is constructed
std::make_shared<Widget>(); // the pointed-to Widget's
// ref count(RC) is 1
// See Item 21 for in on std::make_shared
…
std::weak_ptr<Widget> wpw(spw); // wpw points to same Widget as spw. RC remains 1
…
spw = nullptr; // RC goes to 0, and the
// Widget is destroyed.
// wpw now dangles
```

`std::weak_ptr` 用 `expired` 来表⽰已经 `dangle`。你可以⽤它直接做测试：

```cpp
if (wpw.expired()) … // if wpw doesn't point to an object
```

但是通常你期望的是检查 `std::weak_ptr` 是否已经失效，如果没有失效则访问其指向的对象。这做起来⽐较容易。因为缺少解引⽤操作，没有办法写这样的代码。即使有，将检查和解引⽤分开会引⼊竞态条件：在调⽤ `expired` 和解引⽤操作之间，另⼀个线程可能对指向的对象重新赋值或者析构，并由此造成对象已析构。这种情况下，你的解引⽤将会产⽣未定义⾏为。

你需要的是⼀个原⼦操作实现检查是否过期，如果没有过期就访问所指对象。这可以通过从 `std::weak_ptr` 创建 `std::shared_ptr` 来实现，具体有两种形式可以从 `std::weak_ptr` 上创建 `std::shared_ptr`，具体⽤哪种取决于 `std::weak_ptr` 过期时你希望 `std::shared_ptr` 表现出什么⾏为。⼀种形式是 `std::weak_ptr::lock`，它返回⼀个 `std::shared_ptr`，如果 `std::weak_ptr` 过期这个 `std::shared_ptr` 为空：

```cpp
std::shared_ptr<Widget> spw1 = wpw.lock(); // if wpw's expired, spw1 is null
auto spw2 = wpw.lock(); // same as above, but uses auto
```

另⼀种形式是以 `std::weak_ptr` 为实参构造 `std::shared_ptr`。这种情况中，如果 `std::weak_ptr` 过期，会抛出⼀个异常：

```cpp
std::shared_ptr<Widget> spw3(wpw); // if wpw's expired, throw
std::bad_weak_ptr
```

但是你可能还想知道为什么 `std::weak_ptr` 就有⽤了。考虑⼀个⼯⼚函数，它基于⼀个 UID 从只读对象上产出智能指针。根据 Item18 的描述，⼯⼚函数会返回⼀个该对象类型的 `std::unique_ptr`：

```cpp
std::unique_ptr<const Widget> loadWidget(WidgetID id);
```

如果调⽤ `loadWidget` 是⼀个昂贵的操作（⽐如它操作⽂件或者数据库 I/O）并且对于 ID 来重复使⽤很常⻅，⼀个合理的优化是再写⼀个函数除了完成 `loadWidget` 做的事情之外再缓存它的结果。当请求获取⼀个 `Widget` 时阻塞在缓存操作上这本⾝也会导致性能问题，所以另⼀个合理的优化可以是当 `Widget` 不再使⽤的时候销毁它的缓存。

对于可缓存的⼯⼚函数，返回 `std::unique_ptr` 不是好的选择。调⽤者应该接收缓存对象的智能指针，调⽤者也应该确定这些对象的⽣命周期，但是缓存本⾝也需要⼀个指针指向它所缓的对象。缓存对象的指针需要知道它是否已经悬空，因为当⼯⼚客⼾端使⽤完⼯⼚产⽣的对象后，对象将被销毁，关联的缓存条⽬会悬空。所以缓存应该使⽤ `std::weak_ptr`，这可以知道是否已经悬空。这意味着⼯⼚函数返回值类型应该是 `std::shared_ptr` ，因为只有当对象的⽣命周期由 `std::shared_ptr` 管理时，`std::weak_ptr` 才能检测到悬空。

下⾯是⼀个临时凑合的 `loadWidget` 的缓存版本的实现：

```cpp
std::shared_ptr<const Widget> fastLoadWidget(WidgetID id)
{
    static std::unordered_map<WidgetID,
        std::weak_ptr<const Widget>> cache; // 译者注：这⾥是
    ⾼亮
    auto objPtr = cache[id].lock(); // objPtr is std::shared_ptr
                                    // to cached object
                                    // (or null if object's not in cache)
    if (!objPtr) {                  // if not in cache
        objPtr = loadWidget(id);    // load it
        cache[id] = objPtr;         // cache it
    }
    return objPtr;
}
```

这个实现使⽤了 C++11 的 hash 表容器 `std::unordered_map`，但是需要的 WidgetID 哈希和相等性⽐较函数在这⾥没有展⽰。

`fastLoadWidget` 的实现忽略了以下事实：`cache` 可能会累积过期的 `std::weak_ptr` (对应已经销毁的 `Widget` )。可以改进实现⽅式，但不要花时间在不会引起对 `std::weak_ptr` 的深⼊了解的问题上，让我们考虑第⼆个⽤例：观察者设计模式。此模式的主要组件是 `subjects`（状态可能会更改的对象）和`observers`（状态发⽣更改时要通知的对象）。在⼤多数实现中，每个 `subject` 都包含⼀个数据成员，该成员持有指向其 `observer` 的指针。这使subject很容易发布状态更改通知。subject 对控制 `observers` 的⽣命周期（例如，当它们被销毁时）没有兴趣，但是subject 对确保 `observers` 被销毁时，不会访问它具有极⼤的兴趣 。⼀个合理的设计是每个 `subject `持有其 `observers` 的 `td::weak_ptr`，因此可以在使⽤前检查是否已经悬空。

作为最后⼀个使⽤ `std::weak_ptr` 的例⼦，考虑⼀个持有三个对象 A、B、 C的数据结构，A和C共享B的所有权，因此持有 `std::shared_ptr`：

有三种选择：
- **原始指针**。使⽤这种⽅法，如果 A 被销毁，但是 C 继续指向 B，B 就会有⼀个指向 A 的悬空指针。而且 B 不知道指针已经悬空，所以 B 可能会继续访问，就会导致未定义⾏为。
- **`std::shared_ptr`**。这种设计，A 和 B 都互相持有对⽅的 `std::shared_ptr`，导致 `std::shared_ptr` 在销毁时出现循环。即使A和B⽆法从其他数据结构被访问（⽐如，C不再指向B），每个的引⽤计数都是 1。如果发⽣了这种情况，A 和 B 都被泄露：程序⽆法访问它们，但是资源并没有被回收。
- **`std::weak_ptr`**。这避免了上述两个问题。如果 A 被销毁，B 还是有悬空指针，但是 B 可以检查。尤其是尽管 A 和 B 互相指向，B的指针不会影响 A 的引⽤计数，因此不会导致⽆法销毁。

使⽤ `std::weak_ptr` 显然是这些选择中最好的。但是，需要注意使⽤ `std::weak_ptr` 打破 `std::shared_ptr` 循环并不常⻅。在严格分层的数据结构⽐如树，⼦节点只被⽗节点持有。当⽗节点被销毁时，⼦节点就被销毁。从⽗到⼦的链接关系可以使⽤ `std::unique_ptr` 很好的表征。从⼦到⽗的反向连接可以使⽤原始指针安全实现，因此⼦节点的⽣命周期肯定短于⽗节点。因此⼦节点解引⽤⼀个悬垂的⽗节点指针是没有问题的。

当然，不是所有的使⽤指针的数据结构都是严格分层的，所以当发⽣这种情况时，⽐如上⾯所述 cache 和观察者情况，知道 `std::weak_ptr` 随时待命也是不错的。

从效率⻆度来看，`std::weak_ptr` 与 `std::shared_ptr` 基本相同。两者的⼤小是相同的，使⽤相同的控制块（参⻅ Item 19），构造、析构、赋值操作涉及引⽤计数的原⼦操作。这可能让你感到惊讶，因为本 Item 开篇就提到 `std::weak_ptr` 不影响引⽤计数。我写的是 `std::weak_ptr` 不参与对象的共享所有权，因此不影响指向对象的引⽤计数。实际上在控制块中还是有第⼆个引⽤计数，`std::weak_ptr` 操作的是第⼆个引⽤计数。想了解细节的话，继续看 Item 21 吧。

总结：
- 像 `std::shared_ptr` 使⽤ `std::weak_ptr` 可能会悬空。
- `std::weak_ptr` 的潜在使⽤场景包括：`caching、observer lists`、打破 `std::shared_ptr` 指向循环