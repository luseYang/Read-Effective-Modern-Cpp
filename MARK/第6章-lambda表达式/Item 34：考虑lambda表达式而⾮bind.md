---
title: Item 34 考虑lambda表达式而⾮std::bind
tags: ["lambda", "std::bind"]
categories: [Effective Modern C++]
description: '考虑lambda表达式而⾮std::bind'
date: 2024-08-26 15:50:21
cover: https://images.pexels.com/photos/26653530/pexels-photo-26653530.jpeg
---

# 前言

C++11 中的 `std::bind` 是 C++98 的 `std::bind1st` 和 `std::bind` 的后续，但在 2005 年已经成为了标准库的⼀部分。那时标准化委员采⽤了 TR1 的⽂档，其中包含了 `bind` 的规范。（在 TR1 中，`bind` 位于不同的命名空间，因此它是 `std::tr1::bind`，而不是 `std::bind`，接口细节也有所不同）。这段历史意味着⼀些程序员有⼗年或更⻓时间的使⽤ `std::bind` 经验。如果您是其中之⼀，可能会不愿意放弃⼀个对您有⽤的⼯具。这是可以理解的，但是在这种情况下，改变是更好的，因为在 C++11 中，lambda ⼏乎是⽐ `std::bind` 更好的选择。 从 C++14 开始，lambda 的作⽤不仅强⼤，而且是完全值得使⽤的。

这个条⽬假设您熟悉 `std::bind`。 如果不是这样，您将需要获得基本的了解，然后再继续。 ⽆论如何，这样的理解都是值得的，因为您永远不知道何时会在必须阅读或维护的代码库中遇到 `std::bind` 的使⽤。

与第32项中⼀样，我们将从 `std::bind` 返回的函数对象称为绑定对象。

优先 lambda 而不是 `std::bind` 的最重要原因是 lambda 更易读。 例如，假设我们有⼀个设置闹钟的函数：

```cpp
// typedef for a point in time (see Item 9 for syntax)
using Time = std::chrono::steady_clock::time_point;

// see Item 10 for "enum class"
enum class Sound { Beep, Siren, Whistle };

// typedef for a length of time
using Duration = std::chrono::steady_clock::duration;
// at time t, make sound s for duration d void setAlarm(Time t, Sound s, Durationd);
```

进⼀步假设，在程序的某个时刻，我们已经确定需要设置⼀个小时后响 30 秒的闹钟。 但是，具体声⾳仍未确定。我们可以编写⼀个 lambda 来修改 `setAlarm` 的界⾯，以便仅需要指定声⾳：

```cpp
// setSoundL ("L" for "lambda") is a function object allowing a // sound to be
specified for a 30-sec alarm to go off an hour // after it's set
auto setSoundL =
    [](Sound s)
        {
        // make std::chrono components available w/o qualification
        using namespace std::chrono;
        setAlarm(steady_clock::now() + hours(1), // alarm to go off
            s, // in an hour for
            seconds(30)); // 30 seconds
        };
```

我们在 lambda 中突出了对 `setAlarm` 的调⽤。这看来起是⼀个很正常的函数调⽤，即使是⼏乎没有 lambda 经验的读者也可以看到：传递给 lambda 的参数被传递给了 `setAlarm`。

通过使⽤基于 C++11 对⽤⼾⾃定义常量的⽀持而建⽴的标准后缀，如秒(s)，毫秒(ms)和小时(h)等，我们可以简化 C++14 中的代码。这些后缀在 `std::literals` 命名空间中实现，因此上述代码可以按照以下⽅式重写：

```cpp
auto setSoundL =
    [](Sound s)
    {
        using namespace std::chrono;
        using namespace std::literals; // for C++14 suffixes
        setAlarm(steady_clock::now() + 1h, // C++14, but
            s, // same meaning
            30s); // as above
    };
```

下⾯是我们第⼀次编写对应的 `std::bind` 调⽤。这⾥存在⼀个我们后续会修复的错误，但正确的代码会更加复杂，即使是此简化版本也会带来⼀些重要问题：

```cpp
using namespace std::chrono; // as above
using namespace std::literals;
using namespace std::placeholders; // needed for use of "_1"
auto setSoundB = std::bind(setAlarm, // "B" for "bind"
                            steady_clock::now() + 1h, // incorrect! see below
                            _1,
                            30s);
```

我想像在 lambda 中⼀样突出显⽰对 `setAlarm` 的调⽤，但是没有这么做。这段代码的读者只需知道，调⽤ `setSoundB` 会使⽤在对 `std::bind` 的调⽤中所指定的时间和持续时间来调⽤ `setAlarm`。对于初学者来说，占位符“ _1”本质上是⼀个魔术，但即使是普通读者也必须从思维上将占位符中的数字映射到其在 `std::bind` 参数列表中的位置，以便明⽩调⽤ `setSoundB` 时的第⼀个参数会被传递进 `setAlarm`，作为调⽤时的第⼆个参数。在对 `std::bind` 的调⽤中未标识此参数的类型，因此读者必须查阅 `setAlarm` 声明以确定将哪种参数传递给 `setSoundB`。

但正如我所说，代码并不完全正确。在 lambda 中，表达式 `steady_clock::now() + 1h` 显然是是 `setAlarm` 的参数。调⽤ `setAlarm` 时将对其进⾏计算。这是合理的：我们希望在调⽤ `setAlarm` 后⼀小时发出警报。但是，在 `std::bind` 调⽤中，将 `steady_clock::now() + 1h` 作为参数传递给了 `std::bind`，而不是 `setAlarm`。这意味着将在调⽤ `std::bind` 时对表达式进⾏求值，并且该表达式产⽣的时间将存储在结果绑定对象中。结果，闹钟将被设置为在调⽤ `std::bind` 后⼀小时发出声⾳，而不是在调⽤ `setAlarm` ⼀小时后发出。

要解决此问题，需要告诉 `std::bind` 推迟对表达式的求值，直到调⽤ `setAlarm` 为⽌，而这样做的⽅法是将对 `std::bind` 的第⼆个调⽤嵌套在第⼀个调⽤中：

```cpp
auto setSoundB =
    std::bind(setAlarm,
    std::bind(std::plus<>(), steady_clock::now(), 1h), _1,
    30s);
```

如果您熟悉C++98的 `std::plus` 模板，您可能会惊讶地发现在此代码中，尖括号之间未指定任何类型，即该代码包含 `std::plus<>`，而不是 `std::plus<type>`。 在C ++14中，通常可以省略标准运算符模板的模板类型参数，因此⽆需在此处提供。 C++11没有提供此类功能，因此等效于 lambda 的 C ++11 `std::bind` 使⽤为：

```cpp
using namespace std::chrono; // as above
using namespace std::placeholders;
auto setSoundB =
    std::bind(setAlarm,
        std::bind(std::plus<steady_clock::time_point>(),
steady_clock::now(), hours(1)),
                seconds(30));
```

如果此时 Lambda 看起来不够吸引，那么应该检查⼀下视⼒了。

当 `setAlarm` 重载时，会出现⼀个新问题。 假设有⼀个重载函数，其中第四个参数指定了⾳量：

```cpp
enum class Volume { Normal, Loud, LoudPlusPlus };
void setAlarm(Time t, Sound s, Duration d, Volume v);
```

lambda 能继续像以前⼀样使⽤，因为根据重载规则选择了 `setAlarm` 的三参数版本：

```cpp
auto setSoundL =
    [](Sound s)
    {
        using namespace std::chrono;
        setAlarm(steady_clock::now() + 1h, s,
        30s);
    };
```

然而，`std::bind` 的调⽤将会编译失败：

```cpp
auto setSoundB = // error! which
    std::bind(setAlarm, // setAlarm?
    std::bind(std::plus<>(),
    steady_clock::now(),
    1h),
    _1,
    30s);
```

这⾥的问题是，编译器⽆法确定应将两个 `setAlarm` 函数中的哪⼀个传递给 `std::bind`。 它们仅有的是⼀个函数名称，而这个函数名称是不确定的。
要获得对 `std::bind` 的调⽤能进⾏编译，必须将 `setAlarm` 强制转换为适当的函数指针类型：

```cpp
using SetAlarm3ParamType = void(*)(Time t, Sound s, Duration d);
auto setSoundB = // now
    std::bind(static_cast<SetAlarm3ParamType>(setAlarm), // okay
    std::bind(std::plus<>(),
        steady_clock::now(),
        1h),
        _1,
        30s);
```

但这在 lambda 和 `std::bind` 的使⽤上带来了另⼀个区别。在 `setSoundL` 的函数调⽤操作符（即 lambda 的闭包类对应的函数调⽤操作符）内部，对 `setAlarm` 的调⽤是正常的函数调⽤，编译器可以按常规⽅式进⾏内联：


```cpp
setSoundL(Sound::Siren);    // body of setAlarm may
                            // well be inlined here
```

但是，对 `std::bind` 的调⽤是将函数指针传递给 `setAlarm`，这意味着在 `setSoundB` 的函数调⽤操作符（即绑定对象的函数调⽤操作符）内部，对 `setAlarm` 的调⽤是通过⼀个函数指针。 编译器不太可能通过函数指针内联函数，这意味着与通过 `setSoundL` 进⾏调⽤相⽐，通过 `setSoundB` 对 `setAlarm `的 调⽤，其函数不⼤可能被内联：

```cpp
setSoundB(Sound::Siren);    // body of setAlarm is less
                            // likely to be inlined here
```

因此，使⽤ lambda 可能会⽐使⽤ `std::bind` 能⽣成更快的代码。`setAlarm` ⽰例仅涉及⼀个简单的函数调⽤。如果您想做更复杂的事情，使⽤ lambda 会更有利。 例如，考虑以下 C++14 的 lambda 使⽤，它返回其参数是否在最小值（`lowVal`）和最⼤值（`highVal`）之间的结果，其中 `lowVal` 和 `highVal` 是局部变量：

```cpp
auto betweenL =
    [lowVal, highVal]
    (const auto& val) // C++14
    { return lowVal <= val && val <= highVal; };
```

使⽤ `std::bind` 可以表达相同的内容，但是该构造是⼀个通过晦涩难懂的代码来保证⼯作安全性的⽰例：

```cpp
using namespace std::placeholders; // as above
auto betweenB =
    std::bind(std::logical_and<>(), // C++14
        std::bind(std::less_equal<>(), lowVal, _1),
        std::bind(std::less_equal<>(), _1, highVal));
```

在 C++11 中，我们必须指定要⽐较的类型，然后 `std::bind` 调⽤将如下所⽰：

```cpp
auto betweenB = // C++11 version
std::bind(std::logical_and<bool>(),
std::bind(std::less_equal<int>(), lowVal, _1),
std::bind(std::less_equal<int>(), _1, highVal));
```

当然，在 C++11 中，`lambda` 也不能采⽤ `auto` 参数，因此它也必须指定⼀个类型：

```cpp
auto betweenL = // C++11 version
    [lowVal, highVal]
    (int val)
    { return lowVal <= val && val <= highVal; };
```

⽆论哪种⽅式，我希望我们都能同意，lambda           版本不仅更短，而且更易于理解和维护。之前我就说过，对于那些没有 `std::bind` 使⽤经验的⼈，其占位符（例如_1，_2等）本质上都是 `magic`。 但是，不仅仅占位符的⾏为是不透明的。 假设我们有⼀个函数可以创建 `Widget`的压缩副本，

```cpp
enum class CompLevel { Low, Normal, High }; // compression
                                            // level
Widget compress(const Widget& w,            // make compressed
CompLevel lev);                             // copy of w
```

并且我们想创建⼀个函数对象，该函数对象允许我们指定应将特定 `w` 的压缩级别。这种使⽤ `std::bind` 的话将创建⼀个这样的对象：

```cpp
Widget w;
using namespace std::placeholders;
auto compressRateB = std::bind(compress, w, _1);
```

现在，当我们将 `w` 传递给 `std::bind` 时，必须将其存储起来，以便以后进⾏压缩。它存储在对象 `compressRateB` 中，但是这是如何存储的呢（是通过值还是引⽤）。之所以会有所不同，是因为如果在对 `std::bind` 的调⽤与对 `compressRateB` 的调⽤之间修改了 `w`，则按引⽤捕获的 `w` 将反映其更改，而按值捕获则不会。

答案是它是按值捕获的，但唯⼀知道的⽅法是记住 `std::bind` 的⼯作⽅式；在对 `std::bind` 的调⽤中没有任何迹象。与 lambda ⽅法相反，其中 `w` 是通过值还是通过引⽤捕获是显式的：

```cpp
auto compressRateL = // w is captured by
    [w](CompLevel lev) // value; lev is
        { return compress(w, lev); }; // passed by value
```

同样明确的是如何将参数传递给 lambda。 在这⾥，很明显参数 `lev` 是通过值传递的。 因此：

```cpp
compressRateL(CompLevel::High); // arg is passed
                                    // by value
```

但是在对由 `std::bind` ⽣成的对象调⽤中，参数如何传递？

```cpp
compressRateB(CompLevel::High); // how is arg
                                    // passed?
```

同样，唯⼀的⽅法是记住 `std::bind` 的⼯作⽅式。（答案是传递给绑定对象的所有参数都是通过引⽤传递的，因为此类对象的函数调⽤运算符使⽤完美转发。）

与 lambda 相⽐，使⽤ `std::bind` 进⾏编码的代码可读性较低，表达能⼒较低，并且效率可能较低。 在 C++14 中，没有 `std::bind` 的合理⽤例。但是，在 C++11 中，可以在两个受约束的情况下证明使⽤ `std::bind` 是合理的：

- 移动捕获。 C++11 的 lambda 不提供移动捕获，但是可以通过结合 lambda 和 `std::bind` 来模拟。有关详细信息，请参阅条款32，该条款还解释了在 C++14 中，`lambda`对初始化捕获的⽀持将少了模拟的需求。

- 多态函数对象。 因为绑定对象上的函数调⽤运算符使⽤完全转发，所以它可以接受任何类型的参数（以条款 30 中描述的完全转发的限制为例⼦）。当您要使⽤模板化函数调⽤运算符来绑定对象时，此功能很有⽤。 例如这个类，

```cpp
class PolyWidget {
public:
    template<typename T>
    void operator()(const T& param); ...
};
```

`std::bind` 可以如下绑定⼀个 `PolyWidget` 对象：

```cpp
PolyWidget pw;
auto boundPW = std::bind(pw, _1);
```

`boundPW` 可以接受任意类型的对象了：

```cpp
boundPW(1930);      // pass int to
                        // PolyWidget::operator()

boundPW(nullptr);   // pass nullptr to
                        // PolyWidget::operator()

boundPW("Rosebud"); // pass string literal to
                        // PolyWidget::operator()        
```

这⼀点⽆法使⽤ C++11 的 lambda 做到。 但是，在 C++14 中，可以通过带有 `auto` 参数的 lambda 轻松实现：

```cpp
auto boundPW = [pw](const auto& param) // C++14
{ pw(param); };
```

当然，这些是特殊情况，并且是暂时的特殊情况，因为⽀持 C++14 lambda 的编译器越来越普遍了。当 `bind` 在 2005 年被⾮正式地添加到 C++ 中时，与 1998 年的前⾝相⽐有了很⼤的改进。 在 C++11 中增加了 lambda ⽀持，这使得 `std::bind` ⼏乎已经过时了，从 C++14 开始，更是没有很好的⽤例了。

# 总结

- 与使⽤ `std::bind` 相⽐，Lambda 更易读，更具表达⼒并且可能更⾼效。
- 只有在 C++11 中，`std::bind` 可能对实现移动捕获或使⽤模板化函数调⽤运算符来绑定对象时会很有⽤。
