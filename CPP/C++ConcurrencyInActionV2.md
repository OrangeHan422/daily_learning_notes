# 第一章：你好，C++并发世界

> 本书基于C++11、C++14以及C++17

大多数情况下，使用线程实现并发要优于使用进程实现并发。但是运用独立的进程实现并发，还可以有一个额外的优势，通过网络进行连接。

应用软件使用并发技术的主要原因有两个：分离关注点和性能提升。

C++标准库在大多数情况下都可以到达性能要求，但是在极少数情况下，有必要使用平台专属的工具。

新线程启动后，其实线程继续执行。如果起始线程不等待新线程结束，就会一路执行，知道main()结束，甚至可能直接终止整个程序，新线程根本没有机会启动。（这时可以使用join）

# 第二章：线程管控

## 2.1线程的基本管控

当main()返回时，程序就会退出；同样，当入口函数返回时，对应的线程随之终结。

### 2.1.1 发起线程

函数在自己的线程上运行，等它一返回，线程随之终止。与C++标准库中的很多类型相同，任何可调用(callable type)都使用于`std::thread`。

为了避免"C++最麻烦的解释"，在对线程进行初始化时，最好使用统一初始化，或者使用lambda表达式。

```c++
std::thread my_thread(func());//注意这是一个函数声明！！
//正确做法：
std::thread my_thread((func()));
std::thread my_thread{func()};
std::thread my_thread{[](){
    ...lambda函数体
}};
```

一旦启动了线程，我们就需要明确是要等待它结束（与之汇合(join)）还是任由它独自运行（与之分离(detach)）。假如等到`std::thread`对象销毁之际还没有决定好。那`std::thread`析构函数将调用`std::terminate()`终止整个程序。

### 2.1.2 等待线程完成

join()简单而粗暴，我们抑或一直等待线程结束，抑或干脆完全不等待。如需选取更精细的粒度控制线程等待，如查验线程结束与否，或限定只等待一段时间，那我们边需要改用其他方式，如条件变量和future。只要调用了join()，隶属于该线程的任何存储空间即会因此清除，`std::thread`对象就不再关联到已结束的线程。即，该对象和任何对象都没有关系。

对于某个给定的线程，join()只能调用一次；只要`std::thread`对象曾经调用过join()，线程就不再可汇合（joinable），成员函数joinable()将返回false

### 2.1.3 在出现异常的情况下等待

如果线程启动后有异常抛出，而join()尚未执行，则该join()调用会被略过。解决方法是使用try/catch语句，或者利用RAII手法：

```c++
struct func;
void f()
{
    int local_state = 0;
    func my_func(local_stat);
    std::thread t(my_func);
    try{
        do_something_in_current_thread();
    }catch(...){
        t.join();
        throw;
    }
    t.join();
}
//使用RAII代码会更加简洁
class thread_guard{
    std::thread& t;
    public:
    explicit thread_guard(std::thread& t_)
        :t(t_)
        {}
    ~thread_guard(){
        if(t.joinable()){
            t.join();
        }
    }
    thread_guard(const thread_guard&) = delete;
    thread_guard& operator=(const thread_guard&) = delete;
};
struct func;
void f()
{
    int local_state = 0;
    func my_func(local_stat);
    std::thread t(my_func);
    thread_guard g(t);
    do_something_in_current_thread();
}
```

### 2.1.4 在后台运行线程

调用`std::thread`对象的成员函数detach(),会令线程在后台运行，此时就无法与该线程直接通信。加入线程被分离，就无法等待它完结，也不能获取与它关联的`std::thread`对象，因此无法汇合(join())该线程。

但是分离的线程仍然在后台运行，其归属权和控制权都移交给C++运行时库（runtime library，运行库，动态库），由此来保证线程一旦退出，与之关联的资源都会被正确的回收。

被分离的线程在UNIX操作系统中，就叫做守护线程(daemon thread)

## 2.2 向线程函数传递参数

向线程传递启动函数的参数，直接在`std::thread`的构造函数中增添更多参数即可。

但是需要注意的是，线程具有内部的存储空间，参数默认是复制获得的。即使函数的形参形式是引用形式的，如果向使用，必须使用`std::ref()`传递参数

```c++
void f(int i,const std::string& s);
std::string dood{"hello"};
std::thread t(f,3,std::ref(dood));
```

## 2.3 移交线程的归属权

`std::thread`和`std::unique_ptr`一样仅具有移动语义，而没有复制操作。因此线程的归属权可以在实例之间移动。

```c++
void some_func();
void some_other_func();
std::thread t1(some_func);		//t1入口some_func
std::thread t2=std::move(t1);	//t1是空的，t2入口some_func
t1=std::thread(some_other_func);//t1入口some_other_func,t2入口some_func
std::thread t3;
t3 = std::move(t2);				//t2是空的，t3入口some_func
t1 = std::move(t3);				//t3是空的，t1入口some_func，但是t1在移动时，已经关联了some_other_func，因此std::terminate()会被调用，终止整个程序。
```

## 2.4 在运行时选择线程数量

`std::thread::hardware_concurrency()`函数返回一个指标，表示程序在各次运行中可整整并发的线程数量。如果该值无法获取，则会返回0，因此使用时应该注意：

```c++
const unsigned long hardware_threads = std::thread::hardware_concurrency();
const unsigned long num_threads = hardware_threads != 0 ? hardware_threads:2;//如过无法获取，默认2个
```

截止目前，无法从线程中直接获取返回值，所以我们必须使用`std::ref`来传递一个传入传出参数。但是后续可以通过`std::future`从线程中获取返回值。

## 2.5 识别线程

线程ID的类别是`std::thread::id`，有两种获取方式：

+ 在与线程关联的`std::thread`对象上调用`get_id()`，即可得到该线程的ID。如果`std::thread`对象没有关联任何线程，则会返回一个`std::therad::id`对象，按默认构造函数生成，表示线程不存在。
+ 当前线程的ID可以直接调用`std::this_thread::get_id()`获得。

如果两个`std::thread::id`相等，那么就表示相同的线程，或者都表示线程不存在；C++标准库允许我们随意判断两个线程ID是否相等（STL算法或者比较运算符都可以）。因此，可以将它们硬作关联容器的键值。比如标准库的hash模板能够具体化为`std::hash<std::thread::id>`。因此，`std::thread::id`可以作为无序关联容器的键值。

# 第三章：在线程间共享数据

