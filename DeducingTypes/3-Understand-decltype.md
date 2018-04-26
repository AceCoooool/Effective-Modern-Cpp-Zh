> 说明：
>
> 1. 这部分很多是基于[item3](http://www.cnblogs.com/boydfd/p/4961764.html)修改的
> 2. 下面的代码可以参考[item3-github](https://github.com/AceCoooool/Effective-Modern-Cpp-Code/tree/master/item3)

条款三：理解`decltype`
=========================

`decltype`是一个怪异的发明。给定一个变量名或者表达式，`decltype`会告诉你这个变量名或表达式的类型。大多数情况下，`decltype`的返回类型往往也是你期望的。然而有时候，它提供的结果会使开发者极度抓狂而不得不去参考其他文献或者在线的Q&A网站。

我们从一般情况（没有意外的结果）开始，这种情况下`decltype`不会有令人惊讶的行为。与`templates`和`auto`在类型推导中行为相比（请见条款一和条款二），`decltype`一般只是复述一遍你所给他的变量名或者表达式的类型，如下：

```cpp
   const int i = 0;            // decltype(i) is const int

   bool f(const Widget& w);    // decltype(w) is const Widget&
                               // decltype(f) is bool(const Widget&)
   struct Point{
     int x, y;                 // decltype(Point::x) is int
   };

   Widget w;                   // decltype(w) is Widget

   if (f(w)) ...               // decltype(f(w)) is bool

   template<typename T>        // simplified version of std::vector
   class vector {
   public:
     ...
     T& operator[](std::size_t index);
     ...
   };

   vector<int> v;              // decltype(v) is vector<int>
   ...
   if(v[0] == 0)               // decltype(v[0]) is int&
```
看到没有？毫无令人惊讶的地方。

在C++11中，`decltype`最主要的用处可能就是用来声明**返回值类型依赖于参数类型的**函数模板。举个例子，假设你要写一个函数，这个函数需要一个支持下标操作（"[]"）的容器，然后根据下标操作来识别出用户，函数的返回值类型应该和下标操作的类型相同。

以`T`为元素类型的容器，`operator[]`操作通常返回`T&`。对`std::deque`一般是成立的，例如，对`std::vector`的大多数情况均是成立的。然而，对于`std::vector<bool>`，`operator[]`操作不返回`bool&`，而是返回一个全新的对象。发生这种情况的原理将在条款六中讨论，但是在这里，重点是：**operator[]**函数返回的类型取决于容器的类型。(即和`T`有关，比如此处的`bool`)

decltype让表达式变得简单。我们将写下第一个template，展示如何用decltype计算返回值。template可以进一步精炼，但是我们先写成这样：

```cpp
template<typename Container, typename Index>    // works, but requires refinements
auto authAndAccess(Container& c, Index i)-> decltype(c[i])                                   
{
    authenticateUser();
    return c[i];
}
```
函数前面的那个auto的使用没有做任何类型的推导。当然了，这标志着C++11使用了返回值类型后置的语法。也就是，函数的返回值类型将跟在参数列表（"`->`"的后面）后面声明。一个后置的返回类型拥有一个优点，就是函数的参数能用来确定返回值类型。例如在函数`authAndAccess`中，我们使用了`c`和`i`来定义返回值类型。按传统的使用方法，我们让返回值类型处于函数名的前面，那么`c`和`i`就不能使用了，因为解析到这时，他们还没被声明出来。

使用这个声明方式，就和我们的需求一样，authAndAccess根据我们传入的容器，以这个容器的**operator[]**操作返回的类型来作为它自身的返回值类型。

C++11允许我们推导一些比较简单的lambdas表达式的返回类型，并且在C++14中，把这种推导扩张到了所有的lambdas表达式和所有的函数中，包括那些有多条语句的复杂的函数。在authAndAccess中，这意味着在C++14中，我们能忽略返回值类型，只需使用auto即可。在这样的声明形式下，auto意味着类型推导将会发生。尤其是，它意味着编译器将会根据函数的实现来推导函数的返回值类型。

```cpp
template<typename Container, typename Index>         // C++14;
auto authAndAccess(Container &c, Index i)            // not quite
{                                                    // correct
    authenticateUser();
    return c[i];
}                                 // return type deduced from c[i]
```

条款二解释了一个返回auto的函数，编译器采用template的类型推导规则。在这种情况下，这是有问题的。就像我们讨论的那样，对于大多数容器，operator[]返回的是`T&`，但是**条款1**解释了在template类型推导中，表达式的引用属性会被忽略（情况3），考虑下这对于客户代码意味着什么：

```cpp
std::deque<int> d;
...
authAndAccess(d, 5) = 10;   // 这回返回d[5], 然后把10赋给它。但是这会出现编译错误（给一个右值赋值）
```

这里，`d[5]`返回一个`int&`，但是对authAndAccess的auto返回类型的推导将会去掉引用属性，这产生了一个`int`类型,`int`是函数的返回值类型，是一个右值，然后这段代码尝试给一个右值**int**赋值一个10。这在C++中是禁止的，所以代码无法编译通过。

为了让authAndAccess能工作地像我们预期的那样，我们需要对返回值类型使用decltype类型推导，也就是明确authAndAccess应该返回表达式`c[i]`返回的类型。C++的规则制定者们预料到了在一些类型需要推测的情况下，用decltype类型推导规则来推导的需求，所以在C++14中，通过`decltype(auto)`类型说明符来让之成为可能。这一开始看起来可能有点矛盾的东西（`decltype`和`auto`）确实完美地结合在一起：auto明确了类型需要被推导，decltype明确了推导时使用decltype推导规则。因此我们像这样能写出authAndAccess的代码：

```cpp
template<typename Container, typename Index>          // C++14; works,
decltype(auto) authAndAccess(Container &c, Index i)   // but still requires refinement         
{                                             
    authenticateUser();
    return c[i];
}
```

现在`authAndAccess`的返回类型就是`c[i]`的返回类型。在一般情况下，`c[i]`返回`T&`，`authAndAccess`就返回`T&`；在不常见的情况下，`c[i]`返回一个对象，`authAndAccess`也返回一个对象。

`decltype(auto)`的使用不止局限于函数返回类型，当你对正在初始化的表达式使用decltype类型的类型推导规则时,它也能很方便地用在变量的声明上：

```cpp
Widget w;
const Widget& cw = w;
auto myWidget1 = cw;           // auto type deduction，myWidget1's type is Widget 
decltype(auto) myWidget2 = cw  // decltype type deduction: myWidget2's type is const Widget&
```

我知道，到目前为止会有两个问题困扰着你。一个是我们前面提到的，对`authAndAccess`的改进。我们在这里讨论。

再次看一下`C++14`版本的`authAndAccess`的声明：

```cpp
template<typename Container, typename Index>
decltype(auto) anthAndAccess(Container &c, Index i);
```

这个容器是通过非`const`左值引用传入的，因为一个对元素的引用允许客户更改容器的。但是这也意味着不可能将右值容器传入这个函数。右值不能和一个左值引用绑定（除非是`const`的左值引用，这不是这里的情况）。

诚然，传递一个右值容器给`authAndAccess`是一种极端情况。一个右值容器作为一个临时对象，在 `anthAndAccess` 所在语句的最后被销毁，这意味着对这样一个容器（authAndAccess的返回值）的引用在语句结束时会产生未定义的结果。但是，传递一个临时变量给authAndAccess还是有意义的。一个客户可能简单地想产生临时容器中元素的一份拷贝，举个例子：

```cpp
std::deque<std::string> makeStringDeque();  // factory function
// make copy of 5th element of deque returned
// from makeStringDeque
auto s = authAndAccess(makeStringDeque(), 5);
```

为了支持这样的使用，意味着我们需要修改authAndAccess的声明，让它能同时接受lvalues和rvalues。。重载可以解决这个问题（一个重载负责左值引用参数，另外一个负责右值引用参数），但是我们将有两个函数需要维护。避免这种情况的一个方法是使authAndAccess有一个既可以绑定左值又可以绑定右值的引用参数，条款24将说明这正是universal reference所做的。因此authAndAccess可以像如下声明：

```cpp
template<typename Container, typename Index>            // c is now a
decltype(auto) authAndAccess(Container&& c, Index i);   // universal reference
```

在这个template中，我们不知道容器的类型，所以，这意味着我们同样不知道容器对象使用的索引类型。对于未知对象使用传值（pass-by-value）方式通常会因为没必要的拷贝而对效率产生很大的影响，以及对象切割（条款41）的问题，并会被我们的同事嘲笑，但是在容器索引的使用上，跟随标准库（比如，**std::string**,**std::vector**以及**std::deque**的`operator[]`）的脚步看起来是合理的。所以我们将坚持以传值（pass-by-value）的方式使用它。

然而，我们需要更新这个模板的实现，将`std::forward`应用给universal引用，使得它和条款25中的建议是一致的。（`std::forward`可以保存参数的左值或右值特性）

```cpp
template<typename Container, typename Index>          // final
decltype(auto) authAndAccess(Container&& c, Index i)  // C++14 version
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

这个例子应该会做到所有我们想要的事情了，但是它需要C++14的编译器。如果你没有，你将需要使用C++11版本的template。除了你需要自己明确返回值类型外，它和C++14的版本是一样的：

```cpp
template<typename Container, typename Index>    // final C++11 version
auto authAndAccess(Container&& c, Index i) -> decltype(std::forward<Container>(c)[i])           
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

另外一个容易被你挑刺的地方是我在本条款开头的那句话：`decltype`几乎所有时候都会输出你所期望的类型，但是有时候它的输出也会令你吃惊。诚实的讲，你不太可能遇到这种例外，除非你是一个重型库的实现人员。

为了彻底的理解`decltype`的行为，你必须使你自己对一些特殊情况比较熟悉。这些特殊情况太晦涩难懂，以至于很少有书会像本书一样讨论，但是了解它们同时也可以增加我们对`decltype`的认识。

对一个变量名使用`decltype`得到这个变量名的声明类型。变量名属于左值表达式，但这并不影响`decltype`的行为。然而，对于一个比变量名更复杂的左值表达式，`decltype`保证返回的类型是左值引用。一个lvalue表达式除了变量名会推导出`T`，其他情况decltype推导出来的类型就是`T&`。这很少产生什么影响，因为绝大部分左值表达式的类型有内在的左值引用修饰符。例如，需要返回左值的函数返回的总是左值引用。

这里有一个隐含的行为值得你去意识到：

```cpp
int x = 0;
```
`x`是一个变量名，因此`decltyper(x)`是`int`。但是如果给`x`加上括号"(x)"就得到一个比变量名复杂的表达式。作为变量名，`x`是一个左值，同时C++定义表达式`(x)`也是左值。因此`decltype((x))`是`int&`。给一个变量名加上括号会改变`decltype`返回的类型。

在C++11中，这仅仅是个好奇的探索，但是和C++14中对`decltype(auto)`支持相结合，函数中返回语句的一个细小改变会影响对这个函数的推导类型。
```cpp
decltype(auto) f1()
{
    int x = 0;
    ...
    return x;       // decltype(x) is int, so f1 returns int
}

decltype(auto) f2()
{
    int x = 0;
    return (x);     // decltype((x)) is int&, so f2 return int&
}
```
记住f2不仅仅是返回值和f1不同，它还返回了一个局部变量。这是一种把你推向未定义行为的陷阱代码。

最主要的经验教训就是当使用`decltype(auto)`时要多留心一些。被推导的表达式中看上去无关紧要的细节都可能影响`decltype(auto)`返回的类型。为了保证推导出的类型是你所期望的，请使用条款4中的技术。

同时，不要忽视大局。当然，decltype（无论只有decltype或者还是和auto联合使用）有可能偶尔会产生类型推导的惊奇行为，但是这不是常见的情况。一般情况下，decltype会产生你期望的类型。当decltype应用在name时，它总产生你想要的情况，在这情况下，decltype就像听起来那样：它推导出name的声明类型。

|要记住的东西|
|:--------- |
|`decltype` 大多数情况下总是会不加修改地产出变量和表达式的类型|
|对于非变量名的类型为`T`的左值表达式，`decltype`总是返回`T&`|
|C++14支持`decltype(auto)`，它的行为就像`auto`从初始化操作来推导类型，但是它推导类型时使用`decltype`的规则|
