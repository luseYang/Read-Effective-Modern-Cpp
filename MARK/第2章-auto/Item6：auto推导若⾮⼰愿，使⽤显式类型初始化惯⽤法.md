假设我有一个函数，参数为 `Widget`，返回一个 `std::vector<bool>` ，这⾥的 `bool` 表⽰ `Widget` 是否提供⼀个独有的特性。

```cpp
std::vector<bool> features(const Widget& w);
```

更进⼀步假设表⽰是否 `Widget` 具有⾼优先级，我们可以写这样的代码：

```cpp
bool highPriority = features(w)[5];
...
processWidget(w,highPriority);
```

但是如果我们使⽤auto代替显式指定类型做⼀些看起来很⽆害的改变：

```cpp
auto highPriority = features(w)[5];
...
processWidget(w,highPriority); //未定义⾏为！
```

就像注释说的，这个 `processWidget` 是⼀个未定义⾏为。为什么呢？答案有可能让你很惊讶，使⽤ `auto` 后 `highPriority` 不再是 `bool` 类型。虽然从概念上来说 `std::vector<bool>` 意味着存放bool，但是 `std::vector<bool>` 的 `operator[]` 不会返回容器中元素的引⽤，取而代之它返回⼀个 `std::vector<bool>::reference` 的对象（⼀个嵌套于 `std::vector<bool>` 中的类） 。`std::vector<bool>::reference` 之所以存在是因为 `std::vector<bool>` 指定了它作为代理类。`operator[]` 返回⼀个代理类来扮演 `bool&` 。要想成功扮演这个⻆⾊，`bool&` 适⽤的上下⽂ `std::vector<bool>::reference` 也必须⼀样能适⽤。基于这个特性 `std::vector<bool>::reference` 可以隐式的转化为`bool`（不是 `bool&`，是 `bool！`要想完整的解释 `std::vector<bool>::reference` 能模拟 `bool&` 的⾏为所使⽤的⼀堆技术可能扯得太远了，所以这⾥简单地说隐式类型转换只是这个⼤型⻢赛克的⼀小块）

有了这些信息，我们再来看看原始代码的⼀部分：

```cpp
bool highPriority = features(w)[5]; //显式的声明highPriority的类型
```

这⾥，`feature` 返回⼀个 `std::vector<bool>` 对象后再调⽤ `operator[]` ，` operator[]` 将会返回⼀个 `std::vector<bool>::reference` 对象，然后再通过隐式转换赋值给 `bool` 变量 `highPriority`。`highPriority` 因此表⽰的是 `features` 返回的 `vector` 中的第五个 `bit` ，这也正如我们所期待的那样。然后再对照⼀下当使⽤ `auto` 时发⽣了什么.

```cpp
auto highPriority = features(w)[5]; //推导highPriority的类型
```

同样的，`feature` 返回⼀个 `std::vector<bool>` 对象，再调⽤ `operator[]` ， `operator[]` 将会返回⼀个 `std::vector<bool>::reference` 对象，但是现在这⾥有⼀点变化了，`auto` 推导 `highPriority` 的类型为 `std::vector<bool>::reference` ，但是 `highPriority` 对象没有第五 `bit` 的值。

```cpp
processWidget(w,highPriority); //未定义⾏为！
//highPriority包含⼀个悬置指针
```

## 代理类

所谓代理类，就是以模仿和增强一些类型的行为为目的而存在的类。很多情况下都会使用代理类。 `std::vector<bool>::reference` 展⽰了对 `std::vector<bool>` 使⽤ `operator[]` 来实现引⽤ `bit` 这样的⾏为。

作为⼀个通则，不可⻅的代理类通常不适⽤于 `auto`。这样类型的对象的⽣命期通常不会设计为能活过⼀条语句，所以创建那样的对象你基本上就走向了违反程序库设计基本假设的道路. `std::vector<bool>::reference`  就是这种情况，我们看到违反这个基本假设将导致未定义⾏为

当你不知道这个类型有没有被代理还想使⽤ `auto` 时你就不能单单只⽤⼀个 `auto`。解决⽅案是强制使⽤⼀个不同的类型推导形式，这种⽅法我通常称之为显式类型初始器惯⽤法

显式类型初始器惯⽤法使⽤ `auto` 声明⼀个变量，然后对表达式强制类型转换得出你期望的推导结果。举个例⼦，我们该怎么将这个惯⽤法施加到 `highPriority` 上？

```cpp
auto highPriority = static_cast<bool>(features(w)[5]);
```

这⾥，`feature(w)[5]` 还是返回⼀个 `std::vector<bool>::reference` 对象，就像之前那样，但是这个转型使得表达式类型为 `bool`，然后 `auto` 才被⽤于推导 `highPriority`。在运⾏时，对 `std::vector` 使⽤ `operator[]` 将返回⼀个 `std::vector::reference`，然后强制类型转换使得它执⾏向 `bool `的转型，在这个过
程中指向 `std::vector<bool>` 的指针已经被解引⽤。这就避开了我们之前的未定义⾏为。然后5将被⽤
于指向bit的指针，`bool` 值被⽤于初始化 `highPriorit`。

应⽤这个惯⽤法不限制初始化表达式产⽣⼀个代理类。它也可以⽤于强调你声明了⼀个变量类型，它的类型不同于初始化表达式的类型。举个例⼦，假设你有这样⼀个表达式计算公差值：

```cpp
double calEpsilon();
```

`calEpsilon` 清楚的表明它返回⼀个 `double`，但是假设你知道对于这个程序来说使⽤ `float` 的精度已经⾜够了，而且你很关⼼ `double` 和 `float` 的⼤小。你可以声明⼀个 `float` 变量储存 `calEpsilon` 的计算结果。

```cpp
float ep = calEpsilon();
```

但是这⼏乎没有表明 *“我确实要减少函数返回值的精度”*。使⽤显式类型初始器惯⽤法我们可以这样：

```cpp
auto ep = static_cast<float>(calEpsilon());
```

再比如，你想用一个 `int` 类型存储一个表达式返回的 `float` 类型的结果，也可以使用这个方法

假如你需要计算⼀个随机访问迭代器（⽐如 `std::vector,std::deque,std::array` ）中某元素的下标，你给它⼀个 0.0 到 1.0 的值表明这个元素离容器的头部有多远（0.5 意味着位于容器中间）。进⼀步假设你很⾃信结果下标是 `int`。如果容器是 `c`，`d` 是 `double` 类型变量，你可以⽤这样的⽅法计算容器下标：

```cpp
int index = d * c.size();
```

这种写法并没有明确表明你想将右侧的 `double` 类型转换成 `int` 类型，显式类型初始器可以帮助你正确表意：

```cpp
auto index = static_cast<int>(d * size());
```

总结：
- 不可见的代理类可能会使 `auto` 从表达式中推导出错误的类型
- 显示类型初始器惯用法强制 `auto` 推导处你想要的结果
