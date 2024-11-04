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
> > 正确的翻译应该为：
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

> 橙子注：全特化是将模板类型指定为具体的类型，此时已经可以看做一个具体类
>
> 偏特化是对模板参数形式或者类型进行改变，仍旧存在模板参数。此时的偏特化模板类仍代表一个类族。

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

就像函数模板一样，你可以为类模板参数定义默认值。举例来说，在类`Stack<>`中，你可以将存储元素的容器作为第二个模板参数，并将`std::vector<>`设置为默认值：

```c++
template<typename T, typename Cont = std::vector<T>>
class Stack{
private:
    Cont elems_;

public:
    void push(const T& elem);
    void pop();
    const T& top() const;
    bool empty() const{
        return elems_.empty();
    }
};

template<typename T,typename Cont>
void Stack<T,Cont>::push(const T& elem){
    elems_.push_back(elem);
}

template<typename T,typename Cont>
void Stack<T,Cont>::pop(){
    assert(!elems_.empty());
    elems_.pop_back();
}

template<typename T,typename Cont>
const T& Stack<T,Cont>::top() const{
    assert(!elems_.empty());
    return elems_.back(); // return copy of last element
}
```

需要注意现在我们有两个模板参数，所以每个成员函数定义必须带上这两个参数进行定义：

```c++
template<typename T, typename Cont>
void Stack<T,Cont>::push(const T& elem){
    elems_.push_back(elem);
}
```

你可以像以前一样使用这个自定义stack类。因为如果你只传递了一个参数作为元素类型，`vector`会用来管理这些元素：

```c++
template<typename T,typename Cont = std::vector<T>>
class Stack{
    private:
    	Cont elems_;
...  
};
```

当然，你也可以在你的程序中声明一个`Stack`对象时，指定一个元素容器

```c++
int main() {
    Stack<int> intStack; // stack of ints

    //stack of doubles using a std::deque<> to manage the elements
    Stack<double,std::deque<double>> dblStack;

    //manipulate int stack
    intStack.push(7);
    std::cout << intStack.top() << '\n';
    intStack.pop();

    //manipulate double stack
    dblStack.push(42.42);
    std::cout << dblStack.top() << '\n';
    dblStack.pop();
    return 0;
}
```

通过

```c++
Stack<double,std::deque<double>>
```

你声明了一个使用`std::deque<>`来管理元素的栈，栈中的数据类型为double。

#### 2.8 类型别名

你可以通过为整个类型取一个新名字来更方便的使用类模板

*类型定义*(typedefs)和*别名声明*(alias declarations)

有两种方式为一个完整类型定义一个新名称：

1. 使用关键词`typedef`：

   ```c++
   typedef Stack<int> IntStack;	//typedef
   void foo(const IntStack& s);	//s is stack of ints
   IntStack istack[10]; 			//istack is array of 10 stacks of ints
   ```

   我们将这种方式称为*类型定义*(typedef)，生成的名称为*类型定义名*（typename-name）。

2. C++11后，使用关键词`using`:

   ```c++
   using IntStack = Stack<int>;	//alias declaration 别名声明
   void foo(const IntStack& s);	//s is stack of ints
   IntStack istack[10]; 			//istack is array of 10 stacks of ints
   ```
   由 *DosReisMarcusAliasTemplate*引入，称为别名声明

需要注意，两种方法种，我们都只为已存在的类型定义了一个新名称，而非定义了一个新类型。因此，在类型定义：

```c++
typedef Stack<int> IntStack;
```

或

```c++
using IntStack = Stack<int>;
```

之后，`IntStack`和`Stack<int>`是相同类型的两个通用符号。

对两种为现有类型定义新名称的方式，有一个通用术语*类型别名声明*(type alias declaration)。新名称就称之为*类型别名*(type alias)

由于可读性更好（总是将被定义的类型名称放在等号左边），所以本书的后续章节，都更倾向于使用别名声明的语法（即`using`关键字。橙子注：在《EffectiveMordernC++》还是《ProfessionalC++V5》中好像提及了`typedef`好像会带来隐患，而`using`似乎会有多一层的检查来杜绝这种隐患）

**模板别名(alias templates)**

不同于`typedef`，别名声明可以为一个模板（即一个类型族）提供一个便利的名字。这也是从C++11开始支持的，称为模板别名(alias templates)。

下例中将元素类型T参数化，并展开为一个在`std::deque`中存储元素的栈的模板别名为DequeStack：

```c++
template<typename T>
using DequeStack = Stack<T,std::deuqe<T>>;
```

因此，类模板和模板别名都可以用来作为一个参数化类型。再次提醒，模板别名只是给现存类型定义一个新名字，显存类型仍然可以被使用。`DequeStack<int>`和`Stack<int,std::deque<int>>`都代表同一类型。

再次提醒，一般来说，模板只能声明并定义在全局/命名空间范围内或者在类声明内。

**成员类型的模板别名(Alias Templates for Member Types)**

模板别名对于为模板类成员定义别名来说十分方便。

通过如下方式为类成员定义模板别名后，

```c++
struct C{
  typedef ... iterator;  
};
//or
struct MyType{
  using iterator = ...;  
};
```

我们就可以更方便的声明一个模板类的成员：

```c++
template<typename T>
using MyTypeIterator = typename MyType<T>::iterator;

MyTypeIterator<int> pos;
```

否则，我们就要这样写：

```c++
typename MyType<T>::iterator pos;
```

**类型特征中的后缀`_t`**

从C++14以后，标准库使用该技术对所有会生成一个类型的类型特征定义别名。举例来说，以后可以这么写：

```c++
std::add_const_t<T> //since C++14
```

来替换这种写法：

```c++
typename std::add_const<T>::type //since C++11
```

标准库中的定义：

```c++
namespace std{
    template<typename T>
    using add_const_t = typename add_const<T>::type;
}
```

#### 2.9 类模板的参数推导

在C++17之前，除非设置了默认的模板参数，否则你需要将所有的模板参数类型传递给类模板。到了C++17，这个限制被放松了。如果构造函数可以推导所有模板参数(没有默认值的)，你就可以跳过显式定义模板参数。

举个例子，在前面所有的代码示例中，你可以不指定模板参数使用拷贝构造函数：

```C++
Stack<int> intStack1;
Stack<int> intStack2 = intStack1; // OK in all versions
Stack intStack3 = intStack1;	  // OK since C++17
```

通过提供需要初始化参数的构造函数，你可以使自定义的stack类支持元素类型推导。举例说明，我们可以提供一个可以由单一元素初始化的的stack：

```c++
template<typename T>
class Stack{
  private:
    std::vector<T> elems_;
  public:
    Stack() = default;
    Stack(const T& elem) // initialize stack with one element
        :elems_({elem}){}
    ...
};
```

这样就可以以如下方式声明一个stack

```c++
Stack intStack = 0; // Stack<int> deduced since C++17
```

通过整型数0来初始化一个stack，模板参数T被推导为int，这样就实例化了一个`Stack<int>`

注意事项：

+ 因为定义了带有参数int的构造函数，你需要显式声明默认构造函数为默认（即通过default设置默认构造函数，形式如下）。因为默认构造函数只在没有定义其他构造函数的时候有效。（橙子注：简言之就是，如果用户定义了构造函数，编译器就不会自动生成默认构造函数，如果想要该类有默认构造函数的行为，需要显式指定）

  ```c++
  Stack() = default;
  ```

+ elem参数作为唯一元素使用大括号`{}`括起来，作为一个*初始化列表*(initializer list)来初始化vector类型的`elems_`：

  ```c++
  :elems_({elem})
  ```

  vector没有任何构造函数可以直接使用一个元素进行构造。(注意：`vector<int>{5}`构造的是一个长度为5的数组)

需要注意，和函数模板不同，类模板参数不能只进行部分推导（通过显式声明了部分模板参数）。细节将在15.12讨论

**通过字符串常量(String Literals)进行类模板参数推导**

原则上，你也可以通过字符串常量初始化栈：

```c++
Stack string_stack = "bottom"; // Stack<const char[7]> deduced since C++17
```

但是这会导致很多问题：一般而言，当以引用形式传递一个模板类型T（橙子注：这里当以引用传递指的是前文中自定义的单个参数的构造函数`Stack(const T& elem)`），参数并不会*衰减*(decay，术语，表示将一个原生数组转换为一个指针类型)。这就意味着我们实际的初始化为:

```c++
Stack<const char[7]>
```

并在任何使用T的地方使用const char[7]类型。举例来说，我们不能push不同大小的字符串，因为它们的类型不同(we may not push a string of different size,because it has a different type)。更多细节参见7.4节。

然而，当按值传递一个模板类型T，参数会*衰减*(decay，术语，表示将一个原生数组转换为一个指针类型)。这就意味着，构造函数将参数T推导成了const char*，所以整个类被推导成了`Stack<const char*>`

正因如此，将构造函数声明为按值传递是有价值的：

```c++
template<typename T>
class Stack{
  private:
    std::vector<T> elems_;
  public:
    Stack(T elem)
        :elems_({elem})
    {}
};
```

这样，下述的初始化就可以正常工作：

```c++
Stack string_stack = "bottom";	//Stack<const char*> deduced since C++17
```

此时，为了避免多余的拷贝操作，我们应该使用移动操作：

```c++
template<typename T>
class Stack{
  private:
    std::vector<T> elems_;
  public:
    Stack(T elem)
        :elems_({std::move(elem)})
    {}
};
```

**推导指引**

除了将构造函数声明为按值传递，还有另一个解决方法：因为将裸指针存储在容器中进行操作会造成诸多问题，我们应该禁止为容器类自动推导原生字符指针。

你可以定义指定的*推导指引(deduction guides)来为类模板推导提供附加选择或者修正类模板推导的行为。举例来说，你可以定义推导指引，不管是字符常量还是C风格字符串，stack都实例化为`std::string`:

```c++
Stack(const char*) -> Stack<std::string>;
```

该指引应该和类定义出现在同一作用域（或命名空间）。一般来讲，该指引跟在类定义后面。我们将跟在推导指引`->`后面的类型称之为*指引类型*(guided type)

现在，下述声明将会推导为`Stack<std::string`。

```c++
Stack string_stack{"bottom"}; // OK:Stack<std::string> deduced since C++17
```

然而，下述形式依旧不能推导为`Stack<std::string>`

```c++
Stack string_stack = "bottom"; //Stack<std::string> deduced,but still not valid
```

原因如下，我们推导为`std::string`就相当于实例化了一个`Stack<std::string>`:

```c++
class Stack{
    private:
    	std::vector<std::string> elems_;
    public:
    	Stack(const std::string& elem)
            :elems_({elem}){}
};
```

但是，根据语言规则，不能将字符串常量传递给以`std::string`为参数的构造函数来拷贝初始化（使用`=`来初始化）一个对象。所以必须使用`{}`来初始化stack：

```c++
Stack string_stack{"bottom"}; //Stack<std::string>deduced and valid
```

以防有疑问，类模板参数推导也会复制。在将string_stack声明为`Stack<std::string>`后，下述的初始化为相同类型（调用了拷贝构造），而不是通过将string stack作为元素进行初始化。

```c++
Stack stack2{string_stack};		//Stack<std::string> deduced
Stack stack3(string_stack);		//Stack<std::string> deduced
Stack stack4 = {string_stack};	//Stack<std::string> deduced
```

类模板参数推导细节在15.12小节

#### 2.10 模板聚合

聚合类也可以作为模板。（聚合类：没有用户定义的，显式的或者继承的构造函数；没有私有或保护的非静态数据成员；没有虚函数；没有虚基类、私有基类或者保护基类的类或者结构体）。举个例子：

```c++
template<typename T>
struct ValueWithComment{
    T value;
    std::string comment;
};
```

该结构体定义了参数化了value类型的聚合类。你可以想其他任何类模板一样声明对象，并且继续以聚合的方式使用:

```c++
ValueWithComment<int> vc;
vc.value = 42;
vc.comment = "initial value";
```

从C++17开始，对于聚合类模板，也可以定义推导指引：

```C++
ValueWithComment(const char*,const char*) -> ValueWithComment<std::string>;
ValueWithComment vc2 = {"hello","initial value"};
```

如果没有推导指引，这种初始化是不可能的，因为ValueWithComment没有提供构造函数来帮助推导。

标准库中的`std::array<>`也是一个聚合类，参数化了元素类型和大小。C++17也为其定义了推导指引，将在4.4.4节讨论。

#### 2.11 总结

+ 类模板是一个或者多个类型参数待定的类
+ 使用类模板，需要将类型作为模板参数传递。类模板就会为这些类型实例化并编译
+ 对于类模板，只有被调用的成员函数才会被实例化
+ 可以为类模板指定具体的类型（全特化）
+ 可以使用指定类型对类模板进行偏特化
+ 从C++17开始，类模板参数可以从构造函数中自动推导
+ 可以定义聚合类模板
+ 按值传递的模板类型将会衰退
+ 模板只能在全局、命名空间或者类定义内部进行声明。

### 第三章：非类型模板参数

对于函数和类模板，模板参数不必为类型。它们也可以是一般的值。和使用类型参数作为模板一样，你在代码使用之前为一些细节保留开放（橙子注：就是在没有使用之前，模板类型是不进行实例化的）。非类型模板参数也一样，只不过保持开放的是一个值而不是一个类型。当使用这种模板时，你需要显式指定这个值。之后对应的代码就会被实例化。该章节通过这种特性展示了一个新版本的stack类模板。另外，我们展示了一个非类型函数模板参数的示例并将讨论该技术的一些限制

#### 3.1 非类型类模板参数

和之前章节的stack实现对比，你可以通过一个固定长度的数组实现一个栈。这种方法的优点是，不管是来源于你或者标准容器的内存管理开销被避免了。然而，找到栈的最佳尺寸是一个挑战。你栈设定的越小，就越容易填满。你栈设定的越大，就会有越多不必要的内存被保留。一个较好的方法是让用户决定stack中的最大元素数量是多少。

为了达到这个目的，可以将大小设置为一个模板参数：

```c++
template<typename T, std::size_t MaxSize>
class Stack{
private:
    std::array<T,MaxSize> elems_;
    std::size_t num_elems_;
public:
    Stack();
    void push(const T& elem);
    void pop();
    const T& top() const;
    bool empty() const{
        return num_elems_ == 0;
    }

    std::size_t size() const{
        return num_elems_;
    }
};
template<typename T,std::size_t MaxSize>
Stack<T,MaxSize>::Stack()
:num_elems_(0){}


template<typename T,std::size_t MaxSize>
void Stack<T,MaxSize>::push(const T& elem){
    assert(num_elems_ < MaxSize);
    elems_[num_elems_] = elem;
    ++num_elems_;
}

template<typename T,std::size_t MaxSize>
void Stack<T,MaxSize>::pop(){
    assert(!elems_.empty());
    --num_elems_;
}

template<typename T,std::size_t MaxSize>
const T& Stack<T,MaxSize>::top() const{
    assert(!elems_.empty());
    return elems_[num_elems_-1]; 
}
```

第二个新的模板参数，`MaxSize`是一个整型。它指定了内部数组中栈元素的最大个数：

```c++
template<typename T,std::size_t MaxSize>
class Stack{
  private:
    std::array<T,MaxSize> elems_;
};
```

另外，在`push()`函数中检查了栈是否已满：

```c++
template<typename T,std::size_t MaxSize>
void Stack<T,MaxSize>::push(const T& elem){
    assert(num_elems_ < MaxSize);
    elems_[num_elems_] = elem;
    ++num_elems_;
}
```

为了使用这个类模板，你需要指明元素类型和最大容量：

```c++
int main() {
    Stack<int,20> int20Stack;   //stack of up to 20 ints
    Stack<int,40> int40Stack;   //stack of up to 40 ints
    Stack<std::string,40> stringStack;  //stack of up to 40 strings

    //manipulate stack of up to 20 ints
    int20Stack.push(7);
    std::cout << int20Stack.top() << std::endl;
    int20Stack.pop();

    //manipulate stack of up to 40 strings
    stringStack.push("hello");
    std::cout << stringStack.top() << std::endl;
    stringStack.pop();
    return 0;
}
```

需要注意，每个模板实例都是它自动的类型。因此，int20Stack和int40Stack是两个不同的类型，它们之间是没有隐式转换的。所以它们不能互相替换以及互相赋值。

同样的，模板参数可以设置默认值：

```c++
template<typename = int,std::size_t MaxSize=100>
class Stack{
  ...  
};
```

但是，从设计角度来看，这样做在该实例中并不合适。默认参数应该让使用者在使用时直观上看起来是正确的。但是不管是int类型还是最大值100对一个普通的stack类来说并不直观。因此，最好让用户在声明的时候将两个值都指定出来。

#### 3.2 非类型函数模板参数

你也可以为函数模板定义非类型参数。举个例子，下面的函数模板定义了一组可以累加指定值的函数：

```c++
template<int Val,typename T>
T addValue(T x){
    return x + Val;
}
```

这种方法在函数或者操作别当做参数时十分有用。举例来说，在C++标准库中，你可以使用该函数实例来对一个集合的每个元素增加指定值：

```c++
std::transform(source.begin(),source.end(),
              dest.begin(),
              addValue<5,int>);
```

最后一个参数将`addValue<>()`实例化为将传递进来的int参数加5。source集合里的每个元素会调用对应的函数(即实例化的addValue)，并将结果放在dest集合中。

需要注意，你需要为`addValue<>()`的模板参数T指定为int。推导仅在即时调用时有效，但是`std::transform()`的第四个参数需要一个完整的类型来推导。标准不支持对部分模板指定参数而对剩余的参数进行推导。

同样，你也可以指定模板参数是由前面的参数推导出来的。举个例子，从传递进来的非类型参数派生出返回类型：

```c++
template<auto Val,typename T = decltype(Val)>
T foo();
```

或者保证传递进来的值和传递进来的类型是相同的类型：

```c++
template<typename T,T Val = T{10}>
T bar(T val){
    return val + Val;
}

int main() {
    int num = 10;
    std::cout << bar<int>(num) << std::endl; //20
    return 0;
}
```

#### 3.3 非类型模板参数的限制

需要注意非类型模板参数有一些限制。一般来说，只能是整型常量值（包括枚举），对象、函数、成员的指针，对象或者函数的左值引用，或者`std::nullptr_t`(`nullptr`的类型)。

浮点类型和类对象是不允许作为非类型模板参数的：

```c++
template<double VAT>	//error
double process(double v){
    return v * VAT;
}

template<std::string name>	//error
class MyClass{
    ...
};
```

在向模板参数传递指针或者引用时，对象不能是字符串常量，临时对象，或者数据成员和他的子类。因为在C++17之前，这些限制在逐步放宽，所以下述的限制也需注意：

+ C++11里，对象必须有外部链接（即，可以在另一个翻译单元中使用extern引用）
+ C++14中，对象补习有内部或者外部链接

因此，下例是不可行的：

```c++
template<const char* name>
class MyClass{
    ...
};

MyClass<"hello"> x;	//error:string literal "hello" not allowed
```

但是也有变通方案（依赖于C++的版本）：

```c++
template<const char* name>
class Message{
public:
    void display(){
        std::cout << name << std::endl;
    }
};

extern const char s03[] = "s03";
const char s11[] = "s11";

int main() {
    Message<s03> m03; //ok for all versions
    m03.display();

    Message<s11> m11; // ok since C++11
    m11.display();

    static const char s17[] = "s17";//no linkage
    Message<s17> m17;   // ok since C++17
    m17.display();
    return 0;
}
```

三种场景下，常量字符串被当做一个声明为`const char*`的模板参数。如果该对象有外部链接，那么在所有C++版本中都合法(s03)；如果该对象有内部链接，那么在C++11、C++14中是合法的；如果该对象没有任何链接，那么从C++17开始都合法。

更多细节将在12.3.3小节讨论，并在17.2小节讨论该领域将来可能发送的改变。

**避免无效的表达式**

非类型模板参数的实参可能是任何编译期表达式。举个例子：

```c++
template<int I,bool B>
class C;
...
C<sizeof(int) + 4,sizeof(int) == 4> c;
```

然而，如果在表达式中使用了运算符`>`，你就必须将整个表达式括起来，保证配对的`>`作为结尾：

```c++
C<42, sizeof(int) > 4> c;	//error:first > ends the template argument
C<42, (sizeof(int) > 4)> c;	//ok
```

#### 3.4 模板参数类型auto

从C++17开始，你可以定义一个非类型模板参数来接收任何被允许的非类型参数。通过该特性，我们可以提供一个更泛化的有固定长度的stack类：

```c++
template<typename T, auto MaxSize>
class Stack{
public:
    using size_type = decltype(MaxSize);
private:
    std::array<T,MaxSize> elems_;
    size_type num_elems_;
public:
    Stack();
    void push(const T& elem);
    void pop();
    const T& top() const;
    bool empty() const{
        return num_elems_ == 0;
    }

    size_type size() const{
        return num_elems_;
    }
};


template<typename T, auto MaxSize>
Stack<T,MaxSize>::Stack()
:num_elems_(0){}


template<typename T, auto MaxSize>
void Stack<T,MaxSize>::push(const T& elem){
    assert(num_elems_ < MaxSize);
    elems_[num_elems_] = elem;
    ++num_elems_;
}

template<typename T, auto MaxSize>
void Stack<T,MaxSize>::pop(){
    assert(!elems_.empty());
    --num_elems_;
}

template<typename T, auto MaxSize>
const T& Stack<T,MaxSize>::top() const{
    assert(!elems_.empty());
    return elems_[num_elems_-1]; // return copy of last element
}
```

通过定义

```c++
template<typename T,auto MaxSize>
class Stack{
    ...
};
```

通过使用*占位符类型*(placeholder type)auto,你将MaxSize定义为一个没有指明的类型的值。它可以是任何允许作为非类型模板参数类型的类型。

在类内部，你可以使用这个值：

```c++
std::array<T,MaxSize> elems_;
```

以及其类型：

```c++
using size_type = decltype(MaxSize);
```

然后，比方说，你就可以将这个类型作为一个`size()`成员方法的返回值类型：

```c++
size_type size() const{
    return num_elems_;
}
```

从C++14开始，你也可以直接使用auto作为返回值类型，让编译器帮你找到返回类型。

```c++
auto size() const{
    return num_elems_;
}
```

当使用stack时，在这个类声明中，元素数量的类型由用来记录元素数量的类型的类型决定（With this class declaration the type of the number of elements is defined by the type used for the number of elements,橙子注：绕来绕去的话，简而言之就是，记录元素数量的类型由用户使用时传入的值的类型决定。用例子说明就是，`size_type`是用户在声明`Stack<int,22>`时，由这个`22`决定的。）：

```c++
int main() {
    Stack<int,20u> int20Stack;          //stack of up to 20 unsigned ints
    Stack<std::string,40> stringStack;  //stack of up to 40 strings

    //manipulate stack of up to 20 unsigned ints
    int20Stack.push(7);
    std::cout << int20Stack.top() << '\n';
    auto size1 = int20Stack.size();

    //manipulate stack of up to 40 strings
    stringStack.push("hello");
    std::cout << stringStack.top() << '\n';
    auto size2 = stringStack.size();

    if(!std::is_same<decltype(size1),decltype(size2)>::value){  //C++11 version
        std::cout << "size type differ judged by C++11" << '\n';
    }

    if(!std::is_same_v<decltype(size1),decltype(size2)>){  //C++17 version
        std::cout << "size type differ judged by C++17" << '\n';
    }
    return 0;
}
```

通过

```c++
Stack<int,20u> int20Stack;
```

内部的size type就通过传递进来的20u定义成了`unsigned int`。

通过

```c++
Stack<std::string,40> stringStack;
```

内部的size type就通过传递进来的40定义成了`int`。

两个stack的`size()`方法返回值将会有不同的类型，在

```c++
auto size1 = int20Stack.size();
...
auto size2 = stringStack.size();
```

之后，size1和size2的类型是不同的。通过使用标准类型特征`std::is_same`（参考附录D3.3）和`decltype`，我们可以验证：

```c++
if(!std::is_same<decltype(size1),decltype(size2)>::value){  //C++11 version
    std::cout << "size type differ judged by C++11" << '\n';
}
```

对应的输出为：

```shell
size type differ judged by C++11
```

从C++17开始，对于特征返回值，你也可以直接使用后缀`_v`来跳过`::value`（参见5.6节）

```c++
if(!std::is_same_v<decltype(size1),decltype(size2)>){  //C++17 version
    std::cout << "size type differ judged by C++17" << '\n';
}
```

需要注意，非类型模板参数的其他限制依旧有效。尤其是3.3节讨论的非类型参数可能类型的限制。比如说：

```c++
Stack<int,3.14> sd;	//error:floatting-point nontype argument
```

同样，你也可以字符串作为常量数组传入（从C++17开始本地静态声明也可以(static locally declared)；参见3.3节），下例是可能的：

```c++
template<auto T>
class Message{
public:
    void print(){
        std::cout << T << '\n';
    }
};

int main() {
    Message<42> msg1;
    msg1.print();

    static const char s[] = "hello";
    Message<s> msg2;
    msg2.print();

    return 0;
}
```

注意，`template<decltype(auto) N>`也有可能，这允许N以引用的形式实例化：

```c++
template<decltype(auto) N>
class T{
    ...
};

int i;
T<(i)> x; //N is int&
```

细节将在15.10.1小节介绍

#### 3.5 总结

+ 模板不仅仅可以以类型作为参数，也可以以值作为参数
+ 不能使用浮点类型或者类对象作为非类型模板参数。对于字符串常量的指针/引用，临时变量，子对象，该限制同样适用。
+ 适用`auto`使模板拥有可以以一般类型作为非类型模板参数。

### 第四章：可变参数模板

从C++11开始，模板可以使用可变数量的模板参数。该特性允许你在需要传入任意数量任意类型的地方使用模板。一个典型的应用是将任意数量任意类型的参数传递给一个类或者框架。另一个应用例子是为处理任意数量任意类型的参数提供通用代码。

#### 4.1 可变参数模板

模板参数可以被定义为接收任意数量的模板参数。拥有这种能力的模板被称为可*变参数模板*（variadic templates）。

##### 4.1.1 可变参数模板示例

举例来说，你可以通过以下代码用不同类型的可变参数调用`print()`函数：

```c++
void print(){}

template<typename T,typename... Types>
void print(T first_arg,Types... args){
    std::cout << first_arg << '\n';
    print(args...);
}
```

如果传递一个或者多个参数，就会使用到这个模板函数（在递归调用输出剩余参数之前输出单独声明的第一个参数）。剩余名为args的参数是一个*函数参数包*（function parameter pack）:

```c++
void print(T firstArg,Types... args)
```

使用"Types"定义一个*模板参数包*(template parameter pack)：

```c++
template<typename T,typename... Types>
```

定义了一个非模板重载的`print()`，当参数为空的时候调用来结束递归。

举个例子，以下调用：

```c++
std::string s("world");
print(7.5,"hello",s);
```

输出：

```shell
7.5
hello
world
```

第一次调用相当于：

```c++
print<double,const char*,std::string> (7.5,"hello",s);
```

其中：

+ `firstArg`值为7.5，所以类型T就是`double`
+ `args`是可变模板参数，包括`const char*`类型的`"hello"`以及`std::string`类型的`"world"`

在`firstArg`以7.5输出后，将使用剩下的参数再次调用`print()`，相当于：

```c++
print<const char*,std::string>("hello",s);
```

其中：

+ `firstArg`值为`"hello"`，所以类型T就是`const char*`
+ `args`是可变模板参数，包括`std::string`类型的`"world"`

