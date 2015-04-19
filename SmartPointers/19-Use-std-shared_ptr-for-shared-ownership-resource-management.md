#Item 19:使用std::shared_ptr来管理共享式的资源
使用垃圾回收机制的程序员指责并且嘲笑C++程序员阻止内存泄露的做法。“你们tmd是原始人!”他们嘲笑道。“你们有没有看过1960年Lisp语言的备忘录？应该用机器来管理资源的生命周期，而不是人类。”C++程序员开始翻白眼了:"你们懂个屁，如果备忘录的内容意味着唯一的资源是内存而且回收资源的时机是不确定性的，那么我们宁可喜欢具有普适性和可预测性的析构函数."但是我们的回应部分是虚张声势。垃圾回收确实非常方便，手动来控制内存管理周期听起来像是用原始工具来做一个记忆性的内存回路。为什么我们不两者兼得呢？做出一个既可以想垃圾回收那样自动，且可以运用到所有资源，具有可预测的回收时机(像析构函数那样)的系统。

`std::shared_ptr`就是C++11为了达到上述目标推出的方式。一个通过`std::shared_ptr`访问的对象被指向它的指针通过共享所有权(shared ownership)方式来管理.没有一个特定的`std::shared_ptr`拥有这个对象。相反，这些指向同一个对象的`std::shared_ptr`相互协作来确保该对象在不需要的时候被析构。当最后一个`std::shared_ptr`不再指向该对象时(例如，因为`std::shared_ptr`被销毁或者指向了其他对象)，`std::shared_ptr`会在此之前摧毁这个对象。就像GC一样，使用者不用担心他们如何管理指向对象的生命周期，而且因为有了析构函数，对象析构的时机是可确定的。

一个`std::shared_ptr`可以通过查询资源的引用计数(reference count)来确定它是不是最后一个指向该资源的指针，引用计数是一个伴随在资源旁的一个值，它记录着有多少个`std::shared_ptr`指向了该资源。`std::shared_ptr`的构造函数会自动递增这个计数，析构函数会自动递减这个计数，而拷贝构造函数可能两者都做(比如，赋值操作`sp1=sp2`,sp1和sp2都是`std::shared_ptr`类型，它们指向了不同的对象，赋值操作使得sp1指向了原来sp2指向的对象。赋值带来的连锁效应使得原来sp1指向的对象的引用计数减1，原来sp2指向的对象的引用计数加1.)如果`std::shared_ptr`在执行减1操作后发现引用计数变成了0，这就说明了已经没有其他的`std::shared_ptr`在指向这个资源了，所以`std::shared_ptr`直接析构了它指向的空间。

引用计数的存在对性能会产生部分影响

 * `std::shared_ptrs`是原生指针的两倍大小，因为它们内部除了包含了一个指向资源的原生指针之外，同时还包含了指向资源的引用计数
 
 * 引用计数的内存必须被动态分配.概念上来说，引用计数会伴随着被指向的对象，但是被指向的对象对此一无所知。因此，他们没有为引用计数准备存储空间。(一个好消息是任何对象，即使是内置类型，都可以被`std::shared_ptr`管理。)Item21解释了用`std::make_shared`来创建`std::shared_ptr`的时候可以避免动态分配的开销，但是有些情况下`std::make_shared`也是不能被使用的。不管如何，引用计数都是存储为动态分配的数据

 * 引用计数的递增或者递减必须是原子的，因为在多线程环境下，会同时存在多个写者和读者。例如，在一个线程中，一个`std::shared_ptr`指向的资源即将被析构(因此递减它所指向资源的引用计数)，同时，在另外一个线程中，一个`std::shared_ptr`指向了同一个对象，它此时正进行拷贝操作(因此要递增同一个引用计数)。原子操作通常要比非原子操作执行的慢，所以尽管引用计数通常只有一个word大小，但是你可假设对它的读写相对来说比较耗时。

当我写到:`std::shared_ptr`构造函数在构造时"通常"会增加它指向的对象的引用计数时，你是不是很好奇？创建一个新的指向某对象的`std::sharedptr`会使得指向该对象的`std::sharedptr`多出一个，为什么我们不说构造一个`std::sharedptr`总是会增加引用计数？

Move构造函数是我为什么那么说的原因。从另外一个`std::shared_ptr` move构造(Move-constructing)一个`std::shared_ptr`会使得源`std::shared_ptr`指向为null,这就意味着新的`std::shared_ptr`取代了老的`std::shared_ptr`来指向原来的资源，所以就不需要再修改引用计数了。Move构造`std::shared_ptr`要比拷贝构造`std::shared_ptr`快：copy需要修改引用计数，然而拷贝缺不需要。对于赋值构造也是一样的。最后得出结论，move构造要比拷贝构造快，Move赋值要比copy赋值快。

像`std::unique_ptr`(Item 18)那样,`std::shared_ptr`也把delete作为它默认的资源析构机制。但是它也支持自定义的deleter.然后，它支持这种机制的方式不同于`std::unique_ptr`.对于`std::unique_ptr`,自定义的deleter是智能指针类型的一部分，对于`std::shared_ptr`,情况可就不一样了:

```cpp
auto loggingDel = [](widget *pw)
					{	
						makeLogEntry(pw);
						delete pw;
					}//自定义的deleter(如Item 18所说)
std::unique_ptr<Widget, decltype(loggingDel)>upw(new Widget, loggingDel);//deleter类型是智能指针类型的一部分

std::shared_ptr<Widget> spw(new Widget, loggingDel);//deleter类型不是智能指针类型的一部分
```

std::shared_prt的设计更加的弹性一些，考虑到两个std::shared_ptr<Widget>,每一个都支持不同类型的自定义deleter(例如，两个不同的lambda表达式):

```cpp
auto customDeleter1 = [](Widget *pw) {...};
auto customDeleter2 = [](Widget *pw) {...};//自定义的deleter,属于不同的类型

std::shared_prt<Widget> pw1(new Widget, customDeleter1);
std::shared_prt<Widget> pw2(new Widget, customDeleter2);
```

因为pw1和pw2属于相同类型，所以它们可以放置到属于同一个类型的容器中去:

```cpp
std::vector<std::shared_ptr<Widget>> vpw{ pw1, pw2 };
```

它们之间可以相互赋值，也都可以作为一个参数类型为`std::shared_ptr<Widget>`类型的函数的参数。所有的这些特性，具有不同类型的自定义deleter的`std::unique_ptr`全都办不到，因为自定义的deleter类型会影响到`std::unique_ptr`的类型。

与`std::unique_ptr`不同的其他的一点是，为`std::shared_ptr`指定自定义的deleter不会改变`std::shared_ptr`的大小。不管deleter如何，一个`std::shared_ptr`始终是两个pointer的大小。这可是个好消息，但是会让我们一头雾水。自定义的deleter可以是函数对象，函数对象可以包含任意数量的data.这就意味着它可以是任意大小。涉及到任意大小的自定义deleter的`std::shared_ptr`如何保证它不使用额外的内存呢？

它肯定是办不到的，它必须使用额外的空间来完成上述目标。然而，这些额外的空间不属于`std::shared_ptr`的一部分。额外的空间被分配在堆上，或者在`std::shared_ptr`的创建者使用了自定义的allocator之后，位于该allocator管理的内存中。我之前说过，一个`std::shared_ptr`对象包含了一个指针，指向了它所指对象的引用计数。此话不假，但是却有一些误导性，因为引用计数是一个叫做控制块(control block)的很大的数据结构。每一个由`std::shared_ptr`管理的对象都对应了一个控制块。改控制块不仅包含了引用计数，还包含了一份自定义deleter的拷贝(在指定好的情况下).如果指定了一个自定义的allocator,也会被包含在其中。控制块也可能包含其他的额外数据，比如Item 21条所说，一个次级(secondary)的被称作是weak count的引用计数，在本Item中我们先略过它。我们可以想象出`std::shared_ptr<T>`的内存布局如下所示: