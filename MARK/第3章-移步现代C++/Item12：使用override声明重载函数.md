---
title: Item 12 使⽤ override 声明重载函数
tags: ["现代C++", "override"]
categories: [Effective Modern C++]
description: '使⽤ override 声明重载函数'
date: 2024-07-21 15:10:21
cover: https://s2.loli.net/2024/07/21/5J8kxLaoYrCZTs6.jpg
---

# 条款12：使⽤ override 声明重载函数

在C++⾯向对象的世界里，涉及的概念有类，继承，虚函数。这个世界最基本的概念是派⽣类的虚函数重写基类同名函数。鉴于 **"重写"** 听起来像 **"重载"**，尽管两者完全不相关，下⾯就通过⼀个派⽣类和基类来说明**什么是虚函数重写**

```cpp
class Base {
public:
    virtual void doWork(); // 基类虚函数
    …
};
class Derived: public Base {
public:
    virtual void doWork(); // 重写Base::doWork(这里"virtual"是可以省略的)
    …
};
std::unique_ptr<Base> upb = std::make_unique<Derived>();// 创建基类指针指向派⽣类对象关于std：：make_unique请参⻅Item1
...
upb->doWork(); // 通过基类指针调⽤doWork实际上是派⽣类的 doWork 函数被调⽤
```

> 要想重写⼀个函数，必须满⾜下列要求：
- 基类函数必须是 `virtual`
- 基类和派⽣类函数名必须完全⼀样（除⾮是析构函数
- 基类和派⽣类函数参数必须完全⼀样
- 基类和派⽣类函数常量性必须完全⼀样
- 基类和派⽣类函数的返回值和异常说明必须兼容

除了这些 C++98 就存在的约束外，C++11 ⼜添加了⼀个：
- 函数的引⽤限定符必须完全⼀样。成员函数的引⽤限定符是 C++11 很少抛头露脸的特性，所以如果你从没听过它⽆需惊讶。它可以限定成员函数只能⽤于左值或者右值。成员函数不需要 `virtual` 也能使⽤它们：

```cpp
class Widget {
public:
    …
    void doWork() &;    //只有*this为左值的时候才能被调⽤
    void doWork() &&;   //只有*this为右值的时候才能被调⽤
};
…
Widget makeWidget(); // ⼯⼚函数（返回右值）
Widget w;            // 普通对象（左值）
…
w.doWork(); // 调⽤被左值引⽤限定修饰的Widget::doWork版本(即Widget::doWork &)

makeWidget().doWork(); // 调⽤被右值引⽤限定修饰的Widget::doWork版本(即Widget::doWork &&)
```

如果基类的虚函数有引⽤限定符，派⽣类的重写就必须具有相同的引⽤限定符。如果没有，那么新声明的函数还是属于派⽣类，但是不会重写⽗类的任何函数。

这么多的重写需求意味着哪怕⼀个小小的错误也会造成巨⼤的不同。代码中包含重写错误通常是有效的，但它的意图不是你想要的。因此你不能指望当你犯错时编译器能通知你。⽐如，下⾯的代码是完全合法的，咋⼀看，还很有道理，但是它包含了⾮虚函数重写。你能识别每个case的错误吗，换句话说，为什么派⽣类函数没有重写同名基类函数？

```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
};
class Derived: public Base {
public:
    virtual void mf1();
    virtual void mf2(unsigned int x);
    virtual void mf3() &&;
    void mf4() const;
};
```

- `mf1` 在基类声明为 `const`, 但是派⽣类没有这个常量限定符
- `mf2` 在基类声明为接受⼀个 `int` 参数，但是在派⽣类声明为接受 `unsigned int` 参数
- `mf3` 在基类声明为左值引⽤限定，但是在派⽣类声明为右值引⽤限定
- `mf4` 在基类没有声明为虚函数

这些都是编译器不会报错的现象。这并不是我们所希望的结果

由于正确声明派⽣类的重写函数很重要，但很容易出错，C++11 提供⼀个⽅法让你可以**显式的将派⽣类函数指定为应该是基类重写版本**：将它声明为 `override`。

```cpp
class Derived: public Base {
public:
    virtual void mf1() override;
    virtual void mf2(unsigned int x) override;
    virtual void mf3() && override;
    virtual void mf4() const override;
};
```

代码不能编译，当然了，因为这样写的时候，编译器会指出所有与重写有关的问题。这也是你想要的，以及为什么要在所有重写函数后⾯加上 `override`。

⽐起让编译器告诉你"将要"重写实际不会重写，不如给你的派⽣类成员函数全都加上 `override`。如果你考虑修改修改基类虚函数的函数签名，`override` 还可以帮你评估后果。如果派⽣类全都⽤上 `override`，你可以只改变基类函数签名，重编译系统，再看看你造成了多⼤的问题（即，多少派⽣类不能通过编译），然后决定是否值得如此⿇烦更改函数签名。

C++ 既有很多关键字，C++11引⼊了两个上下⽂关键字, `override` 和 `final` **（向虚函数添加 final 可以防⽌派⽣类重写。 final 也能⽤于类，这时这个类不能⽤作基类）**。**这两个关键字的特点是它们是保留的，它们只是位于特定上下⽂才被视为关键字**。对于 `override`，*它只在成员函数声明结尾处才被视为关键字*。这意味着如果你以前写的代码⾥⾯已经⽤过 `override` 这个名字，那么换到 C++11 标准你也⽆需修改代码.

---
# 关于 override 对于成员函数引⽤限定

如果我们想写⼀个函数只接受左值实参，我们的声明可以包含⼀个左值引⽤形参：

```cpp
void doSomething(Widget& w); // 只接受左值Widget对象
```

如果我们想写⼀个函数只接受右值实参，我们的声明可以包含⼀个右值引⽤形参：

```cpp
void doSomething(Widget&& w); // 只接受右值Widget对象
```

成员函数的引⽤限定可以很容易的区分哪个成员函数被对象调⽤（即 `*this`）。它和在成员函数声明尾部添加⼀个 `const` 暗⽰该函数的调⽤者（即 `*this`）是 `const` 很相似。

对成员函数添加引⽤限定不常⻅，但是可以⻅。

举个例⼦，假设我们的 `Widget` 类有⼀个 `std::vector` 数据成员，我们提供⼀个范围函数让用户可以直接访问它：

```cpp
class Widget {
public:
    using DataType = std::vector<double>;
    …
    DataType& data() { return values; }
    …
private:
    DataType values;
};
```

这是最具封装性的设计，只给外界保留⼀线光。但先把这个放⼀边，思考⼀下下⾯的客⼾端代码：

```cpp
Widget w;
…
auto vals1 = w.data(); // 拷⻉w.values到vals1
```

`Widget::data` 函数的返回值是⼀个左值引⽤（准确的说是 `std::vector<double>&`）,因为左值引⽤是左值，`vals1` 从左值初始化，因此它由 `w.values` 拷⻉构造而得，就像注释说的那样。

现在假设我们有⼀个创建 `Widgets` 的⼯⼚函数，

```cpp
Widget makeWidget();
```

我们想⽤ `makeWidget` 返回的 `std::vector` 初始化⼀个变量：

```cpp
auto vals2 = makeWidget().data(); // 拷⻉Widget⾥⾯的值到vals2
```

再说⼀次， `Widgets::data` 返回的是左值引⽤，左值引⽤是左值。所以，我们的对象(vals2)⼜得从 `Widget` ⾥的 `values` 拷⻉构造。这⼀次， `Widget` 是 `makeWidget` 返回的临时对象（即右值），所以将其中的 `std::vector` 进⾏拷⻉纯属浪费。最好是移动，但是因为 `data` 返回左值引⽤，C++ 的规则要求编译器不得不⽣成⼀个拷⻉。

我们需要的是指明当 `data` 被右值 `Widget` 对象调⽤的时候结果也应该是⼀个右值。

现在就可以使⽤引⽤限定写⼀个重载函数来达成这⼀⽬的：

```cpp
class Widget {
public:
using DataType = std::vector<double>;
…
DataType& data() & // 对于左值Widgets,
{ return values; } // 返回左值
DataType data() && // 对于右值Widgets,
{ return std::move(values); } // 返回右值
…
private:
DataType values;
};
```

注意 `data` 重载的返回类型是不同的，左值引⽤重载版本返回⼀个左值引⽤，右值引⽤重载返回⼀个临时对象。这意味着现在客⼾端的⾏为和我们的期望相符了：

```cpp
auto vals1 = w.data(); //调⽤左值重载版本的Widget::data，拷⻉构造vals1
auto vals2 = makeWidget().data(); //调⽤右值重载版本的Widget::data, 移动构造vals2
```

这个条款的中⼼是只要你在派⽣类声明的是想要重写基类虚函数的函数，就加上 `override`。

总结：
- 为重载函数加上 `override`
- 成员函数限定让我们可以区别对待左值对象和右值对象（即 `*this`）
