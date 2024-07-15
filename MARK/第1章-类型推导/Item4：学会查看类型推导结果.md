# IDE编辑器

在IDE中的代码编辑器通常可以显⽰程序代码中变量，函数，参数的类型，你只需要简单的把⿏标移到它们的上⾯，比如：

```cpp
const int theAnswer = 42;

auto x = theAnswer;
auto y = &theAnswer;
```

一个IDE编辑器可以直接显示 `x` 的推导结果为 `int` ，`y` 的推导结果为 `const int*` ，为此，代码应该尽可能处于可编译状态，因为IDE之所以能提供这些信息是因为⼀个C++编译器（或者⾄少是前端中的⼀个部分）运⾏于IDE中。如果这个编译器对你的代码不能做出有意义的分析或者推导，它就不会显⽰推导的结果。

# 编译器诊断

另⼀个获得推导结果的⽅法是使⽤编译器出错时提供的错误消息。这些错误消息⽆形的提到了造成我们编译错误的类型是什么。

比如：

```cpp
template<typename T> //只对TD进⾏声明
class TD; //TD == "Type Displayer"
```

如果这时候尝试实例化这个类模板，就会出现一个错误消息，因为这里没有用来实例化的类模板定义。为了查看 `x` 和 `y` 的类型，只需要使⽤它们的类型去实例化 `TD`：

```cpp
TD<decltype(x)> xType; //引出错误消息
TD<decltype(y)> yType; //x和y的类型
```

这时候的报错信息就是：

```cpp
error: aggregate 'TD<int> xType' has incomplete type and
    cannot be defined
error: aggregate 'TD<const int *> yType' has incomplete type and
    cannot be defined
```

也可以通过这种方式查看类型

# 运行时输出

使⽤ `printf` 的⽅法使类型信息只有在运⾏时才会显⽰出来（尽管我不是⾮常建议你使⽤ `printf` ），但是它提供了⼀种格式化输出的⽅法。现在唯⼀的问题是只需对于你关⼼的变量使⽤⼀种优雅的⽂本表⽰。“这有什么难的“，你这样想”这正是 `typeid` 和 `std::type_info::name` 的价值所在”。为了实现我们我们想要查看 `x` 和 `y` 的类型的需求，你可能会这样写：

```cpp
std::cout<<typeid(x).name()<<"\n"; //显⽰x和y的类型
std::cout<<typeid(y).name()<<"\n";
```

这种⽅法对⼀个对象如 `x` 或 `y` 调⽤ `typeid` 产⽣⼀个 `std::type_info` 的对象，然后 `std::type_info` ⾥⾯的成员函数 `name()` 来产⽣⼀个 C ⻛格的字符串表⽰变量的名字。

但是调⽤ `std::type_info::name` 不保证返回任何有意义的东西，但是库的实现者尝试尽量使它们返回的结果有⽤。

更复杂的例子：

```cpp
template<typename T>
void f(const T& param);

std::vector<Widget> createVec();

const auto vw = createVec();

if(!vw.empty()){
    f(&vw[0]);
    ...
}
```

在上面这段代码中包含了⼀个⽤⼾定义的类型 `Widget`，⼀个 STL 容器和⼀个 `auto` 变量 `vw`，这个更现实的情况是你可能在会遇到的并且想获得他们类型推导的结果，⽐如模板类型参数 `T`，⽐如函数参数 `param`。从这⾥中我们不难看出 `typeid` 的问题所在。我们添加⼀些代码来显⽰类型：

```cpp
template<typename T>
void f(const T& param){
    using std::cout;
    cout<<"T= "<<typeid(T).name()<<"\n";
    cout<<"param = "<<typeid(param).name()<<"\n";
    ...
}
```

GNU 和 Clang 执⾏这段代码将会输出这样的结果

```cpp
T= PK6Widget
param= PK6Widget
```

