## 理解 auto 类型推导

auto 类型推导和模版类型推导有一个直接的映射关系，他们之间可以通过⼀个⾮常规范⾮常系统化的转换流程来转换彼此。

在Item1中，模版类型推导使用下面这个函数模版来解释：

```cpp
template<typename T>
void f(ParmaType param); //使⽤⼀些表达式调⽤f
```

在 `f` 的调⽤中，编译器使⽤ `expr` 推导 `T` 和 `ParamType` 。当⼀个变量使⽤ `auto` 进⾏声明时， `auto` 扮演了模板的⻆⾊，变量的类型说明符扮演了 `ParamType` 的⻆⾊。

```cpp
auto x = 27;

const auto cx = x;

const auto & rx=cx;
```

在这⾥例⼦中要推导 `x rx cx` 的类型，编译器的⾏为看起来就像是认为这⾥每个声明都有⼀个模板，然后使⽤合适的初始化表达式进⾏处理：

```cpp
template<typename T> //理想化的模板⽤来推导x的类型
void func_for_x(T param);

func_for_x(27);
template<typename T> //理想化的模板⽤来推导cx 的类型
void func_for_cx(const T param);

func_for_cx(x);
template<typename T> //理想化的模板⽤来推导rx的类型

void func_for_rx(const T & param);
func_for_rx(x);
```

auto类型推导除了⼀个例外（我们很快就会讨论），其他情况都和模板类型推导⼀样。样auto类型推断和模板类型推导⼀样⼏乎⼀样的⼯作，它们就像⼀个硬币的两⾯。

讨论完相同点接下来就是不同点，前⾯我们已经说到 `auto` 类型推导和模板类型推导有⼀个例外使得它们的⼯作⽅式不同:  
从一个简单的例子开始：

如果你想⽤⼀个int值27来声明⼀个变量，C++98 提供两种选择：

```cpp
int x1=27;
int x2(27);
```

C++11 由于也添加了⽤于⽀持统⼀初始化（uniform initialization）的语法：

```cpp
int x3={27};
int x47{27};
```

总之，这四种不同的语法只会产⽣⼀个相同的结果：变量类型为 `int` 值为27

但是，如果我们把上面的 `int` 全部改为 `auto` ，会得到不一样的结果：

```cpp
auto x1=27;
auto x2(27);
auto x3={27};
auto x4{27};
```

这些声明都能通过编译，但是他们不像替换之前那样有相同的意义。前⾯两个语句确实声明了⼀个类型为 `int` 值为 27 的变量，但是后⾯两个声明了⼀个存储⼀个元素27的 `std::initializer_list<int>` 类型的变量。

这就造就了 `auto`类型推导不同于模版类型推导的特殊情况，当⽤auto声明的变量使⽤花括号进⾏初始化，`auto` 类型推导会推导出 `auto` 的类型为 `std::initializer_list`。如果这样的⼀个类型不能被成功推导（⽐如花括号⾥⾯包含的是不同类型的变量），编译器会拒绝这样的代码！

```cpp
auto x5={1,2,3.0}; //错误！auto类型推导不能⼯作
```

就像注释说的那样，在这种情况下类型推导将会失败，但是对我们来说认识到这⾥确实发⽣了两种类型推导是很重要的。⼀种是由于 `auto` 的使⽤：`x5` 的类型不得不被推导，因为 `x5` 使⽤花括号的⽅式进⾏初始化， `x5` 必须被推导为 `std::initializer_list` ,但是 `std::initializer_list` 是⼀个模板。`std::initializer_list` 会被实例化，所以这⾥ `T` 也会被推导。 另⼀种推导也就是模板类型推导被归⼊第⼆种推导。在这个例⼦中推导之所以出错是因为在花括号中的值并不是同⼀种类型。

当使⽤auto的变量使⽤花括号的语法进⾏初始化的时候，会推导出std::initializer_list的实例化，但是对于模板类型推导这样就⾏不通：

```cpp
auto x={11,23,9}; //x的类型是std::initializer_list<int>

template<typename T>
void f(T param);

f({11,23,9}); //错误！不能推导出T
```

然而如果指定 `T `是 `std::initializer` 而留下未知 `T`,模板类型推导就能正常⼯作：

```cpp
template<typename T>
void f(std::initializer_list<T> initList);

f({11,23,9}); //T被推导为int，initList的类型被推导为std::initializer_list<int>
```

因此 `auto` 类型推导和模板类型推导的真正区别在于 `auto` 类型推导假定花括号表⽰ `std::initializer_list` , 而模板类型推导不会这样（确切的说是不知道怎么办）。

C++14 允许 `auto` ⽤于函数返回值并会被推导（参⻅Item3），而且 C++14 的 `lambda` 函数也允许在形参中使⽤ `auto` 。但是在这些情况下虽然表⾯上使⽤的是 `auto` 但是实际上是模板类型推导的那⼀套规则在⼯作，所以说下⾯这样的代码不会通过编译：

```cpp
auto createInitList()
{
return {1,2,3}; //错误！推导失败
}
```

同样在C++14的lambda函数中这样使⽤auto也不能通过编译：

```cpp
std::vector<int> v;

auto resetV = [&v](const auto & newValue){v=newValue;}; //C++14
...
reset({1,2,3}); //错误！推导失败
```

## 总结

- `auto` 类型推导通常和模板类型推导相同，但是 `auto` 类型推导假定花括号初始化代表 `std::initializer_list` 而模板类型推导不这样做
- 在 C++14 中 `auto` 允许出现在函数返回值或者 `lambda` 函数形参中，但是它的⼯作机制是模板类型推导那⼀套⽅案。
