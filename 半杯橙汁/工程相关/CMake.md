# CMake官方示例

粗略的说，CMake就是生成编译配置文件，在Linux上就是生成makefile。

构建工程文件需要编写的是CMakeList.txt

## 第一步 起点

### 练习1 最简单的CMake项目

**`CMakeLists.txt`**

```cmake
# TODO 1: 设置CMake最低版本要求为 3.10
cmake_minimum_required(VERSION 3.10)

# TODO 2: 创建一个名为Tutorial的项目
project(Tutorial)

# TODO 3: 为项目添加一个叫做 Tutorial 的可执行文件
# Hint: 一定要指定源文件 tutorial.cxx
add_executable(Tutorial tutorial.cxx)
```

**要点**

①cmake_minimum_required

用于指定所需cmake最低版本

用法与示例：

```cmake
# 用法
cmake_minimum_required(VERSION <版本号>)
# 示例
cmake_minimum_required(VERSION 3.10)
```

如果当前使用的cmake版本低于所指定的版本，则会报错并且终止执行。

②project

指定项目名称

用法与示例：

```cmake
# 用法
project(<项目名>)
# 示例 指定项目名称为Tutorial
project(Tutorial)
```

③add_executable

利用指定的源文件在项目中添加可执行文件

用法与示例：

```cmake
# 用法 源文件可以有多个，用空格隔开
add_executable(<可执行文件名> <源文件列表>)
# 示例 可执行文件名为Tutorial，用到的源文件为tutorial.cxx
add_executable(Tutorial tutorial.cxx)
```

④cmake命令常用执行方法

```bash
# 用法
cmake -G <生成器名称> <CMakeLists.txt所在的目录>

# Linux下一般流程,进入项目根目录后
mkdir build;cd build;cmake ..
```

如果使用默认生成器，则-G这部分可以省略，具体支持哪些生成器可以用cmake --help查看

> 扩展：设置环境变量CMAKE_GENERATOR可以指定默认生成器，简化cmake命令的执行

### 练习2 指定C++标准

**`CMakeLists.txt`**

```cmake
# TODO 1: 设置CMake最低版本要求为 3.10
cmake_minimum_required(VERSION 3.10)

# TODO 2: 创建一个名为Tutorial的项目
project(Tutorial)

# TODO 7: 用上面project命令将项目版本设为 1.0

# TODO 6: 设置变量 CMAKE_CXX_STANDARD 为 11
#          CMAKE_CXX_STANDARD_REQUIRED 为 True
set(CMAKE_CXX_STANDARD 26)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# TODO 8: 用 configure_file 复制 TutorialConfig.h.in 生成
#         TutorialConfig.h

# TODO 3: 为项目添加一个叫做 Tutorial 的可执行文件
# Hint: 一定要指定源文件 tutorial.cxx
add_executable(Tutorial tutorial.cxx)
```

**要点**

①set

用于给变量设置值

用法与示例：

```cmake
# 用法
set(<变量名> <变量值>)
# 示例
set(CMAKE_CXX_STANDARD 26)
set(SRC_DIR /home/src)
```

②CMAKE_CXX_STANDARD

变量，用于指定C++标准

用法与示例：

```cmake
# 用法 截止2023/6 std_num∈{98,11,14,17,20,23,26}
set(CMAKE_CXX_STANDARD <std_num>)
# 示例
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
```

> 在C++中可以通过输出__cplusplus查看当前编译器所用的标准
>
> | __cplusplus的值 | 对应的C++标准 |
> | --------------- | ------------- |
> | 199711          | C++98         |
> | 201103          | C++11         |
> | 201402          | C++14         |
> | 201703          | C++17         |
> | 202002          | C++20         |
> | 202100          | C++23         |

③CMAKE_CXX_STANDARD_REQUIRED

变量，如果设置为True，则通过CMAKE_CXX_STANDARD设置的C++标准是必需的，如果编译器不支持该标准则会输出错误提示信息。如果不设置或者设置为False，则CMAKE_CXX_STANDARD设置的C++标准不是必需的，如果编译器不支持对应的标准，则会使用上一个版本的标准进行编译。

用法与示例：

```cmake
set(CMAKE_CXX_STANDARD_REQUIRED True)
```

### 练习3 添加版本号和配置头文件

有些时候需要让源代码能访问CMakeLIsts.txt当中的数据，比如说在CMakeLists.txt中定义版本号之后，希望能在源程序中对版本号进行输出。本节内容为如何让源代码中能访问CMakeLists.txt中的变量数据。

```cmake
CMakeLists.txt
# TODO 1: 设置CMake最低版本要求为 3.10
cmake_minimum_required(VERSION 3.10)

# TODO 2: 创建一个名为Tutorial的项目
project(Tutorial VERSION 11.25)

# TODO 7: 用上面project命令将项目版本设为 1.0

# TODO 6: 设置变量 CMAKE_CXX_STANDARD 为 11
#          CMAKE_CXX_STANDARD_REQUIRED 为 True
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# set(STR_TEST "Hello World")

# TODO 8: 用 configure_file 复制 TutorialConfig.h.in 生成
#         TutorialConfig.h
configure_file(TutorialConfig.h.in TutorialConfig.h)

# TODO 3: 为项目添加一个叫做 Tutorial 的可执行文件
# Hint: 一定要指定源文件 tutorial.cxx
add_executable(Tutorial tutorial.cxx)

# TODO 9: 用 target_include_directories 添加头文件搜索目录 ${PROJECT_BINARY_DIR}
# PUBLIC PRIVATE INTERFACE
target_include_directories(Tutorial PUBLIC ${PROJECT_BINARY_DIR})
TutorialConfig.h.in
#define Tutorial_VERSION_MAJOR @Tutorial_VERSION_MAJOR@
#define Tutorial_VERSION_MINOR @Tutorial_VERSION_MINOR@
```

**要点**

〇project第二种用法

定义项目名和版本号

```cmake
project(Tutorial VERSION 2.15)
```

①configure_file

将输入文件复制为输出文件，并把其中的变量引用替换为CMakeLists.txt中定义的变量，如果变量未定义，则替换为空串。输入文件中的变量引用方式为**@@变量名@@**或者**${变量名}**。

输入文件(.h.in)默认路径为CMakeLists.txt所在的路径，输出文件(.h)的路径默认为cmake生成文件所在的路径。

用法与示例：

```cmake
# 用法
configure_file(<inputfile> <outputfile>)
# 示例
configure_file(TutorialConfig.h.in TutorialConfig.h)
```

在输出文件(.h)中，用宏定义的方式对变量进行定义

```cmake
#define Tutorial_VERSION_MAJOR @Tutorial_VERSION_MAJOR@
#define Tutorial_VERSION_MINOR ${Tutorial_VERSION_MINOR}

// 因为CMakeLists.txt中定义的字符串都是裸的，所以如果一个变量的值为字符串，需要用双引号包起来
#define STR_VAR "@STR_VAR@"
```

上述定义中@Tutorial_VERSION_MAJOR@、${Tutorial_VERSION_MINOR}、@STR_VAR@在输出文件中会被替换为CMakeLists.txt中定义的对应变量值。

 

②target_include_directories

给指定的目标添加头文件搜索路径。

用法与示例：

```cmake
# 用法
target_include_directories(<target> <INTERFACE|PUBLIC|PRIVATE> <dir1 dir2 ...>)

# 示例
target_include_directories(Tutorial PUBLIC ${PROJECT_BINARY_DIR})
```

> PROJECT_BINARY_DIR:二进制文件存储目录（即可执行文件生成位置）

③_VERSION_MAJOR

版本号第一个组成部分。该变量为cmake自动定义的一个变量，不需要手动定义，值来自于project的定义。其中为用**project**定义的项目名。

> 一般形式为：项目名称_VERSION_MAJOR,或者直接PROJECT_VERSION_MAJOR

④_VERSION_MINOR

版本号第二个组成部分。该变量为cmake自动定义的一个变量，不需要手动定义，值来自于project的定义。其中为用**project**定义的项目名。

> 一般形式为：项目名称_VERSION_MINOR,或者直接PROJECT_VERSION_MINOR

## 第二步 加个库

### 练习1 创建库文件

前面的练习当中创建了可执行文件。本节将学习如何创建库文件以及库文件的使用 。同时也将练习将一个项目划分为多个子目录的方法。

```cmake
CMakeLists.txt
cmake_minimum_required(VERSION 3.10)

# set the project name and version
project(Tutorial VERSION 1.0)

# specify the C++ standard
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# TODO 7: Create a variable USE_MYMATH using option and set default to ON


# configure a header file to pass some of the CMake settings
# to the source code
configure_file(TutorialConfig.h.in TutorialConfig.h)

# TODO 8: Use list() and APPEND to create a list of optional libraries
# called  EXTRA_LIBS and a list of optional include directories called
# EXTRA_INCLUDES. Add the MathFunctions library and source directory to
# the appropriate lists.
#
# Only call add_subdirectory and only add MathFunctions specific values
# to EXTRA_LIBS and EXTRA_INCLUDES if USE_MYMATH is true.

# TODO 2: Use add_subdirectory() to add MathFunctions to this project

add_subdirectory(MathFunctions)




# add the executable
add_executable(Tutorial tutorial.cxx)

# TODO 9: Use EXTRA_LIBS instead of the MathFunctions specific values
# in target_link_libraries.

# TODO 3: Use target_link_libraries to link the library to our executable
# 即三方库名，三方库名在三方库目录中的CMakeLists中使用add_library定义
# 在g++命令中相当于-lMathFunctions
target_link_libraries(Tutorial PUBLIC MathFunctions)

# TODO 4: Add MathFunctions to Tutorial's target_include_directories()
# Hint: ${PROJECT_SOURCE_DIR} is a path to the project source. AKA This folder!

# TODO 10: Use EXTRA_INCLUDES instead of the MathFunctions specific values
# in target_include_directories.

# add the binary tree to the search path for include files
# so that we will find TutorialConfig.h
# 该步骤添加的是头文件搜索路径，即include_dir
# 在g++命令中相当于-I /path/build -I path/MathFunctions
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           "${PROJECT_SOURCE_DIR}/MathFunctions"	
                           )

# MathFunctions/CMakeLists.txt，即三方库中的cmake配置文件
# TODO 1: Add a library called MathFunctions
# Hint: You will need the add_library command
# 通过mysqrt.cxx生成lMathFunctions.a静态库
add_library(MathFunctions mysqrt.cxx)
```

**要点**

①add_subdirectory

为当前项目添加子目录。子目录当中必须包含一个CMakeLists.txt文件，其中可以不写cmake_minimum_required与project。

用法与示例：

```cmake
# 用法
add_subdirectory(<source_dir>)
# 示例
add_subdirectory(MathFunctions)
```

②target_link_libraries

为指定目录指定链接库。

用法与示例：

```cmake
# 用法
target_link_libraries(<target> ... <item>... ...)
# 示例
target_link_libraries(Tutorial PUBLIC MathFunctions)
```

③PROJECT_SOURCE_DIR

最后一次调用project的CMakeLists.txt文件所在的目录。

④add_library

用指定的源文件生成库文件。

用法与示例：

```cmake
# 用法
add_library(<name> [<source>...])
# 示例
add_library(MathFunctions mysqrt.cxx MathFunctions.h)
```

 

 

### 练习2 库文件可选编译

本节内容为设置库文件（子目录）可选编译。

需要注意的是，除了CMakeLists文件需要修改，cmake生成的配置头文件TutorialConfig.h.in也要添加option设置的宏(需要使用`#cmakedefine XXX`定义宏)，在源程序中也要添加宏判断区分使用不同库

```cmake
cmake_minimum_required(VERSION 3.10)

# set the project name and version
project(Tutorial VERSION 1.0)

# specify the C++ standard
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# TODO 7: Create a variable USE_MYMATH using option and set default to ON
# 创建cmake的布尔值，参数分别为变量名，描述，ON|OFF。
# 在进行cmake构建工程的时候，可以使用-DUSE_MYMATH=ON|OFF 来控制生成makefile
option(USE_MYMATH "Use My Math?" OFF)

# configure a header file to pass some of the CMake settings
# to the source code
configure_file(TutorialConfig.h.in TutorialConfig.h)

# TODO 8: Use list() and APPEND to create a list of optional libraries
# called  EXTRA_LIBS and a list of optional include directories called
# EXTRA_INCLUDES. Add the MathFunctions library and source directory to
# the appropriate lists.
#
# Only call add_subdirectory and only add MathFunctions specific values
# to EXTRA_LIBS and EXTRA_INCLUDES if USE_MYMATH is true.

# TODO 2: Use add_subdirectory() to add MathFunctions to this project
if(USE_MYMATH)
    add_subdirectory(MathFunctions)
    # 可选三方库APPEND至EXTRA_LIBS中
    list(APPEND EXTRA_LIBS MathFunctions) 
    # 可选三方库头文件目录APPEND至EXTRA_INCLUDES中
    list(APPEND EXTRA_INCLUDES "${PROJECT_SOURCE_DIR}/MathFunctions")
endif()



# add the executable
add_executable(Tutorial tutorial.cxx)

# TODO 9: Use EXTRA_LIBS instead of the MathFunctions specific values
# in target_link_libraries.

# TODO 3: Use target_link_libraries to link the library to our executable
# 添加的三方库都在EXTRA_LIBS变量中存储，EXTRA_LIBS是一个cmake的list
target_link_libraries(Tutorial PUBLIC ${EXTRA_LIBS})

# TODO 4: Add MathFunctions to Tutorial's target_include_directories()
# Hint: ${PROJECT_SOURCE_DIR} is a path to the project source. AKA This folder!

# TODO 10: Use EXTRA_INCLUDES instead of the MathFunctions specific values
# in target_include_directories.

# add the binary tree to the search path for include files
# so that we will find TutorialConfig.h
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                   # 三方库的头文件目录都在EXTRA_INCLUDES中存储，EXTRA_INCLUDES是一个cmake的list
                           "${EXTRA_INCLUDES}"
                           )
```

**要点**

①option

提供一个布尔变量，可以让用户自行选择。

用法与示例：

```cmake
# 用法
option(<variable> "<help_text>" [value])
# 示例
option(USE_MYMATH "Use MyMath" ON)
```

`value`值为`ON`或`OFF`，默认值为`OFF`。

在执行配置时，可以用`-D`来指定值，例如

```bash
cmake . -DUSE_MYMATH=OFF
```

②if() & endif()

条件判断开始与结束。

语法：

```cmake
if(<condition>)
  <commands>
elseif(<condition>)
  <commands>
else()
  <commands>
endif()
```

| <condition>判断为真的值 | <condition>判断为假的值     |
| ----------------------- | --------------------------- |
| 1                       | 0                           |
| ON                      | OFF                         |
| TRUE                    | FALSE                       |
| YES                     | NO                          |
| Y                       | N                           |
| 其他非0数               | IGNORE                      |
|                         | NOTFOUND或以-NOTFOUND结尾的 |
|                         | 值不是判断为真的字符串      |

③list

列表操作。详细操作见[list](https://gitee.com/unlimited13/cpp/blob/master/cmake/cmake.md#)，这里只讲用到的APPEND操作。将一些元素追加到已有的列表当中。如果列表变量还未定义，则会当做空列表处理。

语法与示例：

```cmake
# 语法
list(APPEND <list> [<element> ...])
# 示例 将MathFunctions追加到EXTRA_LIBS当中
list(APPEND EXTRA_LIBS MathFunctions)
```

④cmakedefine

用法与#define相同，用在configure_file的输入文件当中进行宏定义。

不同点在于，#define本身就是C/C++当中的宏定义，所以不论对应的变量是否在CMakeLists.txt中有定义，都会在输出文件中定义一个宏。而#cmakedfine则会根据变量在CMakeLists.txt中的定义情况来确定是否会在输出文件中定义宏。如果变量在CMakeLists.txt中没有定义或都已定义但是一个判断为假的布尔值，则不会在输出文件中定义对应的宏，如果变量在CMakeLists.txt中有定义且不为布尔值、或者为布尔值但判断为真，则会在输出文件中定义对应的宏。

用法示例：

```cmake
#cmakedefine USE_MYMATH
```



## 第三步 添加使用依赖

### 练习1 为库添加使用依赖

如果三方库过多，每个都要在主项目中添加include的路径（即APPEND EXTRA_INCLUDES，以及target_include_directories）过于麻烦，这些工作我们放在三方库的CMakeLists中，而主项目中仅APPEND EXTRA_LIBS以及target_link_libraries即可。

```cmake
CMakeLists.txt
cmake_minimum_required(VERSION 3.10)

# set the project name and version
project(Tutorial VERSION 1.0)

# specify the C++ standard
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# should we use our own math functions
option(USE_MYMATH "Use tutorial provided math implementation" ON)
message(STATUS "OUT --- ${CMAKE_CURRENT_SOURCE_DIR}")

# configure a header file to pass some of the CMake settings
# to the source code
configure_file(TutorialConfig.h.in TutorialConfig.h)

# TODO 2: 删除EXTRA_INCLUDES

# add the MathFunctions library
if(USE_MYMATH)
  add_subdirectory(MathFunctions)
  list(APPEND EXTRA_LIBS MathFunctions)
endif()

# add the executable
add_executable(Tutorial tutorial.cxx)

target_link_libraries(Tutorial PUBLIC ${EXTRA_LIBS})

# TODO 3: 删除EXTRA_INCLUDES

# add the binary tree to the search path for include files
# so that we will find TutorialConfig.h
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )
                           
                           
# MathFunctions/CMakeLists.txt即三方库中的CMakeLists文件
add_library(MathFunctions mysqrt.cxx)

# TODO 1: 声明所有需要链接MathFunctions库的都要在头文件搜索中加入当前当前目录，但是MathFunctions本身不需要
# Hint: 用target_include_directories和INTERFACE  
# PUBLIC 本目标需要用，依赖这个目标的其他目标也需要用
# INTERFACE  本目标不需要，依赖本目标的其他目标需要
# PRIVATE 本目标需要，依赖这个目标的其他目标不需要
target_include_directories(MathFunctions INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}")
message(STATUS "MathFunction --- ${CMAKE_CURRENT_SOURCE_DIR}")
```

**要点**

①PUBLIC | INTERFACE | PRIVATE

在使用`target_include_directories`和`target_link_libraries`添加搜索目录时，有三个修饰符`PUBLIC | INTERFACE | PRIVATE`，其含义如下：

**PUBLIC**：当前目标和以当前目标为依赖的目标都能能使用添加的目录，都能在对应的目录中进行搜索

**PRIVATE**：只有当前目标能使用添加的目录，以当前目标为依赖的目标不能使用

**INTERFACE**：以当前目标为依赖的目标需要使用添加的目录，但当前目标不需要用这种方式添加对应搜索目录时用INTERFACE。

 

②CMAKE_CURRENT_SOURCE_DIR

变量。当前CMakeLists.txt所在的目录。



## 第四步 生成器表达式

### 练习1 用接口库设置C++标准

就是自己定义一个抽象的库，即接口库，仅用来设置规范相关的内容。

```cmake
CMakeLists.txt
# TODO 4: Update the minimum required version to 3.15

cmake_minimum_required(VERSION 3.10)

# set the project name and version
project(Tutorial VERSION 1.0)

# TODO 1: 将下面的代码替换为:
# * 创建一个interface库tutorial_compiler_flags
#   Hint: use add_library() with the INTERFACE signature
# * 添加编译特性cxx_std_11到tutorial_compiler_flags
#   Hint: Use target_compile_features()
# 就是添加一个接口库用来管理项目的规范，如C++版本等。设计理念类似抽象接口类。
add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_14)



# TODO 5: 创建一些辅助变量用来确定用的是哪个编译器:
# * 创建一个变量gcc_like_cxx如果用的是CXX并且用的是下列任意一个编译器那么值为true
#         ARMClang, AppleClang, Clang, GNU, LCC
# * 创建一个变量msvc_cxx如果用的是CXX和MSVC那么值为true
# Hint: Use set() and COMPILE_LANG_AND_ID

# TODO 6: 向interface库tutorial_compiler_flags中添加警告选项：
# 
# * 如果是gcc_like_cxx, 添加 -Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused
# * 如果是msvc_cxx, 添加 -W3
# Hint: Use target_compile_options()

# TODO 7: 用嵌套生成器表达式, 只在构建的时警告
# 
# Hint: Use BUILD_INTERFACE

# should we use our own math functions
option(USE_MYMATH "Use tutorial provided math implementation" ON)

# configure a header file to pass some of the CMake settings
# to the source code
configure_file(TutorialConfig.h.in TutorialConfig.h)

# add the MathFunctions library
if(USE_MYMATH)
  add_subdirectory(MathFunctions)
  list(APPEND EXTRA_LIBS MathFunctions)
endif()

# add the executable
add_executable(Tutorial tutorial.cxx)

# TODO 2: 链接tutorial_compiler_flags

target_link_libraries(Tutorial PUBLIC ${EXTRA_LIBS} tutorial_compiler_flags)

# add the binary tree to the search path for include files
# so that we will find TutorialConfig.h
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )
                           
                           
# MathFunctions/CMakeLists.txt，即三方库目录下的CMakeLists文件
add_library(MathFunctions mysqrt.cxx)

# state that anybody linking to us needs to include the current source dir
# to find MathFunctions.h, while we don't.
target_include_directories(MathFunctions
          INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}
          )

# TODO 3: 链接tutorial_compiler_flags
# 将接口库链接到实体库
target_link_libraries(MathFunctions PUBLIC tutorial_compiler_flags)
```

**要点**

①INTERFACE库

使用`add_library(<libname> INTERFACE)`可以创建个Interface库，这样的库并不是真实存在的，是一个虚拟的库，通常用来传递一些选项。用法和正常的库一样，可通过`target_link_libraries`链接到目标，可以向指定的目标传递一些指定的参数选项。

 

②target_compile_features

`target_compile_features` 是 CMake 用来指定编译器特性的命令。它可以用来指定编译器需要支持的 C++ 标准或者其他编译器特性。具体支持的特性取决于编译器版本和 CMake 版本。

语法与示例

```cmake
target_compile_features(<target> <PRIVATE|PUBLIC|INTERFACE> <feature> [...])

# 示例
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_11)
```

以下是一些常见的特性：

- `cxx_std_11`：指定 C++11 标准。
- `cxx_std_14`：指定 C++14 标准。
- `cxx_std_17`：指定 C++17 标准。
- `cxx_std_20`：指定 C++20 标准。
- `cxx_constexpr`：启用 C++11 constexpr 函数。
- `cxx_nullptr`：启用 C++11 nullptr 关键字。
- `cxx_auto_type`：启用 C++11 auto 关键字。
- `cxx_lambdas`：启用 C++11 lambda 表达式。
- `cxx_range_for`：启用 C++11 range-based for 循环。
- `cxx_override`：启用 C++11 override 关键字。
- `cxx_final`：启用 C++11 final 关键字。

 

### 练习2 添加编译警告选项

**CMakeLists.txt解析过程**

CMake构建过程分为两个阶段

1. 配置阶段，CMake 会读取项目的 CMakeLists.txt 文件，并根据其中的指令和参数来生成 Makefile 或者 IDE 的项目文件
   - 检查编译器和工具链是否可用，并设置编译器选项和链接选项
   - 检查系统库和第三方库是否可用，并设置库的路径和链接选项
   - 检查项目的源代码文件，并设置编译选项和链接选项
   - 生成 Makefile 或者 IDE 的项目文件
   - 根据不同的平台和编译器生成不同的 Makefile 或者项目文件，以保证项目可以在不同的平台和编译器上构建
2. 生成阶段，CMake 会根据配置阶段生成的 Makefile 或者项目文件来执行实际的构建操作
   - 根据 Makefile 或者项目文件中的指令和参数来编译源代码文件，并生成目标文件
   - 根据 Makefile 或者项目文件中的指令和参数来链接目标文件，并生成可执行文件或者库文件

 

**生成器表达式**

CMake生成器表达式是一种特殊的语法，用于在CMake构建系统中动态地生成构建规则。它们可以用于指定编译器选项、链接选项等。

本节先学习其中两种表达式：

`$<condition:true_string>`

- 如果`condition`为1，则此表达式结果为`true_string`
- 如果`condition`为0，则此表达式结果为空

`$<COMPILE_LANG_AND_ID:language,compiler_ids>`

- 如果当前所用的语言与`language`一致且编译器ID在`compiler_ids`的列表中，则表达式值为1，否则为0
- `language`值主要为`CXX`和`C`
- `compiler_ids`主要有GNU、Clang、MSVC等，有多个时用逗号隔开

生成器表达式因为是在生成阶段可用，所以不能在配置阶段进行输出 ，可用下面方式调式

```cmake
add_custom_target(ged COMMAND ${CMAKE_COMMAND} -E echo "$<1:hello>")
```

配置完成之后，用以下命令进行输出

```bash
cmake --build . --target ged
# 用make可简写
make ged
```

但不是所有的表达式都能这样输出，有的表达式无法输出，比如`$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>`

 

```cmake
# TODO 4: Update the minimum required version to 3.15

cmake_minimum_required(VERSION 3.15)

# set the project name and version
project(Tutorial VERSION 1.0)

# TODO 1: 将下面的代码替换为:
# * 创建一个interface库tutorial_compiler_flags
#   Hint: use add_library() with the INTERFACE signature
# * 添加编译特性cxx_std_11到tutorial_compiler_flags
#   Hint: Use target_compile_features()
add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_14)

# add_custom_target(ged COMMAND ${CMAKE_COMMAND} -E echo "$<COMPILE_LANG_AND_ID:CXX,GNU>")

# TODO 5: 创建一些辅助变量用来确定用的是哪个编译器:
# * 创建一个变量gcc_like_cxx如果用的是CXX并且用的是下列任意一个编译器那么值为true
#         ARMClang, AppleClang, Clang, GNU, LCC
# * 创建一个变量msvc_cxx如果用的是CXX和MSVC那么值为true
# Hint: Use set() and COMPILE_LANG_AND_ID
# 生成表达式会在make命令的时候生效；这两个变量在g++编译器中gcc_like_cxx为1，msvc_cxx为0
set(gcc_like_cxx "$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>")
set(msvc_cxx "$<COMPILE_LANG_AND_ID:CXX,MSVC>")


# TODO 6: 向interface库tutorial_compiler_flags中添加警告选项：
# 
# * 如果是gcc_like_cxx, 添加 -Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused
# * 如果是msvc_cxx, 添加 -W3
# Hint: Use target_compile_options()
# 目标依旧选择之前构建的抽象库，再次说明，抽象库主要用来规范和配置，并没有实际程序用途
# 该语句意思为：为g++添加编译选项
target_compile_options(tutorial_compiler_flags INTERFACE 
  "$<${gcc_like_cxx}:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>"
  "$<${msvc_cxx}:-W3>"
)

# TODO 7: 用嵌套生成器表达式, 只在构建的时警告
# 
# Hint: Use BUILD_INTERFACE

# should we use our own math functions
option(USE_MYMATH "Use tutorial provided math implementation" ON)

# configure a header file to pass some of the CMake settings
# to the source code
configure_file(TutorialConfig.h.in TutorialConfig.h)

# add the MathFunctions library
if(USE_MYMATH)
  add_subdirectory(MathFunctions)
  list(APPEND EXTRA_LIBS MathFunctions)
endif()

# add the executable
add_executable(Tutorial tutorial.cxx)

# TODO 2: 链接tutorial_compiler_flags

target_link_libraries(Tutorial PUBLIC ${EXTRA_LIBS} tutorial_compiler_flags)

# add the binary tree to the search path for include files
# so that we will find TutorialConfig.h
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )
```

**要点**

①target_compile_options

给指定的目标添加编译选项。

语法及示例：

```cmake
target_compile_options(<target> [BEFORE]
  <INTERFACE|PUBLIC|PRIVATE> [items1...]
  [<INTERFACE|PUBLIC|PRIVATE> [items2...] ...])

# 示例
target_compile_options(Tutorial PUBLIC -std=c++11 -Wunused)
```

 

## 第五步 安装与测试

### 练习1 安装规则

```cmake
CMakeLists.txt
cmake_minimum_required(VERSION 3.15)

# set the project name and version
project(Tutorial VERSION 1.0)

# specify the C++ standard
add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_11)

# add compiler warning flags just when building this project via
# the BUILD_INTERFACE genex
set(gcc_like_cxx "$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>")
set(msvc_cxx "$<COMPILE_LANG_AND_ID:CXX,MSVC>")
target_compile_options(tutorial_compiler_flags INTERFACE
  "$<${gcc_like_cxx}:$<BUILD_INTERFACE:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>>"
  "$<${msvc_cxx}:$<BUILD_INTERFACE:-W3>>"
)

# should we use our own math functions
option(USE_MYMATH "Use tutorial provided math implementation" ON)

# configure a header file to pass some of the CMake settings
# to the source code
configure_file(TutorialConfig.h.in TutorialConfig.h)

# add the MathFunctions library
if(USE_MYMATH)
  add_subdirectory(MathFunctions)
  list(APPEND EXTRA_LIBS MathFunctions)
endif()

# add the executable
add_executable(Tutorial tutorial.cxx)
target_link_libraries(Tutorial PUBLIC ${EXTRA_LIBS} tutorial_compiler_flags)

# add the binary tree to the search path for include files
# so that we will find TutorialConfig.h
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )

# TODO 3: 安装 Tutorial 到 bin 目录  ${CMAKE_INSTALL_PREFIX}
# Hint: Use the TARGETS and DESTINATION parameters
# install(TARGETS targets... [DESTINATION <dir>])
# 该命令会将可执行文件安装到默认路劲下bin目录（如果没有会自动创建）
# 可以使用cmake -DCMAKE_INSTALL_PREFIX指定安装目录（类似原始的configure）
install(TARGETS Tutorial DESTINATION bin)
message(STATUS "${CMAKE_INSTALL_PREFIX}")

# TODO 4: 安装TutorialConfig.h到include目录
# Hint: Use the FILES and DESTINATION parameters
install(FILES "${PROJECT_BINARY_DIR}/TutorialConfig.h" DESTINATION include)


# MathFunctions/CMakeLists.txt，即三方库的目录
add_library(MathFunctions mysqrt.cxx)

target_include_directories(MathFunctions
          INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}
          )

target_link_libraries(MathFunctions tutorial_compiler_flags)
# 可以将三方库集合到一个变量中，方便安装
set(installable_libs MathFunctions tutorial_compiler_flags)
if(TARGET SqrtLibrary)
  list(APPEND installable_libs SqrtLibrary)
endif()
# 将库文件安装到默认目录下的lib中，注意，TARGET安装可以直接写在cmake中定义的三方库名
install(TARGETS ${installable_libs} DESTINATION lib)
# 将指定头文件安装到默认目录下的include中，注意，FILE需要写文件的全称（包括相对路径前缀）
install(FILES MathFunctions.h DESTINATION include)
```

**要点**

①if(TARGET target-name)

- 如果`target-name`是一个已经调用`add_executable`、`add_library`、`add_custom_target`创建的目标，则返回True

②install

用于定义安装规则。

语法与示例（简洁版）

```
# 安装生成的目标文件
install(TARGETS <目标名列表> DESTINATION <安装位置>)
# 安装其他文件
install(FILES <文件列表> DESTINATION <安装位置>)
```

安装多个文件时，用空格隔开。安装位置是相对于`CMAKE_INSTALL_PREFIX`的，`CMAKE_INSTALL_PREFIX`是安装时的默认路径，可以自行用`set`设置。

运行安装：

安装到默认路径下

```bash
cmake --install .
```

如果有多个生成版本，指定安装版本

```bash
cmake --install . --config Release
```

如果用的是IDE，用下列命令

```bash
cmake --build . --target install --config Debug
```

自行指定安装路径

```bash
cmake --install . --prefix "/path/to/your/installdir"
```

 

### 练习2 测试支持

`CTest`提供了一些测试管理。本节内容为给可执行文件创建单元测试。

```cmake
cmake_minimum_required(VERSION 3.15)

# set the project name and version
project(Tutorial VERSION 1.0)

# specify the C++ standard
add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_11)

# add compiler warning flags just when building this project via
# the BUILD_INTERFACE genex
set(gcc_like_cxx "$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>")
set(msvc_cxx "$<COMPILE_LANG_AND_ID:CXX,MSVC>")
target_compile_options(tutorial_compiler_flags INTERFACE
  "$<${gcc_like_cxx}:$<BUILD_INTERFACE:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>>"
  "$<${msvc_cxx}:$<BUILD_INTERFACE:-W3>>"
)

# should we use our own math functions
option(USE_MYMATH "Use tutorial provided math implementation" ON)

# configure a header file to pass some of the CMake settings
# to the source code
configure_file(TutorialConfig.h.in TutorialConfig.h)

# add the MathFunctions library
if(USE_MYMATH)
  add_subdirectory(MathFunctions)
  list(APPEND EXTRA_LIBS MathFunctions)
endif()

# add the executable
add_executable(Tutorial tutorial.cxx)
target_link_libraries(Tutorial PUBLIC ${EXTRA_LIBS} tutorial_compiler_flags)

# add the binary tree to the search path for include files
# so that we will find TutorialConfig.h
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )

# TODO 3: 安装 Tutorial 到 bin 目录  ${CMAKE_INSTALL_PREFIX}
# Hint: Use the TARGETS and DESTINATION parameters
# install(TARGETS targets... [DESTINATION <dir>])
# target: add_excutable add_library
install(TARGETS Tutorial DESTINATION bin)
message(STATUS "${CMAKE_INSTALL_PREFIX}")

# TODO 4: 安装TutorialConfig.h到include目录
# Hint: Use the FILES and DESTINATION parameters
install(FILES "${PROJECT_BINARY_DIR}/TutorialConfig.h" DESTINATION include)

# TODO 5: Enable testing
# 开启测试
enable_testing()

# TODO 6: 添加一个Runs测试，运行下面的命令:
# 测试名称叫做Runs，执行的命令是：$ Tutorial 25
add_test(NAME Runs COMMAND Tutorial 25)

# TODO 7: 添加一个叫Usage的测试，执行下面的命令:
# Hint: 用PASS_REGULAR_EXPRESSION属性匹配"Usage.*number"
# 测试名为Usage，执行的命令为$ Tutorial
add_test(NAME Usage COMMAND Tutorial)
#如果满足测试属性，该测试才算通过。该用例是输出内容满足正则表达式"Usage.*number"即通过。
set_tests_properties(Usage PROPERTIES PASS_REGULAR_EXPRESSION "Usage.*number")

# TODO 8: 再添加一个运行下面命令的测试:
# $ Tutorial 4
# 保证输出结果是正确的.
# Hint: 用PASS_REGULAR_EXPRESSION属性匹配"4 is 2"
add_test(NAME Com4 COMMAND Tutorial 4)
set_tests_properties(Com4 PROPERTIES PASS_REGULAR_EXPRESSION "4 is 2")


# TODO 9: 添加更多测试. 创建一个函数do_test完成重复内容
# 测试以下数值: 4, 9, 5, 7, 25, -25 and 0.0001.
# 声明测试函数do_test,参数为num，result
function(do_test num result)
  add_test(NAME Com${num} COMMAND Tutorial ${num})
  set_tests_properties(Com${num} PROPERTIES PASS_REGULAR_EXPRESSION "${num} is ${result}")
endfunction()
# 调用函数
do_test(9 3)
do_test(5 2.236)
do_test(7 2.645)
do_test(-25 "(-nan|nan|0)") # not a number
do_test(0.0001 0.001)


# 5 2.236
# 7 2.645
# -25 "(-nan|nan|0)"
# 0.0001 0.001
# do_test(4 2)
```

**要点**

①enable_testing()

开启当前目录及子目录的测试支持。

②add_test

添加一条测试

简版用法：

```cmake
add_test(NAME <name> COMMAND <command> [<arg>...])
```

- `name`为本条测试名称
- `command`测试用的命令
- `arg`传递测试命令的参数

③set_tests_properties

设置测试的属性。

语法

```
set_tests_properties(test1 [test2...] PROPERTIES prop1 value1 prop2 value2)
```

- `test1...`为用add_test添加的测试名
- `prop1`为需要设置的属性名，本节中只学`PASS_REGULAR_EXPRESSION`，表示测试程序的输出结果需要能匹配`value`所表示的正则表达式才能通过，如果匹配不了则不通过。
- `value`要设置的属性值

示例

```cmake
set_tests_properties(Usage
  PROPERTIES PASS_REGULAR_EXPRESSION "Usage:.*number"
  )
```

表示运行`Usage`这个测试时测试程序的输出结果要能正则匹配到"Usage:.*number"。

④function()与endfunction()

用于在定义函数，分别表示函数开始与函数结束

语法

```cmake
function(<name> [<arg1> ...])
  <commands>
endfunction()
```

- 括号里第一个参数为函数名，后面是参数列表，可以有多个，多个参数用空格隔开

示例：

```cmake
# 定义
function(do_test target arg result)
  add_test(NAME Comp${arg} COMMAND ${target} ${arg})
  set_tests_properties(Comp${arg}
    PROPERTIES PASS_REGULAR_EXPRESSION ${result}
    )
endfunction()

# 调用
do_test(Tutorial 4 "4 is 1")
```

 测试命令：

```bash
$cmake .. #配置
$ctest -N #输出有多少个测试
$ctest -VV #输出测试结果
```



## 第六步 添加测试面板支持

### 练习1 发送测试结果到测试面板

```cmake
CMakeLists.txt
# 将enable_testing()替换为下面这行
include(CTest)
```

在build目录执行

```bash
cmake -G "MinGW Makefiles" ..
```

之后执行

```bash
ctest -VV -D Experimental
```

即可。

完成之后可在[https://my.cdash.org/index.php?project=CMakeTutorial查看提交的测试结果。](https://gitee.com/link?target=https%3A%2F%2Fmy.cdash.org%2Findex.php%3Fproject%3DCMakeTutorial%E6%9F%A5%E7%9C%8B%E6%8F%90%E4%BA%A4%E7%9A%84%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%E3%80%82)

 

 

## 第七步 添加系统特性检查

### 练习1 评估依赖可用性

注意：源代码也要根据check_cxx_source_compiles以及target_compile_definitions中对应的宏，对代码进行更改

```cmake
# 所有修改都在MathFunctions/CMakeLists.txt
add_library(MathFunctions mysqrt.cxx)

# state that anybody linking to us needs to include the current source dir
# to find MathFunctions.h, while we don't.
target_include_directories(MathFunctions
          INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}
          )

# link our compiler flags interface library
target_link_libraries(MathFunctions tutorial_compiler_flags)

# TODO 1: Include CheckCXXSourceCompiles
# 添加cmake的模块文件
include(CheckCXXSourceCompiles)

# TODO 2:用check_cxx_source_compiles和简单C++代码检测
# 以下两个函数是否可用:
# * std::log ln
# * std::exp e^2
# 把结果存在HAVE_LOG 和 HAVE_EXP 中.

# Hint: Sample C++ code which uses log:
# #include <cmath>
# int main() {
#   std::log(1.0);
#   return 0;
# }


#检查std::log是否可用，结果存在HAVE_LOG变量中
check_cxx_source_compiles("
#include <cmath>
int main() {
  std::log(1.0);
  return 0;
}
" HAVE_LOG)

check_cxx_source_compiles("
#include <cmath>
int main() {
  std::exp(1.0);
  return 0;
}
" HAVE_EXP)

# TODO 3: 如果HAVE_LOG和HAVE_EXP为真, 添加预编译定义
# "HAVE_LOG"和"HAVE_EXP"到目标MathFunctions上.
#Hint: Use target_compile_definitions()
# target_compile_definitions相较于之前的configurefile命令好处是，生成的宏可以直接在cpp源代码中使用
if(HAVE_LOG AND HAVE_EXP)
    target_compile_definitions(MathFunctions PRIVATE "HAVE_LOG" "HAVE_EXP")
endif()



# install libs
set(installable_libs MathFunctions tutorial_compiler_flags)
install(TARGETS ${installable_libs} DESTINATION lib)
# install include headers
install(FILES MathFunctions.h DESTINATION include)
```

**要点**

①include

用于导入其他CMake文件或模块。

```cmake
include(<file|module> [OPTIONAL] [RESULT_VARIABLE <var>]
                      [NO_POLICY_SCOPE])
```

②check_cxx_source_compiles

检查给定的C++代码能不能编译及链接成可执行文件。通常用来检查当前环境中是否具有某些特性。

用法

```cmake
check_cxx_source_compiles(<code> <resultVar> [FAIL_REGEX <regex1> [<regex2>...]])
```

- `code`为需要检查的代码，需要包含`main`函数
- `resultVar`为检查结果，如果成功返回布尔真，否则返回布尔假
- `FAIL_REGEX`如果提供，则返回为假的结果需要能匹配上对应的正则表达式

③target_compile_definitions

为指定可执行文件及库文件这类目标添加编译器定义，用来控制代码中的条件编译。有点类似于`#cmakedefine`与`configure_file`的作用，但这两个操作的结果会生成一个文件再进行引用，而`target_compile_definitions`不会生成文件。

用法

```cmake
target_compile_definitions(<target>
  <INTERFACE|PUBLIC|PRIVATE> [items1...]
  [<INTERFACE|PUBLIC|PRIVATE> [items2...] ...])
```

示例

```cmake
target_compile_definitions(MathFunctions PRIVATE "HAVE_LOG" "HAVE_EXP")
```

 

 

## 第八步 添加自定义命令及用自定义命令生成文件

在Linux中，有许多的工具命令，例如`ls`、`mv`、`mkdir`等。在CMake项目中，可以用源代码写一些自定义小工具，然后在CMake中进行调用，来完成一些工作。

本节的内容为自定义一个`MakeTable`命令用来生成指定范围整数的平方根并保存到文件中，在计算的时候可以用这些已经计算好的值来辅助计算。

```cmake
MathFunctions/CMakeLists.txt
# 注意依赖的头文件也要加入
add_library(MathFunctions mysqrt.cxx Table.h)
# 添加一个可执行程序
add_executable(MakeTable MakeTable.cxx)
# 添加可执行命令
add_custom_command(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/Table.h
  COMMAND MakeTable ${CMAKE_CURRENT_BINARY_DIR}/Table.h
  DEPENDS MakeTable
)

# state that anybody linking to us needs to include the current source dir
# to find MathFunctions.h, while we don't.
target_include_directories(MathFunctions
          INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}
          #Table.h头文件地址，因为是通过MakeTable生成在可执行文件目录下的
          PRIVATE ${CMAKE_CURRENT_BINARY_DIR} 
          )

# link our compiler flags interface library
target_link_libraries(MathFunctions tutorial_compiler_flags)

# does this system provide the log and exp functions?
include(CheckCXXSourceCompiles)
check_cxx_source_compiles("
  #include <cmath>
  int main() {
    std::log(1.0);
    return 0;
  }
" HAVE_LOG)
check_cxx_source_compiles("
  #include <cmath>
  int main() {
    std::exp(1.0);
    return 0;
  }
" HAVE_EXP)

# add compile definitions
if(HAVE_LOG AND HAVE_EXP)
  target_compile_definitions(MathFunctions
                             PRIVATE "HAVE_LOG" "HAVE_EXP")
endif()

# install libs
set(installable_libs MathFunctions tutorial_compiler_flags)
install(TARGETS ${installable_libs} DESTINATION lib)
# install include headers
install(FILES MathFunctions.h DESTINATION include)
```

**要点**

①add_custom_command

执行自定义指令。

简版用法

```cmake
add_custom_command(OUTPUT output1
                   COMMAND command1
                   DEPENDS depends)
```

- `OUTPUT`指定输出文件名
- `COMMAND`指定要执行的指令
- `DEPENDS`执行指令需要依赖的内容。如果是由`add_executable`或`add_library`添加的目标名，写这一条可以保证对应目标的生成。

 

 

## 第九步 打包安装程序

发布程序可以有多种形式，比如安装包、压缩包、源文件等。CMake也提供了打包程序`cpack`可将程序打包成多种形式。

只需要在顶层CMakelists.txt中添加以下代码

```cmake
include(InstallRequiredSystemLibraries)
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/License.txt")
set(CPACK_PACKAGE_VERSION_MAJOR "${Tutorial_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${Tutorial_VERSION_MINOR}")
set(CPACK_SOURCE_GENERATOR "TGZ")
include(CPack)
```

在项目构建完成之后，可以直接执行

```
cpack
```

在Windows上默认情况会打包成.exe文件，所以需要先安装一个exe打包程序NSIS(Null Soft Installer)

NSIS下载地址：[https://sourceforge.net/projects/nsis/](https://gitee.com/link?target=https%3A%2F%2Fsourceforge.net%2Fprojects%2Fnsis%2F)

也可以指定生成器打包成对应的格式

```bash
cpack -G ZIP # 打包成ZIP
```

具体生成器各类可以通过`cpack --help`查看

对于多配置项目，可以指定打包配置

```bash
cpack -C Debug # 打包Debug版本
```

也可以打包源代码

```bash
cpack --config CPackSourceConfig.cmake
```

 

## 第十步 选择静态链接库或动态链接库

在add_library的时候是可以指定静态库或者动态库。也可以使用option对BUILD_SHARED_LIBS变量进行设置，让用户选择生成动态库还是静态库

```cmake
CMakeLists.txt
cmake_minimum_required(VERSION 3.15)

# set the project name and version
project(Tutorial VERSION 1.0)

# specify the C++ standard
add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_11)

# add compiler warning flags just when building this project via
# the BUILD_INTERFACE genex
set(gcc_like_cxx "$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>")
set(msvc_cxx "$<COMPILE_LANG_AND_ID:CXX,MSVC>")
target_compile_options(tutorial_compiler_flags INTERFACE
  "$<${gcc_like_cxx}:$<BUILD_INTERFACE:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>>"
  "$<${msvc_cxx}:$<BUILD_INTERFACE:-W3>>"
)

# should we use our own math functions
option(USE_MYMATH "Use tutorial provided math implementation" ON)
# 这里可以选择最后生成动态库还是静态库
option(BUILD_SHARED_LIBS "Use Dynamic? " ON)

# configure a header file to pass some of the CMake settings
# to the source code
configure_file(TutorialConfig.h.in TutorialConfig.h)

# 设置动态库输出路径，注意，需要在添加库目录add_subdirectory之前设置
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}") # .a .lib
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}") # .dll .exe
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}") # .so

# add the MathFunctions library
if(USE_MYMATH)
  add_subdirectory(MathFunctions)
  list(APPEND EXTRA_LIBS MathFunctions)
endif()

# add the executable
add_executable(Tutorial tutorial.cxx)
target_link_libraries(Tutorial PUBLIC ${EXTRA_LIBS} tutorial_compiler_flags)

# add the binary tree to the search path for include files
# so that we will find TutorialConfig.h
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )

# add the install targets
install(TARGETS Tutorial DESTINATION bin)
install(FILES "${PROJECT_BINARY_DIR}/TutorialConfig.h"
  DESTINATION include
  )

# enable testing
include(CTest)

# does the application run
add_test(NAME Runs COMMAND Tutorial 25)

# does the usage message work?
add_test(NAME Usage COMMAND Tutorial)
set_tests_properties(Usage
  PROPERTIES PASS_REGULAR_EXPRESSION "Usage:.*number"
  )

# define a function to simplify adding tests
function(do_test target arg result)
  add_test(NAME Comp${arg} COMMAND ${target} ${arg})
  set_tests_properties(Comp${arg}
    PROPERTIES PASS_REGULAR_EXPRESSION ${result}
    )
endfunction()

# do a bunch of result based tests
do_test(Tutorial 4 "4 is 2")
do_test(Tutorial 9 "9 is 3")
do_test(Tutorial 5 "5 is 2.236")
do_test(Tutorial 7 "7 is 2.645")
do_test(Tutorial 25 "25 is 5")
do_test(Tutorial -25 "-25 is (-nan|nan|0)")
do_test(Tutorial 0.0001 "0.0001 is 0.01")

# setup installer
include(InstallRequiredSystemLibraries)
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/License.txt")
set(CPACK_PACKAGE_VERSION_MAJOR "${Tutorial_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${Tutorial_VERSION_MINOR}")
set(CPACK_SOURCE_GENERATOR "TGZ")
include(CPack)
```

**要点**

①BUILD_SHARED_LIBS

全局为`add_library`设置库的生成类型。`ON`则生成动态链接库，`OFF`则生成静态链接库。

②CMAKE_ARCHIVE_OUTPUT_DIRECTORY

指定静态库文件的生成位置。

③CMAKE_RUNTIME_OUTPUT_DIRECTORY

指定执行文件的生成位置，包括可执行程序和Windows上动态库文件(.dll)

④CMAKE_LIBRARY_OUTPUT_DIRECTORY

非Windows平台上的生成的.so库文件

## CMake使用第三方库

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyPro)
# 指定库的位置
set(MATH_DIR /path)

add_executable(myPro MyPro.cxx)

# 指定头文件位置以及库文件位置
target_include_dirctories(myPro PRIVATE "${MATH_DIR}/include")
target_link_dirctories(myPro PRIVATE "${MATH_DIR}/lib")

#指定需要链接的库
target_link_libraries(myPro PRIVATE ThirdParty)
```

直接使用开源三方库

```cmake
find_package(protobuf CONFIG REQUIRE) #这些库是可以直接通过apt安装，或者自己安装到usr/local中的，这里以protobuf为例
```

# CMAKE构建实战

## 第二章 CMake简介

### 2.3 hello world

创建`hello.cmake`文件：

```cmake
message(hello cmake) # 输出hello cmake
```

之后以脚本形式执行：

```shell
cmake -P hello.cmake
```

> 需要注意，cmake默认是用来构建项目的，如果想要以脚本方式执行，需要加-P参数

## 第三章 基础语法

### 3.1 CMake程序

CMake根据文件名可以分为以下两种：

+ CMakeLists.txt的文件,主要用于构建项目
+ 后缀为`.cmake`的程序,分为脚本程序和模块程序

#### 3.1.1 目录(CMakeLists.txt)

当使用CMake进行构建的时候，构建的入口就是顶层的`CMakeLists.txt`，当使用`add_subdirectory`添加子目录的时候，子目录中也必须要有`CMakeLists.txt`

#### 3.1.2 脚本(\<script>.cmake)

指定`-P`参数运行CMake的命令时，可以执行脚本类型的CMake程序，此时不会执行任何构建，因此构建相关的命令时不允许出现在脚本文件中的。

#### 3.1.3 模块(\<module>.cmake)

目录程序和脚本程序都可以通过`include`等命令添加CMake模块。主要场景在环境检索，搜索使用三方库等

### 3.2 注释

单行注释：`#`开头

多行注释：`#[[comment]]`

### 3.3 命令调用

注意，CMake的命令名称不区分大小写，并且使用空格作为分隔符（等价C的`,`）

```cmake
#[[这是
一个多行注释]] 

message(a b c) #输出abc
```

### 3.4 命令参数

三种类型：

+ 引号参数(quoted argument)
+ 非引号参数(unquoted argument)
+ 括号参数(bracket argument)

#### 3.4.1 引号参数

CMake语法中的引号(必须为双引号)等价于C++中`R("")`,即原生字符，在双引号内的所有字符都会视为字符串，包括换行。支持变量引用和转义字符

```cmake
message("test
        line2") 
# 输出
test
        line2
```

其中使用`\`在每行的末尾，可以去除换行

```cmake
message("test\
    line2")
# 输出
test    line2
```

#### 3.4.2 非引号参数

非引号参数不能包含任何空白，圆括号，`#`,双引号和反斜杠，除非经过转义。非引号参数也支持变量引用和转义字符

非引号参数不总是作为一个整体传递，在被传递前，会被当做CMake列表处理（CMake列表是一种特殊的字符串，由分号分割），以下语句是等价的

```cmake
message(x y z) #多个非引号参数 输出xyz

message(x;y;z) #非引号参数 输出xyz
```

#### 3.4.3 变量引用

语法形式：`${name}`。变量的引用可用在引号和非引号参数中，并且支持嵌套。需要注意的是，如果变量不存在，CMake并不会报错，而是直接以空字符替换

```cmake
set(var1 test1)

set(var2 1)

message(${var${var2}}) # 输出test1
```

CMake中还有缓存变量`$CACHE{name}`和环境变量`$ENV{name}`，其中环境变量形式要求严格，而缓存变量可以使用普通变量的语法来引用，但是当同时存在缓存变量和普通变量（即同名变量）时，会优先匹配到普通变量。

#### 3.4.4 转义字符

CMake的转义字符由以下四种情况：

+ 如果后面跟的不是字母、数字或者分号。转义字符就是该字符本身
+ `\t`,`\r`,`\n`分别为Tab，回车，换行
+ `\;`分为
  - 如果是在变量引用或者非引号参数中，直接转义为`;`，注意，在非引号参数中，转义后为字符，不再用于分隔列表元素
  - 其他情况，不进行转义，保留`\`
+ 其他全为错误转义

```cmake
cmake_minimum_required(VERSION 3.20)
set("a?b""变量a?b") # 变量名为a?b
message(${a\?b}) # 使用变量名的时候需要用转义字符 输出---->变量a?b
message(今天是几号\?)   #在使用非引用参数的时候，需要使用转义字符 输出---->今天是几号?
message(回答：\n\t今天是一号\!) #在使用非引用参数的时候，需要使用转义字符
message(x;y\;z) #在使用非引用参数的时候，需要使用转义字符 输出---->xy;z
message("x;y\;z") #输出---->x;y\;z
set("a;b" "变量a;b")
message("${a\;b}") # 输出----> "变量a;b"
```

总之记住，引号参数就是C++中的`R("")`就可以了，其他地方都要用转义字符

#### 3.4.5 括号参数

括号参数也是原始字符，和多行注释唯一的区别是没有`#`,语法：`[[]]`

```cmake
message([===[
abc
def
]===])
message([===[abc
def
]===])
message([===[
随便写终⽌⽅括号并不会导致⽂本结束，
因此右边这两个括号]]也会包括在原始⽂本中。
下⼀⾏中最后的括号也是原始⽂本的⼀部分，
因为等号的数量与起始括号不匹配。]==]
]===])

# 输出
abc
def

abc
def

随便写终⽌⽅括号并不会导致⽂本结束，
因此右边这两个括号]]也会包括在原始⽂本中。
下⼀⾏中最后的括号也是原始⽂本的⼀部分，
因为等号的数量与起始括号不匹配。]==]
```

需要注意的是，`=`不是必须的

### 3.5 变量

CMake类似shell脚本，数据类型总是文本型的，但是在使用时，可以被解释为数值型等

**变量的分类**

+ 普通变量：具有特定作用域
+ 缓存变量：会被持久化存储到CMakeCache.txt中，这样可以在多次使用的时候提升构建速度
+ 环境变量：系统变量

**变量作用域**

+ 函数作用域：类似栈作用域
+ 目录作用域：CMake子目录会将父目录中所有变量拷贝一份，所以子目录可以访问父目录的所有变量，反之不行

**保留标识符**

即C++中的关键字

+ 以`CMAKE_`开头
+ 以`_CMAKE_`开头
+ `_`加上任意一个关键字，如：`_message`

#### 3.5.1 预定义变量

+ CMAKE_ARGC：表示CMake脚本程序在被`cmake -P`命令行调用执行时，命令行传递的参数个数
+ CMAKE_ARGV0,CMAKE_ARGC1：命令行第一个第二个参数
+ CMAKE_COMMAND：CMake命令行程序所在的路径
+ CMAKE_HOST_SYSTEM_NAME：宿主机操作系统
+ CMAKE_SYSTEM_NAME：目标操作系统
+ CMAKE_CURRENT_LIST_FILE：当前运行中的CMake程序对应文件的绝对路径
+ CMAKE_CURRENT_LIST_DIR：当前运行的CMake程序所在目录的绝对路径
+ MSVC：在构建时使用的编辑器是否是MSVC
+ WIN32：目标操作系统是否为windows
+ APPLE：目标操作系统是否是苹果
+ UNIX：目标操作系统是否是UNIX或类UNIX

#### 3.5.2 定义变量

**定义普通变量**

语法：`set(<变量><值>...[PARENT_SCOP])`

变量的值可以由若干参数来提供，这些参数会被分号分隔连接为一个列表的形式，并作为最终的变量值。值参数为空的时候，相当于调用unset

`PARENT_SCOPE`将变量定义到父级作用域中。对于目录和函数都是仅上级作用域。

```cmake
function(f)
    set(a "修改后的a")
    set(b "b")
    set(c "c" PARENT_SCOPE)
endfunction()

set(a "a")
f()
message("a:${a}") # 输出 a:a
message("b:${b}") # 输出 b:
message("c:${c}") # 输出 c:c
```

**定义缓存变量**

语法：`set(<变量><值>...CACHE<变量类型><变量描述>[FORCE])`

缓存变量拥有全局的作用域，所以不需要`PARENT_SCOPE`参数，其他和普通变量一致。

`<变量类型>`

| 类型参数 | 描述                          |
| -------- | ----------------------------- |
| BOOL     | 布尔型                        |
| FILEPATH | 文件路径类型                  |
| PATH     | 目录路径类型                  |
| STRING   | 文本型                        |
| INTERNAL | 内部使用（隐含设置FORCE参数） |

`<变量描述>`就是这个变量的提示信息，类似vscode中变量注释`/// @brief`

`FORCE`可选参数用于强制覆盖缓存变量的值。默认情况下，如果缓存变量已经存在，那么set并不会实际执行，除非设置了`FORCE`

