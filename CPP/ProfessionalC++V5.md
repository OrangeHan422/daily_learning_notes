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

### ch03 代码规范

> 已经经历过上一家公司的折磨。人教人教不会，事教人记得牢。已经舔过屎山，体会了痛苦。

## Part2 专业的C++设计

> 和软件工程概述差不多，具体的设计在后续章节。该章节内容记录意义不大，更多的需要实践巩固。

SOLID原则：

+ S:SRP(Single Responsibility Principle)单一责任原则
+ O:OCP(Open/Close Principle)开放/关闭原则
+ L:LSP(Liskov Substitution Principle)里氏替换原则
+ I:ISP(Interface Sergregation Principle)接口隔离原则
+ D:DIP(Dependency Inversion Principle)依赖倒置原则

## Part3 C++编码方法

### ch07 内存管理

> 该章节内容较浅，推荐查看《Effective Mordern C++》章节4，更详细，更底层

#### 7.1 使用动态内存

##### 7.1.2 分配和释放

good habit:释放内存后，记得置空(nullptr)         经常忘记-------->或者尽可能的使用智能指针

`new`存在一个`nothrow`版本，在内存分配不足时返回空指针而非抛出异常，但是仍然推荐异常版本

```c++
int *ptr{new(nothrow) int};
```

##### 7.1.3 数组

不要在C++中使用`realloc()`,十分危险

#### 7.2 数组-指针的对偶性

##### 7.2.1 数组就是指针

> 标题容易让人困惑，该章节如果日后困惑，查看《C和指针》，讲述更清晰

C++17提供了计算数组大小的功能

```c++
int arr[]{1,2,3};
auto size = std::size(arr);//C++17
auto size = sizeof(arr)//before C++17
```

#### 7.3 底层内存操作

##### 7.3.2 自定义内存管理

自行管理内存可用于减少开销。当使用`new`分配内存的时候，程序还需要少量的空间来记录分配了多少内存。这样对很小的对象或分配了大量对象的程序来说，这个开销的影响可能会很大。

自定义`new`和`delete`在15章详细介绍

#### 7.4 常见的内存陷阱

虽然很清楚内存问题，但是记不住问题的名字，还是记录一下吧，方便和同事交流或者面试：

+ 内存泄漏:堆上分配的指针没有及时释放
+ 内存越界：经典实例是缓冲区溢出错误，即写入数组尾部后面的内存
+ 野指针：可以是内存泄漏的指针，也可以是释放后没有及时置空的指针

#### 7.5 智能指针

由于原子性和异步性不能保证，在C++17之前，使用智能指针和`new`会导致内存泄漏的问题《Effective Mordern C++》章节4，智能指针。C++17后改善了这个问题，但是为了考虑代码的稳定性，仍然推荐使用智能指针和`make`函数结合使用

C++20以来提供了`make_unique_for_overwrite()`和`make_shared_for_overwrite()`函数来执行默认初始化值的构造函数。而原生的`make`函数使用的是值初始化。

使用`unique_ptr`:

+ 使用`reset()`,可释放唯一指针的底层指针，并且可以根据需要将其改成另一个指针。

+ `release()`返回资源的底层指针，然后将智能指针置为`nullptr`。此时需要自己负责资源的释放

+ 唯一指针不能复制，只能进行移动：

  ```c++
  class Foo
  {
  public:
      Foo(unique_ptr<int> data):m_data(move(data)){}
  private:
      unique_ptr<int> m_data;
  };
  auto myIntSmartPtr {make_unique<int>(42)};
  Foo f{move(myIntSmartPtr)};
  ```

使用`shared_ptr`:

+ 和唯一指针的操作一致，但是不支持`release()`，可以使用`use_count()`检索共享同一资源的智能指针的数量
+ 可以使用`const_pointer_cast()`、`dynamic_pointer_cast()`、`static_pointer_cast()`和`reinterpret_pointer_cast()`对智能指针进行类型转换，使用方法和其他类型的C++类型转换一致

访问`weak_ptr`中保存的指针:

+ 使用虚指针的`lock()`方法，该方法返回一个`shared_ptr`
+ 创建一个共享指针，使用虚指针作为构造函数的参数

> 自C++17开始，虚指针可以像共享指针一样支持C风格的数组

### ch08 熟悉类和对象

#### 8.2 编写类

##### 8.2.1 类定义

C++20模块化的使用实例：

```c++
export module spreadsheet_cell;//当前模块名是spreadsheet_cell，并且通过export允许其他地方进行import

export class SpreadsheetCell//如果该类允许客户使用，则应该使用export关键字
{
    public:
    	void setValue(double val);
    private:
    	doublie m_value{0};//可以直接在定义中初始化成员变量。-------------->不记得在那本书看到，该形式部分编译器不支持，最好还是放在构造函数中使用初始化器进行初始化。
}
```

```c++
module spreadsheet_cell;//实现文件要先声明该实现文件实现的是哪个（些）模块

void SpreadsheetCell::setValue(val)
{
    m_value = val;
}
```

#### 8.3 对象的生命周期

C++支持委托构造函数，允许构造函数初始化器调用同一个类的其他构造函数。

委托构造函数允许构造函数调用同一个类的其他构造函数。然而，这个调用不能放在构造函体内，只能放在初始化器中，且必须是列表中唯一的初始化器

```c++
//委托构造函数示例
SpreadsheetCell::SpreadsheetCell(string_view initialVal)//string_view 可以看做是const string&的简单替换，基本上就是指针和长度(const char* + size)
    :SpreadsheetCell{stringToDouble(initialVal)}
{}
```

需要注意，不使用统一初始化时，调用默认构造函数是不需要写`()`的，否则会因引起最头疼的解析(most vexing parse),比如：

```c++
SpreadsheetCell myCell();//可以编译通过，但是这不是一个对象变量声明。而是一个函数声明(无参，返回值为SpreadsheetCell)！！
```

单参数的构造函数可以将类对象隐式转换为参数类型，这种构造函数称为转换构造函数。如果不想出现隐式转换，则需要`explicit`关键字：

```c++
export class SpreadsheetCell
{
    public:
    	explicit SpreasheetCell(std::string_view initialVal);
}
```

> 建议将任何单参数构造函数标记为`explicit`，避免意外的隐式转换。除非你确定需要这种隐式转换

从C++20开始，可以将布尔值传递给`explicit`,语法如下：

```c++
explicit(true) Myclass(int);
```

仅仅写出`explicit(true)`就等价于`explicit`。它在使用所谓的类型萃取的泛型模板代码中更加有用。使用类型萃取，可以查询给定类型的某些属性，例如某个类型是否可以转换为另一个类型。类型萃取的记过可以用作`explicit()`的参数。更多细节在26章“高级模板”中介绍。

### ch09 精通类和对象

#### 9.2 对象中的动态内存分配

##### 9.2.1 Spreadsheet类

使用C的库，不一定支持模块，所以应该使用`#include`

如果当前类需要使用另一个类，并且另一个类也要向客户提供，在C++20中应该如此定义：

```c++
module;
#include <cstddef>//导入C的库需要执行如此操作

export module spreadsheet;//当前模块为spreadsheet，需要向客户提供

export import spreadsheet_cell;//spreadsheet类需要spreadsheet_cell类，并且spreadsheet_cell类需要向客户提供

export class Spreadsheet
{
  ...  
};
```

旧式的C++分配二维数组十分麻烦：

```c++
SpreadsheetCell ** m_cells{nullptr};
size_t m_width{};
size_t m_height{};
m_cells = new SpreadsheetCell*[m_width];
for(size_t i{0}; i < m_width; ++i){
    m_cells[i] = new SpreadsheetCell[m_height];
}
```

##### 9.2.3 处理复制和赋值

在处理复制或者赋值的时候，我们需要一种全有或者全无的机制（类似数据库中的事务），为此我们需要使用“复制和交换”惯用方法。

```c++
export class Spreadsheet
{
    public:
    	Spreadsheet& operator=(const Spreadsheet& rhs);
    	void swap(Spreadsheet& othre) noexcept;//必须加上nocept，这样在遇到异常时，程序会直接终止，保证该函数要么成功，要么直接退出程序
};

//swap中可以直接使用标准库中的swap
void Spreadsheet::swap(Spreadsheet& other)
{
    std::swap(m_width,other.m_width);
    std::swap(m_height,other.m_height);
    std::swap(m_cells,other.m_cells);
}
```

注意：使用“复制和交换”就不需要执行自我赋值的检查了。

标准库中`swap`的实现：

```c++
template <typename T>
void swap(T& a,T& b)
{
    T temp(std::move(a));
    a = std::move(b);
    b = std::move(temp);
}
```

##### 9.2.4 使用移动语义处理移动

如果源对象是操作结束后会销毁的临时对象，或显式使用`std::move()`时，编译器会选择移动语义的函数进行执行。

需要注意的是，即使是右值引用参数，在函数内，还是一个左值，因为它有名字。示例：

```c++
void helper(string &&msg){}//handleMsg调用该函数
void handleMsg(string &&msg)//一个移动语义的函数
{
    helper(msg);//错误，msg是一个左值，因为它有名字！
    helper(std::move(msg));//正确用法。
}
```

移动语义应该使用`noexcept`限定符标记，因为移动语义是指针的操作，如果没有完成就抛出异常，会导致内存泄漏等问题

```c++
export class Spreadsheet
{
    public:
    	Spreadsheet(Spreadsheet&& src) noexcept;
    	Spreadsheet& operator=(Spreadsheet&& rhs) noexcept;
};
```

定义在`<utility>`中的`std::exchange()`函数可以使用一个新值替换旧值，并且返回旧值。例如：

```c++
int a{11};
int b{22};
int ret{exange(a,b)};//交换后a=22,b=11,ret=11;

//所以在函数中如果需要进行赋值，并且赋值对象需要置空，可以进行类似操作
void moveFrom(Type& rhs) noexcept
{
    m_data = exchange(rhs.m_data,0);
    /*等价于：
    m_data = rhs.m_data;
    rhs.m_data = 0;*/
}
```

对于`return object`形式的语句，如果`object`是局部变量，函数参数，或者临时值，则会进行*返回值优化(RVO)*。如果`object`是一个局部比那辆，则会启动*命名返回值优化(NRVO)*。这两种优化都是复制省略的形式，即编译器自动执行移动语义。这就导致了所谓的*零拷贝值传递语义*。注意，不要自己使用`return std::move(object)`,如果对象不支持移动语义，则会使用复制语义影响性能。

即：当函数返回一个局部变量或者参数的时候，直接写`return obj`就好了，不要使用`std::move()`画蛇添足。

对于无论如何都要复制的参数，可以考虑使用**值传递**统一左值和右值函数。考虑如下情形：

```c++
class DataHolder
{
    public:
    	void SetData(const std::vector<int>& data){m_data = data;}
    	void SetData(const std::vector<int>&& data){m_data = std::move(data);}
    private:
    	std::vector<int> m_data;
}

//可以使用值传递进行统一:
class DataHolder
{
    public:
    	void SetData(std::vector<int> data){m_data = std::move(data);}
    private:
    	std::vector<int> m_data;
}

//原理：实参和形参结合实际上就是赋值操作
/*如果传递左值：
std::vector<int> ldata;
data = ldata;此时执行的是左值的复制操作，将实参内容复制给形参
对于函数体内，执行move操作移动的是形参（即ldata的副本）
m_data = std::move(data);
综上，传递左值 执行了一次复制操作

如果传递右值：
data = std::vector<int>{};
执行的就是一个移动语义，没有复制操作
函数体内也是移动语义，没有复制操作
综上，传递右值 执行了零次复制操作*/
```

##### 9.2.5 零规则

旧式C++因为存在`new`和`delete`类似自己管理内存的操作，所以需要遵循5规则(rule of five)。

但是，如果完全使用现代C++，包括智能指针，标准模板库等，需要应用0原则(rule of zero),因为标准模板库中对于5函数定义更专业，自定义类如果使用的完全现代的C++，使用编译器默认生成的版本足够使用了。

> 除非项目是完全使用现代C++，个人认为还是应该使用5原则。

#### 9.3 与方法有关的更多内容

##### 9.3.2 const方法

应该养成习惯，将不修改对象的所有方法声明为const，这样可在程序中使用const对象的引用。

有时候，const函数中可能需要改变数据成员，可以将数据成员设置为`mutable`，告诉编译器在const函数中，可以修改该变量：

```c++
mutable int dood{};
int func(int param) const
{
    dood++;//mutable变量可以在const函数中进行修改
    return param;
}
```

##### 9.3.3 方法重载

通常情况下，const版本和非const版本的实现是一样的。为了避免代码重复，可以使用Scott Meyer的`const_cast()`模式

```c++
export class Spreadsheet
{
  public:
    SpreadsheetCell& GetCellAt(size_t x,size_t y);
    const SpreadsheetCell& GetCellAt(size_t x,size_t y) const;
};

//对于原始的版本，const和非const函数体实现如下：
const SpreadsheetCell& Spreadsheet::GetCellAt(size_t x,size_t y) const
{
    func(x,y);
    return m_cells[x][y];
}
//使用Scott Meyer的const_cast()模式可以简化为			这样的模式对于函数体复杂时，可以大幅的减少代码量。
SpreadsheetCell& Spreadsheet::GetCellAt(size_t x,size_t y) const
{
    return const_cast<SpreadsheetCell&>(as_const(*this).GetCellAt(x,y));
}
```

当一个函数的两个版本，仅仅是返回值为左值或者右值的区别，我们无法重载，可以使用引用限定符进行重载：

```c++
const string &getText() const &
{
    return m_text;
}
string &&getText() &&
{
    return std::move(m_text);
}
```

#### 9.5 嵌套类

嵌套的类有权方位外围类的所有成员，而外围类只能访问嵌套类的`public`成员

#### 9.7 运算符重载

如果你定义了`operator==`，诸如`10 == MyClass`的表达式，C++20的编译器会帮你重写`MyClass == 10`,此外，编译器还会自动添加`!=`的支持。

#### 9.8 创建稳定的接口

基本原则是为每个类都定义两个类：接口类和实现类。

接口类只有一个数据成员pimpl指针指向实现类

实现类拥有和接口类完全一样的方法，以及实际需要的所有数据成员

即pimpl idiom模式。这样做可以大大减小程序的编译依赖，从而提升编译速度。

为将实现和接口分离，另一种方法是使用抽象接口以及实现该接口的实现类，抽象接口是只有纯虚函数的接口。

### ch10 揭秘继承技术

#### 10.1 使用继承构建类

派生类“**是一个**”基类

##### 10.1.1 扩展类

指向Base对象的指针可以指向Derived对象，对于引用也是如此。但是客户只能访问基类的方法和成员。

C++允许将类标记为`final`，这意味着继承这个类会导致编译报错。

##### 10.1.2 重写方法

>  一旦将方法或者析构函数标记为`virtual`，他们在所有的派生类都是`virtual`，即使没有标明。

> 对象本身“知道”自己所属的类，因此，只要这个方法声明为`virtual`，就会自动调用对应的方法（即多态）。比如基类对象的引用实际引用了派生类的对象，那么调用虚函数就会调用派生类的函数。
>
> 但是需要注意的是，即使基类指针或者引用知道自己指向的是一个派生类，也无法访问没有在基类定义的方法或者成员。

>  好习惯是在重写虚函数的时候都加上`override`函数，这样在重写的时候可以避免很多问题。比如，如果加上`override`关键字，在派生类将参数类型改变的时候就会报错，但是没有该关键字就会创建一个新的虚函数。

> 重写非虚函数，会“隐藏”基类定义的方法，并且重写的这个方法只能在派生类环境中使用。

> C++在编译类的时候，会创建一个包含类中所有方法的二进制对象。
>
> 在非虚情况下，将控制交个正确方法的代码是硬编码，此时根据的是编译期的类型调用方法。称之为“静态绑定”或者“早绑定”
>
> 在虚函数的情况下，会使用虚表(vtable)来调用正确的实现。即每个拥有虚函数的类都有一张虚表，每个对象都包含指向虚表的指针。当某个对象调用方法时，指针进入虚表，根据实际的对象类型在运行时执行正确的方法。称之为“动态绑定”或“晚绑定”

> 存在虚函数的基类，都需要将析构函数声明为`virtual`《Effective C++》
>
> 除非这个类是`final`类

#### 10.3 利用父类

##### 10.3.4 向上转型和向下转型

如果直接使用派生类初始化基类对象，会发生截断

正确做法应该是使用基类的引用或者指针绑定到派生类。称之为向上转型

反之，将基类转换为派生类称之为向下转型，这种做法不被推荐。如果确实需要，应该使用`dynamic_cast`进行转换。注意：向下转换是设计不良的标志。

#### 10.4 继承与多态性

纯虚函数的定义是在方法声明后加上`=0`

对于同辈的对象之间的拷贝构造，拷贝构造函数需要显示声明：

```c++
//  StringSpreadsheetCell ---inherit--->SpreadsheetCell<----inherit---DoubleSpreadsheetCell
export class StringSpreadsheetCell:public SpreadsheetCell
{
    public:
    	StringSpreadsheetCell() = default;
    	StringSpreadsheetCell(const DoubleSpreadsheetCell &cell)
            :m_value(cell.getString())
        {}
};
```

#### 10.6 有趣而晦涩的继承问题

##### 10.6.1 修改重写方法的返回类型

经验之谈的话，重写方法要使用于基类一直的方法声明。实现可以改变，但是原型保持不变。

但是事实并非如此，在C++中，如果原始的饭后类型是某个类的指针或者引用，重写的方法可以将返回类型改为派生类的指针或者引用。这种类型称之为协变返回类型(covariant return types)。示例：

```c++
/*class Cherry		class CherryTree
		^					^
		|					|
  class BingCherry	class BingCherryTree*/

//假设Tree都有虚方法pick()
Cherry* CherryTree::pick(){return new Cherry();}
//这种重写是允许的，因为多态的向上转换是可以的，但是返回类型不能是其他诸如void *的类型
BingCherry* BinCherryTree::pick()
{
    auto theCherry{make_unique<BingCherry>()};
    //do something...
    return theCherry.release();
}

//实际使用
BingCherryTree theTree;
//注意这里返回的是BingCherry，但是可以正常运，因为这里存在多态的向上转换
unique_ptr<Cherry> theCherry{theTree.pick()};
theCherry->printType();
```

##### 10.6.2 派生类中添加虚基方法的重载

> 这种情况比较少见：使用虚基类中的虚方法，在派生类中重新定义一个重载版本的虚函数。可以使用using解决该问题

```c++
class Base
{
    public:
    	virtual void func();
};

class Derived:public Base
{
    public:
    	using Base::func;
    	virtual void func(int i);
};
```

##### 10.6.3 继承的构造函数

因为派生类和基类中数据成员可能存在不同，所以基类构造函数是不可以在派生类直接使用的，因为可能存在未定义行为。如果确定没有未定义行为，可以使用`using Base::Base`形式允许派生类直接使用基类的构造函数。

```c++
class Base
{
    public:
    	Base() = default;
    	Base(std::string_view str);
    	virtual ~Base() = default;
};

class Derived:public Base
{
    public:
    	using Base::Base;
    	Derived(int i);
};

Base base{"hello"};	//OK
Derived d1{1};		//OK
Derived d2{"hello"};//OK only if using Base::Base is declared
Derived d3;			//ERROR,no default ctor declared
```

需要注意的是，如果派生类重写了基类构造函数（即参数一致），和所有重写一样，派生类的构造函数优先级会更高。

另一点需要注意的是，不可以只继承一部分的基类构造函数，如果继承，会继承除了默认构造函数以外的所有构造函数。

如果两个基类存在相同的参数列表版本的构造函数，就不能继承，因为会存在歧义。正确做法是派生类自己声明这样的构造函数（即隐藏基类的构造函数）

##### 10.6.4 重写方法时的特殊情况

在C++中静态方法是不能重写的。即使派生类的静态方法和基类中的静态方法声明完全一致，也是两个独立的函数。因为静态方法仅和类相关，和对象类型无关：

```c++
Derived d;
Base& ref{d};
d.staticFunc();		//call Derived staticFunc();
ref.staticFunc();	//call Based staticFunc();
```

>  实际还是静态类型和动态类型的关系:
>
> 静态类型都是在编译期确定，和类相关
>
> 动态类型是在运行期确定，和对象的实际类型有关

当重写某个方法时，编译器会隐式地隐藏基类中同名方法的所有其他实例。即：如果重写了给定名称的某个方法，应该重写所有同名方法，否则应该当做错误处理。如：

```c++
class Base
{
    public:
    	virtual void overload(){cout << "Base overload()" << endl;}
    	virtual void overload(int i){cout << "Base overload(int i)" << endl;}
    	virtual ~Base() = default;
};

class Derived:public Base
{
    public:
    	void overload() override{
            cout <<"Drived overload()" << endl;
        }
};

//此时使用Derived对象就无法调用基类的有参版本的overload函数
Derived d;
d.overload(2);//ERROR

//只能通过多态进行使用
Base& ref{d};
ref.overload(2);//OK
```

即如果重写拥有重载版本的方法，派生类仅存在已经重写的版本，对于没有重写的版本是没有继承的。所以要么对所有的重载版本进行重写，要么使用`using Derived::overload()`然后重写特定版本。

> 可以理解为虚函数中继承一般只看名称。对于多态的步骤应该是：
>
> + 运行时确定对象类型
> + 到虚表中通过函数名称，确定对应的虚函数是派生类的虚函数还是基类的虚函数
> + 在派生类或者基类寻找对应重载版本

最后一个比较有趣的例子，来加深静态和动态的理解：即存在默认参数的虚函数

```c++
class Base
{
    public:
    	virtual void go(int i = 2){cout << "Base go:" << i << endl;}
    	virtual ~Base() = default;
};

class Derived:public Base
{
    public:
		virtual void go(int i = 7){cout << "Derived go:" << i << endl;}
};

//有趣的东西
Base b;
Derived d;
Base& bRef{d};

b.go();		//Base go:2
d.go();		//Derived go:7
bRef.go();	//Derived go:2----------->hahaha
//个人理解：
//默认参数是静态检查确定的，即编译期确定了默认参数，而编译期是通过类类型确定。
//而具体的虚函数则是动态检查确定的，即运行时确定使用派生类的函数还是基类的函数，即运行时通过对象类型确定的
//对于该例子中，首先bRef在编译期根据类类型确定为Base，所以确定了默认参数为2
//然后再运行时，bReg引用的对象类型为Derived，所以确定了虚函数应该使用Derived中的虚函数，即Derived go
//综上，将函数和参数结合，bRef.go()输出：Derived go:2
```

##### 10.6.6 运行时类型工具

在C++中提供了对象的运行时的特性集，即运行时类型信息(Run Time Type Information,RTTI)的特性集。`dynamic_cast()`就属于该特性集。

另一个特性(在源码中见到过)，就是`typeid`运算符，这个运算符可以在运行时查询对象。示例：

```c++
import <typeinfo>;

class Animal{public:virtual ~Animal() = default;};
class Dog:public Animal{};
class Bird:public Animal{};


void speak(const Animal& animal){
    if(typeid(animal) == typeid(Dog)){
        cout << "Woof!" << endl;
    }else if(typeid(animal) == typeid(Bird)){
        cout << "Chirp!" << endl;
    }
}
```

注意，至少有一个虚函数，`typeid`运算符才能正常运行(毕竟是运行时，所有多态都涉及运行时的类型检查（至少目前所学的是这样的）)。另外`typeid`也会去除引用和CV限定

在大多数情况下，都应该考虑使用虚函数替换`typeid`，比如上例中可以定义虚函数`speak()`在基类中，然后再派生类进行重写。

`typeid`最有用的应该是日志系统中，或者帮助调试的时候。

#### 10.7 类型转换

##### 10.7.1 static_cast()

C++类型规则技术上允许的转化都可以使用

##### 10.7.2 reinterpret_cast()

比`static_cast()`功能更大，但是“能力越大，责任越大”，强转以为着突破规则限制，这时候可能出现的内存问题或其他问题就只能靠自己了。（使用实例可以回想自己工作中修改LiteZip源码，统一文件句柄时，将`void *`转换为`int`的操作）

##### 10.7.3 std::bit_cast()

对于不同类型的对象，进行位级别的复制。（联想工作中数据库中的二进制数据存储结构体的例子）

##### 10.7.4 dynamic_cast()

主要用于类型检查，尤其在向下转换时，即基类向派生类转换时（注意，这是十分不好的设计）

```c++
Base* pBase;
Derived* pDerived{new Derived()};
pBase = pDerived;//派生类向基类转化，向上转换，没有问题
pDerived = dynamic_cast<Derived *>(pBase);//向下转换，如果失败会返回空指针，对于引用会抛出异常
```

##### 10.7.5 类型转换总结

| 场景                                                   | 类型转换                            |
| ------------------------------------------------------ | ----------------------------------- |
| 移除const属性                                          | const_cast()                        |
| 语言支持的显示转化                                     | static_cast()                       |
| 用户定义的构造函数或者转换函数支持的显式强制转换       | static_cast()                       |
| 一个类的对象转换为无关的类对象                         | bit_cast()                          |
| 在同一继承层次结构中，一个类的指针转换为另一个类的指针 | 建议dynamic_cast()，或static_cast() |
| 在同一继承层次结构中，一个类的引用转换为另一个类的引用 | 建议dynamic_cast()，或static_cast() |
| 执行一中类型的指针转换为指向其他不相关类型的指针       | reinterpret_cast()                  |
| 一种类型的引用转换为其他不相关类型的引用               | reinterpret_cast()                  |
| 指向函数的指针转换为指向函数的指针                     | reinterpret_cast()                  |

