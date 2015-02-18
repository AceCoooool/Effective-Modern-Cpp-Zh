条款11：优先使用delete关键字删除函数而不是使用private限制不实现的函数
=========================
如果你要给其他开发者提供代码，并且还不想让他们调用特定的函数，你只需要不声明这个函数就可以了。没有函数声明，没有就没有函数可以调用。这是没有问题的。但是有时候`C++`为你声明了一些函数，如果你想阻止客户调用这些函数，就不是那么容易的事了。

这种情况只有对“特殊的成员函数”才会出现，即这个成员函数是需要的时候`C++`自动生成的。条款17详细地讨论了这种函数，但是在这里，我们仅仅考虑复制构造函数和复制赋值操作子。这一节致力于`C++98`中的一般情况，这些情况可能在`C++11`中已经不复存在。在`C++98`中，如果你想压制一个成员函数的使用，这个成员函数通常是复制构造函数，赋值操作子，或者它们两者都包括。

在`C++98`中阻止这类函数被使用的方法是将这些函数声明为`private`，并且不定义它们。例如，在`C++`标准库中，`IO`流的基础是类模板`basic_ios`。所有的输入流和输出流都继承（有可能间接地）与这个类。拷贝输入和输出流是不被期望的，因为不知道应该采取何种行为。比如，一个`istream`对象，表示一系列输入数值的流，一些已经被读入内存，有些可能后续被读入。如果一个输入流被复制，是不是应该将已经读入的数据和将来要读入的数据都复制一下呢？处理这类问题最简单的方法是定义这类问题不存在，`IO`流的复制就是这么做的。

为了使`istream`和`ostream`类不能被复制，`basic_ios`在`C++98`中是如下定义的（包括注释）：
```cpp
	template <class charT, class traits = char_traits<charT> >
	class basic_ios :public ios_base {
	public:
	  ...

	  private:
	    basic_ios(const basic_ios& );                  // 没有定义
	    basic_ios& operator(const basic_ios&);         // 没有定义
    };
```
将这些函数声明为私有来阻止客户调用他们。故意不定义它们是因为，如果有函数访问这些函数（通过成员函数或者友好类）在链接的时候会导致没有定义而触发的错误。

在`C++11`中，有一个更好的方法可以基本上实现同样的功能：用`= delete`标识拷贝复制函数和拷贝赋值函数为删除的函数`deleted functions`。在`C++11`中`basic_ios`被定义为：
```cpp
	template <class charT, class traits = char_traits<charT> >
	class basic_ios : public ios_base {
	public:
	  ...
	  basic_ios(const basic_ios& ) = delete;
	  basic_ios& operator=(const basic_ios&) = delete;
	  ...
    };
```