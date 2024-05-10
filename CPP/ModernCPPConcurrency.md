# ModernCPPConcurrency

# ch01 使用线程

```c++
#include <thread>
#include <iostream>

void hello_func(){
    std::cout << "hello world" << std::endl;
}

int main(){
    std::thread t{hello_func};
    t.join();
}
```

`std::thread`的入口函数可以是任何可调用的对象。

`join`函数等待线程执行结束，并将线程的`std::thread::joinable()`设置为false

`std::thread`的析构函数就是通过`std::thread::joinable()`判断当前线程是否有关联活跃线程。

## 1.1 当前环境支持的并发线程数

相当于对Linux中的sys_conf中的数据进行封装。

```c++
unsignedint n = std::thread::hardware_concurrency();//返回硬件支持的并发线程数
```

## 1.2 线程管理

### 1.2.1 启动线程

```c++
std::thread t;//默认构造函数，不会有实际的线程和其关联

class Task{
  public:
    void operator()() const {
        std::cout << "callable obj" << std::endl;
    }
};
// std::thread 的入口函数是任何可调用的对象
std::thread t1{ Task{} };//注意这里临时对象的创建，如果使用()会导致类似最难解析的错误（即编译器会将其当做一个函数声明）

std::thread t2{ [](){
    std::cout << "lambda function" << std::endl;
}}

struct X{
    void task(int) const;
};

X x;
std::thread t3{ &X::task,&x,n };//成员函数指针也是可以调用的

std::thread t4{ std::bind(&X::task,&x,n) };//使用std::bind生成的function也是可以的

//注意线程的结束，join或者detach
```

线程detach后，`std::thread::joinable()`也是false,因为已经失去了对线程的所有权。注意，在对象销毁之后再去访问会引起未定义的行为。

一个简单的未定义行为的例子：

```c++
struct Callable{
  int m_data;
  Callable(int& data):m_data(data){}
  void operator()(int n){
      for(int i = 0; i <= n; ++i){
          m_data += i;
      }
  }
};


int main(){
    int n{0};
    std::thread t1{Callable{n},100};
    t1.detach();
}
```

上述例子中，detach后线程变成守护线程继续执行，但是所持有的局部对象n生命周期已经结束，这样做会引起未定义行为。

除非十分确定，一般不要使用detach()，资源的回收引起的问题很难排查。C++的设计给予了程序员绝对的主导权，而detach则是让你放弃线程的所有权。

另外需要注意

```c++
std::thread t{something};
//...可能抛出异常的代码,可能是线程代码，也可能不是线程代码
t.join();//如果发生了异常，这里就会别跳过，资源就无法回收

//正确做法
std::thread t{something};

try{
    //......可能发生异常的代码,可能是线程代码，也可能不是线程代码
}catch(...){
    t.join();
    throw;//注意，这里需要再次抛出，如果不抛出，会再次执行下面的join
}
t.join();
```

### 1.2.2 RAII

```c++
class thread_guard{
    std::thread& m_thread;//注意是引用
public:
    explicit thread_guard(std::thread& t):m_thread(t){}
    ~thread_guard(){
        if(m_thread.joinable()){
            m_thread.join();
        }
    }
    
    thread_guard(const thread_guard&) = delete;
    thread_guard& operator=(const thread_guard&) = delete;
};


int n = 0;
std::thread t{func{n},10};
thread_guard tg(t);
```

### 1.2.3 传递参数

类似`std::bind`的方式，直接在后面加参数就可以了，引用的传递也一样

但是需要注意的一点是，当形参有引用时，传递实参的时候应该使用std::ref或者std::cref来确保参数的传递是按照我们预想的进行引用传递。

```c++
void test(int,int& refInt){
    std::cout << &refInt <<std::endl;
}


int main(){
    int n{1};
    std::cout << &n << std::endl;
    std::thread t(test,1,n);	//打印地址不同
    std::thread t1(test,1,std::ref(n));	//打印地址相同
    t.join();
}
```

## 1.3 std::this_thread

该命名空间包含了管理当前线程的函数：

+ `yield`屈服，即当前线程让出内核，让内核重新调度各个线程(即减少CPU的占用)
+ `get_id`返回当前线程id
+ `sleep_for`当前线程休眠指定时间
+ `sleep_until`当前线程休眠至指定时间

```C++
int main(){
    std::cout << std::this_thread::get_id() << '\n';	//主线程id
    std::thread t{ [](){
        std::cout << std::this_thread::get_id() << '\n';//子线程id
    }};
    t.join();
    
    std::this_thread::sleep_for(std::chrono::seconds(3));//主线程休眠3秒
    //等价于
    using namespace std::chrono_literals;
    std::this_thread::sleep_for(3s);
}
```

```c++
while(!isDone()){
    //线程等待某个操作，如果一直循环判断，会占用大量CPU资源，使用yield让出CPU时间片，让其他线程继续执行
    std::this_thread::yield();
}
```

## 1.4 `std::thread`转移所有权

`std::thread`不可复制，确保对象对线程是一对一的关系，所以所有权的转移只能使用移动操作。

```c++
std::thread t{ [](){
    ...
} }

std::cout << t.joinable() << std::endl;//1
std::thread t2(std::move(t));
std::cout << t.joinable() << std::endl;//0

//或者使用移动赋值来转移所有权
std::thread t3;
t3 = std::move(t);

t.join();//error
t2.join();//error
t3.join();//ok

//或者使用临时对象
t = std::thread([]{});
t.join();
```

另外，线程在作为返回的时候，编译器会进行RVO，所以不需要显示的进行移动操作

```c++
std::thread f(){
    std::thread t{[] {}};
    return t;
}

int main(){
    std::thread rt = f();
    rt.join();
}
```

std::jthread就是joinable thread，C++20引入，工作中使用C++17，暂不处理

# ch02 共享数据

std::cout 的operator<<是线程安全的，即输出顺序不保证，但是不存在数据竞争

## 2.1 使用互斥量

```c++
std::mutex m;
void func(){
    //m.lock();
    std::cout << std::this_thread::get_id() << std::endl;
    //m.unlock();
}

int main(){
    std::vector<std::thread> threads;
    for(auto i = 0; i < 10; ++i){
        threads[i].emplace_back(func);
    }
    
    for(auto& th:threads){
        th.join();
    }
}
```

如果没有上锁，输出的结果是毫无规律的，在本例子中最明显的就是换行符的位置。

反之，格式会相对整齐

### 2.1.1 `std::lock_guard`

就是RAII版本的`std::mutex`,利用管理类管理`mutex`，注意，管理类是nocopyable的

```c++
std::mutex m;

void func(){
    std::lock_guard<std::mutex> lc{ m };
    std::cout << std::this_thread::get_id() << std::endl;
}
```

源码：

```c++
_EXPORT_STD template <class _Mutex>
class _NODISCARD_LOCK lock_guard { // class with destructor that unlocks a mutex
public:
    using mutex_type = _Mutex;

    explicit lock_guard(_Mutex& _Mtx) : _MyMutex(_Mtx) { // construct and lock
        _MyMutex.lock();
    }

    lock_guard(_Mutex& _Mtx, adopt_lock_t) noexcept // strengthened
        : _MyMutex(_Mtx) {} // construct but don't lock

    ~lock_guard() noexcept {
        _MyMutex.unlock();
    }

    lock_guard(const lock_guard&)            = delete;
    lock_guard& operator=(const lock_guard&) = delete;

private:
    _Mutex& _MyMutex;
};
```

需要注意，构造函数的第二版本，提供了一个`std::adopt_lock_t`的参数，该版本不会在构造期间进行上锁。该构造函数的提出主要是为了缩小锁的**粒度**

在C++17中提供了类模板参数推导，`std::lock_guard`可以自行推导，所以不需要再写明模板类型参数：

```c++
std::mutex m;
std::lock_guard lc{m};
```

C++17还提出了`std::scoped_lock`，其作用和`std::lock_guard`是一样的，但是该类可以管理**多个**互斥量

### 2.1.2 `try_lock`

`try_lock`是尝试上锁，如果锁已经被占用，会立刻返回，并不会阻塞。而`lock`则是会阻塞在lock()等待其他线程进行解锁

```c++
std::mutex mtx;

void thread_function(int id){
    if(mtx.try_lock()){
        std::cout << "线程：" << id << "获得锁" << std::endl;
        //模拟临界区操作
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        mtx.unlock();
        std::cout << "线程：" << id << "释放锁 " << std::endl;
    }else{
        std::cout << "线程" << id << "获取锁失败" << std::endl;
    }
}


std::thread t1(thread_func,1);
std::thread t2(thread_func,2);

t1.join();
t2.join();
```

## 2.2 保护共享数据

使用互斥量时，应该注意指针或者引用，它们可能将受保护的数据传递出去，如果是这样，互斥量就相同虚设

```c++
class Data{
    int a{};
    std::string b{};
public:
    void do_something(){
        // 修改数据成员等...
    }
};

class Data_wrapper{
    Data data;
    std::mutex m;
public:
    template<class Func>
    void process_data(Func func){
        std::lock_guard<std::mutex>lc{m};
        func(data);  // 受保护数据传递给函数
    }
};

Data* p = nullptr;

void malicious_function(Data& protected_data){
    p = &protected_data; // 受保护的数据被传递
}

Data_wrapper d;

void foo(){
    d.process_data(malicious_function);  // 传递了一个恶意的函数
    p->do_something();                   // 在无保护的情况下访问保护数据
}
```

### 2.2.1 死锁：问题与解决



