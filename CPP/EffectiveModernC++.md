# Effective Modern C++

## 第一章 类型推导

### 条款1：理解类型推导

> + auto的类型推导是建立在模板类型推导的基础之上的

考虑如下函数模板

```c++
template<typename T>
//函数声明，ParamType可以带有修饰，如：const T&param
void f(ParamType param);
//调用形式如下：
f(expr);
```

对于类型T的推导取决于`ParamType`和`expr`，有三种情况需要考虑：

+ `ParamType`是一个指针或引用，但不是通用引用;

  > 通用引用（T&&,涉及引用折叠，详细介绍在条款24）

+ `ParamType`是一个通用引用

+ `ParamType`既不是指针也不是引用

#### `ParamType`是一个指针或引用，但不是通用引用

该情况下类型推导的两个步骤：

+ 如果`expr`的类型是一个引用，忽略引用部分
+ 然后`expr`的类型与`ParamType`进行模式匹配来决定`T`

引用实例：

```c++
template<typename T>
void f(T& param);               //param是一个引用
int x=27;                       //x是int
const int cx=x;                 //cx是const int
const int& rx=x;                //rx是指向作为const int的x的引用
f(x);                           //T是int，param的类型是int&
f(cx);                          //T是const int，param的类型是const int&
f(rx);                          //T是const int，param的类型是const int&

```

##### 个人总结：

> + 核心是值传递，T只会推导**基本类型**（int,char...）
> + 对于普通的左值引用以及指针，实参的引用以及指针会别忽略
> + 形参的类型（最后推导出的完整类型）中CV属性由形参和实参共同决定，而左值引用和指针则完全由形参决定

#### `ParamType`是一个通用引用

实例：

```c++
template<typename T>
void f(T&& param);              //param现在是一个通用引用类型
        
int x=27;                       //如之前一样
const int cx=x;                 //如之前一样
const int & rx=cx;              //如之前一样

f(x);                           //x是左值，所以T是int&，
                                //param类型也是int&

f(cx);                          //cx是左值，所以T是const int&，
                                //param类型也是const int&

f(rx);                          //rx是左值，所以T是const int&，
                                //param类型也是const int&

f(27);                          //27是右值，所以T是int，
                                //param类型就是int&&
```



##### 个人总结：

> 引用会折叠，如果设置模板类型参数为T&&，即通用引用，则**仅仅**对右值以及右值引用推到为右值引用，其他均为左值引用。CV限定同情景一

#### `ParamType`既不是指针也不是引用

实例：

```c++
template<typename T>
void f(T param);                //以传值的方式处理param
int x=27;                       //如之前一样
const int cx=x;                 //如之前一样
const int & rx=cx;              //如之前一样

f(x);                           //T和param的类型都是int
f(cx);                          //T和param的类型都是int
f(rx);                          //T和param的类型都是int
//较为复杂的情景
const char* const ptr =         //ptr是一个常量指针，指向常量对象 
    "Fun with pointers";
//ptr自身的const限定会被省略，即由const char * const ---->const char *
f(ptr); 					  //T的类型为const char *
```

##### 个人总结：

> 完全的值传递，形参和实参是两个独立的个体。需要注意的是实例中上层和下层const都存在的情况下，部分const属性会忽略（本质还是值传递的问题，只不过场景复杂）

#### 数组实参

> 特别注意，在《c专家编程》中提到过，数组和指针是完全独立的两个类型，只是在大多数情况下，数组会退化为指针。
>
> 数组类型：typename array[N],如： int array[10]代表一个可以存储10个整型的数组类型

数组退化为指针的实例：

```c++
const char name[] = "J. P. Briggs";     //name的类型是const char[13]

const char * ptrToName = name;          //数组退化为指针
```

实例：

```c++
template<typename T>
void f(T param);                        //传值形参的模板
f(name);                                //T推导为const char *

template<typename T>
void f(T& param);                       //传引用形参的模板
f(name);                                //T推导为const char (&)[13]!!!

//因此可以由比较酷的用法推算数组的长度
template<typename T,std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept  //无名参数，有名参数为T (&)name[N]
{
    return N;
}
//使用
int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 };             //keyVals有七个元素
int mappedVals[arraySize(keyVals)];                     //mappedVals也有七个
//C++程序员应该使用的数组
std::array<int, arraySize(keyVals)> mappedVals;         //mappedVals的大小为7
```

##### 个人总结：

> 对于值传递形式的模板类型(T)，和原始的C语言一样，传入数组名会退化为指针。
>
> 但是，对于引用形式的模板类型(T&),数组类型**不会**退化

#### 函数实参

和数组实参一样：

```c++
void someFunc(int, double);         //someFunc是一个函数，
                                    //类型是void(int, double)

template<typename T>
void f1(T param);                   //传值给f1

template<typename T>
void f2(T & param);                 //传引用给f2

f1(someFunc);                       //param被推导为指向函数的指针，
                                    //类型是void(*)(int, double)
f2(someFunc);                       //param被推导为指向函数的引用，
                                    //类型是void(&)(int, double)
```

#### 条款1总结：

+ 模板类型推导时，实参的左值引用和指针会被忽略。形参的类型由模板类型的ref,ptr决定
+ 通用引用仅对右值和右值引用推导为右值引用
+ 值传递的类型推导，实参形参是两个个体，实参的CV限定在形参中会被忽略
+ 模板类型推导时，数组名和函数名会退化为指针(T)，出给时引用类型的模板推导（T&）

### 条款2：理解auto的类型推导

> `auto`类型推导和模板类型推导有一个直接的映射关系

当一个变量使用`auto`声明时，`auto`就扮演了模板中`T`的角色。

唯一不同的是，自`C++11`开始引入了`std::initializer_list<T>`，初始化列表（《C++20高级编程》推荐使用的统一初始化形式）。

这就造成了`auto`进行类型推导不同于模板类型推导的情况，当使用`auto`声明的变量使用花括号进行初始化的时候，推导出的类型是`std::initializer_list`。

但是对于模板类型推导，传入列表初始化是不行的。

实例：

```c++
//C++11推出的统一初始化
int x3 = { 27 };
int x4{ 27 };
//如果对auto声明的变量进行统一初始化，则会出现和模板类型推导不同的情况。
auto x3 = { 27 };               //类型是std::initializer_list<int>，
                                //值是{ 27 }
auto x4{ 27 };                  //同上
//这里错误是因为初始化列表本身就是一个模板方法，列表中的类型不一致，无法推导T
auto x5 = { 1, 2, 3.0 };        //错误！无法推导std::initializer_list<T>中的T

auto x = { 11, 23, 9 };         //x的类型是std::initializer_list<int>

//对于模板类型推导，不能直接传入初始化列表
template<typename T>            //带有与x的声明等价的
void f(T param);                //形参声明的模板
f({ 11, 23, 9 });               //错误！不能推导出T
```

> 需要注意的是，`C++11`中，auto仅用在初始化列表。但是在`C++14`中同样可以用在函数返回值和lambda表达式中

### 条款3：理解decltype

`decltype`只是简单的返回名字或者表达式的类型：

```c++
const int i = 0;                //decltype(i)是const int
bool f(const Widget& w);        //decltype(w)是const Widget&
                                //decltype(f)是bool(const Widget&)
struct Point{
    int x,y;                    //decltype(Point::x)是int
};                              //decltype(Point::y)是int

Widget w;                       //decltype(w)是Widget
if (f(w))…                      //decltype(f(w))是bool
    
template<typename T>            //std::vector的简化版本
class vector{
public:
    …
    T& operator[](std::size_t index);
    …
};

vector<int> v;                  //decltype(v)是vector<int>
…
if (v[0] == 0)…                 //decltype(v[0])是int&
```

`C++11`中`decltype`主要用途：当函数模板的返回类型依赖形参的类型的时候，配合**尾置返回类型**使用

简单的使用实例：

```c++
//该方法中，auto不会做任何推导，主要依靠尾置返回类型来使用形参的类型
template<typename Container, typename Index>    //可以工作，
auto authAndAccess(Container& c, Index i)       //但是需要改良
    ->decltype(c[i])
{
    authenticateUser();
    return c[i];
}
```

对于以上示例，`C++14`中的auto可以对函数返回类型进行推导，所以不需要尾置返回类型：

```c++
template<typename Container, typename Index>    //C++14版本，
auto authAndAccess(Container& c, Index i)       //不那么正确
{
    authenticateUser();
    return c[i];                                //从c[i]中推导返回类型
}
//这里存在的问题是，auto推导和模板类型推导一样，会将实参的引用忽略
//而大多数容器返回的是T&，这就导致在auto推导返回值类型时，推导出来的时T（即元素副本）
std::deque<int> d;
…
authAndAccess(d, 5) = 10;               //认证用户，返回d[5]，这里返回值是一个右值（临时值）
                                        //然后把10赋值给它
                                        //无法通过编译器！
```

为了解决上述问题`C++14`中提供了`decltype(auto)`

```c++
//decltype(auto)可以理解为：
//auto需要进行类型推导，但是在推导时，运用decltype规则
template<typename Container, typename Index>    //C++14版本，
decltype(auto)                                  //可以工作，
authAndAccess(Container& c, Index i)            //但是还需要
{                                               //改良
    authenticateUser();
    return c[i];
}
//该方法对于声明变量也同样适用
Widget w;
const Widget& cw = w;
auto myWidget1 = cw;                    //auto类型推导
                                        //myWidget1的类型为Widget
decltype(auto) myWidget2 = cw;          //decltype类型推导
                                        //myWidget2的类型是const Widget&
```

该用例可以适用大多数情况，但是有一个edge case，即无法将右值作为实参传入，考虑下述情形：

```c++
std::deque<std::string> makeStringDeque();      //工厂函数
//从makeStringDeque中获得第五个元素的拷贝并返回
auto s = authAndAccess(makeStringDeque(), 5);//错误，函数返回值是一个临时变量（右值）
//解决方案:通用引用
template<typename Containter, typename Index>   //现在c是通用引用
decltype(auto) authAndAccess(Container&& c, Index i);
```

所以最终的`C++11`和`C++14`的版本如下：

```c++
template<typename Container, typename Index>    //最终的C++11版本
auto
authAndAccess(Container&& c, Index i)
->decltype(std::forward<Container>(c)[i])
{
    authenticateUser();
    return std::forward<Container>(c)[i];//条款25：对右值引用适用std::move,对通用引用适用std::forward
}

template<typename Container, typename Index>    //最终的C++14版本
decltype(auto)
authAndAccess(Container&& c, Index i)
{
    authenticateUser();
    return std::forward<Container>(c)[i];//条款25：对右值引用适用std::move,对通用引用适用std::forward
}
```

**decltype的极端适用情形**(普通开发者不需要考虑)：对于比单纯的变量名更复杂的左值表达式，`decltype`可以确保报告的类型始终是左值引用。也就是说，如果一个不是单纯变量名的左值表达式的类型是`T`，那么`decltype`会把这个表达式的类型报告为`T&`，尤其是在`C++14`支持了decltype(auto)后会出现以下状况：

```c++
decltype(auto) f1()
{
    int x = 0;
    …
    return x;                            //decltype(x）是int，所以f1返回int
}

decltype(auto) f2()
{
    int x = 0;
    return (x);                          //decltype((x))是int&，所以f2返回int&
}
```

### 条款4：知道如何查看类型推导的结果

#### IDE编辑器

> 在代码可编译的情况下，将鼠标放置在变量上即可。
>
> 但是仅对简单类型适用，对于较为复杂的类型，IDE提供的信息相对鸡肋

#### 编译器诊断

利用编译器提供的错误信息查看类型推导的结果，比如：

```c++
const int theAnswer = 42;
auto x = theAnswer;//const int
auto y = &theAnswer;//const int *

template<typename T>                //只对TD进行声明,因为我们需要利用错误信息
class TD;                           //TD == "Type Displayer"

TD<decltype(x)> xType;              //引出包含x和y
TD<decltype(y)> yType;              //的类型的错误消息
```

那么编译器会报告类似以下的错误：

```c++
error: aggregate 'TD<int> xType' has incomplete type and 
        cannot be defined
error: aggregate 'TD<const int *> yType' has incomplete type and
        cannot be defined
```

#### 运行时输出

对于简单的类型可以直接使用`typeid`和`std::type_info::name`

```c++
const int theAnswer = 42;
auto x = theAnswer;//const int
auto y = &theAnswer;//const int *
std::cout << typeid(x).name() << '\n';  //显示x和y的类型
std::cout << typeid(y).name() << '\n';
```

对于涉及到类型推导的复杂情况，主流编译器输出的都是错误的（尤其是对于上下层const的判断）。如果可以，使用**Boost.TypeIndex**判断出的信息更为可靠。

```C++
template<typename T>                    //要调用的模板函数
void f(const T& param);

std::vector<Widget> createVec();        //工厂函数

const auto vw = createVec();            //使用工厂函数返回值初始化vw

if (!vw.empty()){
    f(&vw[0]);                          //调用f,注意vector[]返回的是引用
    								 //这里返回的是const Widget &
    								 //取地址后是const Widget *&,作为一个整体进行类型推导
    …
}
```

正如**条款1**提到的，如果传递的是一个引用，那么引用部分（reference-ness）将被忽略，如果忽略后还具有`const`或者`volatile`，那么常量性`const`ness或者易变性`volatile`ness也会被忽略。那就是为什么`param`的类型`const Widget * const &`会输出为`const Widget *`，首先引用被忽略，然后这个指针自身的常量性`const`ness被忽略，剩下的就是指针指向一个常量对象

>  更重要的还是理解类型推导的原则

## 第二章 auto

> auto的使用可以提高开发效率，更加注重于抽象层次。
>
> 但是由于auto使用存在坑，本章主要介绍如何正确的使用auto

### 条款5：优先使用auto而不是显式的类型声明

auto可以避免未初始化变量

```c++
auto a;//error
auto a = 1;//correct
```

如果声明一个局部变量，并且局部变量的类型是闭包的（即只有编译器知道是什么类型的），自`C++11`后可以使用auto进行优化，示例如下：

```c++
//不适用auto的话，需要使用萃取器获取类型
//typename std::iterator_traits<It>::value_type意思为，使用迭代器萃取器对模板It进行类型提取，提取到value_type，说人话就是，使用模板类型It的类型声明一个变量
template<typename It>           //对从b到e的所有元素使用
void dwim(It b, It e)           //dwim（“do what I mean”）算法
{
    while (b != e) {
        typename std::iterator_traits<It>::value_type
        currValue = *b;
        …
    }
}

//局部变量，闭包，可以使用auto优化开发效率
template<typename It>           //如之前一样
void dwim(It b,It e)
{
    while (b != e) {
        auto currValue = *b;
        …
    }
}

//auto还可以表示一些“只有编译器知道的类型”
auto derefUPLess = 
    [](const std::unique_ptr<Widget> &p1,       //用于std::unique_ptr
       const std::unique_ptr<Widget> &p2)       //指向的Widget类型的
    { return *p1 < *p2; };                      //比较函数

//C++14中auto可以使用在auto表达式中，所以可以进一步优化
auto derefLess =                                //C++14版本
    [](const auto& p1,                          //被任何像指针一样的东西
       const auto& p2)                          //指向的值的比较函数
    { return *p1 < *p2; };
//而不适用auto的C++11原生版本如下
std::function<bool(const std::unique_ptr<Widget> &,
                   const std::unique_ptr<Widget> &)>
derefUPLess = [](const std::unique_ptr<Widget> &p1,
                 const std::unique_ptr<Widget> &p2)
                { return *p1 < *p2; };
```

>  值得注意的是，auto不仅可以提高开发效率，也可以提升程序的性能。在上述最后一个示例中，用`auto`声明的变量保存一个和闭包一样类型的（新）闭包，因此使用了与闭包相同大小存储空间。实例化`std::function`并声明一个对象这个对象将会有固定的大小。这个大小可能不足以存储一个闭包，这个时候`std::function`的构造函数将会在堆上面分配内存来存储，这就造成了使用`std::function`比`auto`声明变量会消耗更多的内存。

另一个auto的优势是，可以避免一些隐式转换带来的麻烦，示例如下：

```c++
std::vector<int> v;
…
unsigned sz = v.size();
/*v.size()的标准返回类型是std::vector<int>::size_type
在Windows 32-bit上std::vector<int>::size_type和unsigned是一样的大小，
但是在Windows 64-bit上std::vector<int>::size_type是64位，unsigned是32位*/
```

另一个auto效率优势的情形（大多数开发者都不会注意到的问题）

```c++
std::unordered_map<std::string, int> m;
…
//哈希表中，KEY值是const类型的，即每个元素应该是std::pair<const std::string,int>
//所以这里编译器会对每一个KEY创建一个临时对象，以适配std::pair<std::string,int>
//但是，使用auto就不会出现这样的问题
for(const std::pair<std::string, int>& p : m)
{
    …                                   //用p做一些事
}
//使用auto改写，当使用对象的地址时，上述示例会将指针指向一个临时变量，而auto声明则不会
for(const auto& p : m)
{
    …                                   //如之前一样
}
```

### 条款6：当auto推导出期望之外的类型时，使用显式类型声明

情形一涉及STL源码涉及，在`vector<bool>`中，容器中的bool并不是常规类型bool，只是行为和bool相同。

此时，使用auto就会出现未定义行为：

`vector<bool>`声明如下：

```c++
namespace std{                                  //来自于C++标准库
    template<class Allocator>
    class vector<bool, Allocator>{
    public:
        …
        class reference { … };//代理类，模仿或者增强bool类型

        reference operator[](size_type n);//vector模板中返回的一般是T&，而这里是代理类
        …
    };
}
```



```c++
std::vector<bool> features(const Widget& w);
Widget w;
…
bool highPriority = features(w)[5];     //w高优先级吗？
…
processWidget(w, highPriority);         //根据它的优先级处理w
//如果使用auto则会出现未定义行为
auto highPriority = features(w)[5];     //w高优先级吗？
//vector<bool>中的[]返回的是std::vector<bool>::reference，行为类似bool的一个内嵌代理类！！！
processWidget(w,highPriority);          //未定义行为！

//解决方法是使用强制类型转换对代理类进行转换
auto highPriority = static_cast<bool>(features(w)[5]);
```

> 其他收获：`std::vector<bool>::reference`是一种代理类，代理类实际上是对原始类的模仿或者增强。即的`Proxy`设计模式。
>
> 只能指针就是使用了代理类
>
> 工作中自定义的`CString`类就可以看作一钟代理类！无差别替换并且支持替换`wstring`

### 第二章总结

> 综上auto用法，以及《C++20高级编程》中推荐使用统一初始化可以总结出：在局部闭包情况下，或者在确定无异常行为下，可以使用auto。
>
> 想要强制auto推导出自己想要的类型，可以使用强制类型转化
>
> 说人话就是：确定情况下，偷个懒用auto就行了。

## 第三章 移步现代C++

### 条款7：区别使用()和{}创建对象

> 初始化中使用"="可能会误导C++新手，使他们以为这里发生了赋值运算，然而实际并没有。

##### 统一初始化的优势

`C++11`使用统一初始化来解决初始化行为的“混乱”，即可以在任何涉及初始化的地方使用统一初始化。

而统一初始化的语法就是花括号{}

需要注意的是，统一初始化不允许隐式的*变窄转化（narrowing conversion）*，示例如下：

```c++
double x, y, z;
int sum1{ x + y + z };          //错误！double的和可能不能表示为int
```

同时，统一初始化可以对*最令人头疼的解析most vexing parse*天生具有免疫性。示例如下：

```c++
Widget w2();                    //最令人头疼的解析！声明一个函数w2，返回Widget
Widget w3{};                    //调用没有参数的构造函数构造对象
```

##### 统一初始化的弊端

+ 如条款2中提到的问题，`auto`和`std::initializer_list`可以说是“天敌”关系

+ 当类的构造函数参数中存在`std::initializer_list`时，优先级高于任何匹配，只要实参可以隐式转化为`std::initializer_list`中的任意类型，都会调用包含`std::initializer_list`形参的构造函数，即使可能报错。

+ 对于上一条弊端存在一个*edge case*,即空的{}作为实参，会调用默认的构造函数，因为空的{}不会构造为`std::initializer_list`,空的{}就意味着没有实参。

+ 如果你**想**用空`std::initializer`来调用`std::initializer_list`构造函数，你就得创建一个空花括号作为函数实参——把空花括号放在圆括号或者另一个花括号内来界定你想传递的东西。

  ```c++
  Widget w4({});                  //使用空花括号列表调用std::initializer_list构造函数
  Widget w5{{}};                  //同上
  ```

+ 对于最常见的容器`vector`，统一初始化会更令人费解

  ```c++
  std::vector<int> v1(10, 20);    //使用非std::initializer_list构造函数
                                  //创建一个包含10个元素的std::vector，
                                  //所有的元素的值都是20
  std::vector<int> v2{10, 20};    //使用std::initializer_list构造函数
                                  //创建包含两个元素的std::vector，
                                  //元素的值为10和20
  ```

  ##### 个人总结：

  + 尽量不要有`std::initializer_list`构造函数
  + 对于数值类型的`std::vector`来说使用花括号初始化和圆括号初始化会造成巨大的不同
  + 常规的初始化可以使用统一初始化，和auto一样，适量很重要。这都需要经验积累

### 条款8：优先考虑nullptr而不是0或者NULL

0或者NULL都不是指针类型，虽然nullptr也不是真正的指针类型，但是可以将nullptr（std::nullptr_t）认为是所有类型的空指针，因此可以避免很多南辕北辙和涉及auto的代码阅读性的问题：

```c++
void f(int);        //三个f的重载函数
void f(bool);
void f(void*);
f(0);               //调用f(int)而不是f(void*)
f(NULL);            //可能不会被编译，一般来说调用f(int)，
                    //绝对不会调用f(void*)
f(nullptr);         //调用重载函数f的f(void*)版本


auto result = findRecord( /* arguments */ );
if (result == 0) {//函数方法返回是什么？
    …
} 

auto result = findRecord( /* arguments */ );
if (result == nullptr) {//原来是指针
    …
} 
```

使用nullptr对于模板编程也十分有利

```c++
//不使用模板编程
int    f1(std::shared_ptr<Widget> spw);     //只能被合适的
double f2(std::unique_ptr<Widget> upw);     //已锁互斥量
bool   f3(Widget* pw);                      //调用
std::mutex f1m, f2m, f3m;       //用于f1，f2，f3函数的互斥量

using MuxGuard =                //C++11的typedef，参见Item9
    std::lock_guard<std::mutex>;
…

{  
    MuxGuard g(f1m);            //为f1m上锁
    auto result = f1(0);        //向f1传递0作为空指针
}                               //解锁 
…
{  
    MuxGuard g(f2m);            //为f2m上锁
    auto result = f2(NULL);     //向f2传递NULL作为空指针
}                               //解锁 
…
{
    MuxGuard g(f3m);            //为f3m上锁
    auto result = f3(nullptr);  //向f3传递nullptr作为空指针
}                               //解锁 
/*这里都可以正常进行函数调用*/
//但是如果为了减少上述上锁并调用函数的代码冗余，使用模板优化的话
//c++11版本
template<typename FuncType,
         typename MuxType,
         typename PtrType>
auto lockAndCall(FuncType func,                 
                 MuxType& mutex,                 
                 PtrType ptr) -> decltype(func(ptr))
{
    MuxGuard g(mutex);  
    return func(ptr); 
}
//c++14版本
template<typename FuncType,
         typename MuxType,
         typename PtrType>
decltype(auto) lockAndCall(FuncType func,       //C++14
                           MuxType& mutex,
                           PtrType ptr)
{ 
    MuxGuard g(mutex);  
    return func(ptr); 
}
//此时调用代码就会出错，模板类型推导会将NULL和0推到为实际类型，而不是空指针
auto result1 = lockAndCall(f1, f1m, 0);         //错误！
...
auto result2 = lockAndCall(f2, f2m, NULL);      //错误！
...
auto result3 = lockAndCall(f3, f3m, nullptr);   //没问题
/*该实例的作用就是为了说明，统一使用nullptr的好处
在实际编程中应该尽量使用nullptr，而不是NULL和0*/
```

**记住**

- 优先考虑`nullptr`而非`0`和`NULL`
- 避免重载指针和整型

### 条款9：优先考虑别名而不是typedefs

别名（alias）的语法即：`using name = type`,比如：

```c++
using INT = int;
//等价于
typedef int INT;
```

当声明一个*函数指针*的时候，别名声明可读性更强

```c++
//FP是一个指向函数的指针的同义词，它指向的函数带有
//int和const std::string&形参，不返回任何东西
typedef void (*FP)(int, const std::string&);    //typedef
//含义同上
using FP = void (*)(int, const std::string&);   //别名声明
```

别名声明可以被模板化，而typedef则不可以。考虑一个链表的别名情形：

```c++
//使用别名制作模板
template<typename T>                            //MyAllocList<T>是
using MyAllocList = std::list<T, MyAlloc<T>>;   //std::list<T, MyAlloc<T>>
                                                //的同义词

MyAllocList<Widget> lw;                         //用户代码

//使用typedef制作模板
template<typename T>                            //MyAllocList<T>是
struct MyAllocList {                            //std::list<T, MyAlloc<T>>
    typedef std::list<T, MyAlloc<T>> type;      //的同义词  
};

MyAllocList<Widget>::type lw;                   //用户代码
```

如果在模板内使用typedef声明一个链表对象，而这个对象又使用了模板形参，此时该对象的类型就称之为*依赖类型*，依赖类型前面必须加上typename：

```c++
template<typename T>                            //MyAllocList<T>是
struct MyAllocList {                            //std::list<T, MyAlloc<T>>
    typedef std::list<T, MyAlloc<T>> type;      //的同义词  
};
template<typename T>
class Widget {                              //Widget<T>含有一个
private:                                    //MyAllocLIst<T>对象
    typename MyAllocList<T>::type list;     //作为数据成员
    …
}; 
//MyAllocList<T>::type因为依赖模板参数T，所以称之为依赖类型，依赖类型名前面必须加上typename

//但是使用别名就没必要这么麻烦
template<typename T> 
using MyAllocList = std::list<T, MyAlloc<T>>;   //同之前一样

template<typename T> 
class Widget {
private:
    MyAllocList<T> list;                        //没有“typename”
    …                                           //没有“::type”
};
```

在模板元编程中，会遇到这种情况：取模板的类型，然后基于它创建另一种类型（可能是const以及引用的区别），这时候可以使用`<type_traits>`,比如：

```c++
std::remove_const<T>::type          //从const T中产出T
std::remove_reference<T>::type      //从T&和T&&中产出T
std::add_lvalue_reference<T>::type  //从T中产出T&
```

> 注意类型转换尾部的`::type`。如果你在一个模板内部将他们施加到类型形参上（实际代码中你也总是这么用），你也需要在它们前面加上`typename`

为什么这里标准库也使用typedef而不是using，是因为历史原因。在C++14中，则提供了using的版本，但是完全是另一种模板了，在后缀上加了`_t`,如：

```c++
std::remove_const<T>::type          //C++11: const T → T 
std::remove_const_t<T>              //C++14 等价形式

std::remove_reference<T>::type      //C++11: T&/T&& → T 
std::remove_reference_t<T>          //C++14 等价形式

std::add_lvalue_reference<T>::type  //C++11: T → T& 
std::add_lvalue_reference_t<T>      //C++14 等价形式
    
//如果编译器不支持C++14，可以自定义别名：
template <class T> 
using remove_const_t = typename remove_const<T>::type;

template <class T> 
using remove_reference_t = typename remove_reference<T>::type;

template <class T> 
using add_lvalue_reference_t =
  typename add_lvalue_reference<T>::type; 
```

### 条款10： 有限考虑限域enum而非未限域enum

+ 所谓的限域和未限域，在《C++20高级编程》中对应的就是新式的enum class声明和旧式的enum声明

  > 使用新式的声明可以减少命名污染，但是使用相对麻烦，因此可以使用using进行简化。
  >
  > 但是使用using简化，会再次导致命名污染，所以尽量将using使用在局部，如switch中

+ 新式的声明式强类型的枚举类，而旧式声明则会隐式转换为整形，如：

```c++
enum Color { black, white, red };       //未限域enum

std::vector<std::size_t>                //func返回x的质因子
  primeFactors(std::size_t x);

Color c = red;
…

if (c < 14.5) {                         // Color与double比较 (!)
    auto factors =                      // 计算一个Color的质因子(!)
      primeFactors(c);
    …
}
//推荐使用新式声明
enum class Color { black, white, red }; //Color现在是限域enum

Color c = Color::red;                   //和之前一样，只是
...                                     //多了一个域修饰符

if (c < 14.5) {                         //错误！不能比较
                                        //Color和double
    auto factors =                      //错误！不能向参数为std::size_t
      primeFactors(c);                  //的函数传递Color参数
    …
}
//对于新式声明，如果确实需要和数值类型进行计算等，可以使用强制类型转化
if (static_cast<double>(c) < 14.5) {    //奇怪的代码，
                                        //但是有效
    auto factors =                                  //有问题，但是
      primeFactors(static_cast<std::size_t>(c));    //能通过编译
    …
}
```

+ 《C++20高级编程》第一章没有提及的另一个好处是：新式声明可以被前置声明（这个问题在工作中遇到过），如：

  ```c++
  enum Color;         //错误！
  enum class Color;   //没问题
  ```

  > 这里存在一个误导，即使旧式的enum也是可以前置声明的。因为编译器总是选择可以存储所有枚举成员的最小的数据类型作为枚举体的基础类型，因此，如果不指明枚举体的基础数据类型，仅有前置声明，编译器会手足无措。对此，可以这么做来实现旧式的前置声明：
  >
  > ```c++
  > enum Color: std::uint8_t;   //非限域enum前向声明
  >                             //底层类型为
  >                             //std::uint8_t
  > ```

+ 旧式声明唯一的好处是在涉及到`std::tuple`的时候，可读性会好一些，但是仍推荐新式声明。实例如下：

  ```c++
  using UserInfo =                //类型别名，参见Item9
      std::tuple<std::string,     //名字
                 std::string,     //email地址
                 std::size_t> ;   //声望
  //旧式声明
  enum UserInfoFields { uiName, uiEmail, uiReputation };
  
  UserInfo uInfo;                         //同之前一样
  …
  auto val = std::get<uiEmail>(uInfo);    //啊，获取用户email字段的值
  //新式声明
  enum class UserInfoFields { uiName, uiEmail, uiReputation };
  
  UserInfo uInfo;                         //同之前一样
  …
  auto val =
      std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>
          (uInfo);
  //如果该情形出现频繁，可以使用封装一个函数来简化该操作
  //1、因为std::get是一个模板，需要std::size_t值的模板实参,因为需要在编译器直到这个值，所以封装的函数必须是一个constexpr函数
  //2、为了泛化性更好，我们的函数希望对于所有的枚举都有效，相对于返回std::size_t，我们更希望模板函数返回的是枚举类的底层类型
  
  //对于C++11可以通过std::underlying_type这个type trait获得，对于constexpr我们大多要加上noexcept，因此可以这么写：
  template<typename E>
  constexpr typename std::underlying_type<E>::type
      toUType(E enumerator) noexcept
  {
      return
          static_cast<typename
                      std::underlying_type<E>::type>(enumerator);
  }
  
  //对于C++14，利用条款9提到的using版本的type trait，可以如此优化
  template<typename E>                //C++14
  constexpr std::underlying_type_t<E>
      toUType(E enumerator) noexcept
  {
      return static_cast<std::underlying_type_t<E>>(enumerator);
  }
  //由因为C++14允许auto作为函数返回值，因此，可以进一步优化：
  template<typename E>                //C++14
  constexpr auto
      toUType(E enumerator) noexcept
  {
      return static_cast<std::underlying_type_t<E>>(enumerator);
  }
  
  
  //这时候，使用std::get就可以这么写：
  auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
  ```

  > 尽管仍然很长，但是新式声明可以防止隐式转换，同时可以避免命名污染。
  >
  > 因此，无论何时，最好使用新式声明
