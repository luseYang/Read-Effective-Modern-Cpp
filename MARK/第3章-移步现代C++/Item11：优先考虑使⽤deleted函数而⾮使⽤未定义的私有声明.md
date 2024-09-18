---
title: Item 11 优先考虑delete
tags: ["现代C++", "delete", "作用域"]
categories: [Effective Modern C++]
description: '优先考虑delete'
date: 2024-07-21 15:10:21
cover: https://s2.loli.net/2024/07/21/fYeJAjM9On7V5uy.jpg
---

# 条款11：delete 和私有声明
当写的代码不想被其他人调用的时候，通常不会声明这个函数，但是有时 C++ 会自动声明一些函数，如果你想防⽌客⼾调⽤这些函数，事情就不那么简单了。

上述场景⻅于特殊的成员函数，即当有必要时C++⾃动⽣成的那些函数。Item17 详细讨论了这些函数，但是现在，我们只关⼼**拷⻉构造函数和拷⻉赋值运算符重载**。

在 C++98 中防止调用这些函数的方法是将他们声明为私有的成员函数。举个例子：

> 在 C++ 标准库 `iostream` 继承链的顶部是模板类 `basic_ios`。所有 `istream` 和 `ostream` 类都继承此类(直接或者间接)。拷⻉ `istream` 和 `ostream` 是不合适的，因为要进⾏哪些操作是模棱两可的。⽐如⼀个 `istream` 对象，代表⼀个输⼊值的流，流中有⼀些已经被读取，有⼀些可能⻢上要被读取。如果⼀个 `istream` 被拷⻉，需要像拷⻉将要被读取的值那样也拷⻉已经被读取的值吗？解决这个问题最好的⽅法是不定义这个操作。直接禁⽌拷⻉流。

要使 `istream` 和 `ostream` 类不可拷⻉，`basic_ios` 在C++98中是这样声明的(包括注释)：

```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
    …
private:
    basic_ios(const basic_ios& ); // not defined
    basic_ios& operator=(const basic_ios&); // not defined
};
```

将它们声明为私有成员可以防⽌客⼾端调⽤这些函数。故意不定义它们意味着假如还是有代码⽤它们就会在链接时引发缺少函数定义(missing function definitions)错误。

在 C++11 中有⼀种更好的⽅式，只需要使⽤相同的结尾：⽤ `= delete` 将拷⻉构造函数和拷⻉赋值运算符标记为 `deleted` 函数。上⾯相同的代码在 C++11 中是这样声明的：

```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
    …
    basic_ios(const basic_ios& ) = delete;
    basic_ios& operator=(const basic_ios&) = delete;
    …
};
```

删除这些函数(译注：添加 `= delete`)和声明为私有成员可能看起来只是⽅式不同，别⽆其他区别。其实还有⼀些实质性意义。 `deleted` 函数不能以任何⽅式被调⽤，即使你在成员函数或者友元函数⾥⾯调⽤ `deleted` 函数也不能通过编译。这是较之 C++98 ⾏为的⼀个改进，**后者不正确的使⽤这些函数在链接时才被诊断出来。**

通常， **`deleted` 函数被声明为 `public` 而不是 `private`**.这也是有原因的。当客⼾端代码试图调⽤成员函数时，C++ 会**在检查 `deleted` 状态前检查它的访问性。**如果客⼾端代码调⽤⼀个私有的 `deleted` 函数，**⼀些编译器只会给出该函数是 `private` 的错误** (译注：而没有诸如该函数被 `deleted` 修饰的错误)，即使函数的访问性不影响它的使⽤。如果要将⽼代码的"私有且未定义"函数替换为 `deleted` 函数时请⼀并修改它的访问性为 `public`，这样可以让编译器产⽣更好的错误信息。

`deleted` 函数还有⼀个重要的优势是任何函数都可以标记为 `deleted`，而只有 `private` 只能修饰成员函数。假如我们有⼀个⾮成员函数，它接受⼀个整型参数，检查它是否为幸运数：

```cpp
bool isLucky(int number);
```

C++ 有沉重的 C 包袱，使得**含糊的、能被视作数值的任何类型都能隐式转换为 `int`**，但是有⼀些调⽤可能是没有意义的：

```cpp
if (isLucky('a')) … // 字符'a'是幸运数？
if (isLucky(true)) … // "true"是?
if (isLucky(3.5)) … // 难道判断它的幸运之前还要先截尾成3？
```

如果幸运数必须真的是整数，我们该禁⽌这些调⽤通过编译。其中⼀种⽅法就是创建 deleted 重载函数，其参数就是我们想要过滤的类型：

``` cpp
bool isLucky(int number); // 原始版本
bool isLucky(char) = delete; // 拒绝char
bool isLucky(bool) = delete; // 拒绝bool
bool isLucky(double) = delete; // 拒绝float和double
```


上⾯ `double` 重载版本的注释说拒绝 `float` 和 `double` 可能会让你惊讶，但是请回想⼀下：将 `float` 转换为 `int` 和 `double`，C++ 更喜欢转换为 `double` 。使⽤ `floa`t 调⽤ `isLucky` 因此会调⽤ `double` 重载版本，
而不是 `int` 版本。好吧，它也会那么去尝试。事实是调⽤被删除的 `double` 重载版本不能通过编译。

---

# delete 的其他作用
另⼀个 deleted 函数⽤武之地（private成员函数做不到的地⽅）是禁⽌⼀些模板的实例化。
假如你要求⼀个模板仅⽀持原⽣指针（尽管第四章建议使⽤智能指针代替原⽣指针）

```cpp
template<typename T>
void processPointer(T* ptr);
```

在指针的世界⾥有两种特殊情况。**⼀是 `void*` 指针，因为没办法对它们进⾏解引⽤，或者加加减减等。另⼀种指针是 `char*`，因为它们通常代表 C ⻛格的字符串，而不是正常意义下指向单个字符的指针。**这两种情况要特殊处理，在 `processPointer` 模板⾥⾯，我们假设正确的函数应该拒绝这些类型。也即是说， `processPointer` 不能被 `void*` 和 `char*` 调⽤。要想确保这个很容易，使⽤ `delete` 标注模板实例：

```cpp
template<>
void processPointer<void>(void*) = delete;
template<>
void processPointer<char>(char*) = delete;
```

现在如果使⽤ `void*` 和 `char*` 调⽤ `processPointer` 就是⽆效的，按常理说 `const void*` 和 `const void*` 也应该⽆效，所以这些实例也应该标注 `delete` :

```cpp
template<>
void processPointer<const void>(const void*) = delete;

template<>
void processPointer<const char>(const char*) = delete;
```

有趣的是，如果的类⾥⾯有⼀个函数模板，你可能想⽤ `private` `（经典的 C++98 惯例）来禁⽌这些函数模板实例化，但是不能这样做，因为不能给特化的模板函数指定⼀个不同（于函数模板）的访问级别。

```cpp
class Widget {
public:
    …
    template<typename T>
    void processPointer(T* ptr)
    { … }
private:
    template<> // 错误！
    void processPointer<void>(void*);
};
```

问题是模板特例化必须位于⼀个命名空间作⽤域，而不是类作⽤域。 delete 不会出现这个问题，因为
它不需要⼀个不同的访问级别，且他们可以在类外被删除（因此位于命名空间作⽤域）：

```cpp
class Widget {
public:
    …
    template<typename T>
    void processPointer(T* ptr)
    { … }
    …
};
template<>
void Widget::processPointer<void>(void*) = delete; // 还是public，但是已经被删除了
```

总结：
- ⽐起声明函数为 `private` 但不定义，使⽤ `delete` 函数更好
- 任何函数都能 `delete` ，包括⾮成员函数和模板实例
- 函数模板意指未特化前的源码，模板函数则倾向于模板实例化后的函数