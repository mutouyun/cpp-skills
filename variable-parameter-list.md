# 任意个数的参数传递

## 1. 继承自C语言的可变参数列表

大家一定都用过`printf`，这是C语言里常用的格式化输出函数。这个函数的定义看起来像这样：

```cpp
int printf(const char * format, ...);
```

函数的第二个参数“`...`”，即为可变参数列表（Variable Parameter List）。C++也继承了C的这一特征，它可以让一个函数接受任意多个参数，但同时也具有很多不便和限制。

例如，我们定义一个计算多个数字相加的函数`sum`，使用可变参数列表的定义方式如下：

```cpp
#include <iostream>
#include <stdarg.h>

int sum(int num, ...)
{
    /* 定义参数列表 */
    va_list valist;

    /* 为 num 个参数初始化 valist */
    va_start(valist, num);

    /* 访问所有赋给 valist 的参数 */
    int ret = 0;
    for (int i = 0; i < num; ++i)
    {
        ret += va_arg(valist, int);
    }

    /* 清理内存 */
    va_end(valist);

    return ret;
}

int main(void)
{
    std::cout << "2 + 3 = " << sum(2, 2, 3) << std::endl;
    std::cout << "2 + 3 + 4 = " << sum(3, 2, 3, 4) << std::endl;
}
```

可以看出，C/C++的可变参数列表有着如下一些限制：

1. 可变参数列表之前，至少需要一个普通参数；
2. 仅通过可变参数列表自身，无法得到可变参数的个数。比如上面的例子里，我们必须在第一个参数`num`中写入正确的变参个数；
3. 变参传入后无法知道参数的类型。

上面的`sum`，如果写成这样：

```cpp
int sum(...) { /* ... */ }
```

那么通过可变参数列表传递的内容就无法正常的获取了。同样的，参数传递的个数虽然是任意的，但参数的类型却被限制为`int`，若传入的参数类型不同，`sum`就有可能出错。

由于上面的这三点限制，C/C++中的可变参数列表应用并不广泛。众所周知的一个经典应用就是C语言里的`printf`函数了。

到了C++里，考虑到类型安全，以及自动的参数类型解析和匹配，C++用`cout`对象代替了`printf`，可变参数列表的存在感进一步减弱了。甚至到了后来，可变参数列表变成了泛型和模板元编程中运用SFINAE的一种工具和手段，而不是实现一个变参函数。

## 2. 可变参数列表的改进和替代

针对可变参数列表的限制和劣势，我们有多种手段对其进行改进；同时，也有不少替代方案来实现任意个数的参数传递。

### 2.1 使用宏（macro）简化代码

我们可以通过一个宏对常规的可变参数列表进行包装：

```cpp
// Circumvent MSVC __VA_ARGS__ BUG
// See: https://stackoverflow.com/questions/5134523/msvc-doesnt-expand-va-args-correctly
#define PP_VA(...) __VA_ARGS__ /* Expand __VA_ARGS__ */

#define PP_FILTER(   _1,  _2,  _3,  _4,  _5,  _6,  _7,  _8,  _9,  _10, \
                     _11, _12, _13, _14, _15, _16, _17, _18, _19, _20, \
                     _N, ...) _N
#define PP_NUMBER()  20,  19,  18,  17,  16,  15,  14,  13,  12,  11, \
                     10,   9,   8,   7,   6,   5,   4,   3,   2,   1

#define PP_HELPER(...) PP_VA(PP_FILTER(__VA_ARGS__))
#define PP_COUNT(...)  PP_HELPER(__VA_ARGS__, PP_NUMBER())
#define PP_SUM(...)    sum(PP_COUNT(__VA_ARGS__), __VA_ARGS__)

int sum(int num, ...)
{
    // ...
}

int main(void)
{
    std::cout << "2 + 3 = " << PP_SUM(2, 3) << std::endl;
    std::cout << "2 + 3 + 4 = " << PP_SUM(2, 3, 4) << std::endl;
}
```

如上，`PP_SUM`具备了自动计算参数个数的能力，使用起来也稍显友好。缺点则是我们不得不加上一堆晦涩难懂的宏作为函数`sum`的包装；并且参数的个数计算也被限制在20以内。

一般来说，20个参数已经足够了。如果希望一次传递更多参数，可以调整`PP_FILTER`和`PP_NUMBER`宏来实现更多的参数支持。但不得不说，存在上限的“任意”个数总让人心里不舒坦；更何况此方法仍然无法解决参数类型的限制问题。

### 2.2 使用函数或操作符重载

在C++中，我们更倾向于希望参数在传递过程中保留它的一切属性。很显然，单纯的可变参数列表丢失了太多东西；而类似`printf`那样使用一个小的子语言（format specifiers）来定义参数的类型和细节，则无疑让使用者负担了额外的工作，并且也难以对类型的正确性做静态或动态检查。

因此，C++的做法是使用一些特殊的操作符重载，结合对象自身的状态存储，来完成任意个数的参数传递。

如C++中`printf`的替代对象`cout`：

```cpp
#include <iostream>

int main(void)
{
    std::cout << "2 + 3 = " << 5 << std::endl;
    std::cout << "2 + 3 + 4 = " << 9 << std::endl;
}
```

实际上，`cout`是一个全局变量，定义在`iostream`中，一般像这样：

```cpp
typedef basic_ostream<char, char_traits<char> > ostream;
extern ostream cout;
```

而真正赋予它使用`<<`操作符输入内容的，则是一组`operator<<`重载：

```cpp
template <...>
class basic_ostream : ...
{
public:
    basic_ostream& operator<<(bool _Val);
    basic_ostream& operator<<(short _Val);
    basic_ostream& operator<<(unsigned short _Val);
    basic_ostream& operator<<(int _Val);
    basic_ostream& operator<<(unsigned int _Val);
    ...
};
```

我们可以用`operator()`来实现类似的功能：

```cpp
#include <stdio.h>

class Foo
{
public:
    template <typename T>
    explicit Foo(T a)
    {
        (*this)(a);
    }

    Foo& operator()(int a)
    {
        printf("%d", a);
        return (*this);
    }

    Foo& operator()(char c)
    {
        printf("%c", c);
        return (*this);
    }
};

int main()
{
   Foo(1)(2)('3');
   return 0;
}
```

输出：

```bash
123
```

上面的代码除了使用了`operator()`，同时还用了`Foo`的构造函数来模拟第一个`operator()`。

这种做法的关键在于函数返回了类对象自身的引用，因此无所谓是否是操作符重载，普通成员函数也是可以的。只是操作符重载一般看起来更简洁些。使用这个技巧，我们可以通过重载，或者template在编译期萃取出参数的类型特征，进行有针对性的算法选择。

Qt库的`QString`就使用了一组成员函数`arg`的重载来实现字符串的格式化功能（[QString Class \| Qt Core 5.11](http://doc.qt.io/qt-5/qstring.html#arg-1)）：

```cpp
QString i;           // current file's number
QString total;       // number of files to process
QString fileName;    // current file's name

QString status = QString("Processing file %1 of %2: %3")
                .arg(i).arg(total).arg(fileName);

QString str;
str = QString("Decimal 63 is %1 in hexadecimal")
        .arg(63, 0, 16);
// str == "Decimal 63 is 3f in hexadecimal"

QLocale::setDefault(QLocale(QLocale::English, QLocale::UnitedStates));
str = QString("%1 %L2 %L3")
        .arg(12345)
        .arg(12345)
        .arg(12345, 0, 16);
// str == "12345 12,345 3039"
```

### 2.3 多参数的断言（`assert`）宏

成员函数返回对象自身引用的技巧结合宏，可以实现一些好玩且实用的功能。比如我们都使用过`assert`函数或宏，其简单的实现类似这样：

```cpp
#include <iostream>
using namespace std;

#if !defined(NDBUG)
#define assert(p) do { if (!(p)) { \
    fprintf(stderr, \
            "Assertion failed: %s\nfile: %s, line: %d\n", \
            #p, __FILE__, __LINE__); \
    abort(); \
} } while(0)
#else
#define assert(p)
#endif

int main()
{
    assert(false);
    cout << "Hello World";
    return 0;
}
```

输出：

```bash
Assertion failed: false
file: /tmp/340419142/main.cpp, line: 17

signal: aborted (core dumped)
```

常规的`assert`通常只能用来判断一个条件。如果同时判断多个条件，由于传递的表达式只有一条，断言失败，我们将无法直观的判断出是哪个条件失败而引发的断言。

《Modern C++ Design》的作者Andrei Alexandrescu曾经在[Dr Dobb's](http://www.drdobbs.com/)上发布了一篇文章《[Enhancing Assertions](http://www.drdobbs.com/cpp/enhancing-assertions/184403745)》，其中使用了一个很好玩的宏技巧来实现可传递任意个数条件的断言宏。

首先，他定义了`Assert`类如下：

```cpp
class Assert
{
    /*...*/
public:
    Assert& SMART_ASSERT_A;
    Assert& SMART_ASSERT_B;
    // whatever member functions
    Assert& print_current_val(bool, const char*);
    /*...*/
};
```

这里除了能返回自身引用的成员函数`print_current_val`之外，还额外定义了两个public的成员变量`SMART_ASSERT_A`和`SMART_ASSERT_B`，并且在初始化时让它们指向自身（`*this`）。

之后，再定义一组宏如下：

```cpp
#define SMART_ASSERT_A(x) SMART_ASSERT_OP(x, B)
#define SMART_ASSERT_B(x) SMART_ASSERT_OP(x, A)
#define SMART_ASSERT_OP(x, next) \
        SMART_ASSERT_A.print_current_val((x), #x).SMART_ASSERT_ ## next
```

然后`SMART_ASSERT`宏的实现像这样：

```cpp
#define SMART_ASSERT(expr) \
        if ( (expr) ) ; \
        else make_assert(#expr).print_context(__FILE__, __LINE__).SMART_ASSERT_A
```

`make_assert`会返回一个`Assert`对象，其中包含了`#expr`表达式内容的字符串；成员函数`print_context`负责打印当前断言的上下文。最后剩下`SMART_ASSERT_A`，这也是最关键的地方。

若我们像普通的`assert`一样使用这个`SMART_ASSERT`：

```cpp
SMART_ASSERT(ptr != nullptr);
```

此时`SMART_ASSERT_A`为临时`Assert`对象的public成员`SMART_ASSERT_A`，正常编译且没有任何其它作用。

若我们像这样用：

```cpp
SMART_ASSERT(ptr != nullptr && a == 0 && b == 0)(a)(b);
```

那么`SMART_ASSERT_A`会作为宏，被展开成`SMART_ASSERT_OP(a, B)`，之后`SMART_ASSERT_OP`再次展开，并继续处理`(b)`……完整展开后的代码看起来像这样：

```cpp
SMART_ASSERT(ptr != nullptr && a == 0 && b == 0)(a)(b);
=>
if ( (ptr != nullptr && a == 0 && b == 0) ) ;
else make_assert("ptr != nullptr && a == 0 && b == 0").print_context(__FILE__, __LINE__)
    .SMART_ASSERT_A.print_current_val((a), "a")
    .SMART_ASSERT_B.print_current_val((b), "b").SMART_ASSERT_A;
```

这样，`print_current_val`配合两组宏的交替展开，`SMART_ASSERT`实现了任意多个条件同时断言，并且可以分别打印出其中每一个条件的表达式和内容。

### 2.4 使用宏元编程（Macro Metaprogramming）

有人说模板元编程（Template Metaprogramming）是C++中的黑魔法，不过在我看来，宏元编程要比模板元邪恶得多。之前2.1节中定义的宏`PP_COUNT`就是一个宏元编程的例子，大量的鬼畜代码是宏元编程的特点之一。

在这里我并不打算介绍如何进行宏元编程。本节的内容只是一个示例，为了让大家了解到在C++11之前，我们是如何挣扎着实现类似的功能的。

宏元编程的主要目的，是用来自动生成大量的重复代码。如果我们能使用它生成足够多的不同参数的重载函数，那么就可以间接实现“任意”个数参数的支持了。

比如尝试用宏来生成如下代码：

```cpp
template <typename T1>
void func(T1 p1) { std::cout << p1; }

template <typename T1, typename T2>
void func(T1 p1, T2 p2) { std::cout << p1 << p2; }

template <typename T1, typename T2, typename T3>
void func(T1 p1, T2 p2, T3 p3) { std::cout << p1 << p2 << p3; }
```

首先，我们把这段代码里相同的模式抽出来：

* 包含n的重复元素：`typename Tn`、`<< pn`、`Tn pn`
* 传递n的重复元素：`func`的实现

针对包含n的重复部分，定义`PP_REPEAT`模板元：

```cpp
// Expand __VA_ARGS__
#define PP_VA(...)          __VA_ARGS__
#define PP_PROXY(f, ...)    PP_VA(f(__VA_ARGS__))

// Connect two args together
#define PP_CAT(x, ...)      x##__VA_ARGS__
#define PP_JOIN_(x, ...)    PP_CAT(x, __VA_ARGS__)
#define PP_JOIN(x, ...)     PP_JOIN_(x, __VA_ARGS__)

/*
    PP_REPEAT(3, f)
    -->
    f(1) f(2) f(3)
*/

#define PP_REPEAT_0(f1, f2)
#define PP_REPEAT_1(f1, f2) PP_VA(f1(1))
#define PP_REPEAT_2(f1, f2) PP_REPEAT_1(f1, f2) PP_VA(f2(2))
#define PP_REPEAT_3(f1, f2) PP_REPEAT_2(f1, f2) PP_VA(f2(3))
/*...*/

#define PP_REPEATEX(n, f1, f2) PP_PROXY(PP_JOIN(PP_REPEAT_, n), f1, f2)
#define PP_REPEAT(n, f)        PP_REPEATEX(n, f, f)
```

使用MSVC的 `/P` 或gcc的 `-E -P` 参数可以测试宏的展开结果：

```cpp
#define TYPENAME_1(n)   typename T##n
#define TYPENAME_2(n) , typename T##n
// typename T1 , typename T2 , typename T3
PP_REPEATEX(3, TYPENAME_1, TYPENAME_2)

#define OUTPUT(n) << p##n
// << p1 << p2 << p3
PP_REPEAT(3, OUTPUT)

#define PARAM_1(n)   T##n p##n
#define PARAM_2(n) , T##n p##n
// T1 p1 , T2 p2 , T3 p3
PP_REPEATEX(3, PARAM_1, PARAM_2)
```

下面我们来重复生成`func`。这里必须定义一个新的宏`PP_LOOP`，而不能直接使用`PP_REPEAT`，因为宏不支持递归调用：

```cpp
#define PP_LOOP_0(f)
#define PP_LOOP_1(f) f(1)
#define PP_LOOP_2(f) PP_LOOP_1(f) f(2)
#define PP_LOOP_3(f) PP_LOOP_2(f) f(3)
/*...*/

#define FUNC(n, ...) \
    template <PP_REPEATEX(n, TYPENAME_1, TYPENAME_2)> \
    void func(PP_REPEATEX(n, PARAM_1, PARAM_2)) { std::cout PP_REPEAT(n, OUTPUT); }

/*
    template <typename T1> void func(T1 p1) { std::cout << p1; }
    template <typename T1 , typename T2> void func(T1 p1 , T2 p2) { std::cout << p1 << p2; }
    template <typename T1 , typename T2 , typename T3> void func(T1 p1 , T2 p2 , T3 p3) { std::cout << p1 << p2 << p3; }
*/
PP_LOOP_3(FUNC)
```

这样，我们多定义几个`PP_REPEAT_N`和`PP_LOOP_N`，让宏能够支持更大的`n`，即可任意增加支持的参数个数了。但要注意，编译器对宏的嵌套层数是有限的，并且过大的`n`还会导致预处理时间过长。

## 3. C++11的可变参数模板（Variadic Templates）

从C++11开始，我们有了可变参数模板。类模板示例如下：

```cpp
template <typename... Args>
class Foo { /*...*/ };
```

函数模板示例如下：

```cpp
template <typename... Args>
void func(Args... params) { printf(params...); }
```

其中`typename... Args`被称为模板参数包（Template parameter pack），`Args... params`被称为函数参数包（Function parameter pack），`params...`被称为模板参数展开（Parameter pack expansion）；另外，我们可以使用[`sizeof...`](https://en.cppreference.com/w/cpp/language/sizeof...)操作符得到参数包里的参数个数。

有了`template`，我们自然可以完整的保留参数的类型，限定符（qualifier）等信息，“`...`”也被赋予了新的含义。在C语言的可变参数列表里，我们需要通过`va_list`和`va_start`拿到参数列表的头部，并通过循环逐步获得列表的每一项内容。而可变参数模板则和它完全不同。`Args... params`被称为参数包，而非参数列表，因为`params`确实无法像列表一样通过循环来迭代每一项的内容；`params...`会一口气展开所有的参数。

我们来看一组例子（from [Parameter pack\(since C++11\) - cppreference.com](https://en.cppreference.com/w/cpp/language/parameter_pack)）：

```cpp
f(&args...); // expands to f(&E1, &E2, &E3)
f(n, ++args...); // expands to f(n, ++E1, ++E2, ++E3);
f(++args..., n); // expands to f(++E1, ++E2, ++E3, n);
f(const_cast<const Args*>(&args)...);
// f(const_cast<const E1*>(&X1), const_cast<const E2*>(&X2), const_cast<const E3*>(&X3))
f(h(args...) + args...); // expands to 
// f(h(E1,E2,E3) + E1, h(E1,E2,E3) + E2, h(E1,E2,E3) + E3)
```

### 3.1 可变参数模板的基本用法

我们先来讨论下可变参数函数模板：

```cpp
template <typename... Args>
void func(Args... params) { /*...*/ }
```

一般来说，`params...`是无法被直接展开成一个可供迭代的列表的。有种特殊的写法是这样：

```cpp
template <typename... Args>
void func(Args... params)
{
    for (auto v : { params... })
    {
        std::cout << v;
    }
}
```

`{ params... }`在C++中被称为braced-init-list，会在ranged-for中被转化为一个`std::initializer_list`，可以通过它将展开后的参数包逐项迭代。这种用法具有一些缺陷，`std::initializer_list`要求braced-init-list中的每一项参数必须具有相同的类型，不同类型的`params`参数包将无法通过编译。

那么通用的做法是什么呢？我们不妨先试着定义下面这个函数：

```cpp
template <typename Arg, typename... Args>
void func(Arg p1, Args... rest) { /*...*/ }
```

现在，我们有了一个能够将参数列表分为两部分的函数，其中第一部分是列表的第一个参数，而另一部分则是剩下的所有参数组成的参数包。当外部如`func(1, 2, 3, 4)`这样调用时，`p1`将等于1，`rest`则为`2, 3, 4`。

有了这个函数之后，事情变得简单起来。我们只需要通过递归不停地将第一个参数解出来，并给递归过程增加终止条件就好了：

```cpp
void func(void)
{
    // Do Nothing.
}

template <typename Arg, typename... Args>
void func(Arg p1, Args... rest)
{
    std::cout << p1;
    func(rest...);
}
```

当然，也可以用`void func(Arg p1)`来作为终止函数，这样相当于禁止无参调用`func`。这里的基本思想是：

1. 通过函数重载归纳参数包（代替`if-else`区分参数包的模式）
2. 通过递归展开参数包（代替`for/while`循环对参数进行逐项处理）

用这种方式写出来的代码有些类似用数学归纳法列递归式：首先在n取最小值时列出基础，之后对更大的n进行归纳。

有些时候，这种递归的写法是不大方便的。在不使用递归的情况下，能否在展开参数包的同时依次处理每个参数呢？答案是可以的，我们还是需要借助braced-init-list：

```cpp
template <typename... Args>
void func(Args... params)
{
    auto out = [](auto v) { std::cout << v; };
    using swallow = int[];
    (void)swallow{ (out(params), 0)... };
}
```

这里还用到了C++14的generic lambda，将`std::cout`的操作打包成了一个lambda对象`out`；之后构造了一个临时匿名数组，通过数组的列表初始化（list initialization）将参数包展开，并依次传入out求值。

这里有个细节是数组的初始化列表中不能直接使用`out(params)`，因为它的返回值是`void`，无法转换为`int`，所以我们在列表中使用的是`(out(params), 0)`，通过一个逗号表达式让每个子表达式的返回值为0。

我们同样可以用`std::initializer_list`来实现：

```cpp
template <typename... Args>
void func(Args... params)
{
    auto out = [](auto v) { std::cout << v; };
    std::initializer_list<int>{ (out(params), 0)... };
}
```

接下来我们看一个可变参数类模板的例子。可变参数类模板的经典应用应该算STL里的`tuple`了吧。`tuple`的定义一般长这样：

```cpp
template <typename... T>
class tuple;

template <>
class tuple<> { /*...*/ };

template <typename This, typename... Rest>
class tuple<This, Rest...>
    : private tuple<Rest...> // recursive tuple definition
{ /*...*/ };
```

可以看到，这里的思想和之前的可变参数函数模板是一致的，只是类模板是通过特化和偏特化对参数表进行归纳；同样，`tuple<This, Rest...>`继承自身的行为相当于我们之前`func`函数的递归调用。

### 3.2 计算任意个数的数字相加

现在我们可以回到开头的`sum`函数。如何用可变参数模板来计算任意个数的数字相加呢？

有了新的可变参数函数模板，结合之前的`func`示例应该很容易写出来：

```cpp
template <typename T>
T sum(T p1)
{
    return p1;
}

template <typename T, typename... Args>
T sum(T p1, Args... rest)
{
    return p1 + sum(rest...);
}
```

或者，使用`std::initializer_list`：

```cpp
template <typename T, typename... Args>
T sum_v2(T p1, Args... rest)
{
    T ret { p1 };
    // Type of t is std::initializer_list<T>
    auto t = { ret += rest... };
    return ret;
}

template <typename T, typename... Args>
T sum_v3(T p1, Args... rest)
{
    T ret { p1 };
    std::initializer_list<T>{ ret += rest... };
    return ret;
}
```

参数包会在braced-init-list中被展开，并依次执行所有的`+=`操作，它们的返回值类型都是`T`。当然，若参数类型不同，编译器可能会在类型收窄（Narrowing Conversions）时报warning。

接下来，我们尝试改进一下`sum`，让它可以接受一个高阶函数（Higher-order Function）作为算子，来代替加法。函数名称改为`seq`：

```cpp
template <typename F, typename T>
T seq(F, T p1)
{
    return p1;
}

template <typename F, typename T, typename... Args>
T seq(F f, T p1, Args... rest)
{
    return f(p1, seq(f, rest...));
}
```

这是通过`std::initializer_list`展开参数包的版本：

```cpp
template <typename F, typename T, typename... Args>
T seq_v2(F f, T p1, Args... rest)
{
    T ret { p1 };
    std::initializer_list<T>{ ret = f(ret, rest)... };
    return ret;
}
```

下面我们来测试一下这两个版本的计算结果：

```cpp
std::cout << seq   ([](auto a, auto b){ return a + b; }, 1, 2, 3, 4) << std::endl; // 10
std::cout << seq_v2([](auto a, auto b){ return a + b; }, 1, 2, 3, 4) << std::endl; // 10
std::cout << seq   ([](auto a, auto b){ return a - b; }, 1, 2, 3, 4) << std::endl; // -2
std::cout << seq_v2([](auto a, auto b){ return a - b; }, 1, 2, 3, 4) << std::endl; // -8
```

有意思的事情出现了，`seq`和`seq_v2`对减法的计算结果不一致。这是因为减法不满足交换律，对 $$a - b$$ 来说，`seq`展开后的式子如下

$$
\begin{aligned}
seq(f,p_1,p_2,p_3,\cdots,p_n)&=f(p_1,f(p_2,f(p_3,\cdots f(p_{n-1},p_n)\cdots)))\\
&=p_1-(p_2-(p_3-\cdots(p_{n-1}-p_n)\cdots))\\
&=p_1-p_2+p_3-p_4+\cdots
\end{aligned}
$$

而`seq_v2`则是这样

$$
\begin{aligned}
seq_{v2}(f,p_1,p_2,p_3,\cdots,p_n)&=f(\cdots f(f(p_1,p_2),p_3),\cdots,p_n)\\
&=(\cdots((p_1-p_2)-p_3)\cdots-p_n)\\
&=p_1-p_2-p_3-\cdots-p_n
\end{aligned}
$$

所以可以看出来，`seq_v2`的结果才是我们想要的。

### 3.3 C++17的折叠表达式（Fold Expression）

