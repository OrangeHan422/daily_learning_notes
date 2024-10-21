# C++TemplatesV2

> 个人学习笔记总结，感谢[大佬](https://github.com/xiaoweiChen/Cpp-Templates-2nd)翻译。

## 第一部分：基础知识

### 第一章：函数模板

#### 1.1 初识函数模板

##### 1.1.1 定义模板

下面是一个函数模板，返回两个数的最大值：

```c++
template<typename T>
T max(T a,T b){
    return b < a ? a :b;
}
```

关键字typename引入了一个*类型参数*，这是C++程序中最常见的模板参数类型，也可以使用其他参数。

> C++17之前，类型T也必须可复制才能传递参数。C++17以后，即使复制构造函数和移动构造函数都无效，也可以传递临时变量(右值)

在C++98之前，并没有引入typename关键字，所以使用关键字class定义类型参数是一样的：

```c++
template<class T>
T max(T a,T b){
    return b < a ? a :b;
}
```

推荐使用typename，因为这里的class并非C++中的类，容易引起误会。

##### 1.1.2 使用模板

模板不会编译成处理任意类型的实体。相反，对于模板使用的每个类型，会根据模板生成不同的实体。

用具体类型替换模板参数的过程称为实例化，会产生一个模板实例。

> OOP中，实例以及实例化用于类的具体对象。本书针对模板，仅用这两个术语表示对模板的使用。

如果代码有效，void也是有效的模板参数

```c++
template<typename T>
T foo(T*){
}
void* vp = nullptr;
foo(vp);//ok:deduces void foo(void*)
```

##### 1.1.3 两阶段翻译

模板“编译”分为两个阶段：

1.若在定义时不进行实例化，则会检查模板代码的正确性，而忽略模板参数。包括：

+ 语法错误，如：缺少分号
+ 使用位置名称的模板参数（类型名、函数名……）
+ 检查不依赖于模板参数的静态断言(assert)

2.实例化时，再次检查模板代码，确保生成的代码有效。特别是所有依赖于模板参数的部分，会进行重复检查。

```c++
template<typename T>
void foo(T t){
    undeclared(); // 若undeclared未知，则在第一阶段编译时报错
    undeclared(t);// 若undeclared(T)未知，则在第二阶段编译时报错。即，如果不调用foo，这里不会报错
    static_assert(sizeof(int) > 10,"int too small");//若sizeof(int)<=10，始终断言失败
    static_assert(sizeof(T) > 10,"T too small");//若实例T大小小于等于10，则断言失败
}
```

#### 1.2 模板参数推导

在进行模板参数推导时，需要记住，T只是类型的“一部分”。上例中，如果使用常量引用：

```c++
template<typename T>
T max(const T& a,const T& b){
    return b < a ? a : b;
}
```

此时传递int时，函数的形参为const int &

**类型推导时的类型转换**

+ 当通过引用声明参数时，简单的转换也不适用于类型推导。用同一个模板参数T声明的两个实参必须完全匹配。
+ 当按值声明参数时，只支持简单的转换：忽略const或者volatile限定符（CV限定），引用转换为引用的类型，原始数组或函数转换为相应的指针类型。对于使用相同模板参数T声明的两个参数，转换类型必须匹配

```c++
template<typename T>
T max(T a,T b);

int i = 17;
const int c = 42;
max(i,c); // ok:T-->int
max(c,c); // ok:T--->int
int& ir = i;
max(i,ir); // ok:T--->int
int arr[4];
max(&i,arr); // ok:T--->int*
    
max(4,7.2); //error:T类型不一致,int or double?
std::string s;
max("hello",s); // error:const char* or std::string?
```

对于上述错误可以通过一下方式解决：

```c++
max(static_cast<double>(4),7.2); // ok
max<double>(4,7.2);
```

**默认参数的类型推导**

类型推导不适用于默认调用参数。如：

```c++
template<typename T>
void f(T = ""){

}

f(1); // ok:推导为int
f();  // error:未知T类型

//如果需要支持默认参数，必须为  模板形参 也指定一个默认实参
template<typename T = std::string>
void f(T = ""){

}

f(1); // ok:T--->int 
f();  // ok:T--->std::string
```

#### 1.3 多模板参数

函数模板可以有不同的类型参数：

```c++
template<typename T1,typename T2>
T1 max(T1 a,T2 b){
    return b < a ? a : b;
}

auto m = ::max(4,6.2);
```

需要注意的是，该例中返回类型设置为T1，也就是说max(4,7.2)返回的是7，而max(7.2,4)返回的是7.2

对于这个问题，C++提供的解决方案如下：

+ 为返回类型引入第三个模板参数
+ 让编译器找出返回类型
+ 将返回类型声明为两个参数类型的“公共类型”

##### 1.3.1 返回类型的模板参数

C++无法推导函数的返回类型：

```c++
template<typename RT,typename T1,typename T2>
RT max(T1 a,T2 b){
    std::cout << typeid(a).name() << ":" << typeid(b).name << std::endl;
    return T1();
}

auto dood = ::max(2,5); // error,unable to deduce RT
auto dood = ::max<int>(2,5); // ok:explicit set RT 2 int
```

##### 1.3.2 推导返回类型

若返回类型依赖于模板参数，最好和简单的方法是让编译器判断返回类型。从C++14开始，可以不显式声明返回类型（需要声明返回类型为auto）

```c++
template<typename T1,typename T2>
auto max(T1 a,T2 b){
    std::cout << typeid(a).name() << ":" << typeid(b).name() << std::endl;
    return T1();
}
auto dood = ::max(2,5); // ok : auto--->int
```

而在C++14之前，只能让编译器通过将函数的实现作为其声明的一部分来确定返回类型。C++11中，尾部的返回类型允许使用调用时的参数，声明的返回类型是从三元操作符某个分支派生的：

```c++
template<typename T1,typename T2>
auto max(T1 a, T2 b) -> decltype(b < a ? a : b)
{
    return b < a ? a : b;
}
auto dood = ::max(2,5); // ok : auto--->int
```

注意：

```c++
template<typename T1,typename T2>
auto max(T1 a, T2 b) -> decltype(b < a ? a : b);
```

是一个声明。

实际上，编译器使用这里的三元操作符会产生简答的结果，即：若类型不一致，则返回值会确定为一个通用的类型。

所以，这里条件无所谓，即使为true也可以

```c++
template<typename T1,typename T2>
auto max(T1 a, T2 b) -> decltype(true ? a : b)
```

但是这个定义有一个缺陷：返回类型可能是引用类型，在某些条件下T可能是引用。因为这个原因，应该返回T衰变的类型：（某些条件？什么条件，有时候觉得这些大佬就太想当然了，也不举个例子，就这么过去了？为什么要用衰变？什么情况下要考虑使用衰变？我\*\*真的\*\*\*\*）

```c++
#include <type_traits>
template<typename T1,typename T2>
auto max(T1 a, T2 b) -> typename std::decay<decltype(false? a : b)>::type
{
    return b < a ? a : b;
}
```

>  通过Effective Mordern C++可知，C++14 auto会自动去除CV以及引用限定，所以建议直接使用auto。如果必须使用C++11，需要确保返回类型为值返回时，就使用std::decay，至于为啥，没为啥，自己没用过，作者也不给你说为啥，就死记住吧。

##### 1.3.3 返回类型为公共类型

C++11后，标准库提供了一种方法来指定选择“公共类型”。`std::common_type<>::type`或产生由两个或者多个不同类型作为模板类型的公共类型。例如：

```c++
template<typename T1,typename T2>
std::common_type_t<T1,T2> max(T1 a, T2 b){
    return b < a ? a : b;
}
```

`std::common_type`在`<type_traits>`头文件中定义，其生成一个具有返回类型的结构体。核心用法如下：

```c++
typename std::common_type<T1,T2>::type // since C++11
```

C++14起，可以通过在特性后面加`_t`跳过`typename`和`::type`来简化`trait`的使用（详见2.8节），这样返回类型定义就变成了：

```c++
std::common_type_t<T1,T2> // since C++14
```

`std::common_type<>`使用了一些编程技巧，会在第26.5.2节讨论。注意`std::common_type<>`也会衰变，详见D.5节

#### 1.4 默认模板参数

> C++11前，默认模板参数只允许在类模板中使用

使用返回类型作为例子：

1、可以直接使用三元操作符

```c++
//C++14
template<typename T1,typename T2,
typename RT = std::decay_t<decltype(true ? T1() : T2())>> //注意 std::decay_t<> 的用法以确保返回的不是引用类型。
RT max(T1 a,T2 b){
    return b < a ? a : b;
}
//C++11
template<typename T1,typename T2,
typename RT = typename std::decay<decltype(true ? T1() : T2())>::type> //注意 std::decay_t<> 的用法以确保返回的不是引用类型。
RT max(T1 a,T2 b){
    return b < a ? a : b;
}
```

2、使用`std::common_type`

```c++
//C++14
template<typename T1,typename T2,
typename RT = std::common_type_t<T1,T2>> //注意common_type的衰变，无需再使用std::decay_t
RT max(T1 a,T2 b){
    return b < a ? a : b;
}
//C++11
template<typename T1,typename T2,
typename RT = typename std::common_type<T1,T2>::type>
RT max(T1 a,T2 b){
    return b < a ? a : b;
}
```

在使用`max<type,type,type>`时，默认参数不必遵守函数形参的规则，即，拥有默认参数的模板形参可以放在最前面，这样在仅需要指定返回值类型的时候更加方便：

```c++
template<typename RT = long,typename T1,typename T2>
RT max(T1 a,T2 b);

max<double>(1,2); // RT--->double
```

#### 1.5 重载函数模板

```c++
int max(int,int);
template <typename T>
T max(T,T);

int main() {
    ::max(1,2);         //calls the nontemplate for two ints
    ::max(1.0,2.0);     //calls max<double>
    ::max('a','b');     //calls max<char>
    ::max<>(1,2);       //calls max<int>
    ::max<double>(1,2); //call max<double>
    ::max('a',2.0);       //call the nontemplate for two ints
    return 0;
}
```

非模板函数可以与相同名称和相同类型的函数模板共存。重载解析将优先使用非模板方式。如`::max(1,2)`

若模板可以生成更准确的函数，则选择模板。如`::max(1.0,2.0)`

需要推导的模板参数不会进行自动类型转换，所以会自动的选择普通函数进行自动类型转换。如`::max('a',2.0)`

有趣的例子是重载模板：

```c++
template<typename T1,typename T2>
auto max(T1 a,T2 b);

template<typename RT,typename T1,typename T2>
RT max(T1 a,T2 b);

int main() {
    auto a = ::max(1,2.0);  //使用第一个
    auto b = ::max<long,double>(2.0,1);//使用第二个
    auto c = ::max<int>(1,2.0);//error,两个都可以
    return 0;
}
```

一个重载指针和普通C字符串的比较最大值的函数模板：

```c++
template<typename T>
T max(T a,T b);

template<typename T>
T* max(T* a,T* b);

const char* max(const char* a,const char* b);

int main() {
    int a = 1;
    int b = 2;
    auto m1 = ::max(a,b); //max(T,T)

    std::string s1{"hey"};
    std::string s2{"you"};
    auto m2 = ::max(s1,s2); // max(T,T)

    int* p1 = &b;
    int* p2 = &a;
    auto m3 = ::max(p1,p2); //max(T*,T*)

    const char* x = "hello";
    const char* y = "world";
    auto m4 = ::max(x,y); // max(const char*,const char*)
    return 0;
}
```

重载函数模板时，最好不要进行不必要的更改，可以将更改限制在更改参数的数量或者显示地指定模板参数。否则可能会出现意想不到的后果。

需要再函数调用之前声明函数的所有重载版本。因为在进行相应的函数调用时，并非所有的函数都可见。如：

```c++
template<typename T>
T max(T a,T b){
    std::cout << "max<T>()\n";
    return b < a ? a : b;
}

template<typename T>
T max(T a,T b,T c){
    return max(max(a,b),c); //此时会对int,int也调用模板,因为在编译时,看不到下面的实例.
}

int max(int a,int b){
    std::cout << "max(int,int)\n";
    return b < a ? a : b;
}
```

### 第二章：类模板