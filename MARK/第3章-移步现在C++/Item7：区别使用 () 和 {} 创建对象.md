---
title: Item 7
tags: ["现代C++", "初始化"]
categories: [Effective Modern C++]
description: '区别使⽤ () 和 {} 创建对象'
date: 2024-07-17 17:25:21
cover: https://s2.loli.net/2024/07/19/z3JmjnIsgQBhYRq.jpg
---

# 区别使用 () 和 {} 创建对象 

C++ 11 有三种初始化对象的语法选择，一般来说初始化值要用 () 或者 {} 括起来或者放到 = 的右边

```cpp
int x(0); //使⽤小括号初始化

int y = 0; //使⽤"="初始化

int z{0}; //使⽤花括号初始化
```

在很多情况下，可以使用 = 和 {} 的组合

```cpp
int z = {0};
```

在接下里的笔记里，忽略 = 和 {} 的组合初始化语法，因为 C++ 通常把它视作和只有 {} 一样。

混乱地使用 = 初始化可能会有一些误导，让别人以为这里是赋值运算符。对于像 `int` 这样的内置类型，研究两者区别是没有多⼤意义的，但是对于 **⽤⼾定义的类型** 而⾔，**区别赋值运算符和初始化就⾮常重要了**，因为这可能包含不同的函数调⽤：

```cpp
Widget w1; //调⽤默认构造函数

Widget w2 = w1; //不是赋值运算符，调⽤拷⻉构造函数

w1 = w2; //是⼀个赋值运算符，调⽤operator=函数
```

甚至在 C++98 中有一些情况没办法去表达初始化。举个例子，要想直接表⽰⼀些，存放⼀个特殊值的STL容器是不可能的。

C++11使⽤统⼀初始化来整合这些混乱且繁多的初始化语法，所谓统⼀初始化是指使⽤单⼀初始化语法在任何地⽅*(初始化表达式存在的地⽅)*表达任何东西。它基于花括号。统⼀初始化是⼀个概念上的东西，而括号初始化是⼀个具体语法构型。

括号初始化让你可以表达以前表达不出的东西。使⽤花括号，指定⼀个容器的元素变得很容易：

```cpp
std::vector<int> v{1,3,5}; //v包含1,3,5
```

括号初始化也能被⽤于**为⾮静态数据成员指定默认初始值**。C++11 允许 "=" 初始化也拥有这种能⼒：

```cpp
class Widget {
...
private:
    int x{0}; //没问题，x初始值为0
    int y = 0; //同上
    int z(0); //错误！
}
```

另⼀⽅⾯，**不可拷⻉的对象**可以使⽤花括号初始化或者小括号初始化，但是不能使⽤"="初始化：

```cpp
std::vector<int> ai1{0}; //没问题，x初始值为0
std::atomic<int> ai2(0); //没问题
std::atomic<int> ai3 = 0; //错误！
```

因此我们很容易理解为什么括号初始化⼜叫统⼀初始化，在 C++ 中这三种⽅式都被指派为初始化表达式，但是**只有括号任何地⽅都能被使⽤**。

花括号表达式有⼀个异常的特性，它不允许内置类型隐式的窄化转换。如果⼀个使⽤了花括号初始化的表达式的值的类型与某个对象类型不匹配，代码就不会通过编译：

```cpp
double x,y,z;

int sum1{x+y+z}; //错误！三个double的和不能⽤来初始化int类型的变量
```

使⽤小括号和 "=" 的初始化不检查是否转换为变窄转换，因为由于历史遗留问题它们必须要兼容⽼旧代码

```cpp
double x,y,z;

int sum2(x + y +z);     //可以（表达式的值被截为int）

int sum3 = x + y + z;   //可以，同上
```

还有一个值得注意的点是：是括号表达式有对于C++最令⼈头疼的解析问题。

C++ 规定任何能被**决议**为⼀个声明的东西就**一定被决议为声明**。比如想创建⼀个使⽤默认构造函数构造的对象，却不小⼼变成了函数声明。

问题的根源是如果你想使⽤⼀个实参，调⽤⼀个构造函数，你可以这样做：

```cpp
Widget w1(10); //使⽤实参10调⽤Widget的⼀个构造函数
```

但是如果你尝试使⽤⼀个没有参数的构造函数构造对象，它就会变成函数声明：

```cpp
Widget w2(); //最令⼈头疼的解析！声明⼀个函数w2，返回Widget
```

由于函数声明中形参列表不能使⽤花括号，所以**使⽤花括号初始化表明你想调⽤默认构造函数构造对象**就没有问题：

```cpp
Widget w3{}; //调⽤没有参数的构造函数构造对象
```

---

但是花括号初始化有时会有一些出乎意料的行为：比如 Item2 中解释的当 `auto` 声明的变量使⽤花括号初始化，变量就会被推导为 `std::initializer_list`，相同内容的其他初始化⽅式会产⽣正常的结果。

在构造函数调⽤中，只要不包含 `std::initializer_list` 参数，那么花括号初始化和小括号初始化都会产⽣⼀样的结果：

```cpp
class Widget {
public:
    Widget(int i, bool b); //未声明默认构造函数
    widget(int i, double d); // std::initializer_list参数
…
};

Widget w1(10, true); // 调⽤构造函数
Widget w2{10, true}; // 同上
Widget w3(10, 5.0);  // 调⽤第⼆个构造函数
Widget w4{10, 5.0};  // 同上
```

如果有⼀个或者多个构造函数的参数是 `std::initializer_list` 使⽤括号初始化语法绝对⽐传递⼀个 `std::initializer_list` 实参要好。而且只要某个调⽤能使⽤括号表达式编译器就会使⽤它。如果上⾯的Widget的构造函数有⼀个 `std::initializer_list` 实参，就像这样：

```cpp
class Widget {
public:
Widget(int i, bool b); // 同上
Widget(int i, double d); // 同上
Widget(std::initializer_list<long double> il); //新添加的
…
};

Widget w1(10, true); // 使⽤小括号初始化
                    //调⽤第⼀个构造函数

Widget w2{10, true}; // 使⽤花括号初始化
                    // 调⽤第⼆个构造函数
                    // (10 和 true 转化为long double)

Widget w3(10, 5.0); // 使⽤小括号初始化
                    // 调⽤第⼆个构造函数

Widget w4{10, 5.0}; // 使⽤花括号初始化
                    // 调⽤第⼆个构造函数
                    // (10 和 5.0 转化为long double)
```

`w2` 和 `w4` 将会使⽤新添加的构造函数构造，即使另⼀个⾮ `std::initializer_list` 构造函数对于实参是更好的选择。甚⾄普通的构造函数和移动构造函数都会被 `std::initializer_list` 构造函数劫持：

```cpp
class Widget {
public:
    Widget(int i, bool b);
    Widget(int i, double d);
    Widget(std::initializer_list<long double> il);
    operator float() const; // convert to float (译者注：⾼亮)
};

Widget w5(w4); // 使⽤小括号，调⽤拷⻉构造函数

Widget w6{w4}; // 使⽤花括号，调⽤std::initializer_list构造函数

Widget w7(std::move(w4)); // 使⽤小括号，调⽤移动构造函数

Widget w8{std::move(w4)}; // 使⽤花括号，调⽤std::initializer_list构造函数
```

编译器热衷于把花括号初始化与使 `std::initializer_list` 构造函数匹配了，热衷程度甚⾄超过了最佳匹配。⽐如：

```cpp
class Widget {
public:
    Widget(int i, bool b);
    Widget(int i, double d);
    Widget(std::initializer_list<bool> il); // element type is now bool
…                                             // no implicit conversion funcs
};

Widget w{10, 5.0}; //错误！要求变窄转换
```

这⾥，编译器会直接忽略前⾯两个构造函数，然后尝试调⽤第三个构造函数，即 `std::initializer_list` 的构造函数。调⽤这个函数将会把 `int(10)` 和 `double(5.0)` 转换为 `bool` ，由于括号初始化拒绝变窄转换，所以这个调⽤⽆效，代码⽆法通过编译。

只有当没办法把花括号初始化中实参的类型转化为 `std::initializer_list` 时，编译器才会回到正常的函数决议流程中。

⽐如我们在构造函数中⽤ `std::initializer_list<std::string>` 代替
 `std::initializer_list<bool>` ，这时⾮ `std::initializer_list` 构造函数将再次成为函数决议的候选者，因为没有办法把 `int` 和 `bool` 转换为 `std::string` :

 ```cpp
class Widget {
public:
    Widget(int i, bool b);
    Widget(int i, double d);
    Widget(std::initializer_list<std::string> il);
…
};

Widget w1(10, true); // 使⽤小括号初始化，调⽤第⼀个构造函数

Widget w2{10, true}; // 使⽤花括号初始化，调⽤第⼀个构造函数

Widget w3(10, 5.0); // 使⽤小括号初始化，调⽤第⼆个构造函数

Widget w4{10, 5.0}; // 使⽤花括号初始化，调⽤第⼆个构造函数
```

代码的⾏为和我们刚刚的论述如出⼀辙。这⾥还有⼀个有趣的边缘情况。假如你使⽤的花括号初始化是空集，并且你欲构建的对象有默认构造函数，也有 `std::initializer_list` 构造函数。

空的花括号意味着什么?如果它们意味着没有实参，就该使⽤默认构造函数，但如果它意味着⼀个空的 `std::initializer_list`，就该调⽤ `std::initializer_list` 构造函数。

最终会调⽤默认构造函数。空的花括号意味着没有实参，不是⼀个空的 `std::initializer_list`： 

```cpp
class Widget {
public:
    Widget();
    Widget(std::initializer_list<int> il);
    ...
};

Widget w1; // 调⽤默认构造函数

Widget w2{}; // 同上

Widget w3(); // 最令⼈头疼的解析！声明⼀个函数
```

如果你想调⽤ `std::initializer_list` 构造，你就得创建⼀个空花括号的实参来表明你想调⽤⼀个 `std::initializer_list` 构造函数，它的实参是⼀个空值。

```cpp
Widget w4({}); // 调⽤std::initializer_list
Widget w5{{}}; // 同上
```

此时，括号初始化的晦涩规则，`std::initializer_list` 和构造函数重载就会⼀下⼦涌进你的脑袋，你可能会想研究了半天这些东西在你的⽇常编程中到底占多⼤⽐例。

---

可能⽐你想象的要多。因为 `std::vector` 也会受到影响。`std::vector` 有⼀个⾮ `std::initializer_list` 构造函数允许你去指定容器的初始⼤小，以及使⽤⼀个值填满你的容器。但它也有⼀个 `std::initializer_list` 构造函数允许你使⽤花括号⾥⾯的值初始化容器。如果你创建⼀个数值类型的 `vector` ，然后你传递两个实参。把这两个实参放到小括号和放到花括号中是不同：

```cpp
std::vector<int> v1(10, 20);    //使⽤⾮ std::initializer_list
                                //构造函数创建⼀个包含10个元素的 std::vector
                                //所有的元素的值都是20

std::vector<int> v2{10, 20};    //使⽤ std::initializer_list
                                //构造函数创建包含两个元素的 std::vector
                                //元素的值为10和20
```

第⼀，作为⼀个类库作者，你需要意识到如果你的⼀堆构造函数中重载过⼀个或者多个
 `std::initializer_list`，⽤⼾代码如果使⽤了括号初始化，可能只会看到你重载的 `std::initializer_list` 这⼀个版本的构造函数。因此，你最好把你的构造函数设计为不管⽤⼾是小括号还是使⽤花括号进⾏初始化都不会有什么影响。换句话说，现在看到 `std::vector` 设计的缺点以后你设计的时候避免它。

如果⼀个类没有 `std::initializer_list` 构造函数，然后你添加⼀个，⽤⼾代码中如果使⽤括号初始化可能会发现过去被决议为⾮ `std::initializer_list` 构造函数现在被决议为新的函数。当然，这种事情也可能发⽣在你添加⼀堆重载函数的时候， `std::initializer_list` 重载不会和其他重载函数⽐较，它直接盖过了其它重载函数，其它重载函数⼏乎不会被考虑。所以如果你要使⽤ `std::initializer_list` 构造函数，请三思而后⾏。

第⼆个，作为⼀个类库使⽤者，你必须认真的在花括号和小括号之间选择⼀个来创建对象。
⼤多数开发者都使⽤其中⼀种作为默认情况，只有当他们不能使⽤这种的时候才会考虑另⼀种。如果使⽤默认使⽤花括号初始化，会得到⼤范围适⽤⾯的好处，它禁⽌变窄转换，免疫 C++ 最令⼈头疼的解析问题

---

如果你是⼀个模板的作者，花括号和小括号创建对象就更⿇烦了。通常不能知晓哪个会被使⽤。举个例⼦，假如你想创建⼀个接受任意数量的参数，然后⽤它们创建⼀个对象。使⽤可变参数模板(variadic template )可以⾮常简单的解决：

```cpp
template<typename T, typename...Args>
void doSomeWork(Args&&... args){
    create local T object from params...
}
```

在现实中我们有两种⽅式使⽤这个伪代码：

```cpp
T localObject(std::forward<Ts>(params)...); // 使⽤小括号
T localObject{std::forward<Ts>(params)...}; // 使⽤花括号
```

考虑这样调用代码：

```cpp
std::vector<int> v;
…
doSomeWork<std::vector<int>>(10, 20);
```

如果doSomeWork创建localObject时使⽤的是小括号，std::vector就会包含10个元素。
如果doSomeWork创建localObject时使⽤的是花括号，std::vector就会包含2个元素。

哪个是正确的？doSomeWork的作者不知道，只有调⽤者知道。这正是标准库函数 `std::make_unique` 和 `std::make_shared` ⾯对的问题。它们的解决⽅案是使⽤小括号，并被记录在⽂档中作为接口的⼀部分。

