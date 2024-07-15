# 优先考虑 auto 而非显示类型声明

对一个局部变量使用解引用迭代器的方式初始化：

```cpp
template<typename It>
void dwim(It b, It e)
{
    while(b!=e){
        typename std::iterator_traits<It>::value_type
        currValue = *b;
    }
}
```

声明⼀个局部变量，变量的类型只有编译后知道，这⾥必须使⽤ `typename` 指定，因为待决名的原因。

`auto` 变量从初始化表达式中推导出类型，所以我们必须初始化。

```cpp
template<typename It>
void dwim(It b,It e)
{
    while(b!=e){
        auto currValue = *b;
        ...
    }
}
```

因为 `auto` 使用了 Item2 所描述的类型推导技术，它甚至可以表示一些只有编译器才知道的类型：

```cpp
auto derefUPLess = [](const std::unique_ptr<Widget> &p1, //专⽤于Widget类型的⽐
较函数
const std::unique_ptr<Widget> &p2) {return *p1 < *p2;};
```

```cpp
auto derefUPLess = [](const auto& p1,const auto& p2){return *p1 < *p2;};
```

我们不需要使用 `auto` 声明局部变量来保存一个闭包，因为我们可以使用 `std::function` 对象。`std::function` 是⼀个 C++11 标准模板库中的⼀个模板，它泛化了函数指针的概念。与函数指针只能指向函数不同， `std::function` 可以指向任何可调⽤对象，也就是那些像函数⼀样能进⾏调⽤的东西。当你声明函数指针时你必须指定函数类型（即函数签名），同样当你创建 `std::function` 对象时你也需要提供函数签名，由于它是⼀个模板所以你需要在它的模板参数⾥⾯提供。举个例⼦，假设你想声明⼀个 `std::function` 对象 `func` 使他指向⼀个可调⽤对象，⽐如⼀个具有这样函数签名的函数：

```cpp
bool(const std::unique_ptr<Widget> &p1,
const std::unique_ptr<Widget> &p2);
```

就应该这样写：

```cpp
std::function<bool(const std::unique_ptr<Widget> &p1,
const std::unique_ptr<Widget> &p2)> func;
```

 因为 lambda 表达式产生一个可调用对象，所以我们可以把闭包存放到 `std::function` 对象中。这意味着我们可以不使用 `auto` 写出上面的 `derefUPLess` 对象。

 ```cpp
std::function<bool(const std::unique_ptr<Widget> &p1,
const std::unique_ptr<Widget> &p2)>
dereUPLess = [](const std::unique_ptr<Widget> &p1,
const std::unique_ptr<Widget> &p2){return *p1<*p2;};
```

实例化 `std::function` 并声明⼀个对象这个对象将会有固定的⼤小。当使⽤这个对象保存⼀个闭包时它可能⼤小不⾜不能存储，这个时候 `std::function` 的构造函数将会在堆上⾯分配内存来存储，这就造成了使⽤ `std::function` ⽐ `auto` 会消耗更多的内存。并且通过具体实现我们得知通过 `std::function` 调⽤⼀个闭包⼏乎⽆疑⽐ `auto` 声明的对象调⽤要慢。换句话说，`std::function` ⽅法⽐ `auto` ⽅法要更耗空间且更慢.

使用 `auto` 还可以避免一个问题，称之为依赖类型快捷方式的问题。

```cpp
std::vector<int> v;
unsigned sz = v.size();
```

`v.size()` 的标准返回类型是 `std::vector<int>::size_type` ,但是很多程序猿都知道 `std::vector<int>::size_type` 实际上被指定为无符号整型，所以很多⼈都认为⽤ `unsigned` ⽐写那⼀⻓串的标准返回类型⽅便。这会造成⼀些有趣的结果。

举个例子：在 Windows 32-bit上 `std::vector<int>::size_type` 和 `unsigned int` 都是⼀样的类型，但是在 Windows 64-bit 上 `std::vector<int>::size_type` 是64位， `unsigned int` 是32位。这意味着这段代码在 Windows 32-bit 上正常⼯作，但是当把应⽤程序移植到 Windows 64-bit 上时就可能会出现⼀些问题。

使用 `auto` 的话就可以避免这些细枝末节的问题。

再考虑以下代码：

```cpp
std::unordered_map<std::string,int> m;
...
for(const std::pair<std::string,int>& p : m)
{
    ...
}
```

要知道问题的所在，我们需要知道 `std::unordered_map` 的 key 是一个常量，所以 `std::pair` 的类型不是 `std::pair<std::string,int>` 而应该是 `std::pair<const std::string,int>` 。这时候编译器就会找到一种方法，努力把前者转换为后者。即创建一个临时对象，这个临时对象的类型是 **`p` 想绑定到的对象的类型 ** ，即 `m` 中元素的类型，然后把 `p` 的引用绑定到这个临时对象上。在每个循环迭代结束时，临时对象将会销毁，如果你写了这样一个循环，你只是让 `p` 成为了指向 `m` 中各个元素的引⽤而已。使用 `auto` 可以避免这些很难被意识到的类型不匹配的错误：

```cpp
for(const auto & p : m)
{
    ...
}
```

这样无疑是更有效率的，而且更容易书写。在没有 `auto` 的版本中 `p` 会指向⼀个临时变量，这个临时变量在每次迭代完成时会被销毁!

后⾯这两个例⼦说明了显式的指定类型可能会导致你不像看到的类型转换。如果你使⽤ `auto` 声明⽬标变量你就不必担⼼这个问题。

`auto` 是可选项，不是命令，在某些情况下如果你的专业判断告诉你使⽤显式类型声明⽐ `auto` 要更清晰更易维护，那你就不必再坚持使⽤ `auto`。

总结：

- `auto` 变量必须初始化，通常它可以避免⼀些移植性和效率性的问题，也使得重构更⽅便，还能让你少打⼏个字。
- 正如Item2和6讨论的，`auto` 类型的变量可能会踩到⼀些陷阱。

