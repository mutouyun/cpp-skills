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
// Using C++ 98/03, without STL

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
};

string operator+(string const & x, string const & y) {
    return string(x) += y;
}

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
    // error until C++17
    return Handle((id == 0) ? /* ... */ : /* ... */);
}

// 我们只能使用参数，通过引用或指针返回一个Handle
bool open_file(Handle* h, int id) {
    // ......
}
```

## 2. 一些解决方案

### 2.1 返回值优化（RVO、NRVO）

返回值优化（RVO，Return Value Optimization）；具名返回值优化（NRVO，Named Return Value Optimization）。NRVO是RVO的一个补充，它们都是针对返回值性能问题的编译器优化技术。

RVO和NRVO分别指的是下面这两种情况：
```cpp
vector<int> get_vec_rvo(size_t n) {
    return vector<int>(n); // RVO
}

vector<int> get_vec_nrvo(size_t n) {
    vector<int> tmp(n);
    return tmp; // NRVO
}

int main() {
    vector<int> v1 = get_vec_rvo(5);
    vector<int> v2 = get_vec_nrvo(5);
    return 0;
}
```
如上的代码里，临时变量`vector<int>(n)`和`tmp`就仿佛穿透了函数一样，直接分别构建为`v1`和`v2`，不存在任何拷贝动作。这个技术方案被称为“copy elision（复制消除）”，目前的主流编译器都支持这一特征。

我们再看个例子。把上面`string`的`operator+`稍微修改一下：
```cpp
string operator+(string const & x, string const & y) {
    string tmp(x);
    tmp += y;
    return tmp;
}
```
运行结果：
```
string constructor
string constructor: Hello
string constructor: World
string constructor: !
string constructor: -
string copy constructor: Hello
string copy constructor: Hello-
string copy constructor: Hello-World
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
可以很清晰的看到，NRVO生效了，只有3次必须的copy constructor。

RVO和NRVO可以极大地改善返回值的效率，但也存在很大的局限性。比如我们之前的`operator+`写法：
```cpp
string operator+(string const & x, string const & y) {
    return string(x) += y;
}
```
就不会触发任何返回值优化。另外，它对于`string tmp(x)`这样在构建临时变量时触发的拷贝也无能为力。

参考：
1. [Copy elision - cppreference.com](https://en.cppreference.com/w/cpp/language/copy_elision)
2. [RVO V.S. std::move (C/C++ compilers for IBM Z Blog)](https://www.ibm.com/developerworks/community/blogs/5894415f-be62-4bc0-81c5-3956e82276f3/entry/RVO_V_S_std_move?lang=en)
3. [Rvalue References and Move Semantics in C++11 - Cprogramming.com](https://www.cprogramming.com/c++11/rvalue-references-and-move-semantics-in-c++11.html)
4. [C++ 17 最大的改变——Guaranteed copy elision - 知乎](https://zhuanlan.zhihu.com/p/22821671)

### 2.2 写时拷贝（COW，copy-on-write）和引用计数（Reference Counting）

我们可以不需要每次都拷贝数据，只有当数据发生改写的时候，才会出现拷贝操作。这个技巧可以让普通拷贝操作的时间复杂度降到O\(1\)。

体现在代码上，`string`的拷贝构造函数内，就不再直接使用`strcpy`了，取而代之的是内部指针的浅拷贝。直到`+=`这种会改变自身内容的操作时，`string`才会创建一份新内存，对现有的数据进行一次深拷贝。

修改`string`有两个关键点：

1. 我们需要使用“引用计数”，来标记当前的内存块被多少个`string`共有，否则的话一个`string`在析构时删除内存，所有与之相关的其他`string`类内部的指针全部都会成为野指针。
2. 我们需要处理所有的“写”动作，什么时候`string`需要被改写，在这之前我们就必须将它拷贝出来。

使用COW之后的`string`看起来可能像这样：

![](https://yuml.me/diagram/plain;dir:LR/class/[rc_string%7C-data_:char*%7C+~rc_string%28%29;+rc_string*%20clone%28%29],[string%7C-rc_data_:rc_string*%7C+~string%28%29;+string&%20operator+=%28string%20const&%29%7Bbg:tomato%7D],[string]*-1%3E[rc_string],[string]-[note:Multiple%20Str%20objects%20use%20a%20same%20rc_string%7Bbg:cornsilk%7D],[rc_base%7C+~rc_base%28%29;+inc%28%29;+dec%28%29]%5E-[rc_string])

有了COW之后，我们可以不需要通过引用或指针来传递一个对象（因为对象的拷贝其实也是浅拷贝）；可以直接通过返回值返回一个对象，而不必担心效率问题。同时，引用计数也保证了下层资源的所有权问题，上图的字符串完全可以换成某个独占资源的句柄。

我们来尝试简单实现一下基于COW的`string`。首先完成对引用计数的封装：
```cpp
// Using C++ 98/03, without STL

class rc_base {
    int rc_;

    rc_base(rc_base const &);
    rc_base& operator=(rc_base const &);

protected:
    rc_base() : rc_(0) {}

public:
    virtual ~rc_base() {}

    int ref_count() const {
        return rc_;
    }

    void inc() {
        ++rc_;
    }

    void dec() {
        if (--rc_ == 0) delete this;
    }
};

class rc_string : public rc_base {
    char * data_;
    size_t len_;

public:
    rc_string()
        : data_(NULL), len_(0) {
    }

    ~rc_string() {
        if (data_ != NULL) free(data_);
    }

    size_t length() const { return len_; }

    char const * data() const { return data_; }
    char       * data()       { return data_; }

    void extend(size_t new_len) {
        if (new_len <= len_) return;
        data_ = (char *)realloc((void *)data_, (len_ = new_len) + 1);
    }

    rc_string* clone() {
        rc_string* rs = new rc_string;
        if (len_ > 0) {
            rs->extend(len_);
            strcpy(rs->data(), data_);
            printf("string clone: %s\n", data_);
        }
        return rs;
    }
};

template <class T>
class rc_ptr {
    T* rc_data_;

public:
    rc_ptr(T* ptr)
        : rc_data_(ptr) {
        if (rc_data_ != NULL) rc_data_->inc();
    }

    rc_ptr(rc_ptr const & other)
        : rc_data_(other.rc_data_) {
        if (rc_data_ != NULL) rc_data_->inc();
    }

    ~rc_ptr() {
        if (rc_data_ != NULL) rc_data_->dec();
    }

    rc_ptr& operator=(rc_ptr other) {
        swap(other);
        return *this;
    }

    void swap(rc_ptr& other) {
        rc_string* d = rc_data_;
        rc_data_ = other.rc_data_;
        other.rc_data_ = d;
    }

    operator bool() const { return rc_data_ != NULL; }

    T const * operator->() const { return rc_data_; }
    T       * operator->()       { return rc_data_; }
};
```
这里的rc实现类似一个侵入式智能指针。之后，对`string`进行改造：
```cpp
class string {
    rc_ptr<rc_string> rc_data_;

    void try_clone() {
        if (rc_data_->ref_count() > 1) {
            rc_data_ = rc_data_->clone();
        }
    }

public:
    string()
        : rc_data_(new rc_string) {
        printf("string constructor\n");
    }

    string(char c)
        : rc_data_(new rc_string) {
        rc_data_->extend(1);
        rc_data_->data()[0] = c;
        rc_data_->data()[1] = '\0';
        printf("string constructor: %s\n", rc_data_->data());
    }

    string(char const * str)
        : rc_data_(new rc_string) {
        rc_data_->extend(strlen(str));
        strcpy(rc_data_->data(), str);
        printf("string constructor: %s\n", rc_data_->data());
    }

    string(string const & other)
        : rc_data_(other.rc_data_) {
    }

    ~string() {
        printf("string destructor");
        if (!empty()) {
            printf(": %s", rc_data_->data());
        }
        printf("\n");
    }

    string& operator=(string other) {
        swap(other);
        return *this;
    }

    void swap(string& other) {
        rc_data_.swap(other.rc_data_);
    }

    size_t length() const {
        return rc_data_->length();
    }

    bool empty() const {
        return length() == 0;
    }

    void clear() {
        if (!empty()) {
            rc_data_ = new rc_string;
        }
    }

    char const * data () const { return rc_data_->data(); }
    char const * cdata() const { return rc_data_->data(); }

    char operator[](size_t i) const { return data()[i]; }

    char * data() {
        try_clone();
        return rc_data_->data();
    }

    char & operator[](size_t i) {
        try_clone();
        return data()[i];
    }

    string& operator+=(char c) {
        try_clone();
        rc_data_->extend(length() + 1);
        rc_data_->data()[length() - 1] = c;
        rc_data_->data()[length()]     = '\0';
        return *this;
    }

    string& operator+=(string const & other) {
        if (!other.empty()) {
            try_clone();
            rc_data_->extend(length() + other.length());
            strcat(rc_data_->data(), other.data());
        }
        return *this;
    }
};
```
从现在开始，`string`的拷贝再也不会引起性能问题了……然而，对于我们的`operator+`来说，COW和RVO存在一样的问题。

同样的测试代码：
```cpp
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
string clone: Hello
string clone: Hello-
string clone: Hello-World
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
理所当然的，我们看到了3次必须的clone操作。

这是因为虽然`string tmp(x)`生成临时对象的拷贝不存在了，但马上执行的`operator+=`操作必须进行一次深拷贝，在这种情况下原先的性能问题依然存在。

另外，对字符串使用COW也并不总是能提高我们的执行效率。比如上面的简单实现中，并没有考虑多线程下的安全问题；并且对非`const`对象的`operator[]`操作一定会触发一次“写”动作。而为了解决这些问题，实际上基于COW的对象不得不引入一些复杂的设计。

参考：
1. [std::string的Copy-on-Write：不如想象中美好 - PromisE_谢 - 博客园](https://www.cnblogs.com/promise6522/archive/2012/03/22/2412686.html)

### 2.3 Mojo（Move of Joint Objects）

Andrei Alexandrescu，《Modern C++ Design》的作者，2003年曾在Generic上发过一篇著名的文章《Move Constructors》，里面给出了他对于这个问题的解决方案：Mojo (Move of Joint Objects)，核心在于从语义上区分出“临时值”的概念来。

我们来看看Mojo方案提供的几个模板工具：
```cpp
namespace mojo {

template <class T>
class constant /* type sugar for constants */ {
    T const * data_;

public:
    explicit constant(const T& obj) : data_(&obj) {}
    const T& get() const { return *data_; }
};

template <class T>
class temporary /* type sugar for temporaries */ : private constant<T> {
public:
    explicit temporary(T& obj) : constant<T>(obj) {}
    T& get() const { return const_cast<T&>(constant<T>::get()); }
};

template <class T>
struct enabled /* inside mojo */ {

    operator constant<T>() const {
        return constant<T>(static_cast<const T&>(*this));
    }

    operator temporary<T>() {
        return temporary<T>(static_cast<T&>(*this));
    }

protected:
    enabled() {} // intended to be derived from
    ~enabled() {} // intended to be derived from
};

} // namespace mojo
```
然后为`string` **添加** 一个处理临时对象的“移动”构造函数：
```cpp
class string : public mojo::enabled<string> {
    // ......

    string(mojo::temporary<string> other)
        : data_(NULL), len_(other.get().len_) {
        swap(other.get());
        printf("string move constructor: %s\n", data_);
    }

    // ......
};
```
让我们来看看它是怎么工作的。首先，对于函数的参数：
```cpp
// binds to non-const temporaries
void connect(mojo::temporary<string>) {
    std::cout << "mojo::temporary<string>" << std::endl;
}

// binds to all const objects (lvalues AND temporaries)
void connect(mojo::constant<string>) {
    std::cout << "mojo::constant<string>" << std::endl;
}

// binds to non-const lvalues
void connect(string& str) {
    // just forward to the other overload
    connect(mojo::constant<string>(str));
}
```
Mojo通过`constant`、`temporary`和普通左值引用`string&`成功区分了3种不同的语义：
```cpp
string s1("http://moderncppdesign.com");
// invoke connect(string&)
connect(s1);

// invoke operator TemporaryString()
// followed by connect(TemporaryString)
connect(string("http://moderncppdesign.com"));

const string s2("http://moderncppdesign.com");
// invoke operator ConstantString() const
// followed by connect(ConstantString)
connect(s2);
```
那么同样的，`operator+`可以定义为：
```cpp
string operator+(mojo::constant<string> x, string const & y) {
    string tmp(x.get());
    tmp += y;
    return tmp;
}

string operator+(mojo::temporary<string> x, string const & y) {
    x.get() += y;
    return x;
}

string operator+(string & x, string const & y) {
    return mojo::constant<string>(x) + y;
}
```
类似`connect`，使用三个不同的重载对应三种不同的情况。测试结果如下：
```
string constructor
string constructor: Hello
string constructor: World
string constructor: !
string constructor: -
string copy constructor: Hello
string move constructor: Hello-World
string move constructor: Hello-World!
string destructor
string destructor
string destructor
string destructor: -
Hello-World!
string destructor: !
string destructor: World
string destructor: Hello
string destructor: Hello-World!
```
现在，我们的方案工作得非常好。一次必须的拷贝构造，之后每次的字符串追加都是通过`move`动作获得临时对象的内存来完成，完全规避了耗时的拷贝动作。

但这个方案无法涵盖所有的情况。比如，我们为了规避如 `if (s1 + s2 = s3)` 这样的意外赋值，需要为函数的返回值添加`const`限定符：
```cpp
string const operator+(mojo::constant<string> x, string const & y) {
    string tmp(x.get());
    tmp += y;
    return tmp;
}

// ......
```
此时，`string const`会优先匹配到拷贝构造函数`string(string const &)`上，从而让我们的“移动”语义失效。对此，Mojo的解决方案是针对返回值提供`fnresult`类模板：
```cpp
namespace mojo {

// ......

template <class T>
class fnresult : public T {
public:
    // The cast below is valid given that
    // nobody ever really creates a const fnresult object
    fnresult(fnresult const & rhs)
        : T(temporary<T>(const_cast<fnresult&>(rhs))) {}

    explicit fnresult(T& rhs) : T(temporary<T>(rhs)) {}
};

template <class T>
struct enabled /* inside mojo */ {

    operator constant<T>() const {
        return constant<T>(static_cast<const T&>(*this));
    }

    operator temporary<T>() {
        return temporary<T>(static_cast<T&>(*this));
    }

    operator fnresult<T>() {
        return fnresult<T>(static_cast<T&>(*this));
    }

protected:
    enabled() {} // intended to be derived from
    ~enabled() {} // intended to be derived from
};

} // namespace mojo
```
同时，调整`string`的定义，将添加的“移动”构造函数改为两个，并修改`operator+`：
```cpp
class string : public mojo::enabled<string> {
    // ......

    string(mojo::temporary<string> other)
        : data_(NULL), len_(other.get().len_) {
        swap(other.get());
        printf("string move constructor: %s\n", data_);
    }

    string(mojo::fnresult<string> other)
        : data_(NULL), len_(other.len_) {
        swap(other);
        printf("string move constructor: %s\n", data_);
    }

    // ......
};

mojo::fnresult<string> const operator+(string const & x, string const & y) {
    return string(x) += y;
}

mojo::fnresult<string> const operator+(mojo::fnresult<string> x, string const & y) {
    return x += y;
}
```
结果如下：
```
string constructor
string constructor: Hello
string constructor: World
string constructor: !
string constructor: -
string copy constructor: Hello
string move constructor: Hello-
string destructor
string move constructor: Hello-World
string move constructor: Hello-World!
string move constructor: Hello-World!
string destructor
string destructor
string destructor
string destructor
string destructor: -
Hello-World!
string destructor: !
string destructor: World
string destructor: Hello
string destructor: Hello-World!
```
对比之前的结果，我们可以看到多了一次`move constructor`，并且中间的构造、析构过程也有些不同。原因是`fnresult<string>`和`string`对象之间的赋值由于类型不同而不会触发RVO。

Mojo方案确实可以完美的解决返回值的问题，但使用起来还是比较复杂的，并且相同动作的“移动”构造函数可能需要定义多个。我们可以忽略掉一些复杂的情况，比如不去试图区分参数的“all const objects”和“non-const lvalues”，专注于解决“临时值”，即右值的检测和传递。

首先，我们定义`rvalue`类模板，来标记“右值”这一概念：
```cpp
template <typename T>
class rvalue : public T {
public:
    rvalue(rvalue const & t)
        : T(t) {}

    T& get() const {
        return *const_cast<rvalue*>(this);
    }
};
```
对应的，在`string`的定义中添加针对“右值”引用的“移动”构造函数，并修改`operator+`：
```cpp
class string {
    // ......

    string(rvalue<string> const & other)
        : data_(NULL), len_(other.len_) {
        swap(other.get());
        printf("string move constructor: %s\n", data_);
    }

    operator rvalue<string> const & () const {
        return static_cast<rvalue<string> const &>(*this);
    }

    // ......
};

rvalue<string> const operator+(string const & x, string const & y) {
    return string(x) += y;
}

rvalue<string> const operator+(rvalue<string> const & x, string const & y) {
    return x.get() += y;
}
```
现在，在新的`operator+`里，`string(x) += y`的返回值会被传递给`rvalue(T const & t)`，并被`string(rvalue<string> const & other)`接收，完成移动语义；

对于拷贝构造，则由拷贝构造函数`string(string const & other)`完成；

而用于接收“右值”引用`rvalue<string> const &`的`operator+`，则在处理完`+=`操作后直接返回参数的引用。

结果如下：
```
string constructor
string constructor: Hello
string constructor: World
string constructor: !
string constructor: -
string copy constructor: Hello
string move constructor: Hello-
string destructor
string move constructor: Hello-World
string move constructor: Hello-World!
string move constructor: Hello-World!
string destructor
string destructor
string destructor
string destructor
string destructor: -
Hello-World!
string destructor: !
string destructor: World
string destructor: Hello
string destructor: Hello-World!
```

参考：
1. [Move Constructors | Dr Dobb's](https://www.drdobbs.com/move-constructors/184403855)

## 3. 右值引用和移动语义

