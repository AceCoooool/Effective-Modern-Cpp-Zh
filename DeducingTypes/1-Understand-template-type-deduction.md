> 说明：
>
> 1. 这部分很多是基于[item1](http://www.cnblogs.com/boydfd/p/4950334.html)修改的
> 2. 下面的代码可以参考[item1-github](https://github.com/AceCoooool/Effective-Modern-Cpp-Code/tree/master/item1)

条款1：理解模板类型推导
====================

##Understand template type deduction.

一些用户对复杂的系统往往只关心它能完成什么，而忽略它是怎么工作和设计的。从这个方面来度量，C++的模板类型推导是个巨大的成功。数以万计的程序员使用过template函数，并得到了满意的结果。尽管这些程序员很难给出比朦胧的描述更多的东西，比如那些被推导的函数是怎么使用类型来推导的。

如果你也是其中的一员，我这有好消息和坏消息给你。好消息是`template`类型的推导是现代c++最令人惊叹特性之一（`auto`）的基础。如果你熟悉c++98的`template`类型推导，那么你也会熟悉c++怎么使用`auto`来做类型推导。而坏消息是当`template`类型推导规则应用在`auto`上时，比应用在`template`中时，他们有时候看起来会比较难理解。由于这个原因，我们很有必要完全理解`template`类型推导的各个方面，因为`auto`是建立在这个基础上的。这个条款包含了你想知道的东西。

如果你愿意浏览少许伪代码，我们可以把函数模板看成这样子：

```cpp
template<typename T>
void f(ParamType param);
```

对于函数的调用看起来是这样的：

```cpp
f(expr);                    // 用一些表达式来调用f
```

在整个编译期间，编译器用`expr`来推导两个类型：一个是`T` ，另一个是`ParamType`。这两个类型常常是不一样的，因为`ParamType`常常包含一些修饰符。（比如`const`或引用的限定）。比方说，如果一个`template`声明成这样：

```cpp
template<typename T>
void f(const T& param);		// ParamType 是 const T&
```

如果有这样的调用：

```cpp
int x = 0；
f(x)；						// 使用int调用f
```

`T`被推导成`int`，但`ParamType`被推导成`const int&`。

一般会很自然的期望`T`的类型和传递给他的参数的类型一致，也就是说`T`的类型就是`expr`的类型。在上面的例子中，`x`是一个`int`，`T`也就被推导成`int`。但是并不是所有的情况都是如此。对`T`类型的推导，不仅仅取决于`expr`，同时也取决于`ParamType`。这里有三种情况：

* `ParamType`是指针或引用，但不是一个universal引用（universal引用在#24中描述，现在你要知道的就是他们存在，并且左引用和右引用是不相同的）
* `ParamType`是universal引用。
* `ParamType`既不是指针也不是引用

因此我们有三种类型的推导情况需要分析。每一个将建立在下面这个template的基础上进行调用:

```cpp
template<typename T>
void f(ParamType param);

f(expr);					// 从expr推导出T和ParamType的类型
```

###情况1：`ParamType`是指针或引用，但不是一个universal引用

这种情况是最简单的情况，类型推导将这么工作：

1. 如果`expr`的类型是个引用，忽略引用的部分。

2. 然后利用`expr`的类型和`ParamType`对比去判断`T`的类型。

比方说，下面的template：

```cpp
template<typename T>
void f(T& param);           // param是一个引用类型
```

并且我们有这些变量声明：

```cpp
int x = 27;                 // x是一个int
const int cx = x;           // cx是一个const int
const int& rx = x;          // rx是const int的引用
```

`param`和`T`在不同的调用下的类型推导如下：

```cpp
f(x);        // T是int，      param的类型是int&
f(cx);       // T是const int，param的类型是const int&
f(rx);       // T是const int，param的类型时const int&
```

注意在第二个和第三个函数调用中，因为`cx`和`rx`是const变量，`T`被推导成`cosnt int`，因此param类型是`const int&`。这对调用者是很重要的。当他们传入const引用对象参数，他们希望对象是不可改动的。也就是，参数要是一个reference-to-const。这也是为什么传入一个const对象给使用`T&`参数的template是安全的：对象的const属性会被推导成`T`的一部分。

注意在第三个例子中，即使`rx`的类型是引用，`T`还是被推导成非引用。这是因为第一点，在推导时，`rx`的引用属性被忽略了。

这些例子显示了左值引用，但是对右值引用的推导和左值引用是一样的。当然，右值参数只可能传递给右值引用参数，但是这个限制和类型推导没有关系。

如果我们把`f`的参数从`T&`改成 `const T&`，情况将发生小小的变化，但是不会出乎意料。`cx`和`rx`的`const`属性仍然存在，但是因为我们现在假设param是const引用，所以就没有必要把const属性推导到`T`上去了：

```cpp
template<typename T>
void f(const T& param);     // param现在是const的引用

int x = 27;                 // x是一个int
const int cx = x;           // cx是一个const int
const int& rx = x;          // rx是const int的引用

f(x);                       // T是int，param的类型是const int&
f(cx);                      // T是int，param的类型是const int&
f(rx);                      // T是int，param的类型是const int&
```

和之前一样，`rx`的引用特性在类型推导的过程中会被忽略。

如果`param`是一个指针（或者指向`const`的指针）而不是引用，情况也是类似：

```cpp
template<typename T>
void f(T* param);           // param是一个指针

int x = 27;                 // x是一个int
const int *px = &x;         // px是一个指向const int x的指针

f(&x);                      // T是int，       param的类型是int*
f(px);                      // T是const int,  param的类型是const int*
```

现在你可能觉得有些困了，因为对于引用和指针类型，c++的类型推导规则是如此的自然，把他们一个个写出来是如此枯燥，因为所有的事情显而易见，就和你想要的推导系统一样。

###情况2：`ParamType`是一个universal引用

> 关于右值引用可以参考：[右值引用](https://www.zhihu.com/question/22111546)

对于一个采用universal引用参数的template，情况变得复杂。这样的参数大多被声明成右值引用（比如在函数模板上采用一个类型`T`，一个universal引用的声明类型是`T&&`），但是他们表现的和传入左值参数时不同。完整的情况将在item 24讨论，但是这里给出一些大概内容：

* 如果`expr`是一个左值，`T`和`ParamType`都会被推导成左值引用。这有些不同寻常。第一，这是模板类型`T`被推导成一个引用的唯一情况。第二，尽管`ParamType`利用右值引用的语法来进行推导，但是他最终推导出来的类型是一个左值引用。
* 如果`expr`是一个右值，那么就执行“普通”的法则（情况1的规则）

举个例子：

```cpp
template<typename T>
void f(T&& param);			// param现在是一个通用的引用

int x = 27;                 // x是一个int
const int cx = x;           // cx是一个const int
const int& rx = x;          // rx是const int的引用

f(x);				// x是左值， 所以T是int&，      param的类型也是int&
f(cx);				// cx是左值，所以T是const int&， param的类型也是const int&
f(rx);				// rx是左值，所以T是const int&，  param的类型也是const int&
f(27);				// 27是右值，所以T是int，          param的类型是int&&
```

item24解释了为什么这个例子会这样工作。这里的关键点是universal引用的类型推导规则对左值和右值是不同的。尤其是，当universal引用在使用中，类型推导会根据传入的值是左值还是右值进行区分。non-universal引用永远不会发生这样的情况。

###情况3：`ParamType`既不是指针也不是引用

当`ParamType`既不是指针也不是引用，我们把它处理成pass-by-value：

```cpp
template<typename T>
void f(T param);			// param现在是pass-by-value
```

这意味着，`param`将成为传入参数的一份拷贝，一个全新的对象：

1. 和之前一样，如果`expr`的类型是个引用，将会忽略引用的部分。

2. 如果在忽略`expr`的引用特性后，`expr`是个`const`的，也要忽略掉`const`。如果是`volatile`，照样也要忽略掉（`volatile`对象并不常见。它们常常被用在实现设备驱动上面。更多的细节，请参考条款40。）

这样的话：

```cpp
int x = 27;                 // x是一个int
const int cx = x;           // cx是一个const int
const int& rx = x;          // rx是const int的引用

f(x);                       // T和param的类型都是int
f(cx);                      // T和param的类型也都是int
f(rx);                      // T和param的类型还都是int
```

注意：尽管`cx`和`rx`都是`const`类型，`param`却不是`const`的。这是有道理的，`param`是一个完全独立于`cx`和`rx`的对象——一个`cx`和`rx`的拷贝。`cx`和`rx`不能被修改和`param`能不能被修改是没有关系的。这就是为什么`expr`的常量特性（或者是易变性）（在很多的C++书籍上面`const`特性和`volatile`特性被称之为CV特性——译者注）在推导`param`的类型的时候被忽略掉了：`expr`不能被修改并不意味着它的一份拷贝不能被修改。

很重要的是，你要知道`const`（和`volatile`）属性被忽略只适用于按值传递（by-value）。就像我们看到的，在推导类型的时候，传引用（references-to）或传指针（pointers-to）参数的`const`属性是会保留的。但是我们来考虑一下`expr`是`cosnt`指针指向`const`类型的情况，并且`expr`是按值传递（by-value）：

```cpp
template<typename T>
void f(T param);                               // param仍然是按值传递的（pass by value）

const char* const ptr = "Fun with pointers";    // ptr是一个const指针，指向一个const对象;

f(ptr);                                        // 给参数传递的是一个const char * const类型
```

这里，位于星号右边的`const`是表明指针是常量`const`的：`ptr`不能被修改指向另外一个不同的地址，并且也不能置成`null`。（星号左边的`const`表明`ptr`指向的字符串是`const`的，也就是说字符串不能被修改。）当这个`ptr`传递给`f`，ptr会被拷贝赋给`param`。这样的话，指针自己（`ptr`）本身是被按值传递的。按照按值传递的类型推导法则，`ptr`的`const`特性会被忽略，这样`param`的推导出来的类型就是`const char*`，也就是一个可以被修改的指针，指向一个`const`的字符串。`ptr`指向的东西的`const`特性被加以保留，但是`ptr`自己本身的`const`特性会被忽略，因为它要被重新复制一份而创建了一个新的指针`param`。

##数组参数

之前的那些已经涉及到大多数主流的template参数推导了，但是，这里还有一些小部分的情况值得我们去了解。数组类型不同于指针类型，尽管有时候它们看起来可以相互替换。这个错觉主要来自于：在很多时候，一个数组可以退化成（decays）一个指向其第一个元素的指针。这个退化允许代码写成这样：

```cpp
const char name[] = "J. P. Briggs";     // name的类型是const char[13]
const char * ptrToName = name;          // 数组被退化成指针
```

在这里，`const char*`指针`ptrToName`使用`name`初始化，实际的`name`的类型是`const char[13]`。这些类型（`const char*`和`const char[13]`）是不一样的，但是因为数组到指针的退化规则，代码会被正常编译。

但是，把数组按值传递给一个模板函数，会发生什么呢？

```cpp
template<typename T>
void f(T param);            // 模板拥有一个按值传递的参数

f(name);                    // T和param的类型会被推导成什么呢？
```

我们从一个没有模板参数的函数开始。是的，是的，语法是合法的，

```cpp
void myFunc(int param[]);       // 和上面的函数相同
```

但是以数组声明，等价于把它当成一个指针声明，也就是说`myFunc`可以和下面的声明等价：

```cpp
void myFunc(int* param);        // 和上面的函数是一样的
```

这样的数组和指针等价的声明经常会在以C语言为基础的C++里面出现，这也就导致了数组和指针是等价的错觉。

因为数组参数声明会被当做指针参数，传递给模板函数的按值传递的数组参数会被退化成指针类型。这就意味着在模板`f`的调用中，模板参数`T`被推导成`const char*`：

```cpp
f(name);                    // name是个数组，但是T被推导成const char*
```

但是来一个特例。尽管函数的参数不能被真正地定义成数组，但是可以声明参数是数组的引用！所以如果我们修改模板`f`的参数成引用，

```cpp
template<typename T>
void f(T& param);           // 引用参数的模板
```

然后传一个数组给他

```cpp
f(name);                    // 传递数组给f
```

`T`最后推导出来的实际的类型就是数组！类型推导包括了数组的长度，所以在这个例子里面，`T`被推导成了`const char [13]`，函数`f`的参数（数组的引用）被推导成了`const char (&)[13]`。是的，语法看起来怪怪的，但是理解了这些可以升华你的精神（原文knowing it will score you mondo points with those few souls who care涉及到了几个宗教词汇——译者注）。

有趣的是，声明数组的引用可以使我们创造出一个能够推导数组包含的元素长度的template：

```cpp
// 在编译的时候返回数组的长度（数组参数没有名字，因为只关心数组包含的元素的个数）
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept
{
    return N;                   // constexpr和noexcept在随后的条款中介绍
}
```

（`constexpr`是一种比`const`更加严格的常量定义，`noexcept`是说明函数永远都不会抛出异常——译者注）

正如条款15所述，定义为`constexpr`说明函数可以在编译的时候得到其返回值。这使我们可以用一个数组的大小做为另一个新数组的大小来初始化新数组：

```cpp
int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 };     // keyVals有七个元素

int mappedVals[arraySize(keyVals)];             // mappedVals长度也是七
```

当然，作为一个现代的C++开发者，应该优先选择内建的`std::array`：

```cpp
std::array<int, arraySize(keyVals)> mappedVals; // mappedVals长度是七
```

由于`arraySize`被声明称`noexcept`，这会帮助编译器生成更加优化的代码。可以从条款14查看更多细节。

##函数参数

数组并不是C++唯一可以退化成指针的情况。函数类型也可以被退化成函数指针，和我们之前讨论的数组的推导类似，函数可以被退化成函数指针（所有我们对数组的推导的讨论适用于函数类型推导：）：

```cpp
void someFunc(int， double);    // someFunc是一个函数， 类型是void(int, double)

template<typename T>
void f1(T param);               // 在f1中 参数直接按值传递

template<typename T>
void f2(T& param);              // 在f2中 参数是按照引用传递

f1(someFunc);                   // param被推导成函数指针， 类型是void(*)(int, double)

f2(someFunc);                   // param被推导成函数引用， 类型是void(&)(int, double)
```

这看起来几乎没有任何不同，但是，如果你已经知道了array-to-pointer的退化，你也会同样知道function-to-pointer的退化。

现在你明白了吧：函数类型推导的规则，我在开始就说他们相当简单，并且大多数情况下是这样的。对于universal引用，左值当推导的类型时需要特别对待，这使我们有点晕晕的，然而退化成指针（decay-to-pointer）的规则使得我们更加晕了。有时，你简单地想抓住你的编译器和并渴望：“告诉我你推导的类型是什么”。当发生这样的事时，转到条款4，因为这条款专注于欺骗编译器去做你想它做的事。

|要记住的东西|
| :--------- |
| 在template类型推导的时候， references类型的参数被当成non-references。也就是说引用属性会被忽略。 |
| 当推导universal类型的引用参数时，左值参数被特殊对待。(即保持左值特性) |
| 在推导按值传递的参数时候，`const`和/或`volatile`参数会被视为`non-const`和`non-volatile` |
| 当推导类型是数组或函数时会退化成指针，除非形参是引用。 |

