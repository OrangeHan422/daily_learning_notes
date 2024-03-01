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

##### 1.1.24 作用域解析
