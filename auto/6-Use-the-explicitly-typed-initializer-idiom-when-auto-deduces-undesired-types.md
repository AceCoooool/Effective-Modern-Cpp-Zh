> 说明：
>
> 1. 这部分很多是基于[item6](http://www.cnblogs.com/boydfd/p/4970605.html)修改的
> 2. 下面的代码可以参考[item6-github](https://github.com/AceCoooool/Effective-Modern-Cpp-Code/tree/master/item6)

条款6：当auto推导出非预期类型时应当使用显式的类型初始化
===============================================

条款5解释了比起显式指定类型，使用`auto`来声明变量提供了大量技术上的优点，但是有时候`auto`的类型推导会和你想的南辕北辙（比如auto类型推导出zigs，但是你想要的是zag）。举一个例子，假设我有一个函数接受一个`Widget`返回一个`std::vector<bool>`，其中每个`bool`表征`Widget`是否提供了特定的特性：

```cpp
std::vector<bool> features(const Widget& w);
```

进一步的，假设第五个bit表示`Widget`是否有高优先级。我们可以这样写代码：

```cpp
Widget w;
…
bool highPriority = features(w)[5];         // w是不是个高优先级的？
…
processWidget(w, highPriority);             // 配合优先级处理w
```

这份代码没有任何问题。它工作正常。但是如果我们做一个看起来无伤大雅的修改，把`highPriority`的显式的类型换成`auto`：

```cpp
auto highPriority = features(w)[5];         // w是不是个高优先级的？
```

情况变了。所有的代码还是可以编译，但是他的行为变得不可预测：

```cpp
processWidget(w, highPriority);             // 未定义行为
```

正如注释中所提到的，调用`processWidget`现在会导致未定义的行为。但是为什么呢？答案是非常令人惊讶的。在使用`auto`的代码中，`highPriority`的类型已经不是`bool`了。尽管`std::vector<bool>`从概念上说是`bool`的容器，对`std::vector<bool>`的`operator[]`运算符不返回容器中的元素的引用（`std::vector::operator[]`对所有的类型都返回引用，就是除了`bool`）。事实上，他返回的是一个`std::vector<bool>::reference`对象（是一个在`std::vector<bool>`中内嵌的class）。

`std::vector<bool>::reference`之所以会存在是因为`std::vector<bool>`被指定来代表它包装的bools，每个bool占一bit。这就给`std::vector::operator[]`带来了问题，因为`std::vector<T>`的`operator[]`应该返回一个`T&`，但是C++禁止返回bits的引用。没办法返回一个`bool&`，`std::vector<T>`的`operator[]`于是就返回了一个行为上和`bool&`相似的对象。想要这种行为成功，`std::vector<bool>::reference`对象必须能在`bool&`的能用的语境中使用。在`std::vector<bool>::reference`对象的特性中，是他隐式的转换成`bool`才使得这种操作得以成功。（不是转换成`bool&`，而是`bool`。去解释详细的`std::vector<bool>::reference`对象如何模拟一个`bool&`的行为有有些偏离主题，所以我们就只是简单的提一下这种隐式转换只是这种技术中的一部。）

记住了这些信息后，再次阅读原先的代码：

```cpp
bool highPriority = features(w)[5];         // 直接显示highPriority的类型
```

这里，`features`返回了一个`std::vector<bool>`对象，在这里`operator[]`被调用。`operator[]`返回一个`std::vector<bool>::reference`对象，这个对象隐式的转换成`highPriority`需要用来初始化的`bool`类型。于是就以`features`返回的`std::vector<bool>`的第五个bit的数值来结束`highPriority`的数值，这也是我们所预期的。

和使用`auto`的`highPriority`声明进行对比：

```cpp
auto highPriority = features(w)[5];         // 推导highPriority的类型
```

这次，`features`返回一个`std::vector<bool>`对象，而且，`operator[]`再次被调用。`operator[]`继续返回一个`std::vector<bool>::reference`对象，但是现在有一个变化，因为`auto`推导`highPriority`的类型为`std::vector<bool>::reference`。highPriority不再拥有`features`返回的`std::vector<bool>`的第五个bit的数值。

数值和`std::vector<bool>::reference`是如何实现是有关系的。一种实现是这样的，对象包含一个指向包含bit引用的机器word的指针，在word上面加上偏移。考虑一下，这样一个`std::vector<bool>::reference`的实现，对于highPriority的初始化意味着什么。

对于features的调用返回一个`std::vector<bool>`临时对象。这个对象没有名字，但是为了讨论的目的，我将把它叫做`temp`。`operator[]`是对`temp`的调用，并且`std::vector<bool>::reference`返回一个对象，这个对象指向一个字节（word），这个被`temp`管理的数据结构中的字节持有所需的bit，加上对象存储的偏移就能找到bit 5。highPriority是这个`std::vector<bool>::reference`对象的一份拷贝。所以highPriority也持有指向`temp`中某个字节（word）的指针，以及bit 5的偏移。在语句的最后，`temp`销毁了，因为它是一个临时对象。因此，highPriority持有的是一个悬挂的指针，并且这就是在processWidget调用中造成未定义行为的原因：

```cpp
processWidget(w, highPriority);         // 未定义的行为，highPriority包含野指针
```

`std::vector<bool>::reference`是代理类的一个例子：一个类的存在是为了效仿和增加其他类型的行为。代理类被用作各种各样的目的。`std::vector<bool>::reference`为了提供一个错觉：`std::vector<bool>`的`operator[]`返回一个`bool`引用。再举个例子，标准库的智能指针（第4章）是一个对原始指针进行资源管理的代理类。代理类的用法已经慢慢固定下来了。事实上，设计模式“Proxy”就是软件设计模式万神殿（Pantheon）中长期存在的一员。

对于客户来说，一些代理类被设计成可见的。举个例子，这就是`std::shared_ptr`和`std::unique_ptr`的情况。另外一种代理类被设计成或多或少不可见的。`std::vector<bool>::reference`就是这种“不可见”代理类的例子，对于它的同胞`std::bitset`也有这样的代理类：std::bitset::reference。

同时在一些C++库里面的类存在一种被称作表达式模板（expression template）的技术。这些库最开始是为了提高数值运算的效率。提供一个`Matrix`类和`Matrix`对象`m1, m2, m3 and m4`，举一个例子，下面的表达式：

```cpp
Matrix sum = m1 + m2 + m3 + m4;
```

可以计算的更快如果`Matrix`的`operator+`返回一个结果的代理来代替结果本身。这是因为，对于两个`Matrix`，`operator+`可能返回一个类似于`Sum<Matrix, Matrix>`的代理类而不是一个`Matrix`对象。和`std::vector<bool>::reference`一样，这里会有一个隐式的从代理类到`Matrix`的转换，这个可能允许`sum`从由`=`右边的表达式产生的代理对象进行初始化。（其中的对象可能会编码整个初始化表达式，也就是，变成一种类似于`Sum<Sum<Sum<Matrix, Matrix>, Matrix>, Matrix>`的类型。这是一个客户端需要屏蔽的类型。）

作为一个通用的法则，“不可见”的代理类不能和`auto`愉快的玩耍。这样的对象的生命周期常常被设计成不能超过一条简单语句，所以创造这些类型的变量将违反库设计时的假设情况。这就是`std::vector<bool>::reference`的情况，我们看到了，违反假设将导致未定义行为。

因此你要避免使用下面的代码的形式：

```cpp
auto someVar = expression of "invisible" proxy class type; //"不可见的"代理类的表达式
```

但是你要怎么意识到你在使用代理对象呢？创造代码的人不太可能主动解释它（代理类）的存在。至少在概念上，它们被假设为“不可见的”。并且一旦你找到了它们，你就真的能抛弃auto以及在**条款 5**中说明的auto的众多优点吗？

我们先看看怎么解决"如何发现它们"的问题。尽管“不可见”的代理类被设计成”在日常使用中，程序员不会发现它们“，但库生产者在使用它们时常常会用文档标示出他这么做了。你越多地让你自己熟悉你所使用的库的基础设计决策，你就越少地被库中的代理类给偷袭。

当文档出现遗漏的时候，头文件会填补这个缺陷。想要完全隐藏掉代理类对源代码来说是几乎不可能的。典型地，它们就是用户想要调用的函数的返回，所以函数签名常常反映了它们的存在。`std::vector<bool>::reference`的原型如下：

```cpp
namespace std {                     // from C++ Standards
    template <class Allocator>
    class vector<bool, Allocator> {
        public:
        …
        class reference { … };
        reference operator[](size_type n);
        …
    };
}
```

假设你知道对`std::vector<T>`的`operator[]`常常返回一个`T&`，在这个例子中的这种非常规的`operator[]`的返回类型一般就表征了代理类的使用。把注意力放在你使用的函数的接口上常常帮你揭露出代理类的存在。

在实践中，很多开发者只有在当他们尝试追踪神秘的编译错误，或者调试出不正确的单元测试结果时，才能发现代理类的存在。不管你怎么发现他们，一旦`auto`推导出一个代理类类型替换了被代理的类型，解决方法不是禁止`auto`的使用。`auto`它本身没有问题。问题是`auto`不能推导出你想要它推导的类型。解决方法是强制一个不同的类型推导。你可以使用称为“显式类型初始化语法”的方法。

显示类型初始化语法涉及用`auto`声明一个变量，但是需要把初始化表达式的类型转换到你想要`auto`推导的类型。举个例子，这里给出怎么用这个方法来强制highPriority转换到`bool`：

```cpp
auto highPriority = static_cast<bool>(features(w)[5]);
```

这里，`features(w)[5]`还是返回一个`std::vector<bool>::reference`的对象，就和它经常的表现一样，但是强制类型转换改变了表达式的类型成为`bool`，然后`auto`才推导其作为`highPriority`的类型。在运行的时候，从`std::vector<bool>::operator[]`返回的`std::vector<bool>::reference`对象支持转换到`bool`的行为，作为转换的一部分，从`features`返回的任然存活的指向`std::vector<bool>`的指针被间接引用。这样就在运行的开始避免了未定义行为。索引5然后放置在bits指针的偏移上，然后暴露的`bool`就作为`highPriority`的初始化数值。

针对于`Matrix`的例子，显式类型初始化语法看起来将会是这样：

```cpp
auto sum = static_cast<Matrix>(m1 + m2 + m3 + m4);
```

这个语法的应用不止限制在初始化时产生的代理类类型上。它在当你故意创建一个变量，让它的类型不同于初始化表达式产生的类型时同样可以起到一个强调的作用。举个例子，假设你有一个函数计算一些公差值：

```cpp
double calcEpsilon();               // 返回方差
```

`calcEpsilon`明确的返回一个`double`，但是假设你知道你的程序，`float`的精度就够了的时候，而且你要关注`double`和`float`的长度的区别。你可以声明一个`float`变量去存储`calcEpsilon`的结果：

```cpp
float ep = calcEpsilon();           // 隐式转换double到float
```

但是这个会很难表明“我故意减小函数返回值的精度”。这时就可以用显式类型转换语法来声明它：

```cpp
auto ep = static_cast<float>(calcEpsilon());
```
同样的理由可以应用在你想故意用整形来存放浮点型的表达式。假设你需要用随机存取迭代器（比如，`std::vector`，`std::deque`或`std::array`的迭代器）计算容器元素的下标时，并且你给出了一个从`0.0`到`1.0`的`double`值来表示从容器的开始到你需要的元素的位置有多远。（`0.5`将标示着容器的中间位置）进一步假设你确信下标计算的结果适合作为一个`int`。如果容器是`c`，并且`double`值是`d`，你可以这样计算下标：

```cpp
int index = d * c.size();
```

但是这模糊了一个事实，就是你故意地把右边的`double`转换成了一个`int`。显式类型初始化语法让事情变得易懂了：

```cpp
auto index = static_cast<int>(d * c.size());
```

| 要记住的东西                                                 |
| :----------------------------------------------------------- |
| 对于一个初始化表达式，“看不见的”代理类型能造成auto推导出“错误的”类型 |
| 显式类型初始化语法强制auto推导出你想要它推导的类型。         |