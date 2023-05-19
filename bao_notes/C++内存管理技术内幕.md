





[C++内存管理技术内幕](https://blog.csdn.net/LaoJiu_/article/details/49975497)



# 1 内存管理

## 1.1 C++内存管理详解
### 1.1.1 内存分配方式
#### 1.1.1.1 分配方式简介
　　在C++中，内存分成5个区，分别是堆、栈、自由存储区、全局/静态存储区和常量存储区。

　　栈，在执行函数时，函数内局部变量的存储单元都可以在栈上创建，函数执行结束时这些存储单元自动被释放。栈内存分配运算内置于处理器的指令集中，效率很高，但是分配的内存容量有限。

　　自由存储区，就是那些由new分配的内存块，它们的分配与释放由应用程序去控制，一般一个new就要对应一个delete。如果没有释放掉，那么在程序结束后，操作系统会自动回收。

　　堆，就是那些由malloc等分配的内存块，它和堆是十分相似的，不过是用free来释放。

　　全局/静态存储区，全局变量和静态变量被分配到同一块内存中，在以前的C语言中，全局变量又分为初始化的和未初始化的，在C++里面没有这个区分了，它们共同占用同一块内存区。

　　常量存储区，这是一块比较特殊的存储区，里面存放的是常量，不允许修改。


#### 1.1.1.2 堆和栈究竟有什么区别？

主要的区别有以下几点：

　　1、管理方式；

　　2、空间大小；

　　3、能否产生碎片；

　　4、生长方向；

　　5、分配方式；

　　6、分配效率；

　　管理方式：对于栈来讲，是由编译器自动管理，无需手动控制；对于堆来说，释放由程序员控制，容易产生memory leak。

　　空间大小：一般来讲在32位系统下，堆内存可以达到4G的空间，从这个角度来看堆内存几乎是没有什么限制的。但是对于栈来讲，一般都是有一定的空间大小的，例如，在VC6下面，默认的栈空间大小是1M。当然，是可以修改。

　　碎片问题：对于堆来讲，频繁的new/delete势必会造成内存空间的不连续，从而造成大量的碎片。对于栈来讲，则不会存在这个问题。

　　生长方向：对于堆来讲，生长方向是向上的，也就是向着内存地址增加的方向；对于栈来讲，它的生长方向是向下的，是向着内存地址减小的方向增长。

　　分配方式：堆都是动态分配的，没有静态分配的堆。栈有2种分配方式：静态分配和动态分配。静态分配是编译器完成的，比如局部变量的分配。动态分配由alloca函数进行分配，但是栈的动态分配和堆是不同的，它的动态分配是由编译器进行释放，无需我们手工实现。

　　分配效率：栈是机器系统提供的数据结构，计算机会在底层对栈提供支持：分配专门的寄存器存放栈的地址，压栈出栈都有专门的指令执行，这就决定了栈的效率比较高。堆则是C/C++函数库提供的，它的机制是很复杂的，例如为了分配一块内存，库函数会按照一定的算法在堆内存中搜索可用的足够大小的空间，如果没有足够大小的空间（可能是由于内存碎片太多），就有可能调用系统功能去增加程序数据段的内存空间，这样就有机会分到足够大小的内存。显然，堆的效率比栈要低得多。

　　从这里我们可以看到，堆和栈相比，由于大量new/delete的使用，容易造成大量的内存碎片；由于没有专门的系统支持，效率很低；由于可能引发用户态和核心态的切换，内存的申请，代价变得更加昂贵。所以栈在程序中是应用最广泛的，就算是函数的调用也利用栈去完成，函数调用过程中的参数，返回地址，EBP和局部变量都采用栈的方式存放。所以，推荐尽量用栈，而不是用堆。




### 1.1.2 控制C++的内存分配

#### 1.1.2.1 重载全局的new和delete操作符

​      可以很容易地重载new 和 delete 操作符，如下所示:

```c++
void * operator new(size_t size)
{
     void *p = malloc(size);
     return (p);
}
```



```c++
void operator delete(void *p);
{
     free(p);
} 
```

　　这段代码可以代替默认的操作符来满足内存分配的请求。出于解释C++的目的，我们也可以直接调用malloc()和free()。

　　也可以对单个类的new 和 delete 操作符重载。这样能灵活的控制对象的内存分配。
```c++
class TestClass
{
public:
    void * operator new(size_t size);
    void operator delete(void *p);
	// .. other members here ...
};

void *TestClass::operator new(size_t size)
{
	void *p = malloc(size); // Replace this with alternative allocator
	return (p);
}

void TestClass::operator delete(void *p)
{
	free(p); // Replace this with alternative de-allocator
}
```

​      所有 TestClass 对象的内存分配都采用这段代码。更进一步，任何从 TestClass 继承的类也都采用这一方式，除非它自己也重载了new和 delete 操作符。通过重载 new和 delete 操作符的方法，可以自由地采用不同的分配策略，从不同的内存池中分配不同的类对象。



#### 1.1.2.2 为单个类重载 new[ ]和delete[ ]

​      必须小心对象数组的分配。你可能希望调用到被你重载过的new 和 delete 操作符，但并不如此。内存的请求被定向到全局的new[ ]和delete[ ]操作符，而这些内存来自于系统堆。

　　**C++将对象数组的内存分配作为一个单独的操作，而不同于单个对象的内存分配。为了改变这种方式，同样需要重载new[ ]和 delete[ ]操作符。**

```c++
class TestClass
{
public:
	void * operator new[ ](size_t size);
    void operator delete[ ](void *p);
    // .. other members here ..
};

void *TestClass::operator new[ ](size_t size)
{
    void *p = malloc(size);
	return (p);
}

void TestClass::operator delete[ ](void *p)
{
	free(p);
}

int main(void)
{
	TestClass *p = new TestClass[10];
    // ... etc ...
    delete[ ] p;
} 
```





### 1.1.3 指针与数组的对比

数组要么在静态存储区被创建（如全局数组），要么在栈上被创建。数组名对应着（而不是指向）一块内存，其地址与容量在生命期内保持不变，只有数组的内容可以改变。

指针可以随时指向任意类型的内存块，它的特征是“可变”，所以我们常用指针来操作动态内存。指针远比数组灵活，但也更危险。



#### 1.1.3.1 修改内容

下面示例中，字符数组a的容量是6个字符，其内容为hello。a的内容可以改变，如a[0]=‘X’。指针p指向常量字符串“world”（位于静态存储区，内容为world），常量字符串的内容是不可以被修改的。从语法上看，编译器并不觉得语句p[0]=‘X’有什么不妥，但是该语句企图修改常量字符串的内容而导致运行错误。

```c++
char a[] = “hello”;
a[0] = 'X';
cout << a << endl;
char *p = "world"; //注意p指向常量字符串
p[0] = 'X'; //编译器不能发现该错误
cout << p << endl;
```



#### 1.1.3.2 内容复制与比较

不能对数组名进行直接复制与比较。若想把数组a的内容复制给数组b，不能用语句 b = a，否则将产生编译错误。应该用标准库函数strcpy进行复制。同理，比较b和a的内容是否相同，不能用if(b==a)来判断，应该用标准库函数strcmp进行比较。

语句p = a 并不能把a的内容复制指针p，而是把a的地址赋给了p。要想复制a的内容，可以先用库函数malloc为p申请一块容量为strlen(a)+1个字符的内存，再用strcpy进行字符串复制。同理，语句if(p==a)比较的不是内容而是地址，应该用库函数strcmp来比较。
```c++
// 数组…
char a[] = "hello";
char b[10];
strcpy(b, a); // 不能用 b = a;
if(strcmp(b, a) == 0) // 不能用 if (b == a)

// 指针…
int len = strlen(a);
char *p = (char *)malloc(sizeof(char)*(len+1));
strcpy(p,a); // 不要用 p = a;
if(strcmp(p, a) == 0) // 不要用 if (p == a)
```



#### 1.1.3.3 计算内存容量

用运算符sizeof可以计算出数组的容量（字节数）。如下示例中，sizeof(a)的值是12。指针p指向a，但是sizeof(p)的值却是4。这是因为sizeof(p)得到的是一个指针变量的字节数，相当于sizeof(char*)，而不是p所指的内存容量。C++/C语言没有办法知道指针所指的内存容量，除非在申请内存时记住它。

```c++
char a[] = "hello world";
char *p = a;
cout<< sizeof(a) << endl; // 12字节
cout<< sizeof(p) << endl; // 4字节
```

注意当数组作为函数的参数进行传递时，该数组自动退化为同类型的指针。如下示例中，不论数组a的容量是多少，sizeof(a)始终等于sizeof(char *)。

```c++
void Func(char a[100])
{
　cout<< sizeof(a) << endl; // 4字节而不是100字节
}
```



### 1.1.4 指针参数是如何传递内存的？

如果函数的参数是一个指针，不要指望用该指针去申请动态内存。如下示例中，Test函数的语句GetMemory(str, 200)并没有使str获得期望的内存，str依旧是NULL，为什么？

```c++
void GetMemory(char *p, int num)
{
	p = (char *)malloc(sizeof(char) * num);
}

void Test(void)
{
	char *str = NULL;
　  GetMemory(str, 100); // str 仍然为 NULL
　  strcpy(str, "hello"); // 运行错误
}
```

问题出在函数GetMemory中。编译器总是要为函数的每个参数制作临时副本，指针参数p的副本是 \_p，编译器使 \_p = p。如果函数体内的程序修改了\_p的内容，就导致参数p的内容作相应的修改。这就是指针可以用作输出参数的原因。在本例中，\_p申请了新的内存，只是把\_p所指的内存地址改变了，但是p丝毫未变。所以函数GetMemory并不能输出任何东西。事实上，每执行一次GetMemory就会泄露一块内存，因为没有用free释放内存。

如果非得要用指针参数去申请内存，那么应该改用“指向指针的指针”，见示例：
```c++
void GetMemory2(char **p, int num)
{
　*p = (char *)malloc(sizeof(char) * num);
}

void Test2(void)
{
　char *str = NULL;
　GetMemory2(&str, 100); // 注意参数是 &str，而不是str
　strcpy(str, "hello");
　cout<< str << endl;
　free(str);
}
```



由于“指向指针的指针”这个概念不容易理解，可以用函数返回值来传递动态内存。这种方法更加简单，见示例：

```c++
char *GetMemory3(int num)
{
　char *p = (char *)malloc(sizeof(char) * num);
　return p;
}

void Test3(void)
{
　char *str = NULL;
　str = GetMemory3(100);
　strcpy(str, "hello");
　cout<< str << endl;
　free(str);
}
```

用函数返回值来传递动态内存这种方法虽然好用，但是常常有人把return语句用错了。这里强调**不要用return语句返回指向“栈内存”的指针**，因为该内存在函数结束时自动消亡，见示例：

```c++
char *GetString(void)
{
    char p[] = "hello world";
　  return p; // 编译器将提出警告
}

void Test4(void)
{
    char *str = NULL;
　  str = GetString(); // str 的内容是垃圾
　  cout<< str << endl;
}
```



如果把上述示例改写成如下示例，会怎么样？

```c++
char *GetString2(void)
{
    char *p = "hello world";
　  return p;
}

void Test5(void)
{
　  char *str = NULL;
　  str = GetString2();
　  cout<< str << endl;
}
```

函数Test5运行虽然不会出错，但是函数GetString2的设计概念却是错误的。因为GetString2内的“hello world”是常量字符串，位于静态存储区，它在程序生命期内恒定不变。无论什么时候调用GetString2，它返回的始终是同一个“只读”的内存块。



### 1.1.6 杜绝“野指针”

野指针”不是NULL指针，是指向“垃圾”内存的指针。一般不会错用NULL指针，因为用if语句很容易判断。但是“野指针”是很危险的，if语句对它不起作用。“野指针”的成因主要有两种：

（1）指针变量没有被初始化。**任何指针变量刚被创建时不会自动成为NULL指针，它的缺省值是随机的。**所以，指针变量在创建的同时应当被初始化，要么将指针设置为NULL，要么让它指向合法的内存。例如：
```c++
char *p = NULL;
char *str = (char *) malloc(100);
```

（2）指针p被free或者delete之后，没有置为NULL，让人误以为p是个合法的指针。

（3）指针操作超越了变量的作用域范围。这种情况让人防不胜防，示例程序如下：

```c++
class A
{
public:
    void Func(void){ cout << “Func of class A” << endl; }
};

void Test(void)
{
    A *p;
　  {
　　    A a;
　　    p = &a; // 注意a的生命期
　  }
　  p->Func(); // p是“野指针”
}
```

函数Test在执行语句p->Func()时，对象a已经消失，而p是指向a的，所以p就成了“野指针”。



### 1.1.7 有了malloc/free为什么还要new/delete？

malloc与free是C++/C语言的标准库函数，new/delete是C++的运算符。它们都可用于申请动态内存和释放内存。

对于非内部数据类型的对象而言，光用maloc/free无法满足动态对象的要求。对象在创建的同时要自动执行构造函数，对象在消亡之前要自动执行析构函数。由于malloc/free是库函数而不是运算符，不在编译器控制权限之内，不能够把执行构造函数和析构函数的任务强加于malloc/free。

因此C++语言需要一个能完成动态内存分配和初始化工作的运算符new，以及一个能完成清理与释放内存工作的运算符delete。注意new/delete不是库函数。我们先看一看malloc/free和new/delete如何实现对象的动态内存管理。


## 1.2 C++中的健壮指针和资源管理

所有权分为两种——自动的和显式的（automatic and explicit），如果一个对象的释放是由语言本身的机制来保证的，这个对象的就是被自动地所有。例如，一个嵌入在其他对象中的对象，它的清除需要其他对象来在清除的时候进行保证。外面的对象可看作嵌入类的所有者。类似地，每个在栈上创建的对象的释放是在控制流离开对象被定义的作用域时保证的。这种情况下，作用域被看作是对象的所有者。注意所有的自动所有权都是和语言的其他机制相容的，包括异常。无论是如何退出作用域的——正常流程控制退出、一个break语句、一个return、一个goto、或者是一个throw——自动资源都可以被清除。

到目前为止，一切都很好！问题是在引入指针、句柄和抽象的时候产生的。如果通过一个指针访问一个对象的话，比如对象在堆中分配，C++不自动地关注它的释放。程序员必须明确地用适当的程序方法来释放这些资源。比如说，如果一个对象是通过调用new来创建的，它需要用delete来回收。一个文件是用CreateFile(Win32 API)打开的，它需要用CloseHandle来关闭。用EnterCritialSection进入的临界区（Critical Section）需要LeaveCriticalSection退出，等等。一个"裸"指针，文件句柄，或者临界区状态没有所有者来确保它们的最终释放。基本的资源管理的前提就是确保每个资源都有他们的所有者。



### 1.2.1 第一条规则（RAII）

**一个指针，一个句柄，一个临界区状态只有在我们将它们封装入对象的时候才会拥有所有者。这就是我们的第一规则：在构造函数中分配资源，在析构函数中释放资源。**

当你按照规则将所有资源封装的时候，可以保证程序中没有任何的资源泄露。这点在当封装对象（Encapsulating Object）在栈中建立或者嵌入在其他的对象中的时候非常明显。但是对那些动态申请的对象呢？不要急！任何动态申请的东西都被看作一种资源，并且要按照上面提到的方法进行封装。这一对象封装对象的链不得不在某个地方终止。它最终终止在最高级的所有者，自动的或者是静态的。这些分别是对离开作用域或者程序结束时释放资源的保证。

下面是资源封装的一个经典例子。在一个多线程的应用程序中，线程之间共享对象的问题是通过用这样一个对象关联临界区来解决的。每一个需要访问共享资源的客户需要获得临界区。例如，这可能是Win32下临界区的实现方法。

```c++
class CritSect
{
friend class Lock;
public:
	CritSect () { InitializeCriticalSection (&_critSection); }
　　~CritSect () { DeleteCriticalSection (&_critSection); }

private:
	void Acquire ()
　　{
		EnterCriticalSection (&_critSection);
	}
      
　　void Release ()
　　{
		LeaveCriticalSection (&_critSection);
　　}
private:
	CRITICAL_SECTION _critSection;
};
```

这里聪明的部分是我们确保每一个进入临界区的客户最后都可以离开。"进入"临界区的状态是一种资源，并应当被封装。封装器通常被称作一个锁（lock）。

```c++
class Lock
{
public:
    Lock(CritSect &critSect) : _critSect(critSect)
    {
        _critSect.Acquire();
    }

    ~Lock()
    {
        _critSect.Release();
    }

private
    CritSect &_critSect;
};
```

锁一般的用法如下：

```c++
void Shared::Act() throw(char *)
{
    Lock lock(_critSect);
    // perform action —— may throw
    // automatic destructor of lock
}
```

注意无论发生什么，临界区都会借助于语言的机制保证释放。

还有一件需要记住的事情——每一种资源都需要被分别封装。这是因为资源分配是一个非常容易出错的操作。我们会假设一个失败的资源分配会导致一个异常——事实上，这经常会发生。所以**如果试图在一个构造函数中申请两种形式的资源，那很可能会陷入麻烦。只要想想在一种资源分配成功但另一种失败抛出异常时会发生什么。由于构造函数还没有全部完成，析构函数不可能被调用，第一种资源就会发生泄露。**

**这种情况可以非常简单的避免。无论何时，当有一个需要两种以上资源的类时，写两个小的封装器将它们嵌入到类中。每一个嵌入的构造都可以保证删除，即使包装类没有构造完成。**



### 1.2.2 Smart Pointers

我们至今还没有讨论最常见类型的资源——用操作符new分配，此后用指针访问的一个对象。我们需要为每个对象分别定义一个封装类吗？（事实上，C++标准模板库已经有了一个模板类，叫做auto_ptr，其作用就是提供这种封装。稍后会回到auto_ptr。）让我们从一个极其简单、呆板但安全的Smart Pointer模板类开始。
```c++
template <class T>
class SmartPointer
{
public:
    ~SmartPointer() { delete _p; }
    T *operator->() { return _p; }
    T const *operator->() const { return _p; }

protected:
    SmartPointer() : _p(0) {}
    explicit SmartPointer(T *p) : _p(p) {}
    T *_p;
};
```

为什么要把SmartPointer的构造函数设计为protected呢？如果要遵守第一条规则，那么就必须这样做。资源——在这里是class T的一个对象——必须在封装器的构造函数中分配。但是不能只简单地调用new T，因为并不知道T的构造函数的参数。因为，原则上每一个T都有一个不同的构造函数；那就需要为它定义个另外一个封装器。可以通过继承SmartPointer定义一个新的封装器，并且提供一个特定的构造函数。
```c++
class SmartItem : public SmartPointer<Item>
{
public:
    explicit SmartItem(int i)
        : SmartPointer<Item>(new Item(i))
    {
    }
};
```

为每一个类提供一个Smart Pointer真的值得吗？说实话——不！但是它很有教学的价值，一旦你学会如何遵循第一规则，就可以放松规则并使用一些高级的技术。这一技术是让SmartPointer的构造函数成为public，但只是用它来做资源转换（Resource Transfer）我的意思是用new操作符的结果直接作为SmartPointer的构造函数的参数，像这样：
```c++
SmartPointer<Item> item (new Item (i));
```



### 1.2.3 Resource Transfer

到目前为止，我们所讨论的一直是生命周期在一个单独的作用域内的资源。现在我们要解决一个困难的问题——如何在不同的作用域间安全地传递资源。这一问题在处理容器的时候会变得十分明显。我们可以动态地创建一串对象，将它们存放至一个容器中，然后将它们取出。为了能够让这安全地工作——没有内存泄露——对象需要改变其所有者。

这个问题的一个非常显而易见的解决方法是使用Smart Pointer，增加Release方法到Smart Pointer中：
```c++
template <class T>
T *SmartPointer<T>::Release()
{
    T *pTmp = _p;
    _p = 0;
    return pTmp;
}
```

注意在Release调用以后，Smart Pointer就不再是对象的所有者了——它内部的指针指向空。现在，调用了Release就必须迅速隐藏返回的指针到新的所有者对象中。在我们的例子中，容器调用了Release，比如这个Stack的例子：

```c++
void Stack::Push (SmartPointer <Item> & item) throw (char *)
{
    if (_top == maxStack)
    	throw "Stack overflow";
    _arr [_top++] = item.Release();
};
```



### 1.2.4 Strong Pointers

资源传递除了以强指针赋值的形式进行，也可以有Weak Pointer存在。

给SmartPointer增加一个拷贝构造函数，并重载赋值操作符。

```c++
template <class T>
SmartPointer<T>::SmartPointer(SmartPointer<T> &ptr)
{
    _p = ptr.Release();
}

template <class T>
void SmartPointer<T>::operator=(SmartPointer<T> &ptr)
{
    if (_p != ptr._p)
    {
        delete _p;
        _p = ptr.Release();
    }
}
```

此时就可以以值方式传递这种封装指针。看下Stack新的实现：

```c++
class Stack
{
    enum
    {
        maxStack = 3
    };

public:
    Stack() : _top(0) {}

    void Push(SmartPointer<Item> &item) throw(char *)
    {
        if (_top >= maxStack)
            throw "Stack overflow";
        _arr[_top++] = item;
    }

    SmartPointer<Item> Pop()
    {
        if (_top == 0)
            return SmartPointer<Item>();
        return _arr[--_top];
    }

private
    int _top;
    SmartPointer<Item> _arr[maxStack];
};
```

Pop方法以值方式返回一个Strong Pointer，编译器在return时自动进行了一次资源转换，调用了operator = 来从数组中提取一个Item，调用拷贝构造函数将它传递给调用者。



### 1.2.5 Parser

使用Strong Pointer实现一个表达式解析器，如下：

```c++
SmartPointer<Node> Parser::Expression()
{
    // Parse a term
    SmartPointer<Node> pNode = Term();
    EToken token = _scanner.Token();

    if (token == tPlus || token == tMinus)
    {
        // Expr := Term { ('+' | '-') Term }
        SmartPointer<MultiNode> pMultiNode = new SumNode(pNode);
        do
        {
            _scanner.Accept();
            SmartPointer<Node> pRight = Term();
            pMultiNode->AddChild(pRight, (token == tPlus));
            token = _scanner.Token();
        } while (token == tPlus || token == tMinus);

        pNode = up_cast<Node, MultiNode>(pMultiNode);
    }

    // otherwise Expr := Term
    return pNode; // by value!
}
```

最开始，Term方法被调用，传值返回一个指向Node的Strong Pointer。如果下一个符号不是加号或者减号，就简单地把这个SmartPointer以值返回，这样就释放了Node的所有权。另外一方面，如果下一个符号是加号或者减号，则创建一个新的SumMode并且立刻（直接传递）将它储存到MultiNode的一个Strong Pointer中。这里，SumNode是从MultiMode中继承而来的，而MulitNode是从Node继承而来的。原来的Node的所有权转给了SumNode。

只要是被加号和减号分开，就不断的创建terms，并将这些term转移到MultiNode中，同时MultiNode得到了所有权。最后，我们将指向MultiNode的Strong Pointer向上映射为指向Node的Strong Pointer，并且将其返回调用者。

我们需要对Strong Pointers进行显式的向上映射，即使指针是被隐式的封装。例如，一个MultiNode是一个Node，但是相同的is-a关系在SmartPointer\<MultiNode\>和SmartPointer\<Node>之间并不存在，因为它们是分离的类（模板实例）并不存在继承关系。up-cast模板是像下面这样定义的：

```c++
template <class To, class From>
inline SmartPointer<To> up_cast(SmartPointer<From> &from)
{
    return SmartPointer<To>(from.Release());
}
```

如果编译器支持成员模板（member template），那可以为SmartPointer\<T\>定义一个新的构造函数用于接受一个class U。

```c++
template <class T>
template <class U> SmartPointer<T>::SmartPointer(SmartPointer<U> &uptr)
    : _p(uptr.Release()) {}
```

这里的trick是模板在U不是T的子类的时候，就会编译报错（换句话说，只在U is-a T的时候才会编译）。这是因为uptr的缘故，Release()方法返回一个指向U的指针，并被赋值为_p，一个指向T的指针。所以如果U不是一个T的话，赋值会导致一个编译错误。

上述所说的Strong Pointer其实就是标准库中auto_ptr的。



### 1.2.6 Transfer Semantics

以传值方式传递的对象有value semantics 或者称为 copy semantics。Strong Pointers是以值方式传递的，那我们能说它们有copy semantics吗？不是这样的！它们所指向的对象肯定没有被拷贝过。事实上，传递过后，源auto_ptr不应再访问原有的对象，并且目标auto_ptr成为了对象的唯一拥有者。可以将这种新的行为称作Transfer Semantics。

拷贝构造函数（copy construcor）和赋值操作符定义了auto_ptr的Transfer Semantics，它们用了非const的auto_ptr引用作为它们的参数。
```c++
auto_ptr (auto_ptr<T> & ptr);
auto_ptr & operator = (auto_ptr<T> & ptr);
```

这是因为它们确实改变了他们的源——剥夺了其对资源的所有权。

通过定义相应的拷贝构造函数和重载赋值操作符，可以将Transfer Semantics加入到许多对象中。例如，许多Windows中的资源，如动态建立的菜单或者位图，可以用有Transfer Semantics的类封装。


### 1.2.7 Strong Vectors

标准库只在auto_ptr中支持资源管理。甚至连最简单的容器也不支持ownership semantics。你可能想将auto_ptr和标准容器组合到一起可能会管用，但现实并不是这样的。例如，你可能会这样做，但是会发现并不能用标准的方法来进行索引。

```c++
vector<auto_ptr<Item>> autoVector;
```

下面这种用法编译报错。因为类型不匹配。

```c++
Item * item = autoVector[0];
```

另一方面，当返回Strong Pointer时会导致一个从autoVect到auto_ptr的所有权转换（回想下Strong Pointer的赋值操作）。

```
auto_ptr<Item> item = autoVector[0];
```



别无选择，只能实现我们自己的Strong Vector。最小的接口应该如下：

```c++
template <class T>
class auto_vector
{
public:
    explicit auto_vector(size_t capacity = 0);
    T const *operator[](size_t i) const;
    T *operator[](size_t i);
    void assign(size_t i, auto_ptr<T> &p);
    void assign_direct(size_t i, T *p);
    void push_back(auto_ptr<T> &p);
    auto_ptr<T> pop_back();
};
```

你也许会发现这是一个非常防御性的设计。我决定不提供一个对vector的左值索引的访问，取而代之，如果想设置(set)一个值，必须调用assign或者assign_direct方法。我的观点是，资源管理不应该被忽视，同时，也不应该在所有的地方滥用。

Strong vector最好用一个动态的Strong Pointers的数组来实现：

```c++
template <class T>
class auto_vector
{
private
    void grow(size_t reqCapacity);
    auto_ptr<T> *_arr;
    size_t _capacity;
    size_t _end;
};
```

auto_vector的其他实现都是十分直接的，因为所有资源管理的复杂度都在auto_ptr中。例如，assign方法简单的利用了重载的赋值操作符来删除原有的对象并转移资源到新的对象：

```c++
void assign (size_t i, auto_ptr<T> & p)
{
	_arr [i] = p;
}
```

对auto_vector的索引访问是借助auto_ptr的get方法来实现的，get简单的返回一个内部指针。

```c++
T * operator [] (size_t i)
{
	return _arr[i].get();
}
```



没有容器可以没有iterator。我们需要一个iterator让auto_vector看起来更像一个普通的指针向量。

```c++
template <class T>
class auto_iterator : public iterator<random_access_iterator_tag, T *>
{
public:
    auto_iterator() : _pp(0) {}
    auto_iterator(auto_ptr<T> *pp) : _pp(pp) {}
    bool operator!=(auto_iterator<T> const &it) const
    {
        return it._pp != _pp;
    }
    auto_iterator const &operator++(int) { return _pp++; }
    auto_iterator operator++() { return ++_pp; }
    T *operator*() { return _pp->get(); }
private
    auto_ptr<T> *_pp;
};

```

我们给auto_vect提供了标准的begin和end方法来找回iterator：

```c++
class auto_vector
{
public:
    typedef auto_iterator<T> iterator;
    iterator begin() { return _arr; }
    iterator end() { return _arr + _end; }
};
```



### 1.2.8 Code Inspection

如果严格遵照资源管理的条款，就不会在资源泄露或者两次删除的地方遇到麻烦，也降低了访问野指针的几率。同样的，遵循原有的规则，用delete删除用new申请的德指针，不要两次删除一个指针。你也不会遇到麻烦。但是，哪个是更好的注意呢？

这两种方法有一个很大的不同点，就是和寻找传统方法的bug相比，找到违反资源管理的规定要容易的多。后者仅需要一个代码检测或者一个运行测试，而前者则在代码中隐藏得很深，并需要很深的检查。

设想你要做一段传统的代码的内存泄露检查。第一件事，就是grep所有在代码中出现的new，找出被分配空间的指针都作了什么。需要确定导致删除这个指针的所有的执行路径。需要检查break语句，过程返回，异常等。原有的指针可能赋给另一个指针，则对这个指针也要做相同的事。

相比之下，对于一段用资源管理技术实现的代码。也需要grep检查所有的new，但是这次只需要检查邻近的调用：

- 这是一个直接的Strong Pointer转换，还是在一个构造函数的函数体中？
- 调用的返回是否立即保存到对象中，构造函数中是否有可以产生异常的代码？
- 如果这样的话析构函数中是否有delete?

下一步，需要用grep查找所有的release方法，并实施相同的检查。

在资源管理中的错误模式也比较容易调试。最常见的bug是试图访问一个释放过的strong pointer，这将导致一个错误，并且很容易跟踪。




### 1.2.9 共享的所有权

一个共享资源必须为它的所有者保持一个引用计数。另一方面，所有者在释放资源的时候必须通知共享对象。最后一个释放资源的所有者需要在最后负责释放资源的工作。

最简单的共享的实现是共享对象继承引用计数类RefCounted：

```c++
class RefCounted
{
public:
    RefCounted() : _count(1) {}
    int GetRefCount() const { return _count; }
    void IncRefCount() { _count++; }
    int DecRefCount() { return --_count; }
private
    int _count;
};
```



按照资源管理，一个引用计数是一种资源。当你意识到这一事实的时候，剩下的就变得简单了。简单的遵循规则——在构造函数中获得引用计数，在析构函数中释放。甚至有一个RefCounted的Smart Pointer等价物：

```c++
template <class T>
class RefPtr
{
public:
    RefPtr(T *p) : _p(p) {}

    RefPtr(RefPtr<T> &p)
    {
        _p = p._p;
        _p->IncRefCount();
    }

    ~RefPtr()
    {
        if (_p->DecRefCount() == 0)
            delete _p;
    }
    
private
    T *_p;
};
```



### 1.2.10 所有权网络

链表是资源管理分析中的一个很有意思的例子。如果选择表成为链(link)的所有者，则会陷入递归的所有权。每一个link都是它的继承者的所有者，并且，相应的，余下的链表的所有者。下面是用smart pointer实现的一个表单元：

```c++
class Link
{
    // ...
private
    auto_ptr<Link> _next;
};

class DoubleLink
{
    // ...
private
    DoubleLink *_prev;
    auto_ptr<DoubleLink> _next;
};
```

注意不要创建环形链表。

这给我们带来了另外一个有趣的问题——资源管理可以处理环形的所有权吗？它可以，使用mark-and-sweep算法【TODO】。这里是实现这种方法的一个例子：

```c++
template <class T>
class CyclPtr
{
public:
    CyclPtr(T *p) : _p(p), _isBeingDeleted(false) {}

    ~CyclPtr()
    {
        _isBeingDeleted = true;
        if (!_p->IsBeingDeleted())
            delete _p;
    }

    void Set(T *p)
    {
        _p = p;
    }

    bool IsBeingDeleted() const { return _isBeingDeleted; }

private
    T *_p;
    bool _isBeingDeleted;
};
```

注意我们需要用class T来实现方法IsBeingDeleted，就像从CyclPtr继承。对特殊的所有权网络普通化是十分直接的。





# 2 内存泄漏



## 2.1 C++中动态内存分配引发问题的解决方案

假设我们要开发一个String类，它可以方便地处理字符串数据。可以在类中声明一个数组，考虑到有时候字符串极长，可以把数组大小设为200，但一般的情况下又不需要这么多的空间，这样就浪费了内存。我们可以使用new操作符，这样是十分灵活的，但在类中就会出现许多意想不到的问题，本文就是针对这一现象而写的。现在，我们先来开发一个String类，但它是一个不完善的类。

```c++
/* String.h */
#ifndef STRING_H_
#define STRING_H_

class String
{
private:
    char *str; // 存储数据
    int len; // 字符串长度

public:
    String(const char *s); // 构造函数
    String(); // 默认构造函数
    ~String(); // 析构函数
    friend ostream &operator<<(ostream &os, const String &st);
};
#endif

/*String.cpp*/
#include <iostream>
#include <cstring>
#include "String.h"

using namespace std;

String::String(const char *s)
{
    len = strlen(s);
    str = new char[len + 1];
    strcpy(str, s);
} // 拷贝数据

String::String()

{
    len = 0;
    str = new char[len + 1];
    str[0] = '\0';
}

String::~String()
{
    cout << "这个字符串将被删除：" << str << endl; // 为了方便观察结果，特留此行代码。
    delete[] str;
}

ostream &operator<<(ostream &os, const String &st)

{
    os << st.str;
    return os;
}
```

下面是一段测试代码及其输出：

```c++
#include <iostream>
#include <stdlib.h>
#include "String.h"

using namespace std;

void show_right(const String &);
void show_String(const String); // 注意，参数非引用，而是按值传递。

int main()
{
    String test1("第一个范例。");
    String test2("第二个范例。");
    String test3("第三个范例。");
    String test4("第四个范例。");

    cout << "下面分别输入三个范例：" << endl;
    cout << test1 << endl;
    cout << test2 << endl;
    cout << test3 << endl;

    String *String1 = new String(test1);
    cout << *String1 << endl;
    delete String1;

    cout << test1 << endl; // 在Dev-cpp上没有任何反应。

    cout << "使用正确的函数：" << endl;
    show_right(test2);
    cout << test2 << endl;

    cout << "使用错误的函数：" << endl;
    show_String(test2);
    cout << test2 << endl; // 这一段代码出现严重的错误！

    String String2(test3);
    cout << "String2: " << String2 << endl;

    String String3;
    String3 = test4;
    cout << "String3: " << String3 << endl;
    cout << "下面，程序结束，析构函数将被调用。" << endl;

    return 0;
}

void show_right(const String &a)
{
    cout << a << endl;
}

void show_String(const String a)
{
    cout << a << endl;
}
```

测试输出：

```c++
下面分别输入三个范例：
第一个范例。
第二个范例。
第三个范例。
第一个范例。
这个字符串将被删除：第一个范例。
�c]]
使用正确的函数：
第二个范例。
第二个范例。
使用错误的函数：
第二个范例。
这个字符串将被删除：第二个范例。
��e��U
String2: 第三个范例。
String3: 第四个范例。
下面，程序结束，析构函数将被调用。
这个字符串将被删除：第四个范例。
这个字符串将被删除：第三个范例。
这个字符串将被删除：��e��U
free(): double free detected in tcache 2
Aborted
```

可以看到有很多错误。下面，一一分析原因。

首先，大家要知道，C＋＋类有以下这些极为重要的函数：

一：复制构造函数。

二：赋值函数。



先来讲复制构造函数。什么是复制构造函数呢？比如，有这样的代码`String test1(test2);`这是在进行初始化。我们知道，初始化对象需要调用构造函数。那么就应该提供这样的构造函数：`String(const String &);`可是，上面的String实现中并没有定义这个构造函数。结果是，C＋＋提供了默认的复制构造函数，那么问题也就出在这儿。

(1) 什么时候会调用复制构造函数呢？（以String类为例。）

在提供这样的代码`String test1(test2)`时，它会被调用。当函数的参数列表为按值传递，也就是没有用引用和指针作为类型时，如`void show_String(const String)`，它也会被调用。还有一些情况，此处不一一列举了。

(2) 它是什么样的函数。

其作用就是把两个对象进行复制。以上面的String类为例，C＋＋提供的默认复制构造函数是这样的：

```c++
String(const String& a)
{
	str = a.str;
	len = a.len;
}
```

在平时，这样并不会有任何的问题出现，但当用了new操作符，涉及到了动态内存分配，就不得不谈谈浅拷贝和深拷贝了。以上的函数就是实行的浅拷贝，它只是拷贝了指针，而并没有拷贝指针指向的数据，可谓一点儿用也没有。下面我们来具体谈谈：

假如，A对象中存储了字符串"C++"，它的地址为2000。现在，我们把A对象赋给B对象：`String B = A`。现在，A和B对象的str指针均指向2000地址。看似可以使用，但如果B对象的析构函数被调用，则地址2000处的字符串"C++"已经被从内存中抹去，而A对象仍然指向地址2000。这时，如果我们写下这样的代码`cout<<A<<endl;`或是等待程序结束，A对象的析构函数被调用时，A对象的数据能否显示出来呢？只会是乱码。而且程序还会连续对地址2000处使用两次delete操作符，这样的后果是十分严重的！

本例中，有这样的代码：

```c++
String* String1 = new String(test1);
cout<<*String1<<endl;
delete String1;
```

假设test1中str指向的地址为2000，而String中str指针同样指向地址2000，我们删除了2000处的数据，而test1对象呢？已经被破坏了。从运行结果上可以看到，当运行`cout<<test1`时，输出了乱码。

再看看这段代码：

```c++
cout << "使用错误的函数：" << endl;
show_String(test2);
cout << test2 << endl;
```

show_String函数的参数列表`void show_String(const String a)`是按值传递的，所以相当于执行了`String a=test2;`。函数执行完毕，由于生存周期的缘故，对象a被析构函数删除，这样test2也被破坏了。解决的办法很简单，当然是手动定义一个拷贝构造函数。

```c++
String::String(const String& a)
{
    len = a.len;
    str = new char(len+1);
    strcpy(str, a.str);
}
```

此时，在执行代码`String B = A`时，会先开辟出一块内存，假设地址为3000。然后用strcpy函数将地址2000的内容拷贝到地址3000中，再将对象B的str指针指向地址3000。这样，就互不干扰了。

当把这个拷贝构造函数加入到程序中，问题就解决了大半，但还没有完全解决，问题出现在赋值函数上。我们的程序中有段这样的代码：

```c++
String String3;
String3 = test4;
```

当没有重载operator=时，C++也会提供一个默认实现，同样执行的是浅拷贝，那么也就同样会出现上面所述的问题。

平时，我们可以写这样的代码：x=y=z（x,y,z均为整型变量）。而对于类对象，同样可以这样写，因为这很方便。而对象A=B=C，其实就是A.operator=(B.operator=(c))。而这个operator=函数的参数列表应该是 `const String& a`，所以，**要实现这样的功能，返回值也要是String&，这样才能实现A＝B＝C**。我们先来写写看：

```c++
String &String::operator=(const String &a)
{
    delete[] str; // 先删除自身的数据
    len = a.len;
    str = new char[len + 1];
    strcpy(str, a.str); // 进行拷贝
    return *this;       // 返回自身的引用
}
```

是不是这样就行了呢？假如写出了这种代码：A=A，那么岂不是把A对象的数据给删除了吗？这样会引发一系列的错误。所以，我们还要检查是否为自身赋值。只比较两对象的数据是不行的，因为两个对象的数据很有可能相同，应该比较地址。以下是完整的赋值函数：

```c++
String &String::operator=(const String &a)
{
    if (this == &a)
        return *this;
    delete[] str; // 先删除自身的数据
    len = a.len;
    str = new char[len + 1];
    strcpy(str, a.str); // 此三行为进行拷贝
    return *this;       // 返回自身的引用
}
```

当把以上拷贝构造函数和赋值函数添加String的实现中，问题就完全解决了。





## 2.2 如何对付内存泄漏？



当代码中充斥着new操作、delete操作和指针运算，那么必将会在某个地方晕头转向，出现内存泄漏、指针引用错误等诸如此类的问题。这和你如何小心地对待内存分配工作其实完全没有关系：代码的复杂性最终总是会超过所能够付出的时间和努力。于是随后产生了一些成功的技巧，它们依赖于将内存分配（allocations）与重新分配（deallocation）工作隐藏在易于管理的类型之后。标准容器（standard containers）就是一个优秀的例子。它们不是通过你而是自己为元素管理内存，从而避免了产生糟糕的结果。 

如果实在不能将内存分配/重新分配的操作隐藏到对象中时，那么可以使用资源句柄（resource handle），以将内存泄漏的可能性降至最低。在下面个例子中，我们需要通过一个函数，在空闲内存中建立一个对象并返回它。这时候可能忘记释放这个对象。毕竟，我们不能说，仅仅关注当这个指针要被释放的时候，谁将负责去做。使用资源句柄（这里用了标准库中的auto_ptr）使需要为之负责的地方变得明确了。

```c++
#include <memory>
#include <iostream>

using namespace std;

struct S
{
    S() { cout << "make an S" << endl; }

    ~S()
    {
        cout << "destroy an S" << endl;
    }

    S(const S &)
    {
        cout << "copy initialize an S" << endl;
    }

    S &operator=(const S &)
    {
        cout << "copy assign an S" << endl;
    }
};

S *f()
{
    return new S; // 谁该负责释放这个S？
};

auto_ptr<S> g()
{
    return auto_ptr<S>(new S); // 显式传递负责释放这个S
}

int main()
{
    cout << "start main" << endl;
    S *p = f();

    cout << "after f() before g()" << endl;
    auto_ptr<S> q = g();

    cout << "exit main" << endl;
    // *p产生了内存泄漏
    // *q被自动释放
}
```



## 2.3 浅谈C/C++内存泄漏及其检测工具

对于一个c/c++程序员来说，内存泄漏是一个常见的也是令人头疼的问题。已经有许多技术被研究出来以应对这个问题，比如Smart Pointer，Garbage Collection等。Smart Pointer技术比较成熟，STL中已经包含支持Smart Pointer的class，但是它的使用似乎并不广泛，而且它也不能解决所有的问题；Garbage Collection技术在Java中已经比较成熟，但是在c/c++领域的发展并不顺畅，虽然很早就有人思考在C++中也加入GC的支持。现实世界就是这样的，作为一个c/c++程序员，内存泄漏是你心中永远的痛。不过好在现在有许多工具能够帮助验证内存泄漏的存在，找出发生问题的代码。



### 2.3.1 内存泄漏的定义

一般我们常说的内存泄漏是指堆内存的泄漏。从广义上说，内存泄漏不仅仅包含堆内存的泄漏，还包含系统资源的泄漏(resource leak)，比如核心态HANDLE、GDI Object、SOCKET、Interface等。从根本上说这些由操作系统分配的对象也消耗内存，如果这些对象发生泄漏最终也会导致内存的泄漏。而且，某些对象消耗的是核心态内存，这些对象严重泄漏时会导致整个操作系统不稳定。所以相比之下，系统资源的泄漏比堆内存的泄漏更为严重。



### 2.3.2 检测内存泄漏

检测内存泄漏的关键是要能截获住对分配内存和释放内存的函数的调用。截获住这两个函数，就能跟踪每一块内存的生命周期，比如，每当成功的分配一块内存后，就把它的指针加入一个全局的list中；每当释放一块内存，再把它的指针从list中删除。这样，当程序结束的时候，list中剩余的指针就是指向那些没有被释放的内存。这只是简单的描述了检测内存泄漏的基本原理，详细的算法可以参见Steve Maguire的\<\<Writing Solid Code\>\>。


在Windows平台下，检测内存泄漏的工具常用的一般有三种：MS C-Runtime Library内建的检测功能；外挂式的检测工具，诸如，Purify，BoundsChecker等；利用Windows NT自带的Performance Monitor。这三种工具各有优缺点，MS C-Runtime Library虽然功能上较之外挂式的工具要弱，但是它是免费的；Performance Monitor虽然无法标示出发生问题的代码，但是它能检测出隐式的内存泄漏的存在，这是其他两类工具无能为力的地方。





# 3 探讨C++内存回收



## 3.1 C++内存对象大会战

我们知道，C++将内存划分为三个逻辑区域：堆、栈和静态存储区。既然如此，可以称位于其中的对象分别为堆对象，栈对象以及静态对象。那么这些不同的内存对象有什么区别？堆对象和栈对象各有什么优劣了？如何禁止创建堆对象或栈对象了？这些便是今天的主题。



### 3.1.1 基本概念


先来看看栈。栈一般用于存放局部变量或对象，如我们在函数定义中用类似下面语句声明的对象：
```c++
Type stack_object ; 
```
stack_object便是一个栈对象，它的生命期是从定义点开始，当所在函数返回时，生命结束。

另外，几乎所有的临时对象都是栈对象。比如，下面的函数定义：
```c++
Type fun(Type object);
```
这个函数至少产生两个临时对象，首先，参数是按值传递的，所以会调用拷贝构造函数生成一个临时对象object_copy1，在函数内部使用的不是object，而是object_copy1，自然，object_copy1是一个栈对象，它在函数返回时被释放。此外这个函数是值返回的，在函数返回时，如果不考虑返回值优化（NRV），那么也会产生一个临时对象object_copy2，这个临时对象会在函数返回后一段时间内被释放。比如现有如下代码：

```c++
Type tt, result;  //生成两个栈对象
tt = fun(tt);     //函数返回时，生成的是一个临时对象object_copy2
```

上面的第二个语句的执行情况是这样的，首先函数fun返回时生成一个临时对象object_copy2，然后再调用赋值运算符执行

```c++
tt = object_copy2;  //调用赋值运算符
```

看到了吗？编译器在我们毫无察觉的情况下，生成了这么多临时对象，而生成这些临时对象的时间和空间的开销可能是很大的，所以，你也许明白了，为什么对于“大”对象最好用const引用传递代替值传递了。

接下来，看看堆。堆又叫自由存储区，它是在程序执行的过程中动态分配的，所以它最大的特性就是动态性。在C++中，所有堆对象的创建和销毁都要由程序员负责，所以，如果处理不好，就会发生内存问题。如果分配了堆对象，却忘记了释放，就会产生内存泄漏；而如果已释放了对象，却没有将相应的指针置为NULL，该指针就是所谓的“悬挂指针”，再度使用此指针时，就会出现非法访问，严重时会导致程序崩溃。

那么，C++中是怎样分配堆对象的？唯一的方法就是用new（当然，用malloc指令也可获得C式堆内存），只要使用new，就会在堆中分配一块内存，并且返回指向该堆对象的指针。

再来看看静态存储区。所有的静态对象、全局对象都于静态存储区分配。关于全局对象，是在main()函数执行前就分配好了的。其实，在main()函数中的显示代码执行之前，会调用一个由编译器生成的\_main()函数，而\_main()函数会进行所有全局对象的的构造及初始化工作。而在main()函数结束之前，会调用由编译器生成的exit函数，来释放所有的全局对象。

**所以，知道了这个之后，便可以由此引出一些技巧，比如，我们想要在main()函数执行之前做某些准备工作，那么就可以将这些准备工作写到一个自定义的全局对象的构造函数中，这样，在main()函数的显式代码执行之前，这个全局对象的构造函数会被调用，执行预期的动作。**刚才讲的是静态存储区中的全局对象，那么，局部静态对象呢？局部静态对象通常也是在函数中定义的，就像栈对象一样，只不过，其前面多了个static关键字。局部静态对象的生命期是从其所在函数第一次被调用，更确切地说，是当第一次执行到该静态对象的声明代码时，产生该静态局部对象，直到整个程序结束时，才销毁该对象。

还有一种静态对象，那就是作为class的静态成员。考虑这种情况时，就牵涉了一些较复杂的问题。

第一个问题是class的静态成员对象的生命期。class的静态成员对象随着第一个class object的产生而产生，在整个程序结束时消亡。当我们定义了一个class，该类中有一个静态对象作为成员，但是在程序执行过程中，如果没有创建任何一个该class object，那么也就不会产生该class所包含的那个静态对象。还有，如果创建了多个class object，那么所有这些object都共享那个静态对象成员。


第二个问题是，当出现下列情况时：


```c++
class Base
{
public:
    static Type s_object ;
}

class Derived1 : public Base 
{
    ... // other data
}

class Derived2 : public Base
{
    ... // other data
}

Base example;
Derivde1 example1;
Derivde2 example2;

example.s_object = ... ;
example1.s_object = ... ;
example2.s_object = ... ; 
```

它们所访问的s_object是同一个对象吗？答案是肯定的，它们的确是指向同一个对象。

当一个类比如Derived1，继承另一个类比如Base时，那么可以看作一个Derived1对象中含有一个Base型的对象，这就是一个subobject。那么当我们将一个Derived1型的对象传给一个接受非引用Base型参数的函数时会发生切割，即仅取出Derived1型的对象中的subobject，而忽略了所有Derived1自定义的其它数据成员，然后将这个subobject传递给函数（实际上，函数中使用的是这个subobject的拷贝）。

所有继承Base类的派生类的对象都含有一个Base型的subobject（这是能用Base型指针指向一个Derived1对象的关键所在，自然也是多态的关键了），而所有的subobject和所有Base型的对象都共用同一个s_object对象，自然，从Base类派生的整个继承体系中的类的实例都会共用同一个s_object对象了。


### 3.1.2 使用栈对象的意外收获

前面提到，栈对象是在适当的时候创建，然后在适当的时候自动释放的，也就是栈对象有自动管理功能。那么栈对象会在什么会自动释放了？第一，在其生命期结束的时候；第二，在其所在的函数发生异常的时候。

栈对象，自动释放时，会调用它自己的析构函数。如果我们在栈对象中封装资源，而且在栈对象的析构函数中执行释放资源的动作，那么就会使资源泄漏的概率大大降低，因为栈对象可以自动的释放资源，即使在所在函数发生异常的时候。实际的过程是这样的：函数抛出异常时，会发生所谓的stack_unwinding（堆栈回滚），即堆栈会展开，由于是栈对象，自然存在于栈中，所以在堆栈回滚的过程中，栈对象的析构函数会被执行，从而释放其所封装的资源。除非在析构函数执行的过程中再次抛出异常――而这种可能性是很小的，所以用栈对象封装资源是比较安全的。基于此认识，我们就可以创建一个自己的句柄或代理来封装资源了。智能指针（auto_ptr）中就使用了这种技术。在有这种需要的时候，我们就希望资源封装类只能在栈中创建，也就是要限制在堆中创建该资源封装类的实例。



### 3.1.3 禁止产生堆对象

上面提到，当创建一个资源封装类时，限制该类对象只能在栈中产生，禁止产生其堆对象，这样就能在异常的情况下自动释放封装的资源。

**那么怎样禁止产生堆对象了？我们已经知道，产生堆对象的唯一方法是使用new操作，那么禁止使用new不就行了么。再进一步，new操作执行时会调用operator new，而operator new是可以重载的。方法有了，就是使new operator 为private，为了对称，最好将operator delete也重载为private。**

现在，你也许又有疑问了，难道创建栈对象不需要调用new吗？是的，不需要，因为创建栈对象不需要搜索内存，而是直接调整堆栈指针，将对象压栈，而operator new的主要任务是搜索合适的堆内存，为堆对象分配空间。禁止产生堆对象的类的定义如下：

```c++
#include <stdlib.h> //需要用到C式内存分配函数

class Resource; // 代表需要被封装的资源类

class NoHashObject
{
private:
    Resource *ptr; // 指向被封装的资源
    ...... //其它数据成员
    
    void *operator new(size_t size) // 非严格实现，仅作示意之用
    {
        return malloc(size);
    }

    void operator delete(void *pp) // 非严格实现，仅作示意之用
    {
        free(pp);
    }

public:
    NoHashObject()
    {
        // 此处可以获得需要封装的资源，并让ptr指针指向该资源
        ptr = new Resource();
    }

    ~NoHashObject()
    {
        delete ptr; // 释放封装的资源
    }
};
```

此时如果再有这样的代码`NoHashObject *fp = new NoHashObject();`则会编译报错。

那么，在类NoHashObject的定义不能修改的情况下，就一定不能产生该类型的堆对象了吗？不，还是有办法的，可称之为“暴力破解法”。C++是如此的强大，强大到可以用它做想做的任何事情。这里主要用到的技巧是指针类型的强制转换。

```c++
void main(void)
{
    char *temp = new char[sizeof(NoHashObject)];

    // 强制类型转换，现在ptr是一个指向NoHashObject对象的指针
    NoHashObject *obj_ptr = (NoHashObject *)temp;

    temp = NULL; // 防止通过temp指针修改NoHashObject对象

    // 再一次强制类型转换，让rp指针指向堆中NoHashObject对象的ptr成员
    Resource *rp = (Resource *)obj_ptr;

    // 初始化obj_ptr指向的NoHashObject对象的ptr成员
    rp = new Resource();

    // 现在可以通过使用obj_ptr指针使用堆中的NoHashObject对象成员了
    ......

    delete rp; // 释放资源
    temp = (char *)obj_ptr;
    obj_ptr = NULL; // 防止悬挂指针产生
    delete[] temp; // 释放NoHashObject对象所占的堆空间。
}
```

对于上面的这么多强制类型转换，其最根本的是什么？我们可以这样理解：
某块内存中的数据是不变的，而类型就是我们戴上的眼镜，当我们戴上一种眼镜后，我们就会用对应的类型来解释内存中的数据，这样不同的解释就得到了不同的信息。
所谓强制类型转换实际上就是换上另一副眼镜后再来看同样的那块内存数据。

另外要提醒的是，不同的编译器对对象的成员数据的布局安排可能是不一样的，比如，大多数编译器将NoHashObject的ptr指针成员安排在对象空间的头4个字节，这样才会保证下面这条语句的转换动作预期执行：
```c++
Resource* rp = (Resource*)obj_ptr ; 
```

但是，并不一定所有的编译器都是如此。

既然可以禁止产生某种类型的堆对象，那么可以设计一个类，使之不能产生栈对象吗？当然可以。



### 3.1.4 禁止产生栈对象

**前面已经提到了，创建栈对象时会移动栈顶指针以“挪出”适当大小的空间，然后在这个空间上直接调用对应的构造函数以形成一个栈对象，而当函数返回时，会调用其析构函数释放这个对象，然后再调整栈顶指针收回那块栈内存。那么将构造函数或析构函数设为私有的，这样系统就不能调用构造/析构函数了，当然就不能在栈中生成对象了。**

**如果将构造函数设置为私有，那么也不能用new来直接产生堆对象了，因为new在为对象分配空间后也会调用它的构造函数。所以，我们只将析构函数设置为private。再进一步，将析构函数设为private除了会限制栈对象生成外，还有其它影响吗？是的，这还会限制继承。如果一个类不打算作为基类，通常采用的方案就是将其析构函数声明为private。**

**为了限制栈对象，却不限制继承，我们可以将析构函数声明为protected，这样就两全其美了。**

```c++
class NoStackObject
{
protected:
    ~NoStackObject() {}

public:
    void destroy()
    {
        delete this; // 调用保护析构函数
    }
};
```

接着，可以像这样使用NoStackObject类：

```c++
NoStackObject* hash_ptr = new NoStackObject() ;
... ... //对hash_ptr指向的对象进行操作
hash_ptr->destroy() ; 
```

是不是觉得有点怪怪的，我们用new创建一个对象，却不是用delete去删除它，而是要用destroy方法。很显然，用户是不习惯这种怪异的使用方式的。所以，我们决定将构造函数也设为private或protected。这又回到了上面试图避免的问题，即不用new，那么该用什么方式来生成一个对象？我们可以用间接的办法完成，即让这个类提供一个static成员函数专门用于产生该类型的堆对象。（设计模式中的singleton模式就可以用这种方式实现。）

```c++
class NoStackObject
{
protected:
    NoStackObject() {}
    ~NoStackObject() {}

public:
    static NoStackObject *creatInstance()
    {
        return new NoStackObject(); // 调用保护的构造函数
    }

    void destroy()
    {
        delete this; // 调用保护的析构函数
    }
};
```

现在可以这样使用NoStackObject类了：

```c++
NoStackObject* hash_ptr = NoStackObject::creatInstance() ;
...... //对hash_ptr指向的对象进行操作
hash_ptr->destroy() ;
hash_ptr = NULL ; //防止使用悬挂指针 
```



















