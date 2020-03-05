# 返回值、拷贝和移动

## 1. 返回值和拷贝的问题

大凡编程语言，都会有“函数”这个概念。对于外部的调用者来说，一个函数的返回值，往往体现了它的全部功能（有副作用的函数除外）。

返回值其实应该是一个简单的话题。我们一般都希望自己的函数是幂等的，此时使用返回值来返回计算的结果则是理所当然的事情。比如说像下面这样：

```cpp
int add(int a, int b)
{
    return a + b;
}
 
int main(int argc, char* argv[])
{
    int i = add(1, 2);
    std::cout << i << std::endl;
    return 0;
}
```

不论是函数还是调用者都会身心愉悦，因为这是最自然的使用方法了。但是在C++里，通过函数返回值返回处理结果往往是一种奢侈的行为。

让我们看一个新的例子：

```cpp
int main(int argc, char* argv[])
{
    std::string ss;
    std::string s1("Hello"), s2("-"), s3("World"), s4("!");
    ss = s1 + s2 + s3 + s4;
    std::cout << ss << std::endl;
    return 0;
}
```

 相信有经验的C++程序员看到了都会皱眉头，良好的做法应当是使用`+=`来代替之：

```cpp
int main(int argc, char* argv[])
{
    std::string ss;
    std::string s1("Hello"), s2("-"), s3("World"), s4("!");
    ss += s1 += s2 += s3 += s4;
    std::cout << ss << std::endl;
    return 0;
}
```

原因很简单，`+`和`+=`的操作符重载的实现一般而言是像这样的：

```cpp
operator char*(void) const
{
    return str_;
}
 
Str& operator+=(const char* str)
{
    if (str) strcat_s(str_, 1024, str);
    return (*this);
}
 
friend Str operator+(const Str& x, const Str& y)
{
    return Str(x) += y;
}
```

注意到上面，由于`operator+`不能修改任何一个参数，所以必须构建一个临时变量`Str(x)`，并且`Str(x)`在把值传递出去之后，自身马上就销毁了。外面负责接收的变量只能得到并复制一遍`Str(x)`，于是一个简单的返回值就造成了两次`x`的拷贝。当像上文“`ss = s1 + s2 + s3 + s4`”这样连加的时候，拷贝就会像击鼓传花一样，在每一次+调用处发生。

我们也不可能把`operator+`的返回值像`operator+=`一样用引用或指针来代替，否则外部得到的引用将是一个悬空引用，无法通过它拿到处理后的数据。

为了说明问题，我们可以写一个简单的例子来看看这样赋值到底会有多大的损耗：

```cpp
// Using C++ 98/03
class Str
{
private:
    char* str_;
 
public:
    Str(void)                       // Default constructor, do nothing
        : str_(nullptr)
    {}
 
    Str(const char* rhs)            // Constructor declaration
        : str_(nullptr)
    {
        if (!rhs) return;
        str_ = new char[1024];
        strcpy_s(str_, 1024, rhs);
        std::cout << "Str constructor " << str_ << std::endl;
    }
 
    Str(const Str& rhs)             // Copy constructor
        : str_(nullptr)
    {
        if (!rhs) return;
        str_ = new char[1024];
        strcpy_s(str_, 1024, rhs.str_);
        std::cout << "Str copy constructor " << str_ << std::endl;
    }
 
    ~Str(void)                      // Destructor
    {
        if (!str_) return;
        std::cout << "Str destructor " << str_ << std::endl;
        delete [] str_;
    }
 
    const Str& operator=(Str rhs)   // Assignment operator
    {
        rhs.swap(*this);            // Using copy-and-swap idiom
        return (*this);
    }
 
    void swap(Str& rhs)
    {
        std::swap(str_, rhs.str_);
    }
 
    operator char*(void) const
    {
        return str_;
    }
 
    Str& operator+=(const char* rhs)
    {
        if (rhs) strcat_s(str_, 1024, rhs);
        return (*this);
    }
 
    friend Str operator+(const Str& x, const Str& y)
    {
        return Str(x) += y;
    }
};
 
int main(int argc, char* argv[])
{
    Str ss;
    Str s1("Hello"), s2("-"), s3("World"), s4("!");
    std::cout << std::endl;
 
    ss = s1 + s2 + s3 + s4;
 
    std::cout << std::endl;
    std::cout << ss << std::endl;
    std::cout << std::endl;
    return 0;
}
```

 这是一个简单的`Str`类，包装了一个`char*`，并限制字符串长度为1024。程序运行之后，我们得到如下打印信息：

```bash
Str copy constructor Hello
Str copy constructor Hello-
Str destructor Hello-
Str copy constructor Hello-
Str copy constructor Hello-World
Str destructor Hello-World
Str copy constructor Hello-World
Str copy constructor Hello-World!
Str destructor Hello-World!
Str destructor Hello-World
Str destructor Hello-
```

连续6次拷贝构造，并且最终这些临时生成的字符串统统炸鞭炮一样噼里啪啦被销毁掉了。一次拷贝的工作是`new`一个1024的大内存块，再来一次`strcpy`。连续的构造-拷贝-析构，对性能会有相当大的影响。所以尽量选择`+=`其实是不得已而为之的事情。

同样的道理，我们也很少写`Str to_string(int i)`，取而代之是`void to_string(int i, Str& s)`。为了避免返回值的性能问题，我们不得不牺牲掉代码的优雅，用蹩脚的参数来解决。

除了性能之外，对象的所有权也是一个问题。例如我们现在有一个Handle类，它负责管理某个文件的句柄，并且文件的访问是互斥的。很显然，Handle类和打开的文件应该是一对一的，不应该存在多个Handle管理或访问一个文件的情况。那么这样一个Handle类应该如何写呢？

```cpp
class Handle
{
private:
    handle_t h_;

public:
    Handle(const string& name) : h_(0)
    {
        // ... may be open a file
    }

    Handle(const Handle& handle) : h_(0)
    {
        // ... ???
    }
};
```

 或许，我们可以禁止掉拷贝动作？

```cpp
class Handle
{
private:
    handle_t h_;

    Handle(const Handle&);
    Handle& operator=(const Handle&);

public:
    // ......
};
```

 那么怎样返回一个`Handle`对象呢？

```cpp
Handle open_file(const string& name)
{
    return Handle(name); // ???
}
```

## 2. 一些解决方案

### 2.1 std::auto\_ptr

`std::auto_ptr`是C++98/03里提供的智能指针，它在C++11中被废弃，并在C++17中被删除。在C++98/03的时候，会使用它的人也是少数（这里的“会”有双重含义，会用，和会去用）。

`std::auto_ptr`负责管理一块堆内存的生存期，并且，`std::auto_ptr`也是独占式的，每个`std::auto_ptr`指向的内存块都不同。那么`std::auto_ptr`能否作为返回值呢？

```cpp
std::auto_ptr<int> make_int(int a)
{
    return std::auto_ptr<int>(new int(a));
}

int main(void)
{
    std::auto_ptr<int> pi = make_int(123);
    std::cout << *pi << std::endl;
    return 0;
}
```

可以看到，`make_int`工作得很好。这是因为`std::auto_ptr`在拷贝构造函数中强行获取了对方的资源所有权：

```cpp
template <typename T>
class auto_ptr
{
private:
    T* p_;

public:
    // ...

    auto_ptr(auto_ptr& a) : p_(a.release()) {}

    template <typename U>
    auto_ptr(auto_ptr<U>& a) : p_(a.release()) {}

    // ...
};
```

这种违反直觉的行为极容易导致更多的问题。比如：

```cpp
std::auto_ptr<int> pi = make_int(123);
std::auto_ptr<int> pj = pi;

// ...

std::cout << *pi << std::endl; // crash
```

因此，在项目中使用`std::auto_ptr`往往是一个糟糕的决定。我们必须在代码review的时候仔细检查每个`std::auto_ptr`的使用是否正确，或者干脆禁止`std::auto_ptr`之间的赋值行为。

### 2.2 写时拷贝（COW，copy-on-write）和引用计数（Reference Counting）

我们可以不需要每次都拷贝数据，只有当数据发生改写的时候，才会出现拷贝操作。这个技巧可以让普通拷贝操作的时间复杂度降到O\(1\)。

体现在代码上，上文中`Str`的拷贝构造函数内，就不再直接使用`strcpy_s`了，取而代之的是一行短短的指针复制（注意不是内容复制）的浅拷贝。直到`+=`这种会改变自身内容的操作时，`Str`内部才会再创建一份内存，并把现有的数据拷贝一次。

修改`Str`有两个关键点：

1. 我们需要使用“引用计数”，来标记当前的内存块被多少个`Str`共有，否则的话一个`Str`在析构时删除内存，所有与之相关的其他`Str`类内部的指针全部都会成为野指针。
2. 我们需要处理所有的“写”动作，什么时候`Str`需要被改写，在这之前我们就必须将它拷贝出来。

使用COW之后的`Str`看起来可能像这样：

![](https://yuml.me/diagram/plain;dir:LR/class/[rc_string%7C-str_:char*%7C+~rc_string%28%29;+rc_string*%20clone%28%29],[Str%7C-rc_str_:rc_string*%7C+~Str%28%29;+Str&%20operator+=%28const%20char*%29%7Bbg:tomato%7D],[Str]*-1%3E[rc_string],[Str]-[note:Multiple%20Str%20objects%20use%20a%20same%20rc-string%7Bbg:cornsilk%7D],[rc_base%7C+inc%28%29;+dec%28%29]%5E-[rc_string])

有了COW之后，我们可以不需要通过引用或指针来传递一个对象（因为对象的拷贝其实也是浅拷贝）；可以直接通过返回值返回一个对象，而不必担心效率问题。同时，引用计数也保证了下层资源的所有权问题，上图的字符串完全可以换成某个独占资源的句柄（可能需要调整拷贝动作）。

听起来似乎挺好，如果使用COW能够解决所有问题的话……对于`Str to_string(int i)`一类的问题，COW确实有足够高的性能，但对于我们一开始的“`ss = s1 + s2 + s3 + s4`”，COW帮不了太多忙。

这是因为虽然`Str(x)`时，生成临时对象的拷贝不存在了，但马上执行的赋值改写操作不得不让内存被复制一遍，传递到下一层情况也不会有任何改变，连续赋值导致连续复制，在这种情况下原先的性能问题依然存在。

### 2.3 返回值优化（RVO、NRVO）

RVO（Return Value Optimization），中文为返回值优化；NRVO（Named Return Value Optimization）具名返回值优化是RVO的一个补充。它们都是针对返回值性能问题的编译器优化技术，目前主流的编译器对它们都有很好的支持。

RVO和NRVO分别指的是下面这两种情况：

```cpp
Str make_string1(const char* str)
{
    return Str(str); // RVO
}

Str make_string2(const char* str)
{
    Str tmp(str);
    return tmp; // NRVO
}

int main(void)
{
    Str s1 = make_string1("123");
    Str s2 = make_string2("321");
    return 0;
}
```

打印输出如下：

```bash
Str constructor 123
Str constructor 321
Str destructor 321
Str destructor 123
```

### 2.4 我们需要移动

## 3. C++11的右值引用和移动语义

