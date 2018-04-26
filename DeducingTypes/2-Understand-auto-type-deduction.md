> 说明：
>
> 1. 这部分很多是基于[item2](http://www.cnblogs.com/boydfd/p/4959389.html)修改的
> 2. 下面的代码可以参考[item2-github](https://github.com/AceCoooool/Effective-Modern-Cpp-Code/tree/master/item2)

条款二：理解`auto`类型推导
=========================

如果你已经阅读了条款1关于模板相关的类型推导，你就已经知道了几乎所有关于`auto`的类型推导。因为除了一个例外之外，`auto`类型推导和`template`类型推导是一样的。但是它怎么就会是模板类型推导呢？模板类型推导涉及模板和函数以及参数，但是`auto`和上面的这些没有任何的关系（即不涉及模板，函数和参数）。

这是对的，但是没有关系。模板类型推导和`auto`类型推导是有一个直接的映射。有一个直接从一种情况转换成另外一种情况的算法。

在条款1中，模板类型推导是使用下面的通用模板函数来解释的：

```cpp
template<typename T>
void f(ParamType param);
```

在这里简单地调用：

```cpp
f(expr);                    // 使用一些表达式来当做调用f的参数
```

在调用`f`的地方，编译器使用`expr`来推导`T`和`ParamType`的类型。

当一个变量被声明为`auto`时，`auto`相当于模板中的`T`，而对变量做的相关的类型限定就像`ParamType`。这用代码说明比直接解释更加容易理解，所以看下面的这个例子：

```cpp
auto x = 27;
```

这里，对`x`的类型定义就仅仅是`auto`本身。从另一方面，在这个声明中：

```cpp
const auto cx = x;
```

类型被声明成`const auto`，然后：

```cpp
const auto& rx = x;
```

类型被声明称`const auto&`。在这些例子中推导`x`，`cx`，`rx`的类型的时候，编译器表现地好像这里有template的声明，并且用初始化表达式调用相应的template：

```cpp
template<typename T>                // 推导x的类型概念上的模板
void func_for_x(T param);           

func_for_x(27);                     // 概念上的调用：param的类型就是x的类型

template<typename T>
void func_for_cx(const T param);    // 推导cx的概念上的模板

func_for_cx(x);                     // 概念上的调用：param的推导类型就是cx的类型

template<typename T>
void func_for_rx(const T& param);   // 推导rx概念上的模板

func_for_rx(x);                     // 概念上的调用：param的推导类型就是rx的类型
```

正如我所说，对`auto`的类型推导只存在一种情况的例外（这个马上就会进行讨论），其他的就和模板类型推导完全一样了。

条款1基于`ParamType`的特性和`param`的类型声明，把模板类型推导划分成三种情况。用`auto`声明的变量上，类型修饰符（可以理解为对auto的修饰）代替了`ParamType`的作用，所以也有三种情况：

* 情况1：类型修饰符是一个指针或者是一个引用，但不是一个universal引用
* 情况2：类型修饰符是一个universal引用
* 情况3：类型修饰符既不是一个指针也不是一个引用

我们已经看了情况1和情况3的例子：

```cpp
auto x = 27;                        // 情况3（x既不是指针也不是引用）

const auto cx = x;                  // 情况3（cx二者都不是）

const auto& rx = x;                 // 情况1（rx是一个非通用的引用）
```

情况2正如你期待的那样：

```cpp
auto&& uref1 = x;                   // x是int并且是左值,  所以uref1的类型是int&

auto&& uref2 = cx;                  // cx是int并且是左值, 所以uref2的类型是const int&

auto&& uref3 = 27;                  // 27是int并且是右值, 所以uref3的类型是int&&
```

条款1讲解了在非引用类型声明里，数组和函数名称如何退化成指针。这在`auto`类型推导上面也是一样：

```cpp
const char name[] = "R. N. Briggs";   // name的类型是const char[13] 
    
auto arr1 = name;                     // arr1的类型是const char*
auto& arr2 = name;                    // arr2的类型是const char (&)[13]

void someFunc(int, double);           // someFunc是一个函数，类型是void (*)(int, double)

auto func1 = someFunc;                // func1的类型是void (*)(int, double)
auto& func2 = someFunc;               // func2的类型是void (&)(int, double)
```

正如你所见，`auto`类型推导和模板类型推导工作很类似。它们就像一枚硬币的两面。

除了有一种情况，它们是不一样的。观察一个情况，如果你想用27初始化一个`int`变量， C++98允许你有两种语法选择：

```cpp
int x1 = 27;
int x2(27);
```

C++11，通过它对标准初始化的支持（使用花括号初始化——译者注），增加了这些操作：

```cpp
int x3 = { 27 };
int x4{ 27 };
```

总之上述四种语法，都会生成一种结果：一个拥有27数值的`int`。

但是正如条款5所解释的，在很多情况下使用`auto`来声明变量比使用固定的类型更好，所以在上述的声明中把`int`换成`auto`更好。最直白的写法就如下面的代码：

```cpp
auto x1 = 27;
auto x2(27);
auto x3 = {27};
auto x4{ 27 };
```

上面的所有声明都可以编译，但是他们和被替换的相对应的语句的意义并不一样。头两个的确是一样的，声明一个初始化值为27的`int`。然而后面两个，声明了一个类型为`std::initializer_list<int>`的变量，这个变量包含了一个单一的元素27！

```cpp
auto x1 = 27;                       // 类型时int，值是27
auto x2(27);                        // 同上
auto x3 = { 27 };                   // 类型是std::initializer_list<int>, 值是{ 27 }
auto x4{ 27 };                      // 同上? (这部分个人测得结果为int)
```

产生这样的结果是由于`auto`的特殊的类型推导规则。当使用初始化列表的形式来初始化一个`auto`类型的变量的时候，推导出的类型是`std::iniializer_list`。如果这种类型无法被推导（比如在花括号中的变量拥有不同的类型），代码会编译错误。

```cpp
auto x5 = { 1, 2, 3.0 };            // 错误！ 不能将T推导成std::initializer_list<T>
```

正如注释中所说的，在这种情况下类型推导会失败，但是认识到这里实际上是有两种类型推导是非常重要的。一种是来自`auto: x5`的类型推导。因为`x5`的初始化是在花括号里面，`x5`必须被推导成`std::initializer_list`。但是`std::initializer_list`是一个模板。用`T`来实例化`std::initializer_list`意味着`T`的类型也必须被推导出来。这里发生的推导属于第二种类型推导：template类型推导。类型推导就在第二种推导上失败了。在这个例子中，类型推导失败是因为在花括号里面的数值并不是单一类型的。

对待花括号初始化（初始化列表）的行为是`auto`唯一和模板类型推导不一样的地方。当用初始化列表来声明`auto`类型的变量时，推导出来的类型是`std::initializer_list`的一个实例。但是如果传入同样的初始化列表给template，类型推导将会失败，并且代码编译不通过：

```cpp
auto x = { 11, 23, 9 };             // x的类型是std::initializer_list<int>

template<typename T>                // 和x的声明等价的
void f(T param);                    // 模板

f({ 11, 23, 9 });                   // 错误的！没办法推导T的类型
```

然而如果你明确template的参数为`std::initializer_list`，template类型推导将成功推导出`T`：

```cpp
template<typename T>
void f(std::initializer_list<T> initList);

f({ 11, 23, 9 });      // T被推导成int，initList的类型是std::initializer_list<int>
```
所以`auto`和模板类型推导的本质区别就是`auto`假设花括号初始化代表的是`std::initializer_list`，但是模板类型推导不这么做。

你可能想知道为什么`auto`类型推导对初始化列表有这么一个特别的规则，但是template类型推导却没有。我自己也非常奇怪。可是我一直没有能够找到一个有力的解释。但是法则就是法则，这就意味着你必须记住如果使用`auto`声明一个变量并且使用花括号来初始化它，类型推导的就是`std::initializer_list`。你必须习惯这种花括号的初始化哲学——使用花括号里面的数值来初始化是理所当然的。在C++11编程里面的一个经典的错误就是你想声明另外的一种类型，却被意外地声明成`std::initializer_list`，而其实。这个陷阱使得一些开发者仅仅在必要的时候才会在初始化数值周围加上花括号。（什么时候是必要的会在条款7里面讨论。）

对于C++11来说，故事已经完结了，但是对于C++14来说，故事还要继续。C++14允许auto作为函数的返回值（参看条款3），并且在lambdas表达式中可能用auto来作为形参类型。然而，auto的这些用法使用的是template类型推导规则，而不是auto类型推导规则。所以返回一个初始化列表给auto类型的函数返回值将不能通过编译

```cpp
auto createInitList()
{
    return { 1, 2, 3 };             // 编译错误：不能推导出{ 1, 2, 3 }的类型
}
```

在C++14中，这对于作为lambdas表达式的参数声明的auto类型同样适用：

```cpp
std::vector<int> v;
…

auto resetV = [&v](const auto& newValue) { v = newValue; }    // C++14

…
resetV({ 1, 2, 3 });                // 编译错误，不能推导出{ 1, 2, 3 }的类型
```

| 要记住的东西                                                 |
| :----------------------------------------------------------- |
| auto类型推导常常和template类型推导一样，唯一的区别在于auto类型推导假设初始化列表为`std::initializer_list`，但是template类型推导不这么做 |
| auto作为函数返回类型或lambdas表达式的形参类型时，默认使用template类型推导规则，而不是auto类型推导规则。 |

