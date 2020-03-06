# 返回值、拷贝和移动

## 1. 返回值和拷贝的问题

大凡编程语言，都会有“函数”这个概念。对于外部的调用者来说，一个函数的返回值，往往体现了它的全部功能（有副作用的函数除外）。

返回值其实应该是一个简单的话题。我们一般都希望自己的函数是幂等的，此时使用返回值来返回计算的结果则是理所当然的事情。比如说像下面这样：
```cpp
int add(int a, int b) {
    return a + b;
}
 
int main(int argc, char* argv[]) {
    int i = add(1, 2);
    std::cout << i << std::endl;
    return 0;
}
```
这里我们的`add`函数通过`int`类型返回值直接返回了相加的结果。

在这个例子里，我们的返回值是没有什么额外代价的。但是在我们使用了对象以后，直接返回一个实例往往是一种奢侈的行为。

让我们用经典的字符串操作作为例子：
```cpp
int main() {
    string ss;
    string s1("Hello"), s2("World"), s3("!");
    ss = s1 + '-' + s2 + s3;
    std::cout << ss.cdata() << std::endl;
    return 0;
}
```
输出：
```
Hello-World!
```
结果是正确的，但在性能上会非常难受。为了说明问题，我们先给出一个简单的`string`实现：
```cpp
// Using C++ 98/03

#include <stddef.h>
#include <string.h>
#include <stdlib.h>

namespace test {

class string {

    char * data_;
    size_t len_;

    void extend(size_t new_len) {
        if (new_len <= len_) return;
        data_ = (char *)realloc((void *)data_, (len_ = new_len) + 1);
    }

public:
    string()
        : data_(NULL), len_(0) {}

    string(char c)
        : data_(NULL), len_(1) {
        data_ = (char *)malloc(len_ + 1);
        data_[0] = c;
        data_[1] = '\0';
    }

    string(char const * str)
        : data_(NULL), len_(strlen(str)) {
        data_ = strcpy((char *)malloc(len_ + 1), str);
    }

    string(string const & other)
        : data_(NULL), len_(other.len_) {
        if (!other.empty()) {
            data_ = strcpy((char *)malloc(len_ + 1), other.data_);
        }
    }

    ~string() {
        if (data_ != NULL) {
            free(data_);
        }
    }

    string& operator=(string other) {
        swap(other);
        return *this;
    }

    void swap(string& other) {
        char * d = data_;
        size_t l = len_;
        data_ = other.data_;
        len_  = other.len_;
        other.data_ = d;
        other.len_  = l;
    }

    size_t length() const {
        return len_;
    }

    bool empty() const {
        return length() == 0;
    }

    void clear() {
        if ((data_ != NULL) || (len_ != 0)) {
            free(data_);
            len_ = 0;
        }
    }

    char const * data () const { return data_; }
    char       * data ()       { return data_; }
    char const * cdata() const { return data_; }

    char   operator[](size_t i) const { return data_[i]; }
    char & operator[](size_t i)       { return data_[i]; }

    string& operator+=(char c) {
        extend(len_ + 1);
        data_[len_ - 1] = c;
        data_[len_]     = '\0';
        return *this;
    }

    string& operator+=(string const & other) {
        if (!other.empty()) {
            extend(len_ + other.len_);
            strcat(data_, other.data_);
        }
        return *this;
    }

    friend string operator+(string const & x, string const & y) {
        return string(x) += y;
    }
};

} // namespace test
```
如上实现，仅使用标准C函数。

我们可以看到，关键的`operator+`重载，由于不能修改任何一个参数，所以必须在内部构建一个匿名临时变量`string(x)`。这个临时变量导致我们不可能像`operator+=`一样用引用作为`operator+`的返回值，否则外部将得到一个悬空引用。函数调用完毕后，外部接到返回值，必须马上进行拷贝构造，之后这个匿名临时变量就会自动销毁，于是一个简单的返回值造成了两次拷贝。

当然，`operator+`的参数类型是`string const &`，这个引用可以直接绑定返回值，但其本身作为参数`x`还是必须在函数内部拷贝给新的匿名临时变量`string(x)`。出现如 `ss = s1 + '-' + s2 + s3` 这样的连续调用时，拷贝就会像击鼓传花一样，在每一次`operator+`返回时发生。

为了说明问题，我们可以跟踪一下构造函数，来看看这样到底会有多大的损耗：

```cpp
// ......

// 为构造和析构函数加上输出

string()
    : data_(NULL), len_(0) {
    printf("string constructor\n");
}

string(char c)
    : data_(NULL), len_(1) {
    data_ = (char *)malloc(len_ + 1);
    data_[0] = c;
    data_[1] = '\0';
    printf("string constructor: %s\n", data_);
}

string(char const * str)
    : data_(NULL), len_(strlen(str)) {
    data_ = strcpy((char *)malloc(len_ + 1), str);
    printf("string constructor: %s\n", data_);
}

string(string const & other)
    : data_(NULL), len_(other.len_) {
    if (!other.empty()) {
        data_ = strcpy((char *)malloc(len_ + 1), other.data_);
    }
    printf("string copy constructor: %s\n", data_);
}

~string() {
    printf("string destructor");
    if (data_ != NULL) {
        printf(": %s", data_);
        free(data_);
    }
    printf("\n");
}

// ......

using namespace test;

int main() {
    string ss;
    string s1("Hello"), s2("World"), s3("!");
    ss = s1 + '-' + s2 + s3;
    std::cout << ss.cdata() << std::endl;
    return 0;
}
```
得到打印如下：
```
string constructor
string constructor: Hello
string constructor: World
string constructor: !
string constructor: -
string copy constructor: Hello
string copy constructor: Hello-
string destructor: Hello-
string copy constructor: Hello-
string copy constructor: Hello-World
string destructor: Hello-World
string copy constructor: Hello-World
string copy constructor: Hello-World!
string destructor: Hello-World!
string destructor
string destructor: Hello-World
string destructor: Hello-
string destructor: -
Hello-World!
string destructor: !
string destructor: World
string destructor: Hello
string destructor: Hello-World!
```
除去“Hello-World!”的输出，我们可以看到6次拷贝构造，以及对应的析构动作。

除了性能之外，对象的所有权也是一个问题。例如我们现在有一个Handle类，它负责管理某个文件的句柄，并且文件的访问是互斥的。很显然，Handle类和打开的文件应该是一对一的，不应该存在多个Handle管理或访问一个文件的情况。那么这样一个Handle类应该如何写呢？
```cpp
class Handle {
private:
    handle_t h_;

public:
    Handle(const string& name) : h_(0) {
        // ... 可能会在这里打开某个文件
    }

    Handle(const Handle& handle) : h_(0) {
        // ... copy constructor ???
    }
};
```
 或许，我们应该直接禁止拷贝？
```cpp
class Handle {
private:
    handle_t h_;

    Handle(const Handle&);
    Handle& operator=(const Handle&);

public:
    // ......
};
```
那么这个时候，怎样通过函数返回一个`Handle`对象呢？
```cpp
Handle do_open(int id) {
    return Handle((id == 0) ? /* ... */ : /* ... */); // error
}

// 我们只能使用参数，通过引用或指针返回一个Handle
bool open_file(Handle* h, int id) {
    // ......
}
```

## 2. 一些解决方案

### 2.1 std::auto_ptr

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

