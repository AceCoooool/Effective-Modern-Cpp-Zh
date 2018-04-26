> 说明：
>
> 1. 这部分很多是基于[item5](http://www.cnblogs.com/boydfd/p/4966122.html)修改的
> 2. 下面的代码可以参考[item5-github](https://github.com/AceCoooool/Effective-Modern-Cpp-Code/tree/master/item5)

条款五：优先使用`auto`而非显式类型声明
=========================

使用下面语句是简单快乐的
```cpp
int x;
```
等等。见鬼，我忘记初始化`x`了，因此它的值是无法确定的。也许，它会被初始化为0。但是这根据上下文语境决定。悲伤脸～

不要介意。让我们简单并愉快地声明一个局部变量，通过解引用一个iterator来初始化它：

```cpp
template<typename It>
void dwim(It b, It e)
{
    while(b != e){
        typename std::iterator_traits<It>::value_type currValue = *b;
        ...
    }
}
```
额。真的要用`typename std::iterator_traits<It>::value_type`来表示被迭代器指向的值的类型？我已经忘了这有多愉快了，等等，我之前真的说过这愉快吗？

好的，再看一个简单愉快的例子（第三个了）：愉快地声明一个局部变量，让他的类型是一个闭包。噢，对的，闭包的类型只有编译器知道，因此不能被写出来，哎，讨厌。

见鬼，见鬼，见鬼！C++编程，并不是一段愉快的经历（它本应该是愉快的）

好的吧，它曾经不是。但是在C++11中，由于auto的出现，所有的这些麻烦都离去了。auto变量从初始化表达式中推导它们的类型，所以它们必须被初始化。这意味着，当你行驶在现代C++的超级高速公路上时，你能和变量未初始化问题挥手再见了：

``` cpp
int x1;                   // potentially uninitialized
auto x2;                  // error! initializer required
auto x3 = 0;              // fine, x's value is well-defined
```
通过对iterator解引用来声明局部变量时，这个高速公路没有之前那样的困难：

``` cpp
template<typename It>
void dwim(It b, It e)
{
    while(b != e){
        auto currValue = *b;
        ...
    }
}
```
并且，由于`auto`使用类型推导（参见条款2），它可以表示那些仅仅被编译器知晓的类型：
``` cpp
auto dereUPLess =                           // comparison func.
    [](const std::unique_ptr<Widget>& p1,   // for Widgets
       const std::unique_ptr<Widget>& p2)   // pointed to by
    { return *p1 < *p2};                    // std::unique_ptrs
```
非常酷。在C++14中，事情变得更加简单，因为使用`lambda`表达式的参数可以包含`auto`：
``` cpp
auto derefLess =                            // C++14 comparison
    [](const auto& p1,                      // function for
       const auto& p2)                      // values pointed
    { return *p1 < *p2; };
```
尽管非常酷，也许你在想，我们不需要使用`auto`来声明一个变量来包含闭包，因为我们可以使用`std::function`对象。这是对的，我们能这么做，但是事情并不是你想的那样。有的读者可能在想“什么是`std::function`对象” ，那就让我们先来理清这个对象把。

`std::function`是C++11标准库中的模板，这个模板扩展了函数指针的概念。函数指针只能指向函数，但是，`std::function`对象能指向所有可调用对象（也就是，所有能像函数一样用“()”调用的东西）。就像你声明一个函数指针的时候，必须指明这个函数指针指向的函数的类型，你产生一个`std::function`对象时，你也指明它要引用的函数的类型。你可以通过`std::function`的模板参数来完成这个工作。例如，声明一个名为`func`的`std::function`对象，它可以引用有如下特点的可调用对象：

``` cpp
bool(const std::unique_ptr<Widget> &,    // C++11 signature for
     const std::unique_ptr<Widget> &)    // std::unique_ptr<Widget>
                                         // comparison funtion
```

你可以这么写：

``` cpp
std::function<bool(const std::unique_ptr<Widget> &,
                   const std::unique_ptr<Widget> &)> func; 
```

因为`lambda`表达式得到一个可调用对象，封装体可以存储在`std::function`对象里面。这意味着，我们可以不使用`auto`就声明一个C++11版本的`dereUPLess`如下：

``` cpp
std::function<bool(const std::unique_ptr<Widget>&, const std::unique_ptr<Widget>&)>
      derefUPLess = [](const std::unique_ptr<Widget>& p1, const std::unique_ptr<Widget>& p2)
	                {return *p1 < *p2; };
```

你要知道下面的要点，就算把这冗长的语法和重复的变量类型放在一边，使用`std::function`和使用`auto`也是不一样的。一个用`auto`声明的变量存放和闭包同样类型的闭包，并且只使用和闭包所要求的内存一样多的内存。`std::function`声明的变量存放了一个闭包，这个闭包是`std::function`模板的实例，并且他对任何给出的签名都需要调整大小。这个大小可能不够一个闭包来存储，当遇到这样的情况时，`std::function`的构造函数将申请堆内存来存储闭包。结果就是相比`auto`声明的对象，`std::function`对象通常要使用更多的内存。再考虑下实现的细节，由于`inline`函数的限制，间接函数的调用，调用闭包时，通过`std::function`对象调用总是比通过`auto`声明的对象调用要慢。换句话说，`std::function`方法通常比`auto`方法更大更慢，并且可能产生内存溢出异常。更好的是，就像你在上面的例子中看到的那样，写”auto“要做的工作完全少于写`std::function`实例类型。在`auto`和存储闭包的`std::function`对比中，`auto`完胜了。（相似的争论会出现在`std::bind`函数的返回值的存储中，同样可以使用`auto`或`std::function`，但是在条款34中，我尽全力来说服你，尽量使用`lambdas`表达式来代替`std::bind`）

`auto`的**优点除了可以避免未初始化的变量，变量声明引起的歧义，直接持有封装体的能力**。还有一个就是可以避免“类型截断”（type shortcuts）问题。下面有个例子，你可能见过或者写过：

```cpp
std::vector<int> v;
...
unsigned sz = v.size();
```

`v.size()`定义的返回类型是`std::vector<int>::size_type`，但是很少有开发者对此十分清楚。`std::vector<int>::size_type`被指定为一个无符号的整数类型，因此很多程序员认为`unsigned`类型是足够的，然后写出了上面的代码。这将导致一些有趣的后果。比如说在32位Windows系统上，`unsigned`和`std::vector<int>::size_type`有同样的大小，但是在64位的Windows上，`unsigned`是32bit的，而`std::vector<int>::size_type`是64bit的。这意味着上面的代码在32位Windows系统上工作良好，但是在64位Windows系统上时有可能不正确，当应用程序从32位移植到64位上时，谁又想在这种问题上浪费时间呢？
使用`auto`可以保证你不必被上面的东西所困扰：

```cpp
auto sz = v.size()    // sz's type is std::vector<int>::size_type
```

仍然不太确定使用`auto`的高明之处？看看下面的代码：

```cpp
std::unordered_map<std::string, int> m;
...
for (const std::pair<std::string, int>& p : m)
{
	...                  // do something with p
}
```

这看上去很完美。但是这里存在一个问题，你看到了吗？
意识到`std::unorder_map`的`key`部分是`const`类型的（见[unordered_map: value_type](http://zh.cppreference.com/w/cpp/container/unordered_map)），在哈希表中的`std::pair`的类型不是`std::pair<std::string, int>`，而是`std::pair<const std::sting, int>`。但是这不是循环体外变量`p`的声明类型。后果就是，编译器竭尽全力去找到一种方式，把`std::pair<const std::string, int>`对象（正是哈希表中的内容）转化为`std::pair<std::string, int>`对象（`p`的声明类型）。这个过程将通过复制`m`的一个元素到一个临时对象，然后将这个临时对象和`p`绑定来完成。在每个循环结束的时候这个临时对象将被销毁。如果你写下这样的循环，你会被它的效率所震惊，因为大概你想做的只是简单地把`p`的引用绑定到`m`的每个元素`p`上。
这种无意识的类型不匹配可以通过`auto`解决

```cpp
for (const auto& p : m)
{
    ...                     // as before
}
```

这不仅仅更高效，也更容易写。更好的是，这个代码有一个非常吸引人的特征，那就是如果你获取`p`的地址，你肯定能取到一个指向存在于`m`中的`p`的指针，但是如果不使用auto，你会取到一个指向临时变量的指针，而且这个临时变量在每一次循环结尾会销毁。

最后两个例子中——在应该使用`std::vector<int>::size_type`的时候使用`unsigned`和在该使用`std::pair<const std::sting, int>`的地方使用`std::pair<std::string, int>`——演示了写下明确的类型如何导致隐式的转换，并且这种转换是你不想要的。如果你使用`auto`来作为目标变量的类型，你不需要考虑声明的变量和用来初始化的表达式之间的类型不匹配。

因此，这里有很多原因来让你选择`auto`而不是显式的类型声明。到目前为止，`auto`不是完美的。对于每个`auto`变量，类型是从它的初始化表达式推导来的，并且一些初始化表达式拥有的类型不是我们能预料到以及想要的。会出现这样情况的情景，以及你要如何做已经在**条款 2**和**条款 6**中讨论了，我不在此处展开了。相反，我将我的精力集中在你将传统的类型声明替代为`auto`时带来的代码可读性问题。

首先，深呼吸放松一下。`auto`是一个可选项，不是必选项。如果根据你的专业判断，使用显式的类型声明比使用`auto`会使你的代码更加清晰或者更好维护，或者在其他方面更有优势，你可以继续使用显式的类型声明。牢记一点，C++并没有在这个方面有什么大的突破，这种技术在其他语言中被熟知，叫做类型推断(`type inference`)。其他的静态类型过程式语言（像C#，D，Scala，Visual Basic）也有或多或少等价的特点，对静态类型的函数编程语言（像ML，Haskell，OCaml，F#等）另当别论。一定程度上说，这是受到动态类型语言的成功所启发，比如Perl，Python，Ruby，在这些语言中很少显式指定变量的类型。软件开发社区对于类型推断有很丰富的经验，这些经验表明这些技术和创建及维护巨大的工业级代码库没有矛盾。

一些开发者被这样的事实困扰，使用`auto`会消除看一眼源代码就能确定对象的类型的能力。然而，IDE提示对象类型的功能经常能缓解这个问题（甚至考虑到在**条款4**中提到的IDE的类型显示问题），并且，在很多情况下，一个比较抽象的对象类型和明确的对象类型是一样有用的。举个例子，你不需要知道具体的类型，只要知道一个对象是一个容器，或一个计数器，或一个智能指针，这常常就够了。假设使用良好的名字，这些抽象类型的信息就已经能在名字中得知了。

事实是显式地写出类型可能会引入一些难以察觉的错误，导致正确性或者效率问题，或者两者兼而有之。更重要的是，`auto`类型的变量能在初始化的表达式改变时自动改变，这意味着使用`auto`能简化重构。举个例子，如果一个函数的返回类型声明为了**int**，但是之后你觉得使用**long**会更好，在你下次编译程序的时候，如果你把调用这个函数的结果存在`auto`类型变量中，`auto`类型变量会自动改变自己的类型。如果结果被存放在明确声明的类型中，你就需要找到所有调用这个函数的地方并且手动修改它们。

| 要记住的东西                                                 |
| :----------------------------------------------------------- |
| 比起显式类型声明，`auto`变量必须被初始化，它常常免于由类型不匹配造成的可移植性和效率的问题，它能简化程序的重构，并且通常只需要写的更少的代码。 |
| `auto`类型变量的陷阱在**条款 2**和**条款 6**中讨论。         |

