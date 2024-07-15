# 理解decltype

**`decltype`** 可以得出一个名字或者一个表达式的类型，通常是一个精确的结果，但有时候也会出错。

简单的例子：

```cpp
const int i=0; //decltype(i)是const int

bool f(const Widget& w); //decltype(w)是const Widget&
                            //decltype(f)是bool(constWidget&)
struct Point{
    int x; //decltype(Point::x)是int
    int y; //decltype(Point::y)是int
};

template<typename T>
class Vector{
    ...
    T& operator[](std::size_t index);
    ...
}

vector<int> v; //decltype(v)是vector<int>
...
if(v[0]==0) //decltype(v[0])是int&
```

在 C++11 中，`decltype` 最主要的用途就是用于函数模版返回类型，而这个返回类型依赖形参。假定写一个函数，一个参数为容器，一个参数为索引值，这个函数支持使用方括号的方式访问容器中指定索引值的数据，然后**在返回索引操作的结果前执⾏认证⽤⼾操作**。函数的返回类型应该和索引操作返回的类型相同。

对于一个 `T` 类型的容器使用 `operator[]` 通常会返回一个 `T&` 对象，⽐如 `std::deque` 就是这样，但是 `std::vector<bool>` 有⼀个例外，对于 `std::vector`，`operator[]` 不会返回 `bool&` ，它会返回一个有名字的对象类型，（译注：MSVC 的 STL 实现中返回的是  `std::Vb_reference<std::Wrap_alloc<std::allocator>>`）。**这⾥重要的是我们可以看到对⼀个容器进⾏ `operator[]`作返回的类型取决于容器本⾝。**

```cpp
#include <iostream>
#include <vector>

int main() {
	std::vector<bool> vec{ 0,0,0 };
	std::vector<int> vec1{ 1 };
	decltype(vec) vecc = vec;
	std::cout << typeid(vec1[0]).name() << ' ' << typeid(vecc[0]).name();
}
```

打印可以看到结果。

使⽤ `decltype` 使得我们很容易去实现它，这是我们写的第⼀个版本，使⽤ `decltype` 计算返回类型，这个模板之后会 ***进行改良***：

```cpp
template<typename Container,typename Index>
auto authAndAccess(Container& c,Index i)->decltype(c[i])
{
    authenticateUser();
    return c[i];
}
```

通过模版板块的学习，我们得知：函数名称前面的 `auto` 不做任何操作，他在这行代码中志气到了*占位符*的功能，暗示使用了 C++11 的**返回类型后置**语法。即在函数形参列表后⾯使⽤⼀个 `->` 符号指出函数的返回类型，尾置返回类型的好处是我们可以在函数返回类型中使⽤函数参数相关的信息。在 `authAndAccess` 函数中，我们指定返回类型使⽤ `c` 和 `i` 。如果我们按照传统语法把函数返回类型放在函数名称之前， `c` 和 `i` 就未被声明所以不能使⽤。

`authAndAccess` 函数返回 `operator[]` 应⽤到容器中返回的对象的类型，这也正是我们期望的结果。

C++11 允许⾃动推导单⼀语句的 `lambda` 表达式的返回类型， C++14 扩展到允许⾃动推导所有的 `lambda` 表达式和函数，甚⾄它们内含多条语句。对于 `authAndAccess` 来说这意味着在 C++14 标准下我们可以忽略尾置返回类型，只留下⼀个 `auto`。**在这种形式下 `auto` 不再进⾏ `auto` 类型推导，取而代之的是它意味着编译器将会从函数实现中推导出函数的返回类型。**

```cpp
template<typename Container,typename Index> //C++ 14版本
auto authAndAccess(Container& c,Index i)
{
    return c[i];
}
```

Item2 解释了函数返回类型中使⽤ `auto` 编译器实际上是使⽤的模板类型推导的那套规则。如果那样的话就会这⾥就会有⼀些问题，正如我们之前讨论的，`operator[]` 对于⼤多数 `T` 类型的容器会返回⼀个 `T&` , 但是 Item1 解释了在模板类型推导期间，如果表达式是⼀个引⽤那么引⽤会被忽略。基于这样的规则，考虑它会对下⾯⽤⼾的代码有哪些影响：

```cpp
std::deque<int> d;
...
authAndAccess(d,5)=10; //认证⽤⼾，返回d[5]，
                        //然后把10赋值给它
                        //⽆法通过编译器！
```

在这⾥ `d[5]` 本该返回⼀个 `int&` ，但是模板类型推导会剥去引⽤的部分，因此产⽣了 `int` 返回类型。函数返回的值是⼀个右值，上⾯的代码尝试把 10 赋值给右值,C++11 禁⽌这样做，所以代码⽆法编译。

要想让 `authAndAccess` 像我们期待的那样⼯作，我们需要使⽤ `decltype` 类型推导来推导它的返回值，⽐如指定 `authAndAccess` 应该返回⼀个和 `c[i]` 表达式类型⼀样的类型。C++ 期望在某些情况下当类型被暗⽰时需要使⽤ `decltype` 类型推导的规则，C++14 通过使⽤ `decltype(auto)` 说明符使得这成为可能。

我们第⼀次看⻅ `decltype(auto)` 可能觉得⾮常的⽭盾，（到底是 `decltype` 还是 `auto`？），实际上我们可以这样解释它的意义：`auto` 说明符表⽰这个类型将会被推导，`decltype` 说明以 `decltype` 的推导规则引⽤到这个推导过程中。因此我们可以这样写 `authAndAccess`：

```cpp
template<typename Container,typename Index>
decltype(auto)
authAndAccess(Container& c,Index i)
{
    authenticateUser();
    return c[i];
}
```

`decltype(auto)` 的使⽤不仅仅局限于函数返回类型，当你想对初始化表达式使⽤ `decltype` 推导的规则，你也可以使⽤：

```cpp
Widget w;

const Widget& cw = w;

auto myWidget1 = cw;    //auto类型推导
                        //myWidget1的类型为Widget

decltype(auto) myWidget2 = cw;  //decltype类型推导
                                //myWidget2的类型是const Widget&
```

现在来看一下上文提到的 `authAndAccess` 的改良（C++14）。

```cpp
template<typename Container,typename Index>
decltype(auto) authAndAccess(Container& c,Index i);
```

容器通过传引⽤的⽅式传递⾮常量左值引⽤，因为返回⼀个引⽤允许⽤⼾可以修改容器但是这意味着在这个函数⾥⾯不能传值调⽤，右值不能被绑定到左值引⽤上（除⾮这个左值引⽤是⼀个 `const` ，但是这⾥明显不是）。

公认的向 `authAndAccess` 传递⼀个右值是⼀个 `edge case` 。⼀个右值容器，是⼀个临时对象，通常会在 `authAndAccess` 调⽤结束被销毁，这意味 `authAndAccess` 返回的引⽤将会成为⼀个悬置的 `(dangle)` 引⽤。但是使⽤向 `authAndAccess` 传递⼀个临时变量也并不是没有意义，有时候⽤⼾可能只是想简单的获得临时容器中的⼀个元素的拷⻉，⽐如这样：

```cpp
std::deque<std::string> makeStringDeque(); //⼯⼚函数

//从makeStringDeque中或得第五个元素的拷⻉并返回
auto s = authAndAccess(makeStringDeque(),5);
```

要想⽀持这样使⽤ `uthAndAccess` 我们就得修改⼀下当前的声明使得它⽀持左值和右值重载是⼀个不错的选择（⼀个函数重载声明为左值引⽤，另⼀个声明为右值引⽤），但是我们就不得不维护两个重载函数。另⼀个⽅法是使 `authAndAccess` 的引⽤可以绑定左值和右值，Item24 解释了那正是通⽤引⽤(万能引用)能做的，所以我们这⾥可以使⽤通⽤引⽤进⾏声明：

```cpp
template<typename Container,typename Index> //最终的C++14版本
decltype(auto)
authAndAccess(Container&& c,Index i){
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

---

在极少数情况下，`decltype` 可能得不到想要的结果：

对⼀个名字使⽤ `decltype` 将会产⽣这个名字被声明的类型。名字是左值表达式，但那不影响 `decltype` 的⾏为，`decltype` 确保产⽣的类型总是左值引⽤。换句话说，如果⼀个左值表达式除了名字外还有类型，那么 `decltype` 将会产⽣ `T&LEIX` .这⼏乎没有什么太⼤影响，因为⼤多数左值表达式的类型天⽣具备⼀个左值引⽤修饰符。举个例⼦，函数返回左值，⼏乎也返回了左值引⽤。

```cpp
int x = 0;
```

`x` 是⼀个变量的名字，所以 `decltype(x)` 是 `int`。但是如果⽤⼀个小括号包覆这个名字，⽐如这样 `(x)` ，就会产⽣⼀个⽐名字更复杂的表达式。对于名字来说，`x` 是⼀个左值，C++11 定义了表达式 `(x)` 也是⼀个左值。因此 `decltype((x))` 是 `int&` 。⽤小括号覆盖⼀个名字可以改变 `decltype` 对于名字产⽣的结果。

```cpp
decltype(auto) f1()
{
    int x = 0;
    ...
    return x; //decltype(x）是int，所以f1返回int
}

decltype(auto) f2()
{
    int x =0l;
    return (x); //decltype((x))是int&，所以f2返回int&
}
```

注意不仅f2的返回类型不同于f1，而且它还引⽤了⼀个局部变量！

# 总结

- `decltype` 总是不加修改的产⽣变量或者表达式的类型。
- 对于 `T` 类型的左值表达式，`decltype` 总是产出 `T` 的引⽤即 `T&`。
- C++14 ⽀持 `decltype(auto)` ，就像 `auto` ⼀样，推导出类型，但是它使⽤⾃⼰的独特规则进⾏推导。
