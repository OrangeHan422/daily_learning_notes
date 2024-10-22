# C++TemplatesV2

> 个人学习笔记总结，感谢[大佬](https://github.com/xiaoweiChen/Cpp-Templates-2nd)翻译。
>
> 2024-10-22：感谢译者提供翻译，但是翻译的并不适合个人学习，所以只能硬着头皮啃原版了，所以，可能不算笔记了，又是一份翻译。
>
> > eg:2.4节原文：However, when trying to *declare* the friend function and *define* it afterwards, things become more complicated.
> >
> > 而翻译则是：
> >
> > 但当试图声明友元函数并实现时，事情会变得更加复杂
> >
> > 
> >
> > 个人认为，这是完全错误的，afterwards这么关键的词语竟然忽略了。对于技术类书籍，个人认为这是难以忍受的。正确的翻译应该为：
> >
> > 然而，当我们想要声明一个友元函数，并在之后定义该函数，事情就会变的复杂。
> >
> > 
> >
> > 意思为如果将函数的声明和定义分开，问题会变的棘手。而非译文的声明友元并实现。后面的`forward declare`翻译的是“转发声明”，个人理解应该是“前向声明”，而不是翻译为`std::forward`的中文术语“转发”。开始读翻译版可把我搞蒙了，满脑子都是《人在囧途》王宝强的“啥，啥，啥，这写类都是啥”。
>
> TODO：添加C++20模板知识，尤其是概念和约束

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

#### 2.1 实现stack类模板

```c++
#include <vector>
#include <cassert>

template<typename T>
class Stack{
private:
    std::vector<T> elems_;

public:
    void push(const T& elem);
    void pop();
    const T& top() const;
    bool empty() const{
        return elems_.empty();
    }
};

template<typename T>
void Stack<T>::push(const T& elem){
    elems_.push_back(elem);
}

template<typename T>
void Stack<T>::pop(){
    assert(!elems_.empty());
    elems_.pop_back();
}

template<typename T>
const T& Stack<T>::top() const{
    assert(!elems_.empty());
    return elems_.back(); // return copy of last element
}
```

##### 2.1.1 声明类模板

声明类模板类似于声明函数模板。声明钱，必须将一个或多个标识符声明为类型参数。

```c++
template<typename T>
class TemplateClass{
  ...  
};
```

类模板内部，可以像使用其他类型一样使用T来声明成员和成员函数

```c++
template<typename T>
class Stack{
private:
    std::vector<T> elems_;

public:
    void push(const T& elem);
    void pop();
    const T& top() const;
    bool empty() const{
        return elems_.empty();
    }
};
```

这个类的类型是`Stack<T>`,T是模板参数。因此，除了可以推导的模板参数，只要在使用该类的类型时，就必须使用`Stack<T>`声明。如果在类模板中使用类名，但是不带模板参数，表明这个内部类的模板参数类型和模板类的参数类型相同。（详见13.2.3节）

例如，必须声明自定义的复制构造函数和赋值操作符：

```c++
template<typename T>
class Stack{
  Stack(const Stack&);//copy ctor
  Stack& operator=(const Stack&);//assigenment operator
};
//等价与
template<typename T>
class Stack{
  Stack(const Stack<T>&);//copy ctor,注意，在需要类名而非类类型的地方，只能使用Stack。尤其是构造和析构函数名称
  Stack<T>& operator=(const Stack<T>&);//assigenment operator
};
```

##### 2.1.2 成员函数

要定义类模板的成员函数，必须将其指定为模板，并且使用类模板对类型进行限定。如：

```c++
template<typename T>
void Stack<T>::push(const T& elem){ //注意Stack<T>
    elems_.push_back(elem);
}
```

#### 2.2 使用Stack类模板

C++17之前，要使用类模板的对象，必须显示指定模板参数。

> C++17引入类参数模板推导，若模板参数可以从构造函数派生，则可以跳过这些参数。具体在2.9节讨论。

```c++
int main() {
    Stack<std::string> s;
    s.push("test");
    std::cout << s.top() << std::endl;
    s.pop();
    return 0;
}
```

注意，代码支队调用的模板函数实例化。对于类模板，只有在成员函数被使用时才实例化。这样做可以节省时间和空间，细节将会在2.3节讨论。

可以像使用其他类型一样使用实例化的类模板实例。

```c++
void foo(const Stack<int>& s){
    using IntStack = Stack<int>;
    Stack<int> istack[10]; // istack is array of 10 int statcks
    IntStack istack2[10]; // istack2 is also an array of 10 int stacks
}
```

#### 2.3 部分使用Stack类模板

翻译的版本看不懂，实际上就是模板的实例化仅在使用的时候进行。比如在类成员函数中对模板参数T使用了`<<`运算符。如果自己实例化的T不支持该操作符，那么只要不使用这个成员函数就不会报错。如：

```c++
template<typename T>
class Stack{
public:
      void printOn(std::ostream& strm) const{
        for(const T& elem:elems_){
            strm << elem << ' ';
        }
    }  
};

Stack<std::pair<int,int>> pair_stack;//ok
pair_stack.printOn(std::cout);//error
```

##### 2.3.1 概念

“概念”通常用来表示模板库中需要的约束。C++17中，概念只能通过文字进行表达（如代码注释）。这可能导致一个严重的问题，因为不遵守约束可能会导致大量的错误消息。

C++20开始，将概念定义和验证变为了语言特性。

在这之前，C++11之后，可以通过`static_assert`和预定义类型特征来检查约束、如：

```c++
//C++11
template<typename T>
class C{
    static_assert(std::is_default_constructible<T>::value,
                  "Class C requires default-constructible elements");
};
//C++14
template<typename T>
class C{
        static_assert(std::is_default_constructible_v<T>,
    "Class C requires default-constructible elements");
};
```

但是，这样的错误输出往往是令人绝望的。

#### 2.4 友元

使用printOn()打印堆栈内容，需要为堆栈实现<<操作符。通常<<操作符实现为非成员函数，然后内联调用printOn();

```c++
template<typename T>
class Stack{
public:
    void printOn(std::ostream& strm) const{
      ...
    }  
    friend std::ostream& operator<<(std::ostream& strm,const Stack<T>& s){
        s.printOn(strm);
        return strm;
    }
};
```

这意味着用于类`Stack<>`的操作符<<不是一个函数模板，而是在需要时用类模板实例化的“普通”函数。（即模板实体，参见12.1节）

然而，当我们想要声明一个友元函数，并在之后定义该函数，事情就会变的复杂。这里有两种选择：

1.隐式声明一个新的函数模板，但要使用不同的模板参数，比如U：

```c++
template<typename T>
class Stack{
    ...
     //这里使用T会编译报错declaration of template parameter ‘T’ shadows template parameter
    template<typename U>
    friend std::ostream& operator<<(std::ostream& strm,const Stack<U>& s);
};
```

不管再次使用T或者跳过模板参数声明都没有用（包括内部T隐藏外部T或者在命名空间声明一个非模板函数）。

2.将`Stack<T>`的输出操作符前向声明（意味着我们要首先前向声明`Stack<T>`）

```c++
template<typename T>
class Stack;
template<typename T>
std::ostream& operator<< (std::ostream&,const Stack<T>&);

template<typename T>
class Stack{
    ...
     //注意 <T> 如果缺失<T>，编辑器会警告这里是一个非模板函数
    friend std::ostream& operator<< <T>(std::ostream& strm,const Stack<T>& s);
};
```

需要注意函数名后面的`<T>`，通过这个标识我们声明了一个非成员模板函数的特化作为友元。若没有`<T>`,我们将会声明一个新的非模板函数。12.5.2小节将会有更多的细节。

#### 2.5 类模板的特化

你可以使用模板参数特化类模板。和重载函数模板一样（参见1.5节），类模板特化允许你对于特定类型的实现进行优化或者修正特定类型的类模板特化实例的行为。然而，如果你特化了一个类模板，就必须特化所有的成员函数。尽管你可以特化类模板的某个成员函数，但是如果你这么做了，你就不能再特化该成员函数所属的所有类模板实例了。

如果要特化一个类模板，需要在声明类时以`template<>`开头并标明特化类模板的类型。这些类型被用作模板参数并且必须紧跟在类名的后面：

```c++
template<>
class Stack<std::string>{
  ...  
};
```

对于特化来说，所有的成员函数都要按照“普通”函数来定义，即将所有`T`替换为特化的类型：

```c++
void Stack<std::string>::push(const std::string& elem){
    elems_.push_back(elem);
}
```

完整`Stack<>`对于`std::string`的特化示例：

```c++
#include "stack.hpp" //注意，特化可以作为.cc文件
template<>
class Stack<std::string>{
private:
    std::deque<std::string> elems_;
public:
    void push(const std::string&);
    void pop();
    const std::string& top() const;
    bool empty() const{
        return elems_.empty();
    }
};

void Stack<std::string>::push(const std::string& elem){
    elems_.push_back(elem);
}

void Stack<std::string>::pop(){
    assert(!elems_.empty());
    elems_.pop_back();
}

const std::string& Stack<std::string>::top() const{
    assert(!elems_.empty());
    return elems_.back();
}
```

该实例中，该特化使用引用语义将字符串传递给push()，对于该类型更加合适（更好的做法是使用转发引用（forwarding reference,这里是转发，而非前向，指的是`std::forward`，这些内容将在6.1节讨论））。

另一个不同是使用了`deque`而非`vector`在类中管理数据。尽管在这没什么实际用处，但是表明了特化类模板的实现可能和主要模板的实现看起来非常不同。

#### 2.6 偏特化

类模板可以进行片特化。可以在特定情况下提供特殊的实现，但是有些模板参数仍然需要用户（接口使用者）来定义。举例来说，我们可以为指针定义一个特殊的`Stack<>`类实现：

```c++
#include "stack1.hpp"
template<typename T>
class Stack<T*>{
private:
    std::vector<T*> elems_;
public:
    void push(T*);
    T* pop();
    T* top() const;
    bool empty() const{
        return elems_.empty();
    }
};

template<typename T>
void Stack<T*>::push(T* elem){
    elems_.push_back(elem);
}

template<typename T>
T* Stack<T*>::pop(){
    assert(!elems_.empty());
    T* p = elems_.back();
    elems_.pop_back();
    return p;
}

template<typename T>
T* Stack<T*>::top() const{
    assert(!elems_.empty());
    return elems_.back();
}
```

通过

```c++
template<typename T>
class Stack<T*>{};
```

我们定义了一个类模板，仍然使用T为参数，但是为指针类型进行了特化(`Stack<T*>`)。

再次提醒，特化可能会提供一份（轻微）不同的接口。该实例中的`pop()`返回的是一个已经存储的指针，因此类模板的用户如果使用new来创建指针时，可以通过delete来删除这个指针：

```c++
Stack<int*> prt_stack;
ptr_stack.push(new int{42});
std::cout << *prt_stack.top() << '\n';
delete ptr_stack.pop();
```

**多参数的偏特化**

类模板可能也会特化多个模板参数之间的关系。举例来说，对于下述的类模板：

```c++
template<typename T1,typename T2>
class MyClass{
    ...
};
```

下述的偏特化是可能的：

```c++
//偏特化：所有模板参数都是相同类型
template<typename T>
class MyClass<T,T>{};

//偏特化：第二个类型为int
template<typename T>
class MyClass<T,int>{};

//偏特化，两个模板参数都是指针类型
template<typename T1,typename T2>
class MyClass<T1*,T2*>{};
```

下面指明了每个实例中模板与定义的对应关系：

```c++
MyClass<int,float> mif; 	// MyClass<T1,T2>
MyClass<float,float> mff;	// MyClass<T,T>
MyClass<float,int> mfi;		// MyClass<T,int>
MyClass<int*,float*> mp;	// MyClass<T1*,T2*>
```

如果有多个匹配的偏特化实例，将会提示模糊定义:

```c++
MyClass<int,int> m;	//error:MyClass<T,T> or MyClass<T,int> ?
MyClass<int*,int*> m;//error:MyClass<T,T> or MyClass<T1*,T2*> ?
```

为了解决第二个问题，可以为相同类型的指针提供一个偏特化：

```c++
template<typename T>
class MyClass<T*,T*>{};
```

偏特化的细节，参见16.4节

#### 2.7 默认类模板参数

