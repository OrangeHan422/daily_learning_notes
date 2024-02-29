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

别名声明可以被模板化，而typedef则不可以