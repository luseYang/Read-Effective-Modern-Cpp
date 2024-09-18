---
title: Item 16 让const成员函数线程安全
tags: ["现代C++", "const"]
categories: [Effective Modern C++]
description: '让const成员函数线程安全'
date: 2024-07-23 11:55:21
cover: https://images.pexels.com/photos/26986687/pexels-photo-26986687.png
---

# 条款16：让const成员函数线程安全

在数学领域中⼯作，我们就会发现⽤⼀个类表⽰多项式是很⽅便的。在这个类中，使⽤⼀个函数来计算多项式的根是很有⽤的。也就是多项式的值为零的时候。这样的⼀个函数它不会更改多项式。所以，它⾃然被声明为 `const` 函数。

```cpp
class Polynomial {
public:
    using RootsType = std::vector<double>; // 数据结构保存多项式为零的值
    RootsType roots() const;
};
```

计算多项式的根是很复杂的，因此如果不需要的话，我们就不做。如果必须做，我们肯定不会只做⼀次。所以，如果必须计算它们，就缓存多项式的根，然后实现 `roots` 来返回缓存的值。下⾯是最基本的实现：

```cpp
class Polynomial {
public:
    using RootsType = std::vector<double>;
    RootsType roots() const {
        if (!rootsAreVaild) { // 如果缓存不可⽤
        // 计算根
            rootsAreVaild = true; // ⽤`rootVals`存储它们
        }
        return rootVals;
    }
private:
    mutable bool rootsAreVaild{ false }; // initializers 的更多信息
    mutable RootsType rootVals{}; // 请查看条款7
};
```

从概念上讲，`roots` 并不改变它所操作的多项式对象。但是作为缓存的⼀部分，它也许会改变 `rootVals` 和 `rootsAreVaild `的值。这就是 `mutable` 的经典使⽤样例，这也是为什么它是数据成员声明的⼀部分。

假设现在有两个线程同时调⽤ `Polynomial` 对象的 `roots` ⽅法:

```cpp
Polynomial p;
/*------ Thread 1 ------*/          /*-------- Thread 2 --------*/
auto rootsOfp = p.roots();          auto valsGivingZero = p.roots();
```

这些⽤⼾代码是⾮常合理的。`roots` 是 `const` 成员函数，那就表⽰着它是⼀个读操作。它最起码应该做到：在没有同步的情况下，让多个线程执⾏读操作是安全的。在本例中却没有做到线程安全。因为在 `roots` 中，这些线程中的⼀个或两个可能尝试修改成员变量 `rootsAreVaild` 和 `rootVals` 。这就意味着在没有同步的情况下，这些代码会有不同的线程读写相同的内存，这就是 `data race` 的定义。这段代码的⾏为是未定义的。

问题就是 `roots` 被声明为 `const`，但不是线程安全的。`const` 声明在 C++11 和 C++98 中都是正确的（检索多项式的根并不会更改多项式的值），因此需要纠正的是线程安全的缺乏。

解决这个问题最普遍简单的⽅法就是-------使⽤互斥锁：

```cpp
class Polynomial {
public:
    using RootsType = std::vector<double>;

    RootsType roots() const {
        std::lock_guard<std::mutex> g(m); // lock mutex
        if (!rootsAreVaild) { // 如果缓存⽆效
            // 计算/存储roots
            rootsAreVaild = true;
        }
        return rootsVals;
    } // unlock mutex
private:
    mutable std::mutex m;
    mutable bool rootsAreVaild { false };
    mutable RootsType rootsVals {};
};
```

`std::mutex m` 被声明为 `mutable` ，因为锁定和解锁它的都是 `non-const` 函数。在 `roots` （const成员函数）中，`m` 将被视为 `const` 对象。值得注意的是，因为 `std::mutex` 是⼀种 `move-only` 的类型（⼀种可以移动但不能复制的类型），所以将 `m` 添加进多项式中的副作⽤是使它失去了被复制的能⼒。不过，它仍然可以移动。

在某些情况下，互斥量是过度的。例如，你所做的只是计算成员函数被调⽤了多少次。使⽤ `std::atomic` 修饰的 `counter`（保证其他线程视这个操作为不可分割的发⽣，参⻅item40）。（然而它是否轻量取决于你使⽤的硬件和标准库中互斥量的实现。）以下是如何使⽤ `std::atomic` 来统计调⽤次数。

```cpp
class Point { // 2D point
public:
    // noexcept的使⽤参考Item 14
    double distanceFromOrigin() const noexcept {
        ++callCount; // 原⼦的递增
        return std::sqrt((x * x) + (y * y));
    }
private:
    mutable std::atomic<unsigned> callCount{ 0 };
    double x, y;
};
```

与 `std::mutex` ⼀样，`std::atomic` 是 `move-only` 类型，所以在 `Point` 中调⽤ `Count` 的意思就是 `Point` 也是 `move-only` 的。

因为对 `std::atomic` 变量的操作通常⽐互斥量的获取和释放的消耗更小，所以你可能更倾向与依赖 `std::atomic`。例如，在⼀个类中，缓存⼀个开销昂贵的 `int`，你就会尝试使⽤⼀对 `std::atomic` 变量而不是互斥锁.

```cpp
class Widget {
public:
    int magicValue() const{
        if (cacheVaild) return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cachedValue = val1 + val2; // 第⼀步
            cacheVaild = true; // 第⼆步
            return cachedVaild;
        }
    }
private:
    mutable std::atomic<bool> cacheVaild{ false };
    mutable std::atomic<int> cachedValue;
};
```

这是可⾏的，但有时运⾏会⽐它做到更加困难。考虑：
- ⼀个线程调⽤ `Widget::magicValue`，将 `cacheValid` 视为 `false`，执⾏这两个昂贵的计算，并将它们的和分配给 `cachedValue`。
- 此时，第⼆个线程调⽤ `Widget::magicValue`，也将 `cacheValid` 视为 `false`，因此执⾏刚才完成的第⼀个线程相同的计算。（这⾥的“第⼆个线程”实际上可能是其他⼏个线程。）

这种⾏为与使⽤缓存的⽬的背道而驰。将 `cachedValue` 和 `CacheValid` 的顺序交换可以解决这个问题，但结果会更糟：

```cpp
class Widget {
public:
    int magicValue() const {
        if (cacheVaild) return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cacheVaild = true; // 第⼀步
            return cachedValue = val1 + val2; // 第⼆步
        }
    }
};
```

假设 `cacheVaild` 是 `false`，那么：
- ⼀个线程调⽤ `Widget::magicValue`，在 `cacheVaild` 被设置成 `true` 时执⾏到它。
- 在这时，第⼆个线程调⽤ `Widget::magicValue` 随后检查缓存值。看到它是 `true`，就返回 `cacheValue`，即使第⼀个线程还没有给它赋值。因此返回的值是不正确的。

这⾥有⼀个坑。对于需要同步的是单个的变量或者内存位置，使⽤ `std::atomic` 就⾜够了。不过，⼀旦你需要对两个以上的变量或内存位置作为⼀个单元来操作的话，就应该使⽤互斥锁。对于 `Widget::magicValue` 是这样的。

```cpp
class Widget {
public:
    int magicValue() const {
        std::lock_guard<std::mutex> guard(m); // lock m
        if (cacheValid) return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cachedValue = val1 + val2;
            cacheValid = true;
            return cachedValue;
        }
    } // unlock m
private:
    mutable std::mutex m;
    mutable int cachedValue; // no longer atomic
    mutable bool cacheValid{ false }; // no longer atomic
};
```

现在，这个条款是基于，多个线程可以同时在⼀个对象上执⾏⼀个 `const` 成员函数这个假设的。如果你不是在这种情况下编写⼀个 `const` 成员函数。也就是你可以保证在对象上永远不会有多个线程执⾏该成员函数。再换句话说，该函数的线程安全是⽆关紧要的。⽐如，为单线程使⽤而设计类的成员函数的线程安全是不重要的。在这种情况下你可以避免，因使⽤ `mutex` 和 `std::atomics` 所消耗的资源，以及包含它们的类只能使⽤移动语义带来的副作⽤。然而，这种单线程的场景越来越少⻅，而且很可能会越来越少。可以肯定的是，`const` 成员函数应⽀持并发执⾏，这就是为什么你应该确保 `const` 成员函数是线程安全的。

总结：
- 确保 `const` 成员函数线程安全，除⾮你确定它们永远不会在临界区（concurrent context）中使⽤。
- `std::atomic` 可能⽐互斥锁提供更好的性能，但是它只适合操作单个变量或内存位置。