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
