# C++20高级编程
> 该笔记为个人笔记，仅记录主观上有价值或者没有使用过的内容
## Part1 专业的C++简介
### ch01 C++和标准库速成
#### 1.1 C++速成
##### 1.1.1 小程序“Hello Wrold”

+ C++20新特性，从原始的`#include <head>`转换为了`import <head>`,即模块化（向Python看齐）
+ 从C++20开始，输入输出流推荐写法是`std::format`，定义在`<format>`中（同样类似Python的format）

##### 1.1.2 名称空间

+ C++17前后嵌套命名空间的使用

  ```c++
  //C++17之后
  namespace MyLib::Net::FTP{
      ...
  }
  //C++17以前
  namespace MyLib{
      namespace Net{
          namespace FTP{
          	...
          }
      }
  }
  ```

+ 可以使用**名称空间别名**为嵌套命名空间指定一个更简短的名称

  ```c++
  namespace MyFTP = MyLib::Net::FTP;
  ```

##### 1.1.4 变量

+ 推荐使用统一初始化（注意《Effective Modern C++》中，使用auto和模板编程时的情况），即`T t{}`

+ 推荐使用类模板中的最大值，而不是原来的宏定义`INT_MAX`等。即，推荐使用`<limits>`中的类模板`std::numeric_limits`

  ```c++
  cout  << format("Max int value: {}\n",numeric_limits<int>::max());
  cout  << format("Min int value: {}\n",numeric_limits<int>::min());
  cout  << format("Lowest int value: {}\n",numeric_limits<int>::lowest());
  ```

  > 注意`min()`和`lowest()`的区别：对于整数，两者相等。但是对于浮点数来说，最小值表示该类型可以表示的最小正数，最低值表示该类型能表示的最小负数，即`-max()`（有点反常规。。。）

+ 由于统一初始化不允许*narrowing conversion*(《Effective Modern C++》)，所以当需要转换时，推荐使用强制类型转换

  ```c++
  float MyFlt{3.14f};
  int i{static_cast<int>(MyFlt)};//highly recommended
  ```

##### 1.1.6 枚举类型

+ 推荐使用新式的强类型的`enum class`而不是类型不安全的旧式风格`enum`

  > 旧式风格的值会被推导到外层的作用域，这意味着可以在父作用域不通过全名直接使用他们：
  >
  > ```c++
  > //新式风格
  > enum class PieceType{King,Queen,Rook,Pawn};
  > //新式风格的使用
  > PieceType piece{PieceType::King};
  > 
  > //旧式风格
  > enum PieceType{King,Queen,Rook,Pawn};
  > //旧式风格的使用
  > PieceType piece{King};
  > ```
  >
  > 这样就导致容易命名冲突

+ 默认情况下，枚举值是整形的，但是是可以改变的

  ```c++
  enum class PieceType:unsigned long
  {King,Queen,Rook,Pawn};//枚举值变为ul
  ```

+ 从`C++20`开始，可以使用`using enum`声明避免使用枚举值的全名：

  ```c++
  using enum PieceType;//谨慎使用，尽量在小作用域下使用别名
  					 //如果作用域很大，会和旧式风格一样导致命名冲突的问题
  PieceType piece{King};
  ```

##### 1.1.7 结构体

+ 模块的特性在编译器中还未完全支持，暂时不使用

##### 1.1.8 条件语句

+ 新特性中，正在`if`语句以及`switch`语句中支持了初始化列表

  ```c++
  //<initializer>中引入的任何变量只在<conditional_expression>,<if_body>,<else_if_body>,<else_body>中可用
  if(<initializer>;<conditional_expression>){
      <if_body>
  }else if(<else_if_expression>){
      <else_if_body>
  }else{
      <else_body>
  }
  //<initializer>中引入的任何变量只在<expression>,<body>中可用
  switch(<initializer>;<expression>){<body>}
  ```

+ enum别名的正确使用实例（尽量限制在小的作用域，防止命名冲突）

  ```c++
  enum class Mode{Default,Custom,Standard};
  int value{42};
  Mode mode{...};
  switch(mode){
          using enum Mode;//正确使用，该别名仅在局部有效
      case Custom:
          value = 84;
         	[[fallthrough]];//c++11支持的“属性”，该属性是为了告诉编译器，这里的case没有break导致的fallthrough是故意为之
      case Standard:
      case Default:
          break;
  }
  ```

##### 1.1.10 逻辑比较运算符

+ **好习惯**：利用短路逻辑，将代价更低的测试放在前面，可以提升性能

##### 1.1.11 三向比较运算符

> `<=>`也称为太空飞船运算符

+ 返回的事*类枚举(enumeration-like)*类型，定义在`<compare>`和`std`名称空间中。

+ 如果操作数是整数类型，则结果是*强排序*，为以下之一：

  - `strong_ordering::less`:第一个操作数小于第二个
  - `strong_ordering::greater`:第一个操作数大于第二个
  - `strong_ordering::eqial`:两个操作数相等

+ 如果操作数是浮点类型，结果是一个*偏序(partial-ordering)*

  - `partial_ordering::less`:第一个操作数小于第二个
  - `partial_ordering::greater`:第一个操作数大于第二个
  - `partial_ordering::equivalent`:两个操作数相等
  - `partial_ordering::unordered`:操作数中存在非数字

  ```c++
  int i{11};
  strong_ordering result{i <=> 0};
  if(result == strong_ordering::less){cout << "less" << endl;}
  if(result == strong_ordering::greater){cout << "greater" << endl;}
  if(result == strong_ordering::equal){cout << "equal" << endl;}
  ```

+ 还有一种*弱排序*针对自定义类型实现三向比较：

  - `weak_ordering::less`:第一个操作数小于第二个
  - `weak_ordering::greater`:第一个操作数大于第二个
  - `weak_ordering::equivalent`:两个操作数相等

  > 对于比较**昂贵的对象**很有用

+ 对于结果，也可以使用`<compare>`中提供的`std::is_eq()`,`std::is_gt()`,`std::is_lt()`,`std::is_gteq()`

  ```c++
  int i{11};
  strong_ordering result{i <=> 0};
  if is_lt(result){cout << "less" << endl;}
  if is_gt(result){cout << "greater" << endl;}
  if is_eq(result){cout << "equal" << endl;}
  ```

##### 1.1.13 属性

> C++11开始，通过双方括号语法`[[attribute]]`对属性进行了标准化支持

+ `[[nodiscard]]`:在对函数调用，却对返回值没有处理时发出警告

  ```c++
  [[nodiscard]] int func()
  {
      return 1;
  }
  int main()
  {
      func();//warning C4834:discarding return value of function with 'nodiscard' attribute
  }
  ```

  > 从C++20开始，可以添加自定义描述信息：`[[nodiscard("Some explanation")]] int funs();`

+ `[maybe_unused]`:禁止编译器在未使用某些内容的时候发出警告（在GCC编译器好像是默认属性，在MVC中不是）

+ `[[noreturn]]`:意味着永远不会将控制权返回给调用点

+ `[[deprecated]]`:将某些内容标记为已弃用，意味着仍然可以使用，但是不推荐。（最经典的例子是`std::auto_ptr`）

  ```c++
  [[deprecated("Unsafe method,please use xyz")]] void func();
  
  int main()
  {
      func();//warning:'void fund()' is deprecated:Unsage method,please use xyz
  }
  ```

+ `[[likely]]`和`[[unlikely]]`:在对分支判断时，可以根据经验加上，有利用编译器优化。除非性能至关重要的情况下，很少使用该属性

##### 1.1.14 C风格数组

> 要获取基于栈的C风格数组的大小，可以使用`std::size()`函数(需要`<array>`)

##### 1.1.15 std::array

+ 使用`std::array`代替C风格数组可以有很多好处：

  - 它总是知道自己的大小
  - 不会自动退化为指针，从而避免了某些类型的bug
  - 具有迭代器，可方便的遍历元素

+ 必须在尖括号中指定两个参数，类型和大小

  ```c++
  array<int,3> arr{9,8,7};
  ```

+ C++支持*类模板参数推导(CTAD)*,所以允许这样初始化`std::array`:

  ```c++
  array arr{9,8,7};
  ```

##### 1.1.18 std::optional

> 如果允许值时可选的，则可以将`optional`用于函数的参数

```c++
option<int> getData(bool giveIt)
{
    if(giveIt){
        return 42;
    }
    return nullptr;//or return {};
}

optional<int> data1{getData(true)};
optional<int> data2{getData(false)};

//可以使用has_val()判断一个optional是否有值,或直接用在if语句中
cout << "data1.has_value = " << data1.has_value() << endl;
if(data2){
    cout << "data2 has a value." << endl;
}

//如果optional有值，可以使用value()或解引用运算符访问
cout << "data1.value = " << data1.value() << endl;
cout << "data1.value = " << *data1 << endl;
//如果对空的optional使用value(),将会派出std::bad_optional_access异常

//value_or()可以用来返回optional的值，如果optional为空，则返回指定值
cout << "data2.calue = " << data2.value_or(0) << endl;
```

> 注意，不能将引用放在`optional`中，即`optional<T&>`是无效的，但是可以将指针保存在`optional`中

##### 1.1.19 结构化绑定

+ 必须为结构化绑定使用`auto`关键字，不能使用`int`代替

+ 使用结构化绑定声明的变量**必须**与右侧表达式中的值的数量匹配

  ```c++
  auto [x,y,z]{11,22,33};
  
  struct Point{double m_x,m_y,m_z};
  Point point;
  point.m_x = 1.0;point.m_y = 1.0;point.m_z = 1.0;
  auto [x,y,z]{point};
  
  pair myPair{"hello",5};
  auto [theString,theInt]{myPaiar};
  ```

##### 1.1.20 循环

+ 在循环中使用`continue`经常被认为是不良的编码风格

+ 从`C++20`开始，范围for循环也可以使用初始化器

  ```c++
  for(<initializer>;<for-range-declaration> : <for-range-initializer>)
  {
      <body>
  }
  ```

##### 1.1.21 初始化列表

+ `std::initializer_list`是一个模板，要求在尖括号之间指定列表中的元素类型

+ 也支持CTAD，同时拒绝*narrowing conversion*《Effective Mordern C++》

  ```c++
  import <initializer_list>;
  using namespace std;
  int makeSun(initializer_list<int> vals)
  {
      int total{0};
      for(int val:vals)
      {
          total += val;
      }
      return total;
  }
  ```

##### 1.1.25 统一初始化

+ C++20引入了指派初始化器，是为了使用变量的名称来初始化所谓的聚合的数据成员

+ 聚合类型需满足一下条件：

  - 仅public数据成员
  - 无用户声明或继承的构造函数
  - 无虚函数和无虚基函数，private或protected的基类

  > 大多数情况下可以看成C的结构体形式

+ 指派初始化的顺序必须和数据成员的声明顺序一致

+ 不能混合使用指派初始化器和统一初始化

指派初始化实例：

```c++
struct Employee{
    char first;
    char last;
    int a;
    int b{200};
}
//统一初始化：
Employee e{"j","k",22,11};
//指派初始化
Employee e{
    .first = "j",
    .last = "k",
    .a = 22,
    .b = 11
};
//注意  "."
```

使用指派初始化器的好处：

+ 可读性强，可以看到正在初始化那些变量
+ 使用指派初始化器，如果对某些值的默认值满意，可以偷懒不写。而统一初始化必须就需要显示的指定默认值初始化为多少
+ 最后一个较大的好处是，当新成员添加到数据结构的时候，现有的代码不需要更改仍然可以运行，而新添加的数据成员则执行默认初始化

##### 1.1.27 const的用法

官方认证了！！const的解读，可以看做仅对左侧有效，如果左侧没有内容，则对右侧有效。

##### 1.1.28 constexpr关键字

constexpr函数可以在编译器进行执行，在C++20之前(C++引入了consteval)，都是性能优化的一种手段。但是constexpr函数也可以在运行时计算，这在客户端使用的时候会造成一定的麻烦，因为客户可能使用constexpr的“特性”。

##### 1.1.29 constecal关键字

如果确实希望函数或者对象只能在编译期间进行求值，C++20的consteval关键字将函数转换为所谓的*立即函数(immediate function)*,该类型的对象或者函数限制了**只能**在编译期间进行计算。

##### 1.1.31 const_cast()

C++提供了5种类型转换：const_cast(),static_cast(),reinterpret_cast(),dynamic_cast()以及std::bit_cast()(C++20引入)。具体的类型转换在第10章。

const_cast的作用就是去除或者添加变量的const属性。尽管我们希望变量的CV特性在整个程序中保持不变，但是在程序开发中，总会出现意外。尤其是在使用第三方库的时候，如下场景：

```c++
void ThirdParty(char * str);

void f(const char *s)
{
    ThirdParty(const_cast<char *>(s));
}
```

此外，标准库提供了一个名为std::as_const()的辅助方法。该方法定义在`<utility>`中，该方法接收一个引用参数，返回它的const引用版本。基本上就等价与const_cast<const T&>(obj);其优势可能就是代码更短，可读性更强。

```c++
string str{"C++20"};
const string & constStr{as_const(str)};
```

##### 1.1.35类型推断

需要小心和`as_const()`使用的时候：`auto`的类型推导除了个别情况下，和模板类型推导法则一直（《Effective Mordern C++》）。而as_const和类型推导没关系，注意脑袋不要单线程！

auto和统一初始化在C++17以后和C++14/C++11不同：

```c++
//拷贝列表初始化
auto a = {11};//initializer_list<int>
auto b = {11,22};//initializer_list<int>
//直接列表初始化
auto c{11};//int
auto d{11,22};//error,too many elements
auto e = {11,22.33}//compile error,列表初始化禁止变窄转换
//而早期版本C++11/C++14的拷贝列表初始化和直接列表初始化，auto推断出来的都是initializer_lit<int>
//拷贝列表初始化
auto a = {11};//initializer_list<int>
auto b = {11,22};//initializer_list<int>
//直接列表初始化
auto c{11};//initializer_list<int>
auto d{11,22};//initializer_list<int>
```

### ch02 使用string和stirng_view

#### 2.1 动态字符串

##### 2.1.1 C风格字符串

字符串的最后一个字符是`null`字符(`\0`)，官方将这个`null`字符定义为`NUL`，注意，只有一个L，区别`NULL`

##### 2.1.2 字符串字面量

对于字符串字面量，就不需要转义字符了。原始字符串字面量以`R"(`开头，以`)"`结尾。如：

```c++
const char* str{R"(Hello "World"!)};//output: Hello "World"!
//同样的，对于换行也不需要\n，直接回车即可
const char *str {R"(Line1
Line2)};
/*output:
Line1              -------------->enter here
Line2*/
//如果原始字符串中需要输出"()",用法类似sql客户端执行存储过程时开头定义一个特殊符号，结尾再重置，如：
const char * str{R"-(Embedded )" characters)-"};  //"-" 就是标志开始和结束的符号
```

##### 2.1.3 C++std::string类

除了常用的比较运算符重载，C++20还提供了三向运算符提供使用：

```c++
auto result{a <=> b};
if(is_lt(result)){cout << "less" << endl;}
if(is_gt(result)){cout << "greater" << endl;}
if(is_eq(result)){cout << "equal" << endl;}
```

对于data()方法，在C++14以及更早的版本中，始终和c_str()一样返回`const char*`。从C++17开始，在非const字符上调用时，会返回`char*`(类似CString的GetString()和GetBuffer())

注意一点：string调用find查看查找结果时，不应该和其他容器一样通过是否等于尾迭代器，而应该使用`string::npos`,如：

```c++
string str1{"hello"};
string str2{"the world"};
auto pos {str1.find("!!")};
if(pos != string::npos){
    //found "!!
}
```

> 从C++20开始，std::string是constexpr类，这意味着string可用在编译期执行操作，并且可以使用于constexpr函数和类的实现

当和auto配合使用时，如何区分`const char *`和`std::string`呢，答案是使用用户定义的标准字面量`s`将字面量解释为`std::string`：

```c++
auto string1{"hello"};//const char *
auto string2{"hello"s};//std::string
```

标准用户定义字面量`s`在`std::literals::string_literals`名称空间中定义，需要前置声明以下语句之一：

```c++
using namespace std;
using namespace std::literals;
using namespace std::string_literals;
using namespace std::literals::string_literals;
```

当你使用`vector`的`CTAD`特性推导`std::string`时，记得加上用户定义的标准字面量`s`:

```c++
vector names{"jerry","orange"};//vector<const char *>
vector names{"jerry"s,"orange"s};//vector<std::string>
```

##### 2.1.4 数值转换

数值转化为字符串：`std::to_string`

字符串转化为数值，**可以指定起始位置和选择进制**（之前完全没用过）：

+ int stoi(const string &str,size_t *idx=0,int base=10);
+ long stol(const string &str,size_t *idx=0,int base=10);
+ unsigned long stoul(const string &str,size_t *idx=0,int base=10);
+ long long stoll(const string &str,size_t *idx=0,int base=10);
+ unsigned long long stoull(const string &str,size_t *idx=0,int base=10);
+ float stof(const string &str,size_t *idx=0,int base=10);
+ double stod(const string &str,size_t *idx=0,int base=10);
+ long double stold(const string &str,size_t *idx=0,int base=10);

C++也提供了低级数据和字符串转换的函数，这些函数也是为所谓的*完美往返*而设计的，意味着将数值转换为字符串表示，再讲结果字符串反序列化为数值，结果与原始值完全相同。

数值转换为字符串：

```c++
to_chars_result to_chars(char *first,char* last,IntegerT value,int base = 10);
struct to_chars_result{
    char* ptr;
    errc ec;
};
//如果转换成功，ptr成员将等于所写入字符尾后一位置的指针。如果转换失败，即ec=errc::value_too_large,则ptr=last
//使用实例：
const size_t BufferSize{50};
string out(BufferSize,' ');//string with 50 space characters
auto result{to_chars(out.data(),out.data()+out.size(),12345)};//convert 123456 to chars
if(result.ec == errc{}){cout << out << endl;}//conversiong success

//使用结构化绑定，可以更简洁
string out(BufferSize,' ');//string with 50 space characters
auto [ptr,error]{to_chars(out.data(),out.data()+out.size(),12345)};
if(error == errc{}){cout << out << endl;}//conversiong success

//类似也提供了浮点数版本
to_chars_result to_chars(char *first,char* last,FloatT value);
to_chars_result to_chars(char *first,char* last,FloatT value,chars_format format);
to_chars_result to_chars(char *first,char* last,FloatT value,chars_format format,int precision);
enum class chars_format{
    scientific,//(-)d.ddde±dd
    fixed,	   //(-)ddd.ddd
    hex,	   //(-)h.hhhp±d(Note:no 0x!)
    general = fixed | scientific
};
//使用实例：
double val{0.314};
string out{BufferSize,' '};
auto[ptr,error]{to_chars(out.data(),out.data()+out.size(),val)};
if(error == errc{}){cout << out << endl;}//conversiong success

//对于字符串转换为数字也提供一下方法：
from_chars_result from_chars(const char *first,const char *last,IntegerT& value,int base = 10);
from_chars_result from_chars(const char *first,const char *last,FloatT& value,chars_format format= chars_format::general);
struct from_chars_result{
    const char* ptr;
    errc ec;
};
//结果ptr指向第一个未转换字符的指针。注意，from_chars()不会忽略任何前导空白
```

##### 2.1.5 std::string_view类

C++17之前，在设置字符串实参的时候，为了兼容旧代码传入的`const char *`,一般会选择`const string&`作为形参。这样做会在传入`const char *`的时候构造一个临时字符串对象。而如果形参设置为`const char *`，则传入`std::string`的时候需要使用`c_str()`或者`data()`

在C++17中，通过引入`std::string_view`类解决了所有这些问题，`std::string_view`是`std::basic_string_view`

的实例，在`<string_view>`中定义。简而言之，`string_view`就是`const string&`的简单替代品，但不会产生开销。另外，`string_view`提供了`remove_prefix(size_t)`和`remove_suffix(size_t)`来收缩字符串。

使用实例：

```c++
string_view func(string_view filename)
{
    return filename.substr(filename.rfind('.'));
}
//该函数可以用于所有类型的不同字符串
stirng filename{R"(c:\temp)"};
cout << format("C++ string:{}",func(filename)) << endl;

const char *cString{R"(c:\temp)"};
cout << format("C string:{}",func(cString)) << endl;

cout << format("Literal:{}",func(R"(c::\temp)")) << endl;
```

`string_view`还提供一个构造函数，可以从非NUL终止的字符创中指定长度构件`string_view`。如果字符串以NUL结尾，则不需要统计字符数目。

```c++
const char *raw{...};
size_t length{...};
cout << format("Raw:{}",func({raw,length}) << endl;
//等价于：
cout << format("Raw:{}",func(string_view{raw,length}) << endl;
```

> 即，当一个函数的参数使用const string&或者const char *时，可以使用std::string_view进行代替

但是需要注意的是，无法从`string_view`直接转化为`string`。需要通过调用形如`string(string_view)`或者`string_view.data()`的形式构造`string`。因此在`string`重载的运算符中也不能直接使用`string_view`对象当做参数。但是`append()`是可以的。实例：

```c++
string_view func(string_view filename);
void handle(const string&);
//该调用是不允许的
handle(func("test"));//func返回的是string_view，不能直接用来构造string
//允许的调用如下:
handle(func("test").data());
handle(string{func("test")});
//对于string的运算符是无法直接和string_view进行运算的
strintg str{"hello"};
string_view sv{" world"};
auto result{str + sv};//error
//应该是使用如下方式，或者使用append
auto result{str+sv.data()};
string res2{str};
result2.append(sv.data(),sv.size());
```

注意：需要返回字符串的时候，还是需要返回`std::string`

不能使用`string_view`保存一个临时字符串的视图：

```c++
stirng s{"hello"};
string_view sv{s+"world"};
cout << sv;//error sv is dead in line2
```

可以使用标准的用户定义字面量`sv`，将字符串字面量解释为`std::string_view`。如：

```c++
auto sv{"test"sv};
//需要使用命名空间，声明为以下之一
using namespace std::literals::string_view_literals;
using namespace std::string_view_literals;
using namespace std::literals;
using namespace std;
```

#### 2.2 字符串格式化

`format`中使用`{}`来作为占位符，占位符中的形式声明格式为`[index][:specifier]`,如果需要输出`"{or}"`应转义为`{{or}}`.

对于index是可以省略的，但是指明的话可以自己控制顺序，类似QString。

##### 2.2.1 格式说明符

对于`[:specifier]`支持的形式如下：

`[[fill]align][sign][#][0][width][.precision][type]`

+ width:指定参数的占位宽度,宽度可以在占位符中声明，也可以写为`{}`,然后宽度紧跟在填充实参的下一位

  ```c++
  cout << format("|{:5}|",42) << endl;	// output:|   42|
  cout << format("|{:{}}|",42，7) << endl;// output:|     42|
  ```

+ [fill]align:fill指定填充字符，align指定对其方式：

  - `<`左对齐
  - `>`右对齐
  - `^`居中对齐

  ```c++
  cout << format("|{:_5}|",42) << endl;	// output:|___42|使用_填充三个空白
  cout << format("|{:<{}}|",42，7) << endl;// output:|42     |，左对齐
  cout << format("|{:_^5}|",42) << endl;	// output:|_42__|  使用_填充三个空白并居中对齐
  ```

+ sign:是否显示数字符号

  - `-`只显示负数的符号（default）
  - `+`正负数的符号都显示
  - `space`负数显示符号,整数显示空格

  ```c++
  cout << format("|{:<+5}|",42) << endl;  //|+42  |
  cout << format("|{:< 5}|",42) << endl;  //| 42  |
  cout << format("|{:< 5}|",-42) << endl; //|-42  |
  ```

+ #:备用格式，如果整形启用，则会在数字前加入0x,0X,0b,0B等

+ type:指定值要被格式化的类型：

  - 整形：b（二进制），B（二进制，指定#时，使用0B而非0b），d(十进制)，o（八进制），x（十六进制），X（十六进制，当指定#时，使用0X而非0x）。默认使用d
  - 浮点型：e,E:科学计数法 f,F:固定表示法  g,G:通用表示法 a,A:带有字母的十六进制表示法，默认使用g
  - 布尔型：s(以文本形式输出true,false)，b,B,d,o,x,X(整形表示)，默认使用s
  - 字符型：c(输出字符副本)，b,B,d,o,x,X(整形表示)，默认使用c
  - 字符串：s
  - 指针：p（0X前缀的十六进制表示法）

+ precision：只能用于浮点和字符串类型。保留几位小数

+ 0：使用0填充空白width

##### 2.2.3 支持自定义类型

C++20格式库添加了对自定义类型的支持。需要编写`std::formatter`类模板的特化版本，该类模板包含两个方法模板：`parse()`和`format()`(具体使用在12章)

简单示例：

```c++
class KV
{
    public:
    	KV(string_view key,int val):m_key{key},m_val{val}{}
    	int getVal() const{return m_val;}
    private:
    	string m_key;
    	int m_value;
};
//该formatter类的特化版本仅支持格式化形式如：{:a}输出键 {:b}输出值 {:c}{}输出键值对
template<>
class formatter<keyValue>
{
    public:
    	//parse()
    	constexpr auto parse(auto & context)
        {
            auto iter{context.begin()};
            const auto iter{context.end()};
            if(iter == end || *iter == '}'){//default "fmt"
                m_outputType = OutputType::KeyAndValue;
                return iter;
            }
            
            switch(*iter){//"fmt" with [:specifier]
                case 'a':
                    m_outputType = OutputType::KeyOnly;
                    break;
                case 'b':
                    m_outputType = OutputType::ValueOnly;
                    break;
                case 'c':
                    m_outputType = OutputType::KeyAndValue;
                    break;
                defaule:
                    throw format_error{"Invalid KV format specifier"};
            }
            
            ++iter;
            if(iter != end && *iter != '}'){
                throw format_error("Invalid KV format specifier");
            }
            return iter;
        }
    
    	//format()
    	auto format(const KeyCalues& kv,auto &context)
        {
            switch(m_outputType){
                using enum OutputType;
                case KeyOnly:
                    return format_to(context.out(),"{}",kv.getKey());//将格式化好的数据写入context.out()
                case ValueOnly:
                    return format_to(context.out(),"{}",kv.getValue());
                default:
                    return format_to(context.out(),"{} - {}"，kv.getKey(),kv.getValue());
            }
        }
    
    private:
    	enum class OutputType
        {
            KeyOnly,
            ValueOnlu,
            KeyAndValue
		};
    	OutPutType m_outputType{OutputType::KeyAndValue};
};
```

