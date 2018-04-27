> 说明：
>
> 1. 这部分很多是基于[item8](http://www.cnblogs.com/boydfd/p/4989335.html)修改的
> 2. 下面的代码可以参考[item8-github](https://github.com/AceCoooool/Effective-Modern-Cpp-Code/tree/master/item8)

条款8：优先使用`nullptr`而不是`0`或者`NULL`
=========================

先让我们看一些概念：字面上的`0`是一个**int**类型，而不是一个指针。如果C++发现`0`在上下文中只能被用作指针，它会勉强把`0`解释为一个null指针，但这只是一个应变的方案。C++的主要规则还是把`0`视为**int**，而不是一个指针。

实际上，`NULL`也是这样的。`NULL`的情况，在细节方面有一些不确定性，因为C++标准允许它的实现不管是不是**int**只需要给出一个数值类型（比如，**long**）。虽然这和`0`不一样，但是事实上没有关系，因为这里的问题不是`NULL`的具体类型，而是`0`和`NULL`都不是指针类型。

在C++98中，这个问题造成的最主要的影响就是指针和数值类型的重载会导致意外情况。传入一个`0`或者`NULL`给这样的重载，则永远不会调用指针版本的那个重载函数：

```cpp
void f(int);        // 函数f的三个重载
void f(bool);
void f(void*);

f(0);               // 调用 f(int)，而非f(void*)
f(NULL);            // 可能无法编译，但是通常会调用f(int)，不可能调用 f(void*)
```
关于`f(NULL)`行为的不确定性反映了：关于`NULL`类型的实现（编译器）拥有自由的权利。比如，如果`NULL`被定义为`0L`（也就是`long`类型的`0`），那么函数的调用是有歧义的，因为把`long`转换为`int`，`bool`，`void*`都被认为是同样可行的。关于调用`f`的一个有趣的事情是源代码字面上的意义（我使用`NULL`---null指针，调用`f`）和它实际的意义（我使用某种整形---不是一个指针，调用`f`）之间的矛盾。这种反直觉的行为也是使得C++98程序员避免使用重载指针和整数类型的原因。这个方针在C++11中仍然有效，因为，就算有这条**条款**的建议，尽管`nullptr`是更好的选择，有些开发者还是会继续使用`0`和`NULL`。

`nullptr`的优点不只是它不是一个整形类型。老实说，它也不是一个指针类型，但是你能把它想成是一个指向所有类型的指针。`nullptr`的实际类型是`std::nullptr_t`，在一个完美的循环定义（circular definition）中，`std::nullptr_t`被定义为`nullptr`的类型。`std::nullptr_t`类型能隐式转换到所有的原始（raw）指针类型，就是这个原因，让`nullptr`表现得像可以指向任意类型的指针。

使用`nullptr`作为参数去调用重载函数`f`将会调用`f(void*)`重载函数，因为`nullptr`不能被视为任何数值类型：
```cpp
f(nullptr);              //调用f(void*)重载体
```
使用`nullptr`而不是`0`或者`NULL`，可以避免重载解析上令人吃惊的行为，但是它的优势不仅限于此。它可以提高代码的可读性，尤其是牵扯到`auto`类型变量的时候。例如，你在一个代码库中遇到下面的代码：

```cpp
auto result = findRecord( /* arguments */);

if(result == 0){
    ...
}
```
如果你碰巧不知道（或者很难找出）`findRecord`返回的类型是什么，那么这里对于`result`是指针类型还是整形类型就不明确了。毕竟，`0`（被用来测试`result`的）即可以当做指针也可以当做整数类型。但是，如果你看到下面的代码：

```cpp
auto result = findRecord( /* arguments */);

if(reuslt == nullptr){
    ...
}
```
明显就没有歧义了：`result`一定是一个指针类型。

当涉及模板的时候，`nullptr`的光芒则显得更加耀眼了。假设你有一些函数，只有当对应的互斥量被锁定的时候，这些函数才可以被调用。每个函数的参数使用不同类型的指针：
```cpp
int    f1(std::shared_ptr<Widget> spw);             // 只有对应的
double f2(std::unique_ptr<Widget> upw);             // 互斥量被锁定
bool   f3(Widget* pw);                              // 才会调用这些函数
```
想传递空指针调用这些函数看上去像这样：
```cpp
std::mutex f1m, f2m, f3m;                           // 对应于f1, f2和f3的互斥量 

using MuxGuard = std::lock_guard<std::mutex>;       // C++11 版typedef；参加条款9   
...
{
    MuxGuard g(f1m);                                // 为f1锁定互斥量
    auto result = f1(0);                            // 将0当做空指针作为参数传给f1
}                                                   // 解锁互斥量

...

{
    MuxGuard g(f2m);                                // 为f2锁定互斥量
    auto result = f2(NULL);                         // 将NULL当做空指针作为参数传给f2
}                                                   // 解锁互斥量

...

{
    MuxGuard g(f3m);                               // 为f3锁定互斥量
    auto result = f3(nullptr);                     // 将nullptr当做空指针作为参数传给f3
}                                                  // 解锁互斥量
```
在前两个函数调用中没有使用`nullptr`是令人沮丧的，但是上面的代码是可以工作的，这一点是很重要的。然而，代码中重复的模式（锁定互斥量，调用函数，解锁互斥量）不仅令人忧伤，这更是令人不安的。避免这种重复风格的代码正是模板的设计初衷，因此，让我们模板化上面的模式：

```cpp
template<typename FuncType, typename MuxType, typename PtrType>
auto lockAndCall(FuncType func, MuxType& mutex,
                 PtrType ptr) -> decltype(func(ptr))
{
    MuxGuard g(mutex);
    return func(ptr);
}
```
如果这个函数的返回值类型（`auto ...->decltype(func(ptr))`）让你挠头不已，你应该到**条款3**寻求一下帮助，在那里我们已经做过详细的介绍。在C++14中，你可以看到，返回类型能被简化成一个简单的`decltype(auto)`：

```cpp
template<typename FuncType, typename MuxType, typename PtrType>
decltype(auto) lockAndCall(FuncType func, MuxType& mutex,   // C++14 
                           PtrType ptr) 
{
    MuxGuard g(mutex);
    return func(ptr);
}
```
给定`lockAndCall`模板（上边的任意版本），调用者可以写出下面的代码：
```cpp
auto result1 = lockAndCall(f1, f1m, 0);                       // 错误
...
auto result2 = lockAndCall(f2, f2m, NULL);                    // 错误
...
auto result3 = lockAndCall(f3, f2m, nullptr);                 // 正确
```
很好，它们能这么写，但是，正如注释说的，前面两种情况，代码无法通过编译。第一种调用情况的问题是当`0`被传入`lockAndCall`时，模板通过类型推导得知它的类型。`0`的类型总是`int`，这就是对`lockAndCall`的调用实例化时候的类型。不幸地，这意味着在`lockAndCall`中调用`func`时，会传入一个`int`类型，这与`f1`期望接受的参数`std::share_ptr<Widget>`是不兼容的。在`lockAndCall`调用中，尝试用传入`0`来表示一个null指针，但是实际上传入的是一个普通的`int`。尝试将`int`作为`std::share_ptr<Widget>`传给`f1`会导致一个类型冲突错误。对于用`0`调用`lockAndCall`会失败，是因为在template中，一个`int`被传入一个需要`std::shared_ptr`的函数。（译注：理解起来很简单，这里没有`int`到 指针类型的转换，只有将`0`视为指针这一情况。所以调用会失败。）

对调用`NULL`的情况的分析基本上和`0`是一样的。当`NULL`传递给`lockAndCall`时，从参数`ptr`推导出的类型是整数类型，当`ptr`（一个`int`或者类`int`的类型）传给`f2`，一个类型错误将会发生，因为这个函数期望得到的是一个`std::unique_ptr<Widget>`类型的参数。

相反，使用`nullptr`是没有问题的。当`nullptr`传递给`lockAndCall`，`ptr`的类型被推导为`std::nullptr_t`。当`ptr`被传递给`f3`，有一个由`std::nullptr_t`到`Widget*`的隐形转换，因为`std::nullptr_t`可以隐式转换为任何类型的指针。

真正的原因是，对于`0`和`NULL`，模板类型推导出了错误的类型（它们“对的"类型本应该是一个null指针），这是使用`nullptr`代替`0`或者`NULL`最重要的原因。使用`nullptr`，模板不会造成额外的困扰。另外结合`nullptr`在重载中不会导致像`0`和`NULL`那样的诡异行为的事实，胜负已定。当你需要用到空指针时，使用`nullptr`而不是`0`或者`NULL`。

|要记住的东西|
|:--------- |
|相较于`0`和`NULL`，优先使用`nullptr`|
|避免整数类型和指针类型之间的重载|

