# 条款一：理解模版类型推导

考虑这样一个函数模版：

```cpp
template<typename T>
void f(ParamType param);
```

它的调⽤看起来像这样

```cpp
f(expr); //使⽤表达式调⽤f
```

在编译期间，编译器使⽤ `expr` 进⾏两个类型推导：⼀个是针对 `T` 的，另⼀个是针对 `ParamType` 的。这两个类型通常是不同的，因为 `ParamType` 包括了 `const` 和引⽤的修饰。举个例⼦，如果模板这样声明：

```cpp
template<typename T>
void f(const T& param);
```

然后这样进⾏调⽤

```cpp
int x = 0;
f(x); //⽤⼀个int类型的变量调⽤f
```

显而易见的 `T` 被推导为 `int` ，`ParamType` 却被推导为 `const int&`

--- 

还是这个例子

```cpp
template<typename T>
void f(ParamType param);

f(expr); //从expr中推导T和ParamType
```

分别有三种情况：
- `ParamType` 是⼀个指针或引⽤，但不是通⽤引⽤（关于通⽤引⽤请参⻅Item24。在这⾥你只需要知道它存在，而且不同于左值引⽤和右值引⽤）
- `ParamType` ⼀个通⽤引⽤
- `ParamType` 既不是指针也不是引⽤

## 情景⼀：ParamType是⼀个指针或引⽤但不是通⽤引⽤

```cpp
template<typename T>
void f(T & param); //param是⼀个引⽤
```

1. 如果 `expr` 的类型是⼀个引⽤，忽略引⽤部分
2. 然后剩下的部分决定 `T` ，然后T与形参匹配得出最终 `ParamType`

```cpp
int x=27; //x是int
const int cx=x; //cx是const int
const int & rx=cx; //rx是指向const int的引⽤

f(x); //T是int，param的类型是int&
f(cx); //T是const int，param的类型是const int &
f(rx); //T是const int，param的类型是const int &
```

结果也在上面了。在第三个例⼦中，注意即使rx的类型是⼀个引⽤，T也会被推导为⼀个⾮引⽤ ，这是因为如上⾯提到的如果expr的类型是⼀个引⽤，将忽略引⽤部分。

## 情景⼆：ParamType是⼀个通⽤引⽤

通用引用就是我们所说的万能引用，在函数模板中假设有⼀个模板参数 `T` ,那么通⽤引⽤就是 `T&&`

- 如果 `expr` 是左值， `T` 和 `ParamType` 都会被推导为左值引⽤。这⾮常不寻常，第⼀，这是模板类型推导中唯⼀⼀种 `T` 和 `ParamType` 都被推导为引⽤的情况。第⼆，虽然 `ParamType` 被声明为右值引⽤类型，但是最后推导的结果它是左值引⽤。
- 如果expr是右值，就使⽤情景⼀的推导规则

```cpp
template<typename T>
void f(T&& param); //param现在是⼀个通⽤引⽤类型

int x=27; //如之前⼀样
const int cx=x; //如之前⼀样
const int & rx=cx; //如之前⼀样

f(x); //x是左值，所以T是int&
        //param类型也是int&
f(cx); //cx是左值，所以T是const int &
        //param类型也是const int&
f(rx); //rx是左值，所以T是const int &
        //param类型也是const int&
f(27); //27是右值，所以T是int
        //param类型就是int&&
```

## 情景三：ParamType既不是指针也不是引⽤

当 `ParamType` 既不是指针也不是引⽤时，我们通过传值（pass-by-value）的⽅式处理：就是普通的 `T`

```cpp
template<typename T>
void f(T param); //以传值的⽅式处理param
```

这意味着⽆论传递什么 `param` 都会成为它的⼀份拷⻉——⼀个完整的新对象。事实上 `param` 成为⼀个新对象这⼀⾏为会影响 `T` 如何从 `expr` 中推导出结果。
1. 和之前⼀样，如果 `expr` 的类型是⼀个引⽤，忽略这个引⽤部分
2. 如果忽略引⽤之后 `expr` 是⼀个 `const` ，那就再忽略 `const` 。如果它是 `volatile` ，也会被忽略（volatile不常⻅，它通常⽤于驱动程序的开发中。关于 `volatile` 的细节请参⻅[Item:40]())

```cpp
int x=27; //如之前⼀样
const int cx=x; //如之前⼀样
const int & rx=cx; //如之前⼀样

f(x); //T和param都是int
f(cx); //T和param都是int
f(rx); //T和param都是int
```

注意即使 `cx` 和 `rx` 有 `const` 属性， `param` 也不是 `const` 。这是有意义的。 `param` 是⼀个拷⻉⾃ `cx` 和 `rx` 且现在独⽴的完整对象。具有常量性的 `cx` 和 `rx` 不可修改并不代表 `param` 也是⼀样不可修改。**这就是为什么 `expr` 的常量性或易变性（volatileness)在类型推导时会被忽略：因为 `expr` 不可修改并不意味着他的拷⻉也不能被修改。**

⼀个常量指针指向 `const` 字符串，在类型推导中这个指针指向的数据的常量性将会被保留，但是指针⾃⾝的常量性将会被忽略。

## 数组实参

虽然说数组和指针有时候是完全等价的，比如：

```cpp
const char name[] = "J. P. Briggs"; //name的类型是const char[13]
const char * ptrToName = name; //数组退化为指针
```

在这⾥ `const char*` 指针 `ptrToName` 会由 `name` 初始化，而 `name` 的类型为 `const char[13]` ，这两种类型( `const char *`  和 `const char[13]` )是不⼀样的，但是由于数组退化为指针的规则，编译器允许这样的代码。

但是一个数组传值给一个模版会怎么样？

```cpp
template<typename T>
void f(T param);

f(name); //对于T和param会产⽣什么样的类型
```

让我们从一个简单的例子开始，这里有一个函数，他的形参是数组：

```cpp
void myFunc(int param[]);
```

但是数组声明会被视作指针声明，这意味着myFunc的声明和下⾯声明是等价的：

```cpp
void myFunc(int *param); //同上
```

这样的等价是 C 语⾔的产物，C++ ⼜是建⽴在 C 语⾔的基础上，它让⼈产⽣了⼀种数组和指针是等价的的错觉。

因为数组形参会视作指针形参，所以传递给模板的⼀个数组类型会被推导为⼀个指针类型。这意味着在模板函数f的调⽤中，它的模板类型参数T会被推导为 `const char*` :

```cpp
f(name); //name是⼀个数组，但是T被推导为const char *
```

但是现在难题来了，虽然函数不能接受真正的数组，但是可以接受指向数组的引⽤！所以我们修改 `f` 为传引⽤：

```cpp
template<typename T>
void f(T& param);
```

`T` 被推导为了真正的数组！这个类型包括了数组的⼤小，在这个例⼦中 `T` 被推导为 `const char[13]`， `param` 则被推导为 `const char(&)[13]` 。是的，这种语法看起来简直有毒，但是知道它将会让你在关⼼这些问题的⼈的提问中获得⼤神的称号。

有趣的是，对模板函数形参声明为⼀个指向数组的引⽤使得我们可以在模板函数中推导出数组的⼤小：

```cpp
//在编译期间返回⼀个数组⼤小的常量值（数组形参没有名字，因为我们只关⼼数组的⼤小）

template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept
{
    return N;
}
```

在Item15提到将⼀个函数声明为 `constexpr` 使得结果在编译期间可⽤。这使得我们可以⽤⼀个花括号声明⼀个数组，然后第⼆个数组可以使⽤第⼀个数组的⼤小作为它的⼤小，就像这样：

```ccpp
int keyVals[] = {1,3,5,7,9,11,22,25}; //keyVals有七个元素
int mappedVals[arraySize(keyVals)]; //mappedVals也有七个
```

## 函数实参

在C++中不⽌是数组会退化为指针，**函数类型也会退化为⼀个函数指针**，我们对于数组的全部讨论都可以应⽤到函数来：

```cpp
void someFunc(int, double); //someFunc是⼀个函数，类型是void(int,double)

template<typename T>
void f1(T param); //传值

template<typename T>
void f2(T & param); //传引⽤

f1(someFunc); //param被推导为指向函数的指针，类型是void(*)(int, double)
f2(someFunc); //param被推导为指向函数的引⽤，类型为void(&)(int, bouel)
```

## 总结

- 在模板类型推导时，有引⽤的实参会被视为⽆引⽤，他们的引⽤会被忽略
- 对于通⽤引⽤的推导，左值实参会被特殊对待
- 对于传值类型推导，实参如果具有常量性和易变性会被忽略
- 在模板类型推导时，数组或者函数实参会退化为指针，除⾮它们被⽤于初始化引⽤
