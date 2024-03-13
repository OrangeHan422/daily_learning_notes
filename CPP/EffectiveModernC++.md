# Effective Modern C++

> TODO:
>
> + 二刷时，条款24之前的所有引用折叠，更换为引用压缩(*reference collapsing*)，更官方。
> + 对于条款20+的内容，尤其涉及右值引用，移动操作结合type trait时，待结合实操，模板理解更深刻时重刷，一刷仅通过引用压缩大概理解，底层逻辑模糊。

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

### 条款10： 优先考虑限域enum而非未限域enum

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

### 条款11：优先考虑使用deleted函数而非使用未定义的私有声明

主要是针对C++98的改进。旧的C++如果对拷贝构造函数和拷贝赋值函数运算符设置权限主要是将其设置为私有的，而新式C++推荐使用`delete`。比如，输入输出流的基类`basic_ios`的定义(单例模式)：

```c++
//C++98的定义，利用私有化防止访问
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
    …

private:
    basic_ios(const basic_ios& );           // not defined
    basic_ios& operator=(const basic_ios&); // not defined
};
//C++11使用= delete标记函数为deleted function
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
    …

    basic_ios(const basic_ios& ) = delete;
    basic_ios& operator=(const basic_ios&) = delete;
    …
};
```

*deleted*函数不能以任何方式被调用，即使你在成员函数或者友元函数里面调用*deleted*函数也不能通过编译（注意：`private`就会存在这些问题，即类内可调用，友元可调用）。

另一点是，将deleted函数设置为`public`的好处是，编译器可以产生更好的错误信息

`delete`可以用于任何函数。比如，在防止C的隐式转换时：

```c++
bool isLucky(int number);
if (isLucky('a')) …         //字符'a'是幸运数？
if (isLucky(true)) …        //"true"是?
if (isLucky(3.5)) …         //难道判断它的幸运之前还要先截尾成3？
bool isLucky(int number);       //原始版本
bool isLucky(char) = delete;    //拒绝char
bool isLucky(bool) = delete;    //拒绝bool
bool isLucky(double) = delete;  //拒绝float和double
```

再比如，**禁止一些模板的实例化**：

```c++
//假如模板函数接收指针参数，但是拒绝void *和char *类型
template<typename T>
void processPointer(T* ptr);
//注意，这里是模板特化的形式---------->记得回顾现代C++模板编程！
template<>
void processPointer<void>(void*) = delete;

template<>
void processPointer<char>(char*) = delete;

template<>
void processPointer<const void>(const void*) = delete;

template<>
void processPointer<const char>(const char*) = delete;
```

**总结**：坚定不移的使用`delete`而不是`private`私有化。因为你有的我都有，我有点你没有！

### 条款12： 使用override声明重写函数

> 首先需要注意的是，重写（*overriding*）不是重载（*overloading*）！！
>
> 重写：唯一情况，虚函数在派生类被重写。
>
> 重载：同名函数，参数不同的版本可以称之为重载，对虚函数，普通函数等都适用

重写的经典实例：

```c++
class Base {
public:
    virtual void doWork();          //基类虚函数
    …
};

class Derived: public Base {
public:
    virtual void doWork();          //重写Base::doWork
    …                               //（这里“virtual”是可以省略的）
}; 

std::unique_ptr<Base> upb =         //创建基类指针指向派生类对象
    std::make_unique<Derived>();    //关于std::make_unique
…                                   //请参见Item21

upb->doWork();                      //通过基类指针调用doWork，
                                    //实际上是派生类的doWork
                                    //函数被调用
```

重写一个函数，必须满足下列要求：

+ 基类函数必须是`virtual`

+ 基类和派生类的函数名必须完全一样（除非是析构函数）

  > 实际上，析构函数也是一样的，因为所有的析构函数，编译器都会将其转换为destroy()函数

+ 基类和派生类的形参类型必须完全一样（和重载的重要区别）

+ 基类和派生类的常量性`constness`必须完全一样

+ 基类和派生类函数的返回值和异常说明（*exception specifications*）必须**兼容**

+ *(C++11新约束)*：函数的引用限定符(reference qualifiers)必须完全一样。成员函数的引用限定符可以限定函数只能用于左值或者右值，即使非虚函数也可以使用：

  ```c++
  class Widget {
  public:
      …
      void doWork() &;    //只有*this为左值的时候才能被调用
      void doWork() &&;   //只有*this为右值的时候才能被调用
  }; 
  …
  Widget makeWidget();    //工厂函数（返回右值）
  Widget w;               //普通对象（左值）
  …
  w.doWork();             //调用被左值引用限定修饰的Widget::doWork版本
                          //（即Widget::doWork &）
  makeWidget().doWork();  //调用被右值引用限定修饰的Widget::doWork版本
                          //（即Widget::doWork &&）
  ```

那么为什么要使用`override`呢？因为如果没有`override`修饰，那么在派生类中“重写”基类虚函数时，如果没有遵守上述条款，依旧可以通过编译，并且编译器不会告诉你任何warning。即，你以为你重写了，但是你没有，编译器也不知道你是在重写，还是在覆盖。实例如下：

```c++
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
};
//派生类均未重写，但是可以通过编译
class Derived: public Base {
public:
    virtual void mf1();
    virtual void mf2(unsigned int x);
    virtual void mf3() &&;
    void mf4() const;
};
//为了确认是否正确的重写，应该加上override，这样出现错误就不能通过编译
class Derived: public Base {
public:
    virtual void mf1() override;
    virtual void mf2(unsigned int x) override;
    virtual void mf3() && override;
    virtual void mf4() const override;
};
```

C++11引入了两个上下文关键字*contextual keywords*），`override`和`final`（向虚函数添加`final`可以防止派生类重写。`final`也能用于类，这时这个类不能用作基类）。这两个关键字区别于一般的关键字，是可以当做名称使用的，因为他们仅在特定的位置才会别解析为关键字。

对于引用限定，虽然很少用，但是对于C++程序员来说，凡是可以优化性能的点，都有必要了解，考虑以下情形：

```c++
class Widget {
public:
    using DataType = std::vector<double>;  
    …
    DataType& data() { return values; }//仅提供了返回左值引用的成员方法
    …
private:
    DataType values;
};
//对于左值调用，没有问题
Widget w;
…
auto vals1 = w.data();  //拷贝w.values到vals1
//但是对于临时对象(右值)调用，编译器会将临时对象的值拷贝一次。
//但是，右值完全可以使用移动来提高性能，减少拷贝
Widget makeWidget();
auto vals2 = makeWidget().data();   //拷贝Widget里面的值到vals2

//因此，应该做如下修改：
class Widget {
public:
    using DataType = std::vector<double>;
    …
    DataType& data() &              //对于左值Widgets,
    { return values; }              //返回左值
    
    DataType data() &&              //对于右值Widgets,
    { return std::move(values); }   //返回右值
    …

private:
    DataType values;
};
```

### 条款13： 有限const_iterator而不是iterator

> 仍然是《Effective C++》中的条款，即尽可能的使用const属性，可以使代码更加稳定和健壮

`C++11`中使用实例如下：

```c++
std::vector<int> values;                                //和之前一样
…
auto it =                                               //使用cbegin
    std::find(values.cbegin(), values.cend(), 1983);//和cend
values.insert(it, 1998);
```

但是`C++11`标准存在一个疏漏，那就是没有提供非成员函数(*free function*)版本的cbegin()和cend()，那么在模板编程中就会存在一些障碍;（在`C++14`中进行了修正）

```c++
template<typename C, typename V>
void findAndInsert(C& container,            //在容器中查找第一次
                   const V& targetVal,      //出现targetVal的位置，
                   const V& insertVal)      //然后在那插入insertVal
{
    using std::cbegin;
    using std::cend;
	//为什么使用非成员版本的cbegin,cend,是为了考虑非容器的情况
    auto it = std::find(cbegin(container),  //非成员函数cbegin
                        cend(container),    //非成员函数cend
                        targetVal);
    container.insert(it, insertVal);
}
//以上代码在C++14中没有任何问题，但是如果想在C++11中支持，需要自定义cbegin,cend
template <class C>
auto cbegin(const C& container)->decltype(std::begin(container))
{
    return std::begin(container);   //解释见下
}
//注意，const的限定是以模板参数的形式展现的，即const容器调用非成员函数begin将产生出const_iterator
//对于C数组，C++11提供了特化版本，返回的是pointer-to-const
```

### 条款14： 如果函数不抛出异常，请使用noexcept

`noexcept`和`const`一样重要，如果确定函数不会抛出异常，就声明为`noexcept`。

```c++
int f(int x) throw();   //C++98风格，没有来自f的异常
int f(int x) noexcept;  //C++11风格，没有来自f的异常
```

`noexcept`的好处是，方便排查（比如工作中遇到的sleep_hook问题）。另一个好处是，`noexcept`函数编译器会进行尽可能的优化（比如遇到异常不再执行反序析构），而旧式的声明就不会优化

```c++
RetType function(params) noexcept;  //极尽所能优化
RetType function(params) throw();   //较少优化
RetType function(params);           //较少优化
```

该设计在源码中就有体现，在C++98中vector需要扩容时，是通过将旧元素复制到新内存中实现的，这样做虽然安全（因为只有所有元素都复制完毕，旧元素才析构），但是性能没有移动操作好。所以在C++11中，替换为了移动操作。但是移动操作如果中途出现了异常，就无法保持操作的“原子性”，也无法返回原始状态。所以就使用了`noexcept`(以及move_if_noexcept，较为复杂，暂时忽略)。

**总结**：`noexcept`的设计可能被调用者依赖；同时会带来性能的提升；但是大多数函数都是*异常中立*的，所以不应该使用`noexcept`；

综上，如果没有十足的把握，不要使用，不然可能本末倒置。

### 条款15：尽可能的使用constexpr

constexpr声明的函数或者类，必须在编译期可以计算。constexpr函数只能调用constexpr函数

> 不能假设`constexpr`函数的结果是`const`的，也不能保证他们是在编译期是可知的

所有的`constexpr`对象都是`const`的，但不是所有的`const`对象都是`constexpr`

对于constexpr函数：

+ constexpr函数可以用于需求*编译期常量*的上下文。如果传给constexpr函数的**实参**在编译期可知，那么结果将在编译期计算。如果**实参**的值自编译期不知道，代码将会被拒绝。
+ 当一个constexpr函数被一个或者多个编译期不可知的值调用时，它就像普通函数一样，运行时计算他的结果。这就意味着不需要书写两个函数，constexpr全做了（就像const类型的参数可以接受const和非const参数一样）

简单的实例：

```c++
constexpr                                   //pow是绝不抛异常的
int pow(int base, int exp) noexcept         //constexpr函数
{
 …                                          //实现在下面
}
constexpr auto numConds = 5;                //（上面例子中）条件的个数
std::array<int, pow(3, numConds)> results;  //结果有3^numConds个元素
//但是如果实参不是编译期常量，那么就是一个正常的运行时的函数
auto base = readFromDB("base");     //运行时获取这些值
auto exp = readFromDB("exponent"); 
auto baseToExp = pow(base, exp);    //运行时调用pow函数
```

在C++11中，constexpr函数只能有一行语句，但是C++14解除了这样的限制，比如上述pow函数的实现：

```c++
//c++11
constexpr int pow(int base, int exp) noexcept
{
    return (exp == 0 ? 1 : base * pow(base, exp - 1));
}
//c++14
constexpr int pow(int base, int exp) noexcept   //C++14
{
    auto result = 1;
    for (int i = 0; i < exp; ++i) result *= base;
    
    return result;
}
```

**总结**：尽可能的使用constexpr，就像const一样，constexpr可以最大化对象和函数可以使用的场景；

但是因为客户可能使用constexpr的特性，所以，如果你确定该函数可以在编译期计算出，就使用吧。

> 在C++20中，提出了新的关键字`consteval`,`consteval`可以保证函数只能在编译期使用，而不能在运行期使用---->《C++20高级编程》

### 条款16：让const成员函数线程安全

> 新知识：`mutable`关键字，即使在const函数中也可以改变。该关键字的意思是该变量永远是可变的

主要是涉及`mutable`的情况，可以使用`std::mutex`来进行互斥计算，同时也可以使用`std::atomic`变量，可以比互斥量性能更好。

note：该条款存在错误，且情况罕见（就目前而言）

### 条款17 ： 理解特殊成员函数的生成

特殊成员函数即编译器会在*需要*的时候自动生成的函数：默认构造函数，析构函数，拷贝构造函数，拷贝赋值运算符。生成的函数默认是`public`且`inline`的。现代C++（自C++11开始）加入了移动构造函数和移动赋值函数。

```c++
class Widget {
public:
    …
    Widget(Widget&& rhs);               //移动构造函数
    Widget& operator=(Widget&& rhs);    //移动赋值运算符
    …
};
```

对于默认的移动函数，C++会对逐成员进行“移动”，核心是基于`std::move`的。但是，对于旧的C++类或者不支持移动操作的类，会在函数决议的时候选择复制的操作而不是移动的操作。

有一点需要注意，对于两种拷贝函数，它们是相互独立的，即，如果声明了拷贝构造函数，那么编译器会在需要的时候自动生成拷贝赋值函数。两者没有直接的影响。但是，对于两个移动操作不是相互独立的，如果设声明了一个，另一个编译器并不会自动帮你生成。

原因是，当自定义了移动函数，说明编译器自动生成的逐成员移动操作不一定正确了，所以编译器不再自动生成移动函数。

更进一步的说，如果类显式声明了拷贝函数，那么编译器也不会生成移动操作。根本原因仍然是逐成员移动的原因。

在*Rule of Three*的理论中，如果自定义了析构函数，那么逐成员拷贝或者移动操作都不再使用于该类。

所以综上来说，仅当下述条件成立的时候才会生成移动操作（当需要的时候）：

+ 类没有拷贝操作
+ 类没有移动操作
+ 类没有自定义析构函数

在C++11中对于已经声明了拷贝操作或者析构函数的类 的拷贝操作的自动生成。如果想要继续使用自动生成这个特性，需要添加`= default`关键字，如下：

```c++
class Widget {
    public:
    … 
    ~Widget();                              //用户声明的析构函数
    …                                       //默认拷贝构造函数
    Widget(const Widget&) = default;        //的行为还可以

    Widget&                                 //默认拷贝赋值运算符
        operator=(const Widget&) = default; //的行为还可以
    … 
};
```

综合《Effective C++》，最好的习惯是将Rule of Three 扩展为Rule of Five。即使想要使用编译器自动生成的特殊成员函数，也要通过函数签名+`= default`的形式声明。

## 第四章 智能指针

> 为什么使用智能指针？原始指针太过强大，但是要看自己是否能够驯服。
>
> 智能指针应该是开发中首选的指针，除非你对性能有着痴狂的要求，否则尽量不要使用原始指针。

### 条款18：对于独占资源使用std::unique_ptr

比较令人意外的是，`std::unique_ptr`的大多数操作，和原生的指针的指令是相同的。这就意味着，即使你在内存和时间都比较紧张的情况下使用`std::unique_ptr`。

`std::unique_ptr`是一个可移动(*move_only type*)类型。

最经典的使用场景是作为继承层次结构中对象的工厂函数返回类型。即工厂函数在堆上分配一个对象并返回指针。示例：

```c++
class Investment { … };
class Stock: public Investment { … };
class Bond: public Investment { … };
class RealEstate: public Investment { … };
//工程函数应该声明如下
template<typename... Ts>            //返回指向对象的std::unique_ptr，
std::unique_ptr<Investment>         //对象使用给定实参创建
makeInvestment(Ts&&... params);
//用户调用如下：
{
    …
    auto pInvestment =                  //pInvestment是
        makeInvestment( arguments );    //std::unique_ptr<Investment>类型
    …
}                                       //销毁 *pInvestment
```

另一个使用的场景是，涉及指针的移动时。比如，将工厂函数的返回值放在容器中，再将容器作为一个对象的数据成员，在这种多级传递的情况下，智能指针可以保证对象被正常销毁。除非时类似`std::abort`这种中断时没有释放局部资源，这些情况并不是程序涉及可以控制的。

默认情况下，销毁通过`delete`进行，但是`std::unique_ptr`可以使用**自定义删除器**：当函数销毁的时候，可以调用任何可以调用的对象，包括匿名函数，可调用类，函数等。简答的例子，在删除的时候打印日志：

```c++
auto delInvmt = [](Investment* pInvestment)         //自定义删除器
                {                                   //（lambda表达式）
                    makeLogEntry(pInvestment);
                    delete pInvestment; 
                };

template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)>     //更改后的返回类型
makeInvestment(Ts&&... params)
{
    std::unique_ptr<Investment, decltype(delInvmt)> //应返回的指针
        pInv(nullptr, delInvmt);
    if (/*一个Stock对象应被创建*/)
    {
        pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if ( /*一个Bond对象应被创建*/ )   
    {     
        pInv.reset(new Bond(std::forward<Ts>(params)...));   
    }   
    else if ( /*一个RealEstate对象应被创建*/ )   
    {     
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));   
    }   
    return pInv;
}
```

> 注意代码中auto,decltype,lambda函数的使用，这些都是可以学习的

需要注意的几点：

+ `std::unique_ptr`禁止使用裸指针进行初始化。如果没有完全适应现代C++，那么使用的流程应该是：初始化一个空的智能指针，然后通过reset将new出来的指针赋值给`std::unique_ptr`(更推荐现代化的方式：`std::make_unique`)

+ 通过上述代码可以知道，匿名函数仅对基类指针进行了释放。该方法可行的原因是多态，所以基类析构函数必须为虚函数（《Effective C++》有提及）,即

  ```c++
  class Investment {
  public:
      …
      virtual ~Investment();          //关键设计部分！
      …
  };
  ```

  由于C++14以后`auto`支持函数返回类型的推导，所以工厂函数代码可以优化为：

  ```c++
  template<typename... Ts>
  auto makeInvestment(Ts&&... params)                 //C++14
  {
      auto delInvmt = [](Investment* pInvestment)     //现在在
                      {                               //makeInvestment里
                          makeLogEntry(pInvestment);
                          delete pInvestment; 
                      };
  
      std::unique_ptr<Investment, decltype(delInvmt)> //同之前一样
          pInv(nullptr, delInvmt);
      if ( … )                                        //同之前一样
      {
          pInv.reset(new Stock(std::forward<Ts>(params)...));
      }
      else if ( … )                                   //同之前一样
      {     
          pInv.reset(new Bond(std::forward<Ts>(params)...));   
      }   
      else if ( … )                                   //同之前一样
      {     
          pInv.reset(new RealEstate(std::forward<Ts>(params)...));   
      }   
      return pInv;                                    //同之前一样
  }
  ```
为什么使用匿名函数作为删除器？因为函数指针形式的删除器，会使`std::unique_ptr`的大小从一个字变为两个字（因为存储了裸指针和函数指针）；如果以函数对象的形式作为删除器，大小取决于函数对象中存储的状态有多少。无状态函数对象则对大小没有影响（比如不捕获变量的lambda表达式），所以，在可以使用lambda函数的时候，请尽量使用。

另一种使用`std::unique_ptr`的经典场景是,使用`Pimpl Idiom`,即通过接口隐藏实现,从而减弱编译依赖性的一种设计。《Effective C++》条款31有叙述。

`std::unique_ptr`使用的两种形式：`std::unique_ptr<T>`和指向数组的`std::unique_ptr<T[]>`,但是，对于第二种用法，现代C++程序员应该尽量不适用，因为有更好的`std::array`等替换方案。

`std::unique_ptr`更吸引人的功能之一是，它可以轻松高效的转化为`std::shared_ptr`（类似编译器中的隐式转换），比如：

```c++
std::shared_ptr<Investment> sp =            //将std::unique_ptr
    makeInvestment(arguments);              //转为std::shared_ptr
```

### 条款19：对于共享资源使用std::shared_ptr

引用计数暗示性能问题：

+ `std::shared_ptr`大小是原始指针大小的两倍：两个指针，裸指针，和指向引用计数的指针
+ 引用计数的内存必须是动态分配的
+ 引用计数的递增递减都必须是原子的（多线程环境下尤为重要）

`std::shared_ptr`构造函数**通常**递增引用计数。这里的通常主要是因为移动构造函数的存在。当从另一个`std::shared_ptr`移动构造新的`std::shared_ptr`时，原`std::shared_ptr`会置为空，同时由新的`std::shared_ptr`接管资源，这个时候引用计数是不需要增加的。

虽然`std::shared_ptr`也提供删除器的操作，但是和`std::unique_ptr`不同的是，删除器在`std::shared_ptr`中并不是智能指针的一部分。共享指针的删除器更加的灵活，使用实例如下：

```c++
auto loggingDel = [](Widget *pw)        //自定义删除器
                  {                     //（和条款18一样）
                      makeLogEntry(pw);
                      delete pw;
                  };

std::unique_ptr<                        //删除器类型是
    Widget, decltype(loggingDel)        //指针类型的一部分
    > upw(new Widget, loggingDel);
std::shared_ptr<Widget>                 //删除器类型不是
    spw(new Widget, loggingDel);        //指针类型的一部分
//对于同一类的只能指针，可以对不同的对象定制化删除器
auto customDeleter1 = [](Widget *pw) { … };     //自定义删除器，
auto customDeleter2 = [](Widget *pw) { … };     //每种类型不同
std::shared_ptr<Widget> pw1(new Widget, customDeleter1);
std::shared_ptr<Widget> pw2(new Widget, customDeleter2);
```

这样做和唯一指针的区别是，尽管删除器不同，不同的共享指针也可以互相赋值。但是唯一指针就不可以，因为唯一指针中，删除器作为了唯一指针的成员。

另一个不同的是，自定义删除器不会改变共享指针的大小。一个共享指针永远是**两个指针**大小。

**注意**：两个指针指向的内容（奇怪的作者），该条款前面说的第二个指针指向引用计数不正确，应该指向的是一个控制块。控制块中存储的有引用计数，次级引用计数（*weak count*）,以及其他数据（包括但不限于自定义删除器，分配器，甚至还有虚函数相关的东西）

控制块创建的原则：

+ `std::make_shared`(item 21)总是创建一个控制块。它创建一个要只想的新对象，所以可以确定`std::make_shared`调用时的对象没有控制块
+ 当从唯一指针转换为共享指针的时候，会创建控制块
+ 当从原始指针上创建共享指针的时候会创建控制块

注意最后一条会造成的错误，当一个裸指针被用来创建多个共享指针时，会有多个控制块。考虑以下情形：

```c++
auto pw = new Widget;                           //pw是原始指针
…
std::shared_ptr<Widget> spw1(pw, loggingDel);   //为*pw创建控制块
…
std::shared_ptr<Widget> spw2(pw, loggingDel);   //为*pw创建第二个控制块
```

这是因为有两个控制块的存在，所以就会出现未定义行为

解决方案有两个：

+ 使用`std::make_shared`构建函数原始指针，但是定义了自定义删除器时，`std::make_shared`就无能为力了

+ 如果必须传给共享指针原始指针来构造，那么最好直接将new的结果传入，再使用共享指针之间的赋值进行赋值。即代码应该做出如下更改：

  ```
  std::shared_ptr<Widget> spw1(new Widget,    //直接使用new的结果
                               loggingDel);
  std::shared_ptr<Widget> spw2(spw1);         //spw2使用spw1一样的控制块

另一个特殊情形是，直接传递this指针给共享指针，考虑下述情形：

```c++
std::vector<std::shared_ptr<Widget>> processedWidgets;//用来记录已经处理过的Widget
class Widget {
public:
    …
    void process();
    …
};
void Widget::process()
{
    …                                       //处理Widget
    processedWidgets.emplace_back(this);    //然后将它加到已处理过的Widget
}                                           //的列表中，这是错的！
//如果外部已经有共享指针管理对象了呢，这个时候直接使用this构造（emplace_back）就会导致存在两个控制块。
```

对于该情形，C++已经提供了处理该情形的措施，就是`std::enable_shared_from_this`，使用实例：

```c++
class Widget: public std::enable_shared_from_this<Widget> {
public:
    …
    void process();
    …
};
void Widget::process()
{
    //和之前一样，处理Widget
    …
    //把指向当前对象的std::shared_ptr加入processedWidgets
    processedWidgets.emplace_back(shared_from_this());
}
```

`std::enable_shared_from_this`模板类的设计模式称之为奇异递归模板模式（*The Curiously Recurring Template Pattern*（*CRTP*）），心脏能力不好，不建议深究

从内部来说，`shread_from_this`是先找到对象的控制块，然后创建新的共享指针关联这个控制块。所以，该函数的使用前提是外部已经有共享指针对对象进行了管理。

为了防止客户端使用时出现上述问题，解决方案是将类的构造函数设置为`private`的，并通过返回共享指针的工厂函数进行创建对象，如（有点像工作中自己定义的MFC的日期类）：

```c++
class Widget: public std::enable_shared_from_this<Widget> {
public:
    //完美转发参数给private构造函数的工厂函数
    template<typename... Ts>
    static std::shared_ptr<Widget> create(Ts&&... params);
    …
    void process();     //和前面一样
    …
private:
    …                   //构造函数
};
```

共享指针虽然比唯一指针的性能差，但是对于相应的场景来说，这些开销是可以忍受的。如果不需要考虑共享性的问题，可以直接使用唯一指针，否则使用共享指针比裸指针加互斥，或者计数更高效和安全。

需要注意的是，共享指针和控制块是不离不弃，同生共死的，所以共享指针不可以转换为唯一指针。

共享指针针对的是单个对象，也就意味着不支持对C风格的数组进行管理。如果需要数组，请使用`std::array`

### 条款20： 当std::shared_ptr可能悬空时使用std::weak_ptr

注意，`std::weak_ptr`并不完全属于一种智能指针，它只是`std::shared_ptr`的增强，它没有解引用等操作

`std::weak_ptr`通常从`std::shared_ptr`上创建。当从`std::shared_ptr`上创建`std::weak_ptr`时两者指向相同的对象，但是`std::weak_ptr`不会影响所指对象的引用计数：

```c++
auto spw =                      //spw创建之后，指向的Widget的
    std::make_shared<Widget>(); //引用计数（ref count，RC）为1。
                                //std::make_shared的信息参见条款21
…
std::weak_ptr<Widget> wpw(spw); //wpw指向与spw所指相同的Widget。RC仍为1
…
spw = nullptr;                  //RC变为0，Widget被销毁。
                                //wpw现在悬空
```

悬空的`std::weak_ptr`是过期的(*expired*):

```c++
if (wpw.expired()) …            //如果wpw没有指向对象…
```

如果想通过`std::weak_ptr`构造共享指针，可以使用成员函数lock(),而不应该直接将弱指针作为共享指针的构造函数的参数：

```c++
std::shared_ptr<Widget> spw1 = wpw.lock();  //如果wpw过期，spw1就为空
auto spw2 = wpw.lock();                     //同上，但是使用auto

std::shared_ptr<Widget> spw3(wpw);          //如果wpw过期，抛出std::bad_weak_ptr异常
```

什么时候使用`std::weak_ptr`?

**情形一**：一个工厂函数，基于一个唯一ID从只读对象中产出智能指针：

```c++
std::unique_ptr<const Widget> loadWidget(WidgetID id);
```

如果该函数的调用涉及数据库或者文件操作，并且重复使用ID的场景很常见，一个合理的优化是使用缓存设计，并且在使用完毕后释放缓存。

对于缓存， 唯一指针不是最佳选择，因为用户需要接受缓存的智能指针，同时也要知道缓存的生命周期（即用户在使用结束后去释放缓存）。缓存指针本身也需要一个指针来管理缓存的对象，并且需要知道缓存指针是否悬空，因为客户端使用缓存对象后并释放缓存，关联的缓存条目就会悬空。

综上，应该使用`std::weak_ptr`来关联缓存对象，而工厂函数的返回类型则应该是`std::shared_ptr`,因为只有对象被`std::shared_ptr`管理时，`std::weak_ptr`才能检测是否悬空。下面是一个简答的示例：

```c++
std::shared_ptr<const Widget> fastLoadWidget(WidgetID id)
{
    static std::unordered_map<WidgetID,
                              std::weak_ptr<const Widget>> cache;
                                        //译者注：这里std::weak_ptr<const Widget>是高亮
    auto objPtr = cache[id].lock();     //objPtr是去缓存对象的
                                        //std::shared_ptr（或
                                        //当对象不在缓存中时为null）

    if (!objPtr) {                      //如果不在缓存中
        objPtr = loadWidget(id);        //加载它
        cache[id] = objPtr;             //缓存它
    }
    return objPtr;
}
```

> 题外话，自定义类的哈希，以及相等性函数都没有给出

**情形二**：观察者设计模式（Observer design pattern）

观察者设计模式一般有两种组件：subjects(状态可能会更改的对象)和observers(状态发生更改时要通知的对象)。

在大多数场景下,subject对象会有一个指向observer的指针，方便发布状态更改的通知。subject对observer的生命周期没有兴趣，唯一关心的是observer被销毁后，就不再访问。因此，合理的设计是使用`std::weak_ptr`指向observer

**情形三**：环装结构

考虑下述场景：A -------->B<-------------C，即A和C都通过共享指针指向B

此时，B需要一个指针指向A，可能的选择以及结果有三种:

+ 原始指针：如果A销毁了，那么B会指向一个悬空的指针，但是B不知道，可能会继续访问，导致未定义行为。
+ 共享指针：环状结构，AB将永远无法释放
+ 弱指针：正确的选择，因为及时A释放了，B是可以通过是否悬空来判断A是否还有效

实际上，`std::weak_ptr`不影响引用计数只是为了方便理解，在上个条款提到的控制块中是存在次级引用的，次级引用就是针对`std::weak_ptr`设计的

### 条款21：有限考虑使用std::make_shared,std::make_unique而非new

前置：C++14才引入的`std::make_unique`,对于C++11，可以自己实现基础版本

```c++
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params)
{
    return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
```

总共有**三个make函数**：`std::make_shared`和`std::make_unique`都是接受任意个参数，然后*完美转发*到构造函数去动态分配一个对象，然后返回这个对象的指针。第三个make函数是`std::allocate_shared`和`std::make_shared`作用是一样的，但是第一个参数是一个用来动态分配内存的`allocater`

使用make函数的原因：

**原因一**：一个简单的例子区别使用与否make的差别：

```c++
auto upw1(std::make_unique<Widget>());      //使用make函数
std::unique_ptr<Widget> upw2(new Widget);   //不使用make函数,没有办法使用auto
auto spw1(std::make_shared<Widget>());      //使用make函数
std::shared_ptr<Widget> spw2(new Widget);   //不使用make函数,没有办法使用auto
```

可以看到使用make相对简洁一些，并且避免了重复写类型导致的编译器的额外工作。

**原因二**：使用make函数和异常安全有关。考虑下述场景：

```c++
//假设该函数按照某种优先级处理widget
void processWidget(std::shared_ptr<Widget> spw, int priority);
//有一个函数计算相关的优先级
int computePriority();
//并且我们在调用的时候使用new来创建共享指针
processWidget(std::shared_ptr<Widget>(new Widget),  //潜在的资源泄漏！
              computePriority());
```

为什么会存在资源泄漏的可能呢？原因是编译器将源码转换为目标代码的时候，一个函数的实参必须先被计算，然后才是函数调用。所以在执行processWidget函数前，会执行以下操作：

+ 表达式`new Widget`必须计算
+ 负责管理`new`出来的共享指针的构造函数必须被执行
+ `computePriority`必须运行

但是`computePriority`和new以及共享指针的构造函数的执行顺序是不定的（就像多线程执行一样，没有存在竞态关系），所以如果在执行`computePriority`时，产生了异常，那么`new`出来的资源可能还没来得及被共享指针管理，也就没办法释放了。

而使用`std::make_shared`相当于将new和共享指针的构造函数包在了一起，所以就算`make_shared`和`computePriority`任意一个函数抛出了异常，内存资源也可以得到妥善的释放。

同样的，对于`std::make_unique`也是适用的。这些事情在编写**异常安全**的代码的时候十分重要。

**原因三**

直接使用`std::make_shared`和使用`new`相比会有效率上的提升。使用`std::make_shared`允许编译器生成更小更快的代码，并使用更简洁的数据结构。考虑下属情形：

```c++
std::shared_ptr<Widget> spw(new Widget);
```

这样的调用实际上分配了两次内存：一次是`new`，另一次则是在共享指针构造函数中创建的控制块。如果使用`std::make_shared`：

```c++
auto spw = std::make_shared<Widget>();
```

程序只进行了一次内存分配，同时容纳了对象和控制块。这种优化也减小的程序的静态大小，也提高了可执行代码的速度，也可以减少程序的内存占用。

对于`std::make_shared`的效率分析同样适用`std::allocate_shared`

*既然`make`函数这么好，为什么条款中说的是优先，而不像`constexpr`那样尽可能的使用呢？*

**限制一**：`make`函数都不允许自定义删除器，但是唯一指针和共享指针有构造函数可以这么做。比如：

```c++
auto widgetDeleter = [](Widget* pw) { … };
std::unique_ptr<Widget, decltype(widgetDeleter)>
    upw(new Widget, widgetDeleter);

std::shared_ptr<Widget> spw(new Widget, widgetDeleter);
```

这种情况下，只能使用new，而`make`函数没有办法做同样的事情。

**限制二**：该限制来自于`make`实现的语法细节，仍然是前面的`initialiezer_list`构造函数相关的问题,`make`函数的完美转发使用的是小括号版本的函数，而不是花括号。这么做尽管可以避免函数决议出现问题，但是意味着如果你想用花括号初始化指向的对象，你需要直接`new`。但是花括号初始化无法完美转发，可以使用条款30介绍的方法：使用`auto`类型推导从花括号初始化创建`std::initializer_list`对象（条款2），然后将对象传递给`make`函数:

```c++
//创建std::initializer_list
auto initList = { 10, 20 };
//使用std::initializer_list为形参的构造函数创建std::vector
auto spv = std::make_shared<std::vector<int>>(initList);
```

以上两个限制是`make`函数都会遇到的问题，以下的限制则仅是共享指针和它的`make`函数可能出现的问题：

**共享指针make限制**：对于重载了`operator new`和`operator delete`类的成员不适合使用`std::allocate_shared`和释放（通过自定义删除器），这是因为重载的类对内存分配有特殊的要求，一般仅分配对象大小的内存，但是`std::allocate_shared`分配到内存还要包括控制块。

虽然使用`std::make_shared`会比直接`new`快，但是对于释放，因为引用指针以及弱指针的存在，控制块不能立即释放，因为`make_shared`将对象和控制块分配在了同一块内存中，所以对象也不能立刻释放，直到控制块完成工作才能统一释放。

如果既定义了删除器，有想要编写异常安全的代码，那么最好的方法是将共享指针的函数以及new操作放在一个**不做其他事情的语句中**（即最好单独拎出来）。比如：

```c++
void processWidget(std::shared_ptr<Widget> spw,     //和之前一样
                   int priority);
void cusDel(Widget *ptr);                           //自定义删除器
//如果不能使用make_shared
processWidget( 									    //和之前一样，
    std::shared_ptr<Widget>(new Widget, cusDel),    //潜在的内存泄漏！
    computePriority() 
);
//正确做法：
std::shared_ptr<Widget> spw(new Widget, cusDel);
processWidget(spw, computePriority());  // 正确，但是没优化，见下
```

该代码在性能上有一些缺陷，在非异常安全的版本中，我们传递的是一个右值，而对于异常安全版本我们使用的左值。这样会导致额外的复制开销，并且对于共享指针，复制还是移动的区别是有意义的。所以代码应该优化如下：

```c++
processWidget(std::move(spw), computePriority());   //高效且异常安全
```

### 条款22：当使用Pimpl惯用法，请在实现文件中定义特殊成员函数

Pimpl:pointer to implememtation，即将类的数据成员替换成一个指向包含具体实现的类（或者结构体）的指针，并将放在主类的数据成员门移动到实现类中，这些数据成员通过指针访问，实例：

```c++
class Widget() {                    //定义在头文件“widget.h”
public:
    Widget();
    …
private:
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;              //Gadget是用户自定义的类型
};
```

在该类中要包含的类型有：`std::string`,`std::vector`,和`Gadget`，定义这些类的头文件在编译的时候都要包含进来，那么就会增加`Widget`的编译时间。同样的，如果这些类的一个头文件改变了，类`Widget`也要重新编译。

在C++98中使用Pimpl惯用法，可以将`Widget`的数据成员替换为一个原始指针，指向一个已经被声明但是还没有被实现的结构体，如下：

```c++
class Widget                        //仍然在“widget.h”中
{
public:
    Widget();
    ~Widget();                      //析构函数在后面会分析
    …

private:
    struct Impl;                    //声明一个 实现结构体
    Impl *pImpl;                    //以及指向它的指针
};
```

这样做最大的好处就是可以加速编译，即使依赖的头文件更改了，`Widget`的使用者不需要关心这些变动。

Pimpl惯用法的第一步，声明一个数据成员，它是一个指正，指向一个不完整的类型。

第二步是动态分配和回收一个对象，该对象包含以前在原来的类中的数据成员。内存的分配和回收都写在实现文件中。对于`Widget`而言，写在`Widget.cpp`中：

```c++
#include "widget.h"             //以下代码均在实现文件“widget.cpp”里
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl {           //含有之前在Widget中的数据成员的
    std::string name;           //Widget::Impl类型的定义
    std::vector<double> data;
    Gadget g1,g2,g3;
};

Widget::Widget()                //为此Widget对象分配数据成员
: pImpl(new Impl)
{}

Widget::~Widget()               //销毁数据成员
{ delete pImpl; }
```

古老的代码，现代C++应该这么写头文件：

```c+
class Widget {                      //在“widget.h”中
public:
    Widget();
    …

private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;    //使用智能指针而不是原始指针
};
```

源文件：

```c++
#include "widget.h"                 //在“widget.cpp”中
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl {               //跟之前一样
    std::string name;
    std::vector<double> data;
    Gadget g1,g2,g3;
};

Widget::Widget()                    //根据条款21，通过std::make_unique
: pImpl(std::make_unique<Impl>())   //来创建std::unique_ptr
{}
```

但是此时我们调用该类会出错：

```c++
#include "widget.h"

Widget w;                           //错误！
```

原因是，唯一指针回收资源的时候会使用`static_assert`来确保原始指针指向的不是一个不完整的类型。我们要做的是，在编译器执行管理对象的时候，将被管理的对象的完整声明放在析构函数之前（或者说析构函数放在实现类或者结构体的后面）：

```c++
class Widget {                  //跟之前一样，在“widget.h”中
public:
    Widget();
    ~Widget();                  //只有声明语句
    …

private:                        //跟之前一样
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};
```

```c++
#include "widget.h"                 //跟之前一样，在“widget.cpp”中
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl {               //跟之前一样，定义Widget::Impl
    std::string name;
    std::vector<double> data;
    Gadget g1,g2,g3;
}

Widget::Widget()                    //跟之前一样
: pImpl(std::make_unique<Impl>())
{}

Widget::~Widget()                   //析构函数的定义（译者注：这里高亮）
{}
//或者这里可以更现代
Widget::~Widget() = default;        //同上述代码效果一致
```

对于自动生成的移动操作，也是正合我意，但是不能再头文件使用关键字直接声明：

```c++
class Widget {                                  //仍然在“widget.h”中
public:
    Widget();
    ~Widget();

    Widget(Widget&& rhs) = default;             //思路正确，
    Widget& operator=(Widget&& rhs) = default;  //但代码错误
    …

private:                                        //跟之前一样
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};
```

因为移动构造函数中可能会抛出异常的事件，这个事件里会生成销毁`pImpl`的代码。但是，我们还不知道`Impl`的完整定义。因此，应该和析构函数的做法一样，将关键字都放在源文件中：

```c++
class Widget {                          //仍然在“widget.h”中
public:
    Widget();
    ~Widget();

    Widget(Widget&& rhs);               //只有声明
    Widget& operator=(Widget&& rhs);
    …

private:                                //跟之前一样
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};
```

```c++
#include <string>                   //跟之前一样，仍然在“widget.cpp”中
…
    
struct Widget::Impl { … };          //跟之前一样

Widget::Widget()                    //跟之前一样
: pImpl(std::make_unique<Impl>())
{}

Widget::~Widget() = default;        //跟之前一样

Widget::Widget(Widget&& rhs) = default;             //这里定义
Widget& Widget::operator=(Widget&& rhs) = default;
```

对于复制函数，可能我们也要自己书写，因为第一，对包含有只可移动(*move-only*)类型，比如`std::unique_ptr`的类，编译器不会发生复制操作；第二，即使编译器帮我们生成了，生成的复制操作也仅仅是浅拷贝，而我们实际需要的是深拷贝

因此，我们对于拷贝也应该实现：

```c++
class Widget {                          //仍然在“widget.h”中
public:
    …

    Widget(const Widget& rhs);          //只有声明
    Widget& operator=(const Widget& rhs);

private:                                //跟之前一样
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};
```

```c++
#include <string>                   //跟之前一样，仍然在“widget.cpp”中
…
    
struct Widget::Impl { … };          //跟之前一样

Widget::~Widget() = default;		//其他函数，跟之前一样

Widget::Widget(const Widget& rhs)   //拷贝构造函数
: pImpl(std::make_unique<Impl>(*rhs.pImpl))
{}

Widget& Widget::operator=(const Widget& rhs)    //拷贝operator=
{
    *pImpl = *rhs.pImpl;
    return *this;
}
```

以上情形，仅仅是针对`std::unique_ptr`，对于`std::shared_ptr`则使用最原始的版本即可：

```c++
class Widget {                      //在“widget.h”中
public:
    Widget();
    …                               //没有析构函数和移动操作的声明

private:
    struct Impl;
    std::shared_ptr<Impl> pImpl;    //用std::shared_ptr
};                                  //而不是std::unique_ptr
```

原因是，`std::unique_ptr`的删除器是自身的一部分，因此在编译器生成删除器之前，必须知道管理类型的完整定义。而`std::shared_ptr`而言，删除器并非自身的一部分，因此当编译器生成特殊成员函数被使用的时候，执行的对象不必是一个完整的类型。

**总结**：对于Pimpl管用法，使用`std::unique_ptr`的时候，需要写出特殊成员函数，并且声明和实现要分别放在头文件和源文件中，即使是`= default`这种默认形式。

但是对于`std::shared_ptr`，并没有太多的限制。

## 第五章 右值引用，移动语义，完美转发

**移动语义**：可以使用廉价的移动操作替代昂贵的拷贝操作

**完美转发**：可以将实参转发到其他的函数，使目标函数接收到的实参与被传递给转发函数的实参保持一致

**右值引用**：是连接这两个不同概念的胶合剂。

但是移动语义不一定执行移动操作，也不一定总是比拷贝快。完美转发也并不完美，右值引用也不一定是右值

### 条款23：理解std::move和std::forward

`std::move`不移动任何东西，`std::forward`也不转发任何东西。在运行时，它们不做任何事情，也不产生任何可执行代码，一字节也没有。

`std::move`和`std::forward`仅仅是执行转换(cast)的函数(模板).`std::move`无条件将实参转换为右值，`std::forward`只在特定情况满足时进行转换。

一个`std::move`的简单示例实现:

```c++
template<typename T>                            //在std命名空间
typename remove_reference<T>::type&&			//返回一个指向同对象的引用
move(T&& param)									//注意，这里是通用引用
{
    using ReturnType =                          //别名声明，见条款9
        typename remove_reference<T>::type&&;

    return static_cast<ReturnType>(param);
}
```

由于引用压缩的原因，如果`T`恰好是一个左值引用，那么实参也会变成一个左值引用。为了避免如此，*type trait*`std::remove_reference`应用到了类型T上，确保`&&`被正确的应用到了一个不是引用的类型。这保证了`move`返回的真的就是右值引用。

C++14可以更简单的实现：

```c++
template<typename T>
decltype(auto) move(T&& param)          //C++14，仍然在std命名空间
{
    using ReturnType = remove_referece_t<T>&&;
    return static_cast<ReturnType>(param);
}
```

需要记住的核心是：`std::move`只进行转换，不移动任何东西。仅仅是将实参转换为右值，仅此而已。

并且`std::move`并不代表一定就是移动，考虑以下情形：

```c++
class Annotation {
public:
    explicit Annotation(const std::string text)
    ：value(std::move(text))    //“移动”text到value里；这段代码执行起来
    { … }                       //并不是看起来那样
    
    …

private:
    std::string value;
};
```

该例子中，执行的实际上是复制操作，原因是`std::move(text)`返回的是`const string &&`，当编译器进行函数决议时，会遇到两种可能:

```c++
class string {                  //std::string事实上是
public:                         //std::basic_string<char>的类型别名
    …
    string(const string& rhs);  //拷贝构造函数
    string(string&& rhs);       //移动构造函数
    …
};
```

即使此时`text`已经被转换为了右值，但是因为`const`属性，编译器调用的依旧是带有`const`参数限定的拷贝构造函数。

综上总结：一：如果你希望移动对象，就不要声明（形参）对象为`const`，否则对`const`对象会转换为拷贝操作。二：`std::move`不仅不移动任何东西，它也不保证它执行转换的对象可以移动。

`std::forward`和`std::move`情况相似，但是`std::forward`的转换时有条件的转换。考虑以下常见情形：

```c++
void process(const Widget& lvalArg);        //处理左值
void process(Widget&& rvalArg);             //处理右值

template<typename T>                        //用以转发param到process的模板
void logAndProcess(T&& param)
{
    auto now =                              //获取现在时间
        std::chrono::system_clock::now();
    
    makeLogEntry("Calling 'process'", now);
    process(std::forward<T>(param));
}
//两次调用，分别是左值和右值调用
Widget w;

logAndProcess(w);               //用左值调用
logAndProcess(std::move(w));    //用右值调用
```

`std::forward`是一个有条件的转换：当它的实参用右值初始化的时候，转换为右值。（困惑：如果引入引用压缩的概念不就不用这么麻烦的理解了吗，或许当时没有引用压缩的概念。该条款似乎将简单的问题复杂化了，尤其对于普通开发者来说）

### 条款24：区分通用引用和右值引用

> 前面所有提到的引用压缩实际上均应该叫做：引用折叠(*reference collapsing*),之前的困惑可能更应该归因于自己模板编程的薄弱。一刷暂时忽略，模板的理解需要结合实际去积累。

大体上，**通用引用**指的是模板编程中`T&&`的形式，仅对右值或者右值引用推导为右值引用，其他情况均为左值引用。**右值引用**则是普通编程中的一个基础概念。

具体有一些细节：

+ 通用引用的形式只能是`T&&`,甚至CV限定都不可以声明，通用引用会自己加上实参的CV限定。

+ 通用引用除了形式上的要求，另一点是仅发生在类型推导时(注意`auto&&`也属于通用引用)。注意vector中存在`T&&`形参，但是并非通用引用的情况，原因就是没有发生类型推导：

  ```c++
  template<class T, class Allocator = allocator<T>>   //来自C++标准
  class vector
  {
  public:
      void push_back(T&& x);
      …
  }
  //原因是使用push_back时，不管声明时是否使用了CTAD，都已经确定了类型。即：
  std::vector<Widget> v;
  //因此调用push_back已经被实例化
  class vector<Widget, allocator<Widget>> {
  public:
      void push_back(Widget&& x);             //右值引用
      …
  };
  
  
  //但是emplace_back存在通用引用
  template<class T, class Allocator = allocator<T>>   //依旧来自C++标准
  class vector {
  public:
      template <class... Args>
      void emplace_back(Args&&... args);
      …
  };
  //原因是emplace_back是构造，并且模板参数包实际上也是模板类型加上“模式”，满足通用模板的形式T&&
  ```

### 条款25 对右值引用使用std::move,对通用引用使用std::forward

当*右值引用*转发给其他函数的时候，右值引用应该别**无条件转换**为右值(通过`std::move`)，因为它们**总是**绑定凹右值；当转发*通用引用*时，通用引用应该有条件转换为右值(通过`std::forward`)，因为它们只是**有时**绑定到右值；

```c++
//对于右值引用，使用std::move
class Widget {
public:
    Widget(Widget&& rhs)        //rhs是右值引用
    : name(std::move(rhs.name)),
      p(std::move(rhs.p))
      { … }
    …
private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};
//对于通用引用，使用std::forward
class Widget {
public:
    template<typename T>
    void setName(T&& newName)           //newName是通用引用
    { name = std::forward<T>(newName); }

    …
};
```

为什么要使用通用引用，而不是选择对左值和右值的函数进行重载呢？抛开源代码数量和代码运行的性能，单单从业务需求就可能无法实现。考虑一个函数需要接收**无限制**个数的参数，每个参数都可以是左值或者右值。经典实例就是`std::make_shared`（C++11），和`std::make_unique`（C++14）：

```c++
template<class T, class... Args>                //来自C++11标准
shared_ptr<T> make_shared(Args&&... args);

template<class T, class... Args>                //来自C++14标准
unique_ptr<T> make_unique(Args&&... args);
```

**返回值优化**：可以避免复制局部变量的需要，通过在分配给函数返回值的内存中直接构造来实现。

编译器可能在按值返回的函数中消除对局部对象的拷贝(或者移动)，如果满足以下条件：

+ 局部对象与函数返回值的类型相同
+ 局部对象就是函数要返回的东西

函数形参则不适合。

RVO通过局部对象是否命名分为RVO和NRVO(命名返回值优化)

为什么自己使用`return std::move(obj)`对局部变量进行移动再返回不会启用编译器的返回值优化呢？因为`std::move`返回的是一个引用，这和返回值优化的条件一(局部对象与函数返回值的类型相同)相违背。

综合《C++20高级编程》第9章，如果需要返回局部对象，直接返回就好，不要考虑`std::move()`或者`std::forward()`，因为大佬比你考虑的更多且经过了大量测试。

### 条款26：避免在通用引用上重载

总结：通用引用可以说是C++中最贪婪的匹配，几乎可以匹配任何类型。所以如果重载，需要考虑很多情况，比如：

```c++
template<typename T>
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}

std::string petName("Darla");           //跟之前一样
logAndAdd(petName);                     //跟之前一样，拷贝左值到multiset
logAndAdd(std::string("Persephone"));	//移动右值而不是拷贝它
logAndAdd("Patty Dog");                 //在multiset直接创建std::string
                                        //而不是拷贝一个临时std::string


//如果需要使用索引获取名字并写入日志呢，应该如下重载
std::string nameFromIdx(int idx);   //返回idx对应的名字

void logAndAdd(int idx)             //新的重载
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(nameFromIdx(idx));
}
```

但是存在的问题是，如果传入的参数类型为`short`,`size_t`，函数决议依旧会选择通用引用实例化出的函数，因为它们不需要隐式转换，更加精确。

### 条款27：熟悉通用引用重载的替代方法

#### 方案一：放弃重载

放弃重载，通过函数名称来区分。比如条款26中的函数，可以更改为`logAndAddName`和`logAndAddIdx`来区分。

#### 方案二：传递const T&

退回到旧式C++，虽然效率没有通用引用高，但是更加正确。

#### 方案三：传值

同《C++20高级编程》ch09中提倡的一致，可以使用值传递来统一拷贝和移动函数：

```c++
class Person {
public:
    explicit Person(std::string n)  //代替T&&构造函数，
    : name(std::move(n)) {}         //std::move的使用见条款41
  
    explicit Person(int idx)        //同之前一样
    : name(nameFromIdx(idx)) {}
    …

private:
    std::string name;
};
```

#### 方案四：使用*tag dispatch*

如果使用通用引用的动机是完美转发，那么就只能使用通用引用了，此时重载不可避免。

解决方法是：Pimpl＋工作中自己定义的CString中使用的tag区分宽窄字符

```c++
template<typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(
        std::forward<T>(name),
        //这里使用remove_reference是因为对于int&，使用is_integral返回的是false，所以需要去除引用
        std::is_integral<typename std::remove_reference<T>::type>()//C++11
        //std::is_integral<typename std::remove_reference_t<T>>()//C++14
    );
}


template<typename T>                            //非整型实参：添加到全局数据结构中
void logAndAddImpl(T&& name, std::false_type)	//译者注：高亮std::false_type
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}

std::string nameFromIdx(int idx);           //与条款26一样，整型实参：查找名字并用它调用logAndAdd
void logAndAddImpl(int idx, std::true_type) //译者注：高亮std::true_type
{
  logAndAdd(nameFromIdx(idx)); 
}
```

需要注意的的是上述的`std::false_type`和`std::true_type`:因为我们的`true`和`false`是运行时得到的，但是我们的函数重载是在编译时决策。如果单单一个`bool`类型，是无法重载函数的。所以标准库提供了`std::false_type`和`std::true_type`两个类，代表`is_integral`结果的两个类型（是整型，非整型）。

`std::false_type`和`std::true_type`就是所谓的“标签”(tag),通过标签“分发”(dispatch)给正确的重载。因此这个设计称之为：*tag dispatch*。这是模板元编程的标准构件模块。

#### 方案五：约束使用通用引用的模板

通过tag dispatch是可以解决很多问题，但是当遇到构造函数结合通用引用时，就会出现问题。因为编译器会自动生成一些函数。此时，编译器生成的函数就*可能*绕过分发。

此时，需要一种技术，可以让你确定使用通用引用模板的条件，那就是`std::enable_if`

`std::enable_if`可以提供一种强制编译器执行行为的一种方法，就像是特定模板不存在一样。默认情况下，所有模板都是**启动(enabled)**的，但是，使用`std::enable_if`可以使得仅在`std::enable_if`的条件满足的情况下才启用。

使用示例：

```c++
class Person {
public:
    template<typename T,
             typename = typename std::enable_if<condition>::type>   //condition为某其他特定条件
    explicit Person(T&& n);
    …
};
```

对于`std::enble_if`具体发生了什么，需要自行学习“SFINAE”

这里我们向表示的条件是确认`T`不是`Person`，即模板函数应该在`T`不是`Person`的时候启用。此时我们需要type trait中的`std::is_same`来确定类型是否相同。但是该函数和上个方案中的`std::is_interger`一样（对于引用返回的是false），对于引用类型和非引用类型比较，总是返回false。即`std::is_same<Person,Person&>::value`返回的是false。（对于CV限定也是一样的，都需要注意）。

对此type trait也提供了`std::decay`。`std::decay<T>::value`会去除CV限定和引用的修饰。

所以，我们`std::enable_if<condition>::type`中的`condition`就应该是

```c++
!std::is_same<Person,typename std::decay<T>::type>::value
```

更新后的代码应该如下：

```c++
class Person {
public:
    template<
        typename T,
        typename = typename std::enable_if<
                       !std::is_same<Person, 
                                     typename std::decay<T>::type
                                    >::value
                   >::type
    >
    explicit Person(T&& n);
    …
};
```

此时还需考虑另一个问题，如果存在继承呢，派生类的构造函数想要调用基类的构造函数来构造基类部分内容，而此时传入基类构造函数的类型不再是`Person`,而是`SpecialPerson`，示例：

```c++
class SpecialPerson: public Person {
public:
    SpecialPerson(const SpecialPerson& rhs) //拷贝构造函数，调用基类的
    : Person(rhs)                           //完美转发构造函数！
    { … }
    
    SpecialPerson(SpecialPerson&& rhs)      //移动构造函数，调用基类的
    : Person(std::move(rhs))                //完美转发构造函数！
    { … }
    
    …
};
```

所以说，站在巨人的肩膀上是很舒服的，*type trait*提供了`std::is_base_of`来判断是否存在继承。如果`std::is_base_of(T1,T2)`是`true`，则代表`T2`派生自`T1`（即：T1 is base of T2）。类型也可以被认为从他们自己派生。因此，我们可以直接使用`std::is_base_of`来代替`std::is_same`就可以了：

```c++
//C++11版本
class Person {
public:
    template<
        typename T,
        typename = typename std::enable_if<
                       !std::is_base_of<Person, 
                                        typename std::decay<T>::type
                                       >::value
                   >::type
    >
    explicit Person(T&& n);
    …
};
//也可以使用C++14的别名模板来简化代码
class Person  {                                         //C++14
public:
    template<
        typename T,
        typename = std::enable_if_t<                    //这儿更少的代码
                       !std::is_base_of<Person,
                                        std::decay_t<T> //还有这儿
                                       >::value
                   >                                    //还有这儿
    >
    explicit Person(T&& n);
    …
};
```

至此，如果构造函数中需要解决和`logAndAdd`中的问题一样，我们只需要结合使用两种方法即可：

```c++
class Person {
public:
    template<
        typename T,
        typename = std::enable_if_t<
            !std::is_base_of<Person, std::decay_t<T>>::value
            &&
            !std::is_integral<std::remove_reference_t<T>>::value
        >
    >
    explicit Person(T&& n)          //对于std::strings和可转化为
    : name(std::forward<T>(n))      //std::strings的实参的构造函数
    { … }

    explicit Person(int idx)        //对于整型实参的构造函数
    : name(nameFromIdx(idx))
    { … }

    …                               //拷贝、移动构造函数等

private:
    std::string name;
};
```

最后需要提及的是，前三种方法虽然比较“笨拙”且综合性能不好，但是如果出现特殊类型，如`char16_t`的字符串

```c++
Person p(u"Konrad Zuse");   //“Konrad Zuse”由const char16_t类型字符组成
```

前三种方法编译器会很清楚的报告，表示没有可以从`const char16_t[12]`转换为`int`或者`std::string`的方法。但是后两种就比较头疼了，至少是上百行的错误信息（模板元编程的经典特征，报错你看不懂）。

### 条款28：理解引用折叠

总结就一句话：只有右值和右值引用推导为右值引用，其他均推导为左值引用

### 条款29：认识移动操作的确定

第一点是代码历史遗留问题，对于一些旧式C++的代码风格，大多遵守的是三原则(rule of three)，或者自定义了一些默认函数，导致编译器无法自动生成移动函数，因此很多可以移动的操作，仍然执行的复制操作。

第二个是问题是：一些类的移动并不是单单“像移动一个指针”一样快。最经典的例子是`std::array`，内部并不是存储一个指针，而是将所有数据存在了对象中，这样对`std::array`对象进行移动的时候，依旧会对每个对象进行移动，远没有我们期待的那样：“将对象的指针移动一下”就好了。

第三个问题：对于`std::string`，许多字符串的实现采用了小字符串优化(*small string optimization*，SSO)。小字符串指的是类似长度小于15字符这种，都是存储在了`std::string`的缓冲区中，而没有存储在堆内存，移动这种字符串的速度并不比复制快。

### 条款30：熟悉完美转发失败的情况

完美转发意味着我们不仅转发对象，还要转发对象的显著特征：类型，左值还是右值，以及CV特性。因此，很容易和通用引用结合使用。

```c++
template<typename... Ts>
void fwd(Ts&&... params)            //接受任何实参
{
    f(std::forward<Ts>(params)...); //转发给f
}
```

给定我们的目标函数`f`和转发函数`fwd`，如果`f`使用某特定实参会执行某个操作，但是`fwd`使用相同的实参会执行不同的操作，完美转发就会失败

#### 花括号初始化

```c++
void f(const std::vector<int>& v);
f({ 1, 2, 3 });         //可以，“{1, 2, 3}”隐式转换为std::vector<int>
fwd({ 1, 2, 3 });       //错误！不能编译
```

该问题的错误原因：在`f`函数的调用时，接收到初始化列表作为参数，编译器会和声明的形参进行对比，从而将初始化列表用作`vector`的初始化。而使用`fwd`间接调用f函数的时候，会先进行**类型推导**，但是类型推导拒绝推导`std::initializer_list`类型。因此对于`fwd`函数的类型推导被阻止，编译器就拒绝该调用。

因为`auto`是可以推导出`std::initializer_list`的，所以提供一种简单的解决方法：

```c++
auto il = { 1, 2, 3 };  //il的类型被推导为std::initializer_list<int>
fwd(il);                //可以，完美转发il给f
```

#### 0或者NULL作为空指针

仍旧是旧式代码的问题，在类型推导时，会推导为`int`，然后对`int`完美转发。解决方法还是将`0`或者`NULL`代表的空指针替换为`nullptr`

#### 仅有声明的static const数据成员

对于仅有声明的static const数据成员，即在声明时赋值：

```c++
class Widget {
public:
    static const std::size_t MinVals = 28;  //MinVal的声明
    …
};
…                                           //没有MinVals定义

std::vector<int> widgetData;
widgetData.reserve(Widget::MinVals);        //使用MinVals
```

这样做可以通过编译，但是有些编译器并不会为`static const`数据成员分配内存，而是直接在使用的地方替换为其声明值。这样做的问题是，当你需要对`static const`成员取地址的时候，就会出现编译报错。而通用引用中的引用，在二进制文件中，可以看做就是一个指针。

解决方法就是，将声明和定义分开（即使你的编译器支持上述形式）。即，在源文件中对`static const`进行定义。

#### 当遇到重载函数

考虑下属情形：

```c++
void f(int (*pf)(int));             //pf = “process function”
void f(int pf(int));                //与上面定义相同的f

//现在有重载的函数
int processVal(int value);
int processVal(int value, int priority);

f(processVal);                      //可以
fwd(processVal);                    //错误！那个processVal？
```

这个错误的原因和花括号初始化的原因类似。在直接调用`f`函数的时候，会根据形参来进行函数决议。但是在通用类型推导的时候，没有这些信息。

同样的，对于模板函数，该问题也存在（因为一个模板函数代表的是一个函数族）：

```c++
template<typename T>
T workOnVal(T param)                //处理值的模板
{ … }

fwd(workOnVal);                     //错误！哪个workOnVal实例？
```

解决方法是声明参数的类型，或者进行强制转换：

```c++
using ProcessFuncType =                         //写个类型定义；见条款9
    int (*)(int);

ProcessFuncType processValPtr = processVal;     //指定所需的processVal签名

fwd(processValPtr);                             //可以
fwd(static_cast<ProcessFuncType>(workOnVal));   //也可以
```

但是这么做就比较“脱裤子放屁”，我使用完美准发就是为了“无脑”转发任何类型，这里我需要直到具体的转发类型，还要转换，那还完美什么。

#### 位域

和底层有关，C++无法指定一个引用或指针指向`bit`

## 第六章 Lambda表达式

简单回顾一个lambda表达式：

```c++
std::find_if(container.begin(), container.end(),
             [](int val){ return 0 < val && val < 10; });  
```

术语解释：

+ **闭包**(enclosure)是lambda创建的运行时的对象。依赖捕获模式，闭包持有被捕获数据的副本或者引用。在上述`std::find_if`调用中，闭包作为第三个参数传入`std::find_if`
+ **闭包类**(closure class)是从中实例化闭包的类。每个lambda都会使编译器生成唯一的闭包类。lambda中的语句成为其闭包类的成员函数中的可执行指令

闭包一般可以被拷贝，所以可能有多个闭包对应一个lambda。如：

```c++
{
    int x;                                  //x是局部对象
    …

    auto c1 =                               //c1是lambda产生的闭包的副本
        [x](int y) { return x * y > 55; };

    auto c2 = c1;                           //c2是c1的拷贝

    auto c3 = c2;                           //c3是c2的拷贝
    …
}
```

`c1`，`c2`，`c3`都是*lambda*产生的闭包的副本。

如果简单的使用lambda，这些概念是不必要的。但是如果深入研究，区分什么存在于编译期（*lambdas* 和闭包类），什么存在于运行时（闭包）以及它们之间的相互关系是重要的。

> 一刷理解：闭包可以看做多态中的对象，其作用主要在运行期间发挥。而闭包类则顾名思义，就是一个类，起作用在编译期发挥。

### 条款31：避免使用默认的捕获模式

C++11两种默认的捕获模式：按引用捕获和按值捕获。按引用捕获可能带来悬空引用的问题。那直接按值捕获不就可以解决悬空引用的问题了吗？答案是否定的，并且按值捕获还会让你以为你的闭包是独立的（事实上也不是独立的）。

首先是按引用捕获的问题：如果引用对象是局部的，而闭包的声明周期超过了被引用对象的声明周期，就会出现悬空引用。考虑以下情形：

```c++
//vector中的元素都是可调用对象。返回值为bool，参数为int
using FilterContainer = std::vector<std::function<bool(int)>>;  

FilterContainer filters;                    //过滤函数

//添加一个可调用对象，用来过滤5的倍数
filters.emplace_back(                       
    [](int value) { return value % 5 == 0; }
);
//但是实际上我们可能不止需要过滤5的倍数，即除数可能需要外界传入
void addDivisorFilter()
{
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();

    auto divisor = computeDivisor(calc1, calc2);

    filters.emplace_back(                               //危险！对divisor的引用
        [&](int value) { return value % divisor == 0; } //将会悬空！
    );
    //即使显式捕获变量引用也不行
    filters.emplace_back(
        [&divisor](int value) 			    //危险！对divisor的引用将会悬空！
        { return value % divisor == 0; }
    );
}
```

上述的问题显而易见，divisior的生命周期仅存在该函数栈中，但是filter的生命周期大于该函数栈，而捕获的形式是引用，所以在函数结束后，引用就悬空了。

> 该条款没有提供任何**解决**上述描述的两个问题，只是建议捕获列表将需要捕获的变量写出来（如`[&divisor]`）,这样能帮助自己观察是否出现了引用悬空的情况

另一个值得总结的是，**lambda表达式只能捕获局部变量和函数形参！**

错误实例，使用类内成员：

```c++
//类定义
class Widget {
public:
    …                       //构造函数等
    void addFilter() const; //向filters添加条目
private:
    int divisor;            //在Widget的过滤器使用
};
//函数实现1
void Widget::addFilter() const
{
    filters.emplace_back(
        [divisor](int value)                //错误！没有名为divisor局部变量可捕获
        { return value % divisor == 0; }
    );
}

//函数实现2
void Widget::addFilter() const
{
    //使用默认的值捕获，错误（可以通过编译，但是有大问题），局部变量和形参都是空的，所以该捕获列表仅仅捕获到了一个this！！！
    //至于为什么匿名函数内的divisior没有报错，是因为编译器默认添加了this
    filters.emplace_back(
        [=](int value) { return value % divisor == 0; }
    );
}	
//上述函数的编译器展开版本
void Widget::addFilter() const
{
    auto currentObjectPtr = this;

    filters.emplace_back(
        [currentObjectPtr](int value)
        { return value % currentObjectPtr->divisor == 0; }
    );
}
```

上述版本看似可以使用，但是任然存在悬空问题：

```c++
using FilterContainer = 					//跟之前一样
    std::vector<std::function<bool(int)>>;

FilterContainer filters;                    //跟之前一样

void doSomeWork()
{
    auto pw =                               //创建Widget；std::make_unique
        std::make_unique<Widget>();         //见条款21

    pw->addFilter();                        //添加使用Widget::divisor的过滤器

    …
}                                           //销毁Widget；filters现在持有悬空指针！
```

解决方法是添加一个局部变量，再使用值捕获：

```c++
void Widget::addFilter() const
{
    auto divisorCopy = divisor;                 //拷贝数据成员

    filters.emplace_back(
        [=](int value)                          //捕获副本
        { return value % divisorCopy == 0; }	//使用副本
    );
}

//C++14也可以这样
void Widget::addFilter() const
{
    filters.emplace_back(                   //C++14：
        [divisor = divisor](int value)      //拷贝divisor到闭包
        { return value % divisor == 0; }	//使用这个副本
    );
}
```

再次强调：**lambda表达式只能捕获局部变量和函数形参！**

需要注意的另一个情况是局部`static`变量

```c++
void addDivisorFilter()
{
    static auto calc1 = computeSomeValue1();    //现在是static
    static auto calc2 = computeSomeValue2();    //现在是static
    static auto divisor =                       //现在是static
    computeDivisor(calc1, calc2);

    filters.emplace_back(
        [=](int value)                          //什么也没捕获到！
        { return value % divisor == 0; }        //引用上面的static
    );

    ++divisor;                                  //调整divisor
}
```

条款的核心是：**lambda表达式只能捕获局部变量和函数形参！**（重要的事情说三遍，所有的生命周期问题都可以从这里出发分析）

### 条款32：使用初始化捕获来移动对象到闭包中

前言：C++11无法完美实现，但是C++14可以实现。

C++14提供了**初始化捕获**，初始化捕获的另一个名称是**通用\*lambda\*捕获**（*generalized lambda capture*），使用初始化捕获可以让你指定：

+ 从lambda生成的闭包类中的数据成员名称
+ 初始化该成员的表达式

我们直到`std::unique_ptr`只支持移动而不支持复制，所以以下是一个简单的将`std::unique_ptr`移动到闭包的方法：

```c++
class Widget {                          //一些有用的类型
public:
    …
    bool isValidated() const;
    bool isProcessed() const;
    bool isArchived() const;
private:
    …
};

auto pw = std::make_unique<Widget>();   //创建Widget；使用std::make_unique
                                        //的有关信息参见条款21

…                                       

auto func = [pw = std::move(pw)]        //使用std::move(pw)初始化闭包数据成员
            { return pw->isValidated()
                     && pw->isArchived(); };

//如果不需要对唯一指针管理的内容进行更改，也可以直接创建临时对象来使用lambda的初始化捕获
auto func = [pw = std::make_unique<Widget>()]   //使用调用make_unique得到的结果
            { return pw->isValidated()          //初始化闭包数据成员
                     && pw->isArchived(); };
```

“`=`”的左侧是指定的闭包类中数据成员的名称，右侧则是初始化表达式。有趣的是，“`=`”左侧的作用域不同于右侧的作用域。左侧的作用域是闭包类，右侧的作用域和*lambda*定义所在的作用域相同

注意，开始说的C++11无法做到，指的是C++11使用lambda无法做到。但是你完全可以使用一个可调用类来实现（联想工作中自定义的CA2W类）：

```c++
class IsValAndArch {                            //“is validated and archived”
public:
    using DataType = std::unique_ptr<Widget>;
    
    explicit IsValAndArch(DataType&& ptr)       //条款25解释了std::move的使用
    : pw(std::move(ptr)) {}
    
    bool operator()() const
    { return pw->isValidated() && pw->isArchived(); }
    
private:
    DataType pw;
};

auto func = IsValAndArch(std::make_unique<Widget>());
```

如果坚持C++11使用lambda函数实现移动语义，因为可以考虑使用`std::bind`来帮忙：

```c++
std::vector<double> data;           

…                                     

auto func =
    std::bind(                              //C++11模拟初始化捕获
        [](const std::vector<double>& data) 
        { /*使用data*/ },
        std::move(data)   //注意，这里是bind的参数                  
    );
```

对于`std::bind`,每个左值实参都是复制构造的，每个右值实参都是移动构造的。

总结下来，使用`std::bind`解决C++11lambda无法移动构造的要点：

- 无法移动构造一个对象到C++11闭包，但是可以将对象移动构造进C++11的bind对象。
- 在C++11中模拟移动捕获包括将对象移动构造进bind对象，然后通过传引用将移动构造的对象传递给*lambda*。
- 由于bind对象的生命周期与闭包对象的生命周期相同，因此可以将bind对象中的对象视为闭包中的对象。

### 条款33：对于auto&&形参使用decltype以std::forward它们

在C++14中，lambda中的参数可以支持auto：即在闭包类中的`operator()`函数是一个函数模板：

```c++
auto f = [](auto x){ return func(normalize(x)); };
//就相当于
class SomeCompilerGeneratedClassName {
public:
    template<typename T>                //auto返回类型见条款3
    auto operator()(T x) const
    { return func(normalize(x)); }
    …                                   //其他闭包类功能
};
```

这样的问题是，如果normalize对待左值和右值的方式不同，这里就不合适了。因为不管左值还是右值作为匿名函数的实参，最终传入normalize的都是左值。

为了解决这个问题，应该将参数作为通用引用，并且使用完美转发：

```c++
auto f = [](auto&& x)
         { return func(normalize(std::forward<???>(x))); };
```

这里存在另一个问题，如何确定`auto`的类型并传递给forward呢？答案是`decltype`:递给*lambda*的是一个左值，`decltype(x)`就能产生一个左值引用；如果传递的是一个右值，`decltype(x)`就会产生右值引用:

```c++
auto f =
    [](auto&& param)
    {
        return
            func(normalize(std::forward<decltype(param)>(param)));
    };
```

同样的也可以拓展到可变参数包（毕竟只是一个模式）：

```c++
auto f =
    [](auto&&... params)
    {
        return
            func(normalize(std::forward<decltype(params)>(params)...));
    };
```



