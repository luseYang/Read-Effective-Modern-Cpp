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



