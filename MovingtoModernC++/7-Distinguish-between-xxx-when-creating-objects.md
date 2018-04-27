> 说明：
>
> 1. 这部分很多是基于[item7](http://www.cnblogs.com/boydfd/p/4981504.html)修改的
> 2. 下面的代码可以参考[item7-github](https://github.com/AceCoooool/Effective-Modern-Cpp-Code/tree/master/item7)

条款7:创建对象时使用()和{}的区别
===============================

在C++11中，你可以有多种语法选择用以对象的初始化，这样的语法显得混乱不堪并让人无所适从（这体现了语法丰富造成的“烂摊子”），`()，=，{}`均可以用来进行初始化：
```cpp
int x(0);	//使用()进行初始化
int y = 0;	//使用=进行初始化
int z{0};	//使用{}进行初始化
```
在很多情况下，可以同时使用=和{}组合的方式
```cpp
int z = {0};	//使用{}和=进行初始化
```
（在这个**条款**中，我通常忽略“等号加花括号”的语法，因为C++通常把它和“只使用花括号”的情况同样对待。）

”烂摊子“指的是使用等号来初始化常常误导C++初学者，这里发生了赋值操作（尽管不是这样的）。对于built-in类型比如int，只是术语上的不同，但是对于user-defined类型（用户自定义类型），把初始化和赋值区分开来是很重要的，因为它们调用不同的函数：

```cpp
Widget w1;	    //调用了构造函数
Widget w2 = w1;	//不是赋值语句，调用了拷贝构造函数
w1 = w2;	    //赋值语句，调用了operator=
```

尽管这里有好几种不同的初始化语法了，C++98在很多情况下仍然没有办法实现想要的初始化。举个例子，我们没有办法直接指示一个STL容器使用特定的集合来创建（比如1，3，和5）。

了解决多种初始化语法之间的混乱，也为了解决他们无法涵盖的初始化情况，C++11引入了一种标准初始化：至少在概念上，它是一个单一的初始化语法，可以用在任何地方，做到任何事情。它基于花括号，所以因为这个原因，我更喜欢用术语花括号初始化来形容它。”标准初始化“是一个想法，”花括号初始化“是一个语法概念。

花括号初始化能让你做到之前你做不到的事，比如使用花括号，明确容器的初始内容是很简单的：

```cpp
std::vector<int> v{1,3,5};  //v的初始值是1,3,5
```

花括号同样可以用来明确non-static成员变量的初始值。C++11中的一项新语法就是支持`{}`像`=`一样初始化，但是`()`不行：

```c++
class Widget{
   ...
   private:
	int x{0};	//正确，x默认值为0
	int y = 0;	//正确
	int z(0);	//错误
}
```

另一方法，非拷贝对象(std:atomics-参考第40条)可以使用{}和()初始化，但是"="不行:

另外，不能拷贝的对象（比如，`std::atomics`--看 **条款 40**）能用花括号和圆括号初始化，但是不能用`=`初始化：

```cpp
std::atomic<int> ai1{0};	//正确
std::atomic<int> ai2(0);	//正确
std::atomic<int> ai3 = 0;	//错误
```
因此这很容易理解为什么花括号初始化被称为”标准“。因为，C++中指定初始化值的三种方式中，只有花括号能用在每个地方。

花括号初始化有一个新奇的特性，它阻止在built-in类型中的隐式类型转换（narrowing conversation）。如果表达式的值不能保证被初始化对象表现出来，代码将无法通过编译：

```cpp
double x,y,z;
...
int sum1{x + y + z};	//错误，双精度类型数值相加无法表达为整形
```

使用`()`和`=`进行初始化时不会检查隐式类型转换，因为这么做的话，会让历史遗留的代码无法使用：

```cpp
int sum2(x + y + z);	//正确，表达式的值为转换成整形
int sum3 = x + y + z;	//同上
```

花括号初始化的另外一个值得一谈的特性是它能避免C++最令人恼火的解析（注：此处解析你可以理解为编译器对语句的理解）。一方面，C++的规则中，所有能被解释为声明的东西肯定会被解释为声明，经常困扰开发者的一个问题是当开发者想用默认构造函数构造一个对象时，他常常会不小心声明了一个函数。问题的根本就是你想使用一个参数调用一个构造函数，你能这么做：

```cpp
Widget w1(10);		//调用Widget的构造函数并传递参数10
```

但是如果你使用类似的语法，尝试使用0个参数调用Widget的构造函数，你会声明一个函数，而不是一个对象：

```cpp
Widget w2();		//非常的歧义，定义了一个返回Widget对象的w2函数
```

因为函数声明不可能使用`{}`传递参数列表，所以使用`{}`来调用构造函数不会带来这个问题：

```cpp
Widget w3{};		//调用Widget无参构造函数
```

对于花括号初始化还有很多值得一提的地方：这种语法能用在最宽广的范围，它能阻止隐式收缩转换（narrowing convertions），并且它能避免C++最让人恼火的解析。三个优点！但是为什么这个条款的标题不是”优先使用花括号初始化语法“呢？

因为花括号初始化的缺点是它会伴随一些意外的行为。这些行为来自于花括号初始化与`std::initializer_lists`以及构造函数的重载解析之间异常纠结的关系。它们常常导致一个结果，那就是代码看来本“应该”是这么做的，事实上却做了别的事。举个例子，**条款 2**解释了当auto声明的变量使用花括号初始化时，它的类型被推导为`std::initializer_list`，而使用一样的初始化表达式，别的方式声明的变量能产生更符合实际的类型。结果就是，你越喜欢auto，你就越不喜欢花括号初始化。

在调用构造函数时，只要参数不涉及`std::initializer_list`，圆括号和花括号是一样的：

```cpp
class Widget{
public:
    Widget(int i,bool b);	//构造函数没有声明为std::initializer_list的参数
    Widget(int i,double d);	
}；
Widget w1(10,true);		    //调用第一个构造函数
Widget w2{10,true};		    //同样调用第一个构造函数
Widget w3(10,5.0);		    //调用第二个构造函数
Widget w4{10,5.0};		    //同样调用第二个构造函数
```
但是，如果有一个或多个构造函数声明了一个类型为`std::initializer_list`的参数，使用花括号初始化语法将强烈地偏向于调用带`std::initializer_list`参数的构造函数。强烈意味着，当使用花括号初始化语法时，编译器只要有任何机会能调用带`std::initializer_list`参数的构造函数，编译器就会采用这种解释。举个例子，如果给上面`Widget`类增加一个带`std::initializer_list`参数的构造函数：

```cpp
class Widget{
public:
   Widget(int i,bool b);	                       //和上面一样
   Widget(int i,double d);	                       //和上面一样
   Widget(std::initializer_list<long double> il);  //新加的构造函数
   ...
};
```

如下，尽管`std::initializer_list`元素的类型是`long double`，Widget w2和w4还将使用“新的构造函数”来构造对象（比起`non-std::initializer_list`构造函数，两个参数都是更加糟糕的匹配）：

```cpp
Widget w1(10,true);	//使用()构造函数
Widget w2{10,true};	//使用{}构造函数,调用std::initializer_list参数，10和true被转换成long dobule型
Widget w3(10,5,0);	//使用()构造函数
Widget w4{10,5.0};	//使用{}，调用std::initializer_list参数，10和5.0被转换为long double
```
甚至普通的拷贝和移动构造函数也会被`std::initializer_list`构造函数所劫持：

```cpp
class Widget{
public:
    Widget(int i,bool b);				           //同上
    Widget(int i,double d);				           //同上
    Widget(std::initializer_list<long double> il);	//同上

    operator float() const;				//转换成float型
    ...
};
Widget w5(w4);					//使用()，调用拷贝构造函数
Widget w6{w4};					//使用{}，调用std::initializer_list参数类型构造函数,(w4被转换成                                     //float，然后float再转换成long double)
Widget w7(std::move(w4));		 //使用()，调用move构造函数
Widget w8{std::move(w4)};		 //使用{}，调用std::initializer_list构造函数，和w6原因一样
```
同样情况下，编译器会优先用```std::initializer_lists```来匹配用{}初始化的对象，哪怕有更加匹配的非```std::initializer_lists```构造函数存在。例如

```cpp
class Widget{
public:
   Widget(int i,bool b);			       //同上
   Widget(int i,double d);			       //同上
   Widget(std::initializer_list<bool> il);	//元素类型是bool

   ...
};
Widget w{10,5.0};				           //错误！，要求类型收窄的转换
```
这里，编译器将忽视前两个构造函数（第二个构造函数提供了最合适的两个参数类型）并且尝试调用带`std::initializer_list`参数的构造函数。调用这个构造函数需要把一个`int`（10）和一个`double`（5.0）转换到`bool`。两个转换都要求收缩（narrowing）（bool不能显式地代表这两个值），并且收缩转换（narrowing conversions）在花括号初始化中是被禁止的，所以这个调用是无效的，然后代码编译不通过。

只有当无法将`{}`初始化的参数转换成`std::initializer_list`的时候，编译器才会回到正常的重载解析。举个例子，如果我们把`std::initializer_list<bool>`构造函数替换为带`std::initializer_list<string>`参数的构造函数，`non-std::initializer_list`构造函数才能重新成为候选人，因为这里没有任何办法把`bools`转换为`std::string`：

```cpp
class Widget{
public:
    Widget(int i,bool b);			//同上
    Widget(int i,double d);			//同上
    
    //std::initalizer_list元素类型是std::string
    Widget(std::initializer_list<std::string> il); //没有隐式转换
    ...
};

Widget w1(10,true);				//使用()初始化，调用第一个构造函数
Widget w2{10,true};				//使用{}初始化，调用第一个构造函数
Widget w3(10,5.0);				//使用()初始化，调用第二个构造函数
Widget w4{10,5.0};				//使用{}初始化，调用第二个构造函数
```
我们现在已经接近完成探索`{}`初始化和重载构造函数了，但是这里还有一种有趣的情况值得一提。假设你使用空的花括号来构造对象，对象同时支持`std::initializer_list`作为参数的构造函数。此时，空的`{}`参数指什么呢？如果表示空的参数，那么你将调用默认构造函数，如果表示空的`std::initializer_list`，那么调用无实际传入参数的`std::initializer_list`构造函数。

规则是你将调用默认构造函数。空的{}意味着无参数，并不是空的`std::initializer_list`：

```cpp
class Widget{
public:
    Widget();					//默认构造函数

    Widget(std::initializer_list<init> il);	//std::initializer_list构造函数
    ...
};
Widget w1;					    //调用默认构造函数
Widget w2{};					//调用默认构造函数
Widget w3();					//令人恼火的解析，声明了一个函数！
```
如果你想使用空的`initializer_list`参数来调用`std::initializer_list`参数的构造函数，你可以使用空的{}作为参数--把空的`{}`放在小`()`之中来传递你的内容:

```cpp
Widget w4({});					//使用空列表作为参数调用std::initializer_list型的构造函数
Widget w5{{}};					//如上
```
在这种情况下，看起来很神秘的大括号初始化`std::initializer_list`，和构造函数重载等东西萦绕在你的脑海中，你可能好奇，在日常编程中这些信息会造成多大的麻烦。比你想的更多，因为一个直接被影响的class就是`std::vector`。`std::vector`有一个`non-std::initializer_list`的构造函数允许你明确初始的容器大小以及每个元素的初始值，但是它也有一个`std::initializer_list`构造函数允许你明确初始的容器值。如果你创建一个数值类型的`std::vector`（比如`std::vector<int>` ）并且传入两个参数给构造函数，根据你使用的是圆括号或者花括号，会造成很大的不同：

```cpp
std::vector<int> v1(10,20)			//使用非std::initializer_list参数的构造函数，结果构造了10个元                                      //素的std::vector对象，每个对象的值都是20
std::vector<int> v2{10,20}			//使用std::initializer_list参数的构造函数，结果构造了2个元素的                                      //std::vector对象，两个元素分别是10和20
```
让我们退一步来看`std::vector`，圆括号，花括号以及构造函数重载解析规则的细节。在这个讨论中有两个重要的问题：第一个问题，作为一个class的作者，你需要意识到如果你设置了一个或多个`std::initializer_list`构造函数，客户代码使用花括号初始化时将会只看见`std::initializer_list`版本的重载函数。因此，最好在设计你的构造函数时考虑到，客户使用圆括号或者花括号都不会影响到构造函数的调用。换句话说，就像你现在看到的`std::vector`接口设计是错误的，并且在设计你的类的时候应该避免这个错误。

另一个问题是，如果你有一个类，这个类没有`std::initializer_list`构造函数，然后你想增加一个，客户代码中使用花括号初始化的代码会找到这个构造函数，并把从前本来解析为调用`non-std::initializer_list`构造函数的代码重新解析为调用这个构造函数。当然，这种事情经常发生，只要你增加一个新的函数来重载：原来解析为调用旧函数的代码可能会被解析为调用新函数。`std::initializer_list`构造函数重载的不同之处在于它不同别的版本的函数竞争，它直接占据领先地位以至于其它重载函数几乎不会被考虑。所以只有经过仔细考虑过后你才能增加这样的重载（`std::initializer_list`构造函数）。

第二个要考虑是，作为类的客户，你必须在选择用圆括号或花括号创建对象的时候仔细考虑。很多开发者使用一种符号作为默认选择，只有在必要的时候才使用另外一种符号。默认使用花括号的人被它的大量优点（无可匹敌的应用宽度，禁止收缩转换，避免C++最令人恼火的解析）所吸引。这些人知道一些情况（比如，根据容器大小和所有初始化元素值创建`std::vector`时）圆括号是必须的。另外一方面，一些向圆括号看齐的人使用圆括号作为他们的默认符号。他们喜欢的是，圆括号同C++98传统语法的一致性，避免错误的`auto`推导问题，以及创建对象时不会不小心被`std::initializer_list`构造函数所偷袭。他们有时候会只考虑花括号（比如，使用特定元素值创建一个容器）。这里没有一个标准，没有说用哪一种更好，所以我的建议是选择一种，并坚持使用它。

如果你是**template**的作者，在创建对象时选择使用圆括号还是花括号时会尤其沮丧，因为通常来说，这里没办法知道哪一种会被使用。举个例子，假设你要创建一个对象使用任意类型任意数量的参数。一个概念上的可变参的**template**看起来像这样：

```cpp
template<typename T, typename...Ts>         //对象的类型是T，参数的类型Ts
void doSomeWork(Ts&&... params)
{
    使用params创建局部T对象...
    ...
}
```

这里有两种形式把伪代码转换成真正的代码（要知道`std::forward`的信息，请看**条款 25**）：

```cpp
T localObject(std::forward<Ts>(params)...);		//使用小()
T localObject{std::forward<Ts>(params)...};		//使用{}
```
所以考虑下面的代码：

```cpp
std::vector<int> v;
...
doSomeWork<std::vector<int>>(10,20);
```
如果`doSomeWork`使用圆括号来创建`localObject`，最后的`std::vector`将有10个元素。如果`doSomeWork`使用花括号，最后的`std:vector`将有两个元素。哪一种是对的？doSomeWork的作者不知道，只有调用者知道。

| 要记住的东西                                                 |
| :----------------------------------------------------------- |
| 花括号初始化是用途最广的初始化语法，它阻止收缩转换，能避免C++最令人恼火的解析 |
| 在构造函数重载解析中，只要有可能，花括号初始化都对应`std::initializer_list`构造函数，就算其他的构造函数看起来更匹配。 |
| 选择圆括号或花括号创建对象能得到完全不同结果的例子是用两个参数创建一个`std::vector`的对象。 |
| 在`template`内部，创建对象时选择圆括号或花括号是一个难题。   |