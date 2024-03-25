# Makefile

## 基础

makefile的三要素：依赖，目标，命令

```makefile
目标:依赖
	命令  //注意前面是tab
```

makefile支持增量编译,make会通过文件的时间戳判断是否需要进行增量编译以及对哪些文件进行增量编译

查看版本

```shell
make -v
```

简单示例：

```makefile
all:
	echo "Hello World"
```

执行结果：

```bash
$make
echo "Hello World"
Hello World

$make all
echo "Hello World"
Hello World

$make test
make: *** No rule to make target `test'. Stop.
```

可以为makefile加入test规则：

```makefile
all:
	echo "Hello World"
test:
	echo "Just for test!"
```

注意，make命令默认执行的是第一个规则，该示例中是all，但是对规则名称没有要求:

```bash
$make
echo "Hello World"
Hello World

$make test
echo "Just for test!"
Just for test!
```

我们可以对makefile做出更改，使得终端不输出执行的命令，。即在命令前加入`@`：

```makefile
all:
	@echo "Hello World"
test:
	@echo "Just for test!
```

```bash
$make
Hello World
```

再次更改makefile，使all依赖于test（该示例中，test也称之为all的*先决条件*）：

```makefile
all: test
	@echo "Hello World"
test:
	@echo "Just for test!"
```

```bash
$make
Just for test!
Hello World

$make test
Just for test!
```

对于规则，可以由多个目标名称，以空格隔开

```c++
all test:
	@echo "Hello World"
```

```bash
$make
Hello World

$make test
Hello World
```

### 简单应用：

**foo.c**:

```c
#include <stdio.h>
void foo ()
{
	printf ("This is foo   ()!\n");
}
```

**main.c**

```c
extern void foo();
int main ()
{
	foo();
	return 0;
}
```

**Makefile**

```makefile
all: main.o foo.o
	gcc -o simple main.o foo.o
main.o: main.c
	gcc -o main.o -c main.c
foo.o: foo.c
	gcc -o foo.o -c foo.c
clean:
	rm simple main.o foo.o
```

执行流程：

```bash
$make
gcc -c main.c -o main.o
gcc -c foo.c -o foo.o
gcc -o simple main.o foo.o

$./simple
This is foo ()!

$make clean
rm simple main.o foo.o
```

即使更改了规则的目标名，也不会影响makefile的增量编译：

```makefile
simple: main.o foo.o
	gcc -o simple main.o foo.o
main.o: main.c
	gcc -o main.o -c main.c
foo.o: foo.c
	gcc -o foo.o -c foo.c
clean:
	rm simple main.o foo.o
```

```bash
$make
gcc -c main.c -o main.o
gcc -c foo.c -o foo.o
gcc -o simple main.o foo.o

$make
make: `simple' is up to date.
```

### 假目标

加入touch了一个clean文件，在上述实例中，clean文件没有改变，make就永远不会像我们期待的那样执行rm指令。解决该问题的方法是使用**假目标**，即声明目标名并且加上`.PHONY:`,如：

```makefile
.PHONY: clean
simple: main.o foo.o
	gcc -o simple main.o foo.o
main.o: main.c
	gcc -o main.o -c main.c
foo.o: foo.c
	gcc -o foo.o -c foo.c
clean:
	rm simple main.o foo.o
```

### 变量

简单示例：

```makefile
.PHONY: clean
CC = gcc
RM = rm
EXE = simple
OBJS = main.o foo.o
$(EXE): $(OBJS)
	$(CC) -o $(EXE) $(OBJS)
main.o: main.c
	$(CC) -o main.o -c main.c
foo.o: foo.c
	$(CC) -o foo.o -c foo.c
clean:
	$(RM) $(EXE) $(OBJS)
```

变量的命名类似PYTHON，直接命名就可以。而变量的使用则是以`$(变量名)`的形式（类似预处理，直接替换文本）。

#### 自动变量

当目标或者依赖名称改变时，往往需要更改很多命令。对此，Makefile提供了自动变量(根据规则的上下文推导)：

- **$@**用于表示一个规则中的目标。当我们的一个规则中有多个目标时，$@所指的是其中**任何造成命令被运行的目标。**
- **$^**则表示的是规则中的所有先决条件。
- **$<**表示的是规则中的第一个先决条件。

需要注意的是，在 Makefile 中‘$’具有特殊的意思，因此，如果想采用 echo 输出‘$’，则必需用两个连着的‘$’。还有就是，$@对于 Shell 也有特殊的意思，我们需要在“$$@”之前再加一个脱字符‘\’。

简单示例：

```makefile
.PHONY: all
all: first second third
	@echo "\$$@ = $@"
	@echo "$$^ = $^"
	@echo "$$< = $<"
first second third:
```

```bash
$make
$@ = all				//目标
$^ = first second third	//依赖
$< = first				//首个依赖
```

对此可以对simple的makfile进行更改：

```c++
.PHONY: clean
CC = gcc
RM = rm
EXE = simple
OBJS = main.o foo.o
$(EXE): $(OBJS)
	$(CC) -o $@ $^				//原命令$(CC) -o $(EXE) $(OBJS)
main.o: main.c
	$(CC) -o $@ -c $^			//原命令：$(CC) -o main.o -c main.c
foo.o: foo.c
	$(CC) -o $@ -c $^			//原命令：$(CC) -o foo.o -c foo.c
clean:
	$(RM) $(EXE) $(OBJS)
```

#### 特殊变量

+ MAKE变量：代表make命令名，主要是为了增加可移植性

  ```makefile
  .PHONY: all
  all:
  	@echo "MAKE = $(MAKE)"
  ```

  ```bash
  $make
  MAKE = make
  ```

+ MAKECMDGOALS:make的目标名

  ```makefile
  Makefile
  .PHONY: all clean
  all clean:
      @echo "\$$@ = $@"
      @echo "MAKECMDGOALS = $(MAKECMDGOALS)"
  ```

  ```bash
  $make
  $@ = all
  MAKECMDGOALS =
  
  $make all
  $@ = all
  MAKECMDGOALS = all
  
  $make clean
  $@ = clean
  MAKECMDGOALS = clean
  
  $make all clean
  $@ = all
  MAKECMDGOALS = all clean
  $@ = clean
  MAKECMDGOALS = all clean
  ```

#### 变量类别

makefile提供了**递归扩展变量**的操作：

```makefile
.PHONY: all
foo = $(bar)
bar = $(ugh)
ugh = Huh?
all:
	@echo $(foo)
```

```bash
$make
Huh?
```

其实还是类似预处理的操作，核心还是变量的使用，需要注意的是防止死循环：

```makefile
CFLAGS = $(CFLAGS) -O	//死循环
```

同时makefile也提供了`:=`形式的**简单扩展变量**，这种形式的变量只会进行一次扫描和替换

```makefile
.PHONY: all
x = foo
y = $(x) b
x = later
xx := foo
yy := $(xx) b
xx := later
all:
	@echo "y = $(y), yy = $(yy)"
```

```bash
$make
y = later b, yy= foo b
```

最后makefile也提供了`?=`形式的条件赋值（即当前变量如果没有定义的话就进行赋值，否则无操作）：

```makefile
.PHONY: all
foo = x
foo ?= y
bar ?= y
all:
	@echo "foo = $(foo), bar = $(bar)"
```

```bash
$make
foo = x, bar = y
```

综上，我们可以书写可扩展，易读性强的makefile:

```makefile
.PHONY: all
objects = main.o foo.o bar.o utils.o
objects := $(objects) another.o
all:
	@echo $(objects)
```

```bash
$make
main.o foo.o bar.o utils.o another.o
```

#### 其他方式更改变量值

+ 执行make 并指定值，如:

  ```bash
  $make foo=haha
  foo = haha, bar = x
  ```

+ 通过export导入：

  ```bash
  $make
  foo = x, bar = y
  
  $export bar=x
  
  $make
  foo = x, bar = x
  ```

#### 高级变量引用功能

可以对变量赋值的同时完成后缀替换的操作。语法为`$(变量名:原后缀名=目标后缀名)`

```makefile
.PHONY: all
foo = a.o b.o c.o
bar := $(foo:.o=.c)
all:
	@echo "bar = $(bar)"
```

```bash
$make
bar = a.c b.c c.c
```

#### override

和C++中的override正好反过来，意思是这个变量不允许其他值覆盖：

```makefile
.PHONY: all
override foo = x
all:
	@echo "foo = $(foo)"
```

```bash
$make foo=haha
foo = x
```

### 模式

其实就是使用`%`作为通配符，如：

```makefile
.PHONY: clean
CC = gcc
RM = rm
EXE = simple
OBJS = main.o foo.o
$(EXE): $(OBJS)
	$(CC) -o $@ $^
%.o: %.c				//	main.o: main.c
	$(CC) -o $@ -c $^   //      gcc -o main.o -c main.c
                        //  foo.o: foo.c
                        //      gcc -o foo.o -c foo.c

clean:
	$(RM) $(EXE) $(OBJS)
```

### 函数

#### addprefix函数

为字符串中每个子串前加上一个前缀：`$(addprefix prefix,names...)`

```makefile
.PHONY: all
without_dir = foo.c bar.c main.o
with_dir := $( addprefix objs/, $(without_dir))
all:
	@echo $(with_dir)
```

```bash
$make
objs/foo.c objs/bar.c objs/main.o
```

#### filter函数

根据模式得到满足模式的字符串（简化正则）：`$(filter pattern...,text)`

```makefile
.PHONY: all
sources = foo.c bar.c baz.s ugh.h
sources := $(filter %.c %.s, $(sources))
all:
	@echo $(sources)
```

```bash
$make
foo.c bar.c baz.s		//可以看到过滤掉了.h文件
```

#### filter-out函数

过滤掉满足模式的字符串：`$(filter-out pattern...,text)`

```makefile
.PHONY: all
objects = main1.o foo.o main2.o bar.o
result = $(filter-out main%.o, $(objects))
all:
	@echo $(result)
```

```bash
$make
foo.o bar.o 	//过滤掉了满足main%.o模式的文件
```

#### patsubst 函数

根据模式进行字符串替换：`$(patsubst pattern,replacement,text)`

```makefile
.PHONY: all
mixed = foo.c bar.c main.o
objects := $(patsubst %.c, %.o, $(mixed))  # 将.c文件替换为.o文件
all:
	@echo $(objects)
```

```bash
$make
foo.o bar.o main.o
```

#### strip函数

去除字符串中多余空格：`$(strip string)`

```makefile
original = foo.c   bar.c
stripped := $(strip $(original))
all:
    @echo "original = $(original)"
    @echo "stripped = $(stripped)"
```

```bash
$make
original = foo.c   bar.c
stripped = foo.c bar.c
```

#### wildcard函数

通配符函数：`$(wildcard pattern)`

```makefile
.PHONY: all
SRCS = $(wildcard *.c)
all:
	@echo $(SRCS)
```

```bash
$make
bar.c foo.c main.c
```