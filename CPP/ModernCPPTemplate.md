# ModernCPPTemplate

## ch01 函数模板

### 1.1 定义模板

```c++
template<typename T>
T max(T a,T b){
    return a > b ? a : b;
}
//历史版本,也可以使用class作为类型声明符
template<class T>
...
```

> C++17之前，类型T必须是可复制或者移动的才能传递参数。C++17以后，即使复制构造函数和移动构造函数都无效，因为C++17强制的复制消除，也可以传递临时纯右值。（自C++17开始，非必须不会将右值实质化，并且它会被直接构造到其最终目标的存储中。）
>
> 复制消除：省略复制以及移动构造函数，导致零复制的按值传递语义

### 1.2 使用模板

```c++
template<typename T>
T max(T a,T b){
    return a > b ? a : b;
}

struct Test{
    Test() = default;
    Test(int v) :m_v(v){}
    bool operator>(const Test& t){
        return m_v > t.m_v;
    }
    int m_v;
};

int main(){
	int a{1};
    int b{2};
    std::cout <<"max(a,b): " << ::max(a,b) << '\n';
    
    Test t1{20};
    Test t2{1};
    std::cout <<"max(t1,t2): " << ::max(t1,t2) << '\n';
}
```

需要注意的是，模板只有使用了才会进行实例化。

对于没有实际应用的模板，并不会实例化，所以应该注意，编译通过的代码，可能存在模板错误。

另外，我们也可以显示声明模板函数的模板类型：

```c++
max<double>(a,b);//max模板函数实例化为double版本
```

### 1.3 模板参数推导

C++的查找规则是*实参依赖查找*(ADL)，即，根据函数的实参的作用域或者命名空间来确定函数的具体版本

所以在调用时，需要注意作用域的问题，对于和标准库冲突的函数，可以使用作用域限定符进行确认函数的调用

```c++
::max("str"s,std::string("test"));//调用自定义版本的函数
max("str"s,std::string("test"));//冲突，无法确定。
```

#### 1.3.1 万能引用和引用折叠

万能引用，又称为转发引用，即接收左值表达式就推导为左值引用，接收右值表达式，就推导为右值引用，如：

```c++
template<typename T>
void f(T&& t){}

int a{10};
f(a);	//f<int&>,形参类型int&
f(10);	//f<int>，形参类型int&&
```

引用折叠：右值引用的右值引用折叠为右值引用，其他所有组合均折叠为左值引用

```c++
typedef int& lref;
typedef int&& rref;
int n;
lref& r1 = n;//r1 的类型为int&
lref& r2 = n;//r1 的类型为int&
rref& r3 = n;//r1 的类型为int&
rref& r4 = 1;//r1 的类型为int&&
```

源码中的例子：

```c++
template <class Ty>
constexpr Ty&& forward(Ty& Arg) noexcept {
    return static_cast<Ty&&>(Arg);
}

int a = 10;            
::forward<int>(a);     // 返回 int&& 因为 Ty 是 int，Ty&& 就是 int&&
::forward<int&>(a);    // 返回 int& 因为 Ty 是 int&，Ty&& 就是 int&
::forward<int&&>(a);   // 返回 int&& 因为 Ty 是 int&&，Ty&& 就是 int&&
```

### 1.4 有默认实参的模板类型形参

模板编程中的**类型形参**也可以有默认值

```c++
template<typename T = int>
void f();

f();  // f<int>
f<double>(); // f<double>
```

**funny things:**设置不同的类型形参，并将返回值设置为类型形参的“公共”类型（如，int和double的公共类型就是double）

前置知识：三目运算符要求第二项和第三项之间可以隐式转换，然后整个表达式的类型就是公共类型

利用这种性质，就可以不使用auto，不使用后置返回类型实现需求：

```c++
template<typename T1,typename T2,typename RT = decltype(true ? T1{} : T2{})>
RT func(const T1&,const T2&);
```

C++11可以使用后置返回类型实现需求：

```c++
template<typename T1,typename T2>
auto func(const T1& a,const T2& b) ->decltype(true ? a : b){};
```

注意，这里是`true ? a : b`推导出来的是包括CV限定的，因为在使用后置返回类型时，实参的类型已经确定了，而在前面的例子中的三目运算符`true ? T1{} : T2{}`和实参没有关系，推导出来的是T1，T2的公共类型，没有CV限定（因为本质上就是利用了临时对象的类型）

而在C++14中引入了两个新特性:

+ 返回类型推导：函数的返回也可以直接写auto或者decltype(auto)
+ decltype(auto):和直接使用auto的区别是，auto就相当于上述例子的`true?T1{}:T2{}`,是没有CV限定的，而decltype(auto)则是根据实参类型进行推导的，包括CV限定。（auto在大多数情况下和模板类型推导规则是一致的。《Effective Modern C++》）

所以在C++14以后，上述需求可以更简洁的写为：

```c++
decltype(auto) func(const auto& a, const auto& b){}
```

### 1.5 非类型模板形参

顾名思义，模板不接受类型，而是接受值或者对象。

```c++
template<std::size_t N>
void f(){
    std::cout << N << std::endl;
}

f<100>();
```

也可有默认值

```c++
template<std::size_t N = 11>
void f(){
    std::cout << N << std::endl;
}

f(); //f<11>
f<22>(); //f<22>
```

###  1.6 重载函数模板

函数模板可以重载，这会涉及到重载决议。普通应用场景可以理解为，有限匹配最精确的函数。非模板函数>特化函数>实例化模板函数

```c++
template<typename T>
void test(T) { std::puts("template"); }

void test(int) { std::puts("int"); }

test(1);        // 匹配到test(int)
test(1.2);      // 匹配到模板
test("1");      // 匹配到模板
```

### 1.7 可变参数模板

由于C++11对于可变参数模板只支持了解包和sizeof，没有特殊的用途，所以这里以C++14标准解释

前置知识：`...`符号是一种模式，是对多个参数统一同一种操作的简写，如：

```c++
template<typename... Args>  // 等价：typename T1,typename T2....
void func(Args... arg);		//等价：T1 t1,T2 t2...
```

其中Args就是类型形参包，args就是函数形参包。

形参包的展开也利用到了模式：

```c++
void func(const char*,int,double);
void func(const char**,int*,double*);
template<typename... Args>
void f(Args... args){
    func(args...);	//func(t1,t2...)
    func(&arg...);	//func(&t1,&t2....)
}
```

**funny things**:

```c++
template<typename...Args>
void print(const Args&...args){    // const char (&args0)[5], const int & args1, const double & args2
    int dood[]{ (std::cout << args << ' ' ,0)... };
}

int main() {
    print("luse", 1, 1.2);
}
```

注意这里的`(std::cout << args << ' ',0)...`这里利用是模式，加上逗号表达式。借此生成了一个执行语句的数组，这个数组没有任何意义，就是直接执行`std::cout<<args<<' '`并且返回0，使用0对数组进行了初始化。

需要注意的是，如果print函数没有参数，会造成编译错误，解决方法是在数组中加入默认参数

```c++
int dood[]{ 0,(std::cout << args << ' ', 0)... };
//如果没有可变参数，那么该数组的形式为int dood[]{0,};该声明方式一直都是允许的。
```

对于这个函数，还可以进行性能上的优化，就是生成一个临时数组，而不是让数组存在于整个函数生命周期里：

```c++
template<typename... Args>
void print(const Args&...args){
    using Arr = int[];
    Arr{0,(std::cout << args << ' ',0)...};
    //这里如果存在编译器警告，可以使用弃值表达式
    (void)Arr{0,(std::cout << args << ' ',0)...}; // 表示表达式计算的值被舍弃，我们只需要使用它的附带功能。这里就是执行输出语句
}
```

非类型模板形参有一个重要的用处就是用来接收数组的长度：

```c++
template<typename... Args>
void print(const Args&...args){
    using Arr = int[];
    Arr{0,(std::cout << args << ' ',0)...};
}

template<typename T,std::size_t N,typename... Args>
void f(const T(&arr)[N], Args...index){
    print(arr[index]...);
}

int main() {
    //输出指定位置的数组元素
    int array[10]{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
    f(array, 1, 3, 5);
}
```

综上可以写一个自定义的求和函数：

```c++
template<typebame... Args,typename RT = std::common_type_t<Args...>>
RT sum(const Args&... args){
    RT dood[]{ static_cast<RT>(args)... };
    RT dummy{};
    for(int i = 0; i < sizeof...(args); ++i){
        dummy += dood[i];
    }
    return dummy;
}

//使用STL版本简化的sum
template<typebame... Args,typename RT = std::common_type_t<Args...>>
RT sum(const Args&... args){
    RT dood[]{ static_cast<RT>(args)... };
    return std::accumulate(std::begin(dood),std::end(dood),RT{});
}

int main() {
    double ret = sum(1, 2, 3, 4, 5, 6.7);
    std::cout << ret << '\n';       // 21.7
}
```

其中，`std::common_type_t`就是上述三木表达式附带属性的加强版，会返回一个所有参数都能隐式转换的类型，即返回一个公共类型

非类型模板参数也可使用形参包：

```c++
template<std::size_t... N>
void f(){
    std::size_t _[]{ N... }; // 展开相当于 1UL, 2UL, 3UL, 4UL, 5UL
    std::for_each(std::begin(_), std::end(_), 
        [](std::size_t n){
            std::cout << n << ' ';
        }
    );
}
f<1, 2, 3, 4, 5>();
```

## ch02 类模板

### 2.1 定义类模板

```c++
template<typename T>
struct Test{};
//旧式
template<class T>
struct Test{};
```

### 2.2 使用类模板

```c++
template<typename T>
struct Test{};

int main(){
    Test<void> t;	//ok
    Test<int> t2;	//ok
    Test t;			//error
}
```

如果存在模板数据成员

```c++
template<typename T>
struct Test{
    T t;
};

// Test<void> t;  // Error! ----> void t;错误的定义
Test<int> t2;     
// Test t3;       // Error!
Test t4{ 1 };     // C++17 OK！----->c++17增加了类模板实参推导
```

### 2.3 类模板参数推导

对于普通的应用，和模板函数的实参类型推导一样使用(CV限定的规则也同样适用)

```c++
template<typename T>
struct A{
    A(T,T);
};

auto y = new A{1,3}; //A<int>
```

#### 2.3.1 用户定义的推导指引

语法（前面是调用函数的形式，后面是模板声明的形式）：

```c++
模板名称(类型a)->模板名称<类型b>
```

需求1：如果类模板推导为int，就让它推导为size_t，如果是指针类型就推导为数组

```c++
template<typename T>
struct Test{
    Test(T t):m_t(t){}
private:
    T m_t;
};

Test(int) -> Test<std::size_t>;

//将指针类型推导为数组
template<typename T>
Test(T*)->Test<T[]>;

Test t(1);//Test<std::size_t>

char *p =nullptr;
Test t1(p);//Test<char[]>
```

需求2：

```c++
template<typename T,std::size_t size>
struct array{
    T arr[size];
};

::array arr{1,2,3}; //error
```

solution:

```c++
template<typename T,typename... Args>
array(T t,Args...)->array<T,sizeof...(Args)+1>;//注意形式
```

### 2.4 有默认实参的模板形参

和函数模板一样

```c++
template<typename T = int>
struct X{};

X x;	//CTAD since C++17
X<> x2;	//before C++17
```

但是注意一种情况，如果在类内声明一个*有默认实参的类模板类型*的数据成员，不管是否达到C++17，都不能省略`<>`。尽管GCC中可以，但是为了代码的健壮性和跨平台考虑，这种情形不要省略`<>`

```c++
template<typename T = int>
struct X{};

struct Test{
    X x;	// maybe error (clang msvc)
    X<> x2;	// ok
    static inline X x3;//maybe error(clang msvc)
};
```

非类型模板形参也可以加默认值，但是并不常见

```c++
template<typename T,std::size_t N = 10>
struct Arr{
    T arr[N];
};

Arr<int> x;	//Arr<int,10>
```

### 2.5 模板模板形参

语法形式：

```txt
template < 形参列表 > typename(C++17)|class 名字(可选)            
template < 形参列表 > typename(C++17)|class 名字(可选) = default   
template < 形参列表 > typename(C++17)|class ... 名字(可选)  (C++11 起)
```

使用实例：

```c++
template<typename T>
struct X{};

template<template<typename T> typename C>
struct Test{};

Test<X> test;
```

#### 2.5.1 模板模板形参

```c++
template<typename T>
struct my_array{
    T arr[10];
};

template<typename Ty,template<typename T> typename C>
struct Array{
    C<Ty> array;
};

Array<int,my_array>arr; //arr 中是my_array<int>
```

#### 2.5.2 有默认模板参数的模板模板形参

```c++
template<typename T>
struct my_array{
    T arr[10];
};

template<typename Ty,template<typename T> typename C = my_array>
struct Array{
    C<Ty> array;
};

Array<int>arr; //arr 中是my_array<int>
```

#### 2.5.3 有模板参数包的模板模板形参

```c++
template<typename T>
struct X{};
template<typename T>
struct X2{};

template<template<typename T>typename... TArgs>
struct Test{};

Test<X,X2,X> test;
```

配合非类型模板形参也是可以的

```c++
template<std::size_t N>
struct X{};

template<template<std::size_t> typename C>
struct Test{};

Test<X> test;


template<typename... Args>
struct my_array{
    int arr[sizeof...(Args)];
};

template<typename T,template<typename... Args> typename C = my_array>
struct Array{
    C<T> array;
};

Array<int> arr;	// arr中的数据成员为my_array<int>
```

### 2.6 成员函数模板

需要注意，以下情形**不**是成员函数模板,只是普通的成员函数，在模板类实例化的时候实例化为具体的函数

```c++
template<typename T>
class Test{
    void f(T){}
};
```

#### 2.6.1 类模板中的成员函数模板

```c++
template<typename T>
struct Test{
    template<typename... Args>
    void f(Args&&... args){}	//注意这里是万能引用，而不是右值引用
};
```

#### 2.6.2 普通类中的成员函数模板

```c++
struct Test{
    template<typename... Args>
    void f(Args&&... args){}	//注意这里是万能引用，而不是右值引用
};
```

### 2.7 可变参数类模板

```c++
template<typename... Args>
struct X{
    X(Args... args)
        :m_value(args...)
        {}
    
    std::tuple<Args...> m_value;
};

X x{ 1,"2",'3',4. };    // x 的类型是 X<int,const char*,char,double>
std::cout << std::get<1>(x.value) << '\n'; // 2
```

