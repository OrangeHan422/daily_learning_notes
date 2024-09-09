# ProC++

## 遍历相关trick

`std::for_each(iter,iter,func)`：对每个迭代器指向的对象执行func

```c++
int main() {
  std::vector<std::thread> v;
  for (int i = 0; i < 10; ++i) {
    v.emplace_back([] {});
  }
  //对每个线程执行join
  std::for_each(std::begin(v), std::end(v), std::mem_fn(&std::thread::join));
}
```

遍历并求和（求积）:

```c++
//默认+求和
template< class InputIt, class T >
T accumulate(InputIt first, InputIt last, T init);
//自定义求和
template< class InputIt, class T, class BinaryOperation >
T accumulate(InputIt first, InputIt last, T init, BinaryOperation op);

//默认实例
std::vector<int> numbers = {1, 2, 3, 4, 5};
int sum = std::accumulate(numbers.begin(), numbers.end(), 0);
std::cout << "Sum: " << sum << std::endl; // 输出: Sum: 15
//自定义实例：
std::vector<int> numbers = {1, 2, 3, 4, 5};
int product = std::accumulate(numbers.begin(), numbers.end(), 1, std::multiplies<int>());
std::cout << "Product: " << product << std::endl; // 输出: Product: 120
```



## 函数对象

`std::mem_fn(&ClassName::FuncName)`：将类中的成员函数变成一个可执行对象

```c++
//原型
template <typename Ret, typename Class, typename... Args>
std::mem_fn(Ret (Class::*)(Args...));

//实例
MyClass obj;
auto func = std::mem_fn(&MyClass::foo);
func(obj, 42);  // 调用 obj.foo(42);
```

## C

查看硬件支持的并发线程数量：

```c++
unsigned int n = std::thread::hardware_concurrency();
```

计算两个迭代器之间的距离：

```c++
long len = std::distance(first, last);
```

移动迭代器

```c++
template<class InputIterator, class Distance>  
void advance(InputIterator& it, Distance n);
//Distance：要移动的次数，可以是正数（向前移动）或负数（向后移动）。
//it：要移动的迭代器引用。

//实例
std::vector<int> vec = {1, 2, 3, 4, 5};  
std::vector<int>::iterator it = vec.begin();  
std::advance(it, 3); // 移动到第4个元素  
std::cout << *it << std::endl; // 输出: 4  
```

## 资源利用

cpu亲和性（结合std::thread）

```c++
#include <pthread.h>
#include <sched.h>
#include <string.h>
//Linux中的句柄就是int
void affinity_cpu(std::thread::native_handle_type t, int cpu_id) {
  cpu_set_t cpu_set;
  CPU_ZERO(&cpu_set);
  CPU_SET(cpu_id, &cpu_set);
  int res = pthread_setaffinity_np(t, sizeof(cpu_set), &cpu_set);
  if (res != 0) {
    errno = res;
    std::cerr << "fail to affinity" << strerror(errno) << std::endl;
  }
}
void affinity_cpu_on_current_thread(int cpu_id) {
  cpu_set_t cpu_set;
  CPU_ZERO(&cpu_set);
  CPU_SET(cpu_id, &cpu_set);
  int res = pthread_setaffinity_np(pthread_self(), sizeof(cpu_set), &cpu_set);
  if (res != 0) {
    errno = res;
    std::cerr << "fail to affinity" << strerror(errno) << std::endl;
  }
}
void f() { affinity_cpu_on_current_thread(0); }

int main() {
  std::thread t1{[] {}};
  affinity_cpu(t1.native_handle(), 1);
  std::thread t2{f};
  t1.join();
  t2.join();
}
```

## 多线程

### 互斥锁

`std::lock_guard`:其实就是RAII封装mutex

```c++
class A {
 public:
  void lock() { std::cout << "lock" << std::endl; }
  void unlock() { std::cout << "unlock" << std::endl; }
};

int main() {
  A a;
  {
    std::lock_guard<A> l(a);  // lock
  }                           // unlock
}
```

C++17提供`std::scoped_lock`,可对多个mutex进行上锁，上锁顺序不定。对一个锁进行lock，其他均为try_lock,如果失败，则对已经上锁的互斥量进行unlock，然后从头开始上锁。（C++17最优上多个锁的方式）

```c++

class A {
 public:
  void lock() { std::cout << 1; }
  void unlock() { std::cout << 2; }
  bool try_lock() {
    std::cout << 3;
    return true;
  }
};

class B {
 public:
  void lock() { std::cout << 4; }
  void unlock() { std::cout << 5; }
  bool try_lock() {
    std::cout << 6;
    return true;
  }
};

int main() {
  A a;
  B b;
  {
    std::scoped_lock l(a, b);  // 16
    std::cout << std::endl;
  }  // 25
}
```

`std::lock`可以对多个mutex上锁，要么都锁，要么都不锁

```c++
struct A {
  explicit A(int n) : n_(n) {}
  std::mutex m_;
  int n_;
};

void f(A &a, A &b, int n) {
  if (&a == &b) {
    return;  // 防止对同一对象重复加锁
  }
  std::lock(a.m_, b.m_);  // 同时上锁防止死锁
  // 下面按固定顺序加锁，看似不会有死锁的问题
  // 但如果没有 std::lock 同时上锁，另一线程中执行 f(b, a, n)
  // 两个锁的顺序就反了过来，从而可能导致死锁
  std::lock_guard<std::mutex> lock1(a.m_, std::adopt_lock);
  std::lock_guard<std::mutex> lock2(b.m_, std::adopt_lock);

  // 等价实现，先不上锁，后同时上锁
  //   std::unique_lock<std::mutex> lock1(a.m_, std::defer_lock);
  //   std::unique_lock<std::mutex> lock2(b.m_, std::defer_lock);
  //   std::lock(lock1, lock2);

  a.n_ -= n;
  b.n_ += n;
}

int main() {
  A x{70};
  A y{30};

  std::thread t1(f, std::ref(x), std::ref(y), 20);
  std::thread t2(f, std::ref(y), std::ref(x), 10);

  t1.join();
  t2.join();
}
```

`std::lock_guard` 未提供任何接口且不支持拷贝和移动，而 `std::unique_lock` 多提供了一些接口，使用更灵活，占用的空间也多一点。一种要求灵活性的情况是转移锁的所有权到另一个作用域

```c++
std::unique_lock<std::mutex> get_lock() {
  extern std::mutex m;//使用外部定义的m，不要自定义局部变量m
  std::unique_lock<std::mutex> l(m);
  prepare_data();
  return l;  // 不需要 std::move，编译器负责调用移动构造函数
}

void f() {
  std::unique_lock<std::mutex> l(get_lock());
  do_something();
}
```

代码层次避免死锁，层级锁：

```c++
#include <iostream>
#include <mutex>
#include <stdexcept>

class HierarchicalMutex {
 public:
  explicit HierarchicalMutex(int hierarchy_value)
      : cur_hierarchy_(hierarchy_value), prev_hierarchy_(0) {}

  void lock() {
    validate_hierarchy();  // 层级错误则抛异常
    m_.lock();
    update_hierarchy();
  }

  bool try_lock() {
    validate_hierarchy();
    if (!m_.try_lock()) {
      return false;
    }
    update_hierarchy();
    return true;
  }

  void unlock() {
    if (thread_hierarchy_ != cur_hierarchy_) {
      throw std::logic_error("mutex hierarchy violated");
    }
    thread_hierarchy_ = prev_hierarchy_;  // 恢复前一线程的层级值
    m_.unlock();
  }

 private:
  void validate_hierarchy() {
    if (thread_hierarchy_ <= cur_hierarchy_) {
      throw std::logic_error("mutex hierarchy violated");
    }
  }

  void update_hierarchy() {
    // 先存储当前线程的层级值（用于解锁时恢复）
    prev_hierarchy_ = thread_hierarchy_;
    // 再把其设为锁的层级值
    thread_hierarchy_ = cur_hierarchy_;
  }

 private:
  std::mutex m_;
  const int cur_hierarchy_;
  int prev_hierarchy_;
  static thread_local int thread_hierarchy_;  // 所在线程的层级值
};

// static thread_local 表示存活于一个线程周期
thread_local int HierarchicalMutex::thread_hierarchy_(INT_MAX);

HierarchicalMutex high(10000);
HierarchicalMutex mid(6000);
HierarchicalMutex low(5000);

void lf() {  // 最低层函数
  std::lock_guard<HierarchicalMutex> l(low);
  // 调用 low.lock()，thread_hierarchy_ 为 INT_MAX，
  // cur_hierarchy_ 为 5000，thread_hierarchy_ > cur_hierarchy_，
  // 通过检查，上锁，prev_hierarchy_ 更新为 INT_MAX，
  // thread_hierarchy_ 更新为 5000
}  // 调用 low.unlock()，thread_hierarchy_ == cur_hierarchy_，
// 通过检查，thread_hierarchy_ 恢复为 prev_hierarchy_ 保存的 INT_MAX，解锁

void hf() {
  std::lock_guard<HierarchicalMutex> l(high);  // high.cur_hierarchy_ 为 10000
  // thread_hierarchy_ 为 10000，可以调用低层函数
  lf();  // thread_hierarchy_ 从 10000 更新为 5000
  //  thread_hierarchy_ 恢复为 10000
}  //  thread_hierarchy_ 恢复为 INT_MAX

void mf() {
  std::lock_guard<HierarchicalMutex> l(mid);  // thread_hierarchy_ 为 6000
  hf();  // thread_hierarchy_ < high.cur_hierarchy_，违反了层级结构，抛异常
}

int main() {
  lf();
  hf();
  try {
    mf();
  } catch (std::logic_error& ex) {
    std::cout << ex.what();
  }
}
```

#### 读写锁

c++14提供`std::shared_timed_mutex`，c++17提供接口更少，性能更高的`std::shared_mutex`

如果多个线程调用 `shared_mutex.lock_shared()`，多个线程可以同时读，如果此时有一个写线程调用 `shared_mutex.lock()`，则读线程均会等待该写线程调用 `shared_mutex.unlock()`(即，写的优先级最高)。C++11 没有提供读写锁，可使用 `boost::shared_mutex`

C++14 提供了 `std::shared_lock`，它在构造时接受一个 `mutex`，并会调用 `mutex.lock_shared()`，析构时会调用 `mutex.unlock_shared()`

对于 `std::shared_mutex`，通常在读线程中用 `std::shared_lock`管理，在写线程中用 `std::unique_lock`管理

```c++
class A {
 public:
  int read() const {
    std::shared_lock<std::shared_mutex> l(m_);
    return n_;
  }

  int write() {
    std::unique_lock<std::shared_mutex> l(m_);
    return ++n_;
  }

 private:
  mutable std::shared_mutex m_;
  int n_ = 0;
};
```

#### 递归锁

`std::recursive_mutex`,可以在一个线程上多次获取锁，但是其他线程获取锁之前必须释放所有锁

大多数情况下，使用了递归锁，代表程序设计有问题

```c++
#include <mutex>

class A {
 public:
  void f() {
    m_.lock();
    m_.unlock();
  }

  void g() {
    m_.lock();
    f();
    m_.unlock();
  }

 private:
  std::recursive_mutex m_;
};

int main() {
  A{}.g();  // OK
}
```

可以使用`std::once_flag`和`std::call_once`实现应用层面的原子操作

```c++
#include <memory>
#include <mutex>
#include <thread>

class A {
 public:
  void f() {}
};

std::shared_ptr<A> p;
std::once_flag flag;

void init() {
  std::call_once(flag, [&] { p.reset(new A); });
  p->f();
}

int main() {
  std::thread t1{init};
  std::thread t2{init};

  t1.join();
  t2.join();
}
```

> static 局部变量在声明后就完成了初始化，这存在潜在的 race condition，如果多线程的控制流同时到达 static 局部变量的声明处，即使变量已在一个线程中初始化，其他线程并不知晓，仍会对其尝试初始化。为此，C++11 规定，如果 static 局部变量正在初始化，线程到达此处时，将等待其完成，从而避免了 race condition。只有一个全局实例时，可以直接用 static 而不需要 std::call_once

### 条件变量以及信号量(20)以及屏障(20)

#### 条件变量

`std::condition_variable`基础使用

```c++
class A {
 public:
  void step1() {
    {
      std::lock_guard<std::mutex> l(m_);
      step1_done_ = true;
    }
    std::cout << 1;
    cv_.notify_one();
  }

  void step2() {
    std::unique_lock<std::mutex> l(m_);
    cv_.wait(l, [this] { return step1_done_; });
    step2_done_ = true;
    std::cout << 2;
    cv_.notify_one();
  }

  void step3() {
    std::unique_lock<std::mutex> l(m_);
    cv_.wait(l, [this] { return step2_done_; });
    std::cout << 3;
  }

 private:
  std::mutex m_;
  std::condition_variable cv_;
  bool step1_done_ = false;
  bool step2_done_ = false;
};

int main() {
  A a;
  std::thread t1(&A::step1, &a);
  std::thread t2(&A::step2, &a);
  std::thread t3(&A::step3, &a);
  t1.join();
  t2.join();
  t3.join();
}  // 123
```

当有多个任务等待时，`notify_one()`会随机唤醒一个,`notify_all`会唤醒所有的任务

`std::condition_variable`只能和`std::mutex`或者`std::unique_lock`绑定使用，但是`std::condition_variable_any`可以和弱锁类型绑定（即只要实现了锁的基本功能的类型都可以绑定，会更加灵活）

```c++
class Mutex {
 public:
  void lock() {}
  void unlock() {}
};

class A {
 public:
  void signal() {
    std::cout << 1;
    cv_.notify_one();
  }

  void wait() {
    Mutex m;
    cv_.wait(m);
    std::cout << 2;
  }

 private:
  std::condition_variable_any cv_;
};

int main() {
  A a;
  std::thread t1(&A::signal, &a);
  std::thread t2(&A::wait, &a);
  t1.join();
  t2.join();
}  // 12
```

#### 信号量(semaphore)

C++20提供了`std::counting_semaphore`,`acquire()`为P操作，减一；`release()`为V操作，加一

`std::binary_semaphore`是`std::counting_semaphore`模板为1的特例化(和`std::condition_variable`似乎一样，但是使用更简单)

```c++
class A {
 public:
  void wait1() {
    sem_.acquire();
    std::cout << 1;
  }

  void wait2() {
    sem_.acquire();
    std::cout << 2;
  }

  void signal() { sem_.release(2); }

 private:
  std::counting_semaphore<2> sem_{0};  // 初始值 0，最大值 2
};

int main() {
  A a;
  std::thread t1(&A::wait1, &a);
  std::thread t2(&A::wait2, &a);
  std::thread t3(&A::signal, &a);
  t1.join();
  t2.join();
  t3.join();
}  // 12 or 21
```

#### 屏障(barrier，C++20)

`std::barrier`使用一个值作为要等待的线程数量来构造，`std::barrier::arrive_and_wait`会阻塞至所有线程都完成任务。如果想移除某个线程，可以在线程中调用`std::barrier::arrive_and_drop`.构造 `std::barrier` 时可以额外设置一个 noexcept 函数，当所有线程到达阻塞点时，由其中一个线程运行该函数

```c++
class A {
 public:
  void f() {
    std::barrier sync_point{3, [&]() noexcept { ++i_; }};
    for (auto& x : tasks_) {
      x = std::thread([&] {
        std::cout << 1;
        sync_point.arrive_and_wait();
        assert(i_ == 1);
        std::cout << 2;
        sync_point.arrive_and_wait();
        assert(i_ == 2);
        std::cout << 3;
      });
    }
    for (auto& x : tasks_) {
      x.join();  // 析构 barrier 前 join 所有使用了 barrier 的线程
    }  // 析构 barrier 时，线程再调用 barrier 的成员函数是 undefined behavior
  }

 private:
  std::thread tasks_[3] = {};
  int i_ = 0;
};

int main() {
  A a;
  a.f();
}
```

#### 栅栏(latch，C++20)

#### future

通过`std::async`创建异步任务的`std::future`，只可以进行一次`get()`获取异步任务的执行结果。

```c++
StatusStr IOTaskHandler::getSpecificTopic(const std::string& topic_name){
    sensor_sub_ =  nh_.subscribe("/all_" + device_type_ +"_state",10,&IOTaskHandler::getDeviceStatus, this);
    auto fut =  std::async(std::launch::async,[this,&topic_name](){
        std::this_thread::sleep_for(std::chrono::seconds(5));
        if(this->monitor_msg_.size() == 0){
            this->sensor_sub_.shutdown();
            return std::make_pair(Status::SUCCESS,std::string("check:no_topic"));
        }else{
            std::string response_str{"check:check_specific_topic:"};
            response_str += (topic_name + "-" + monitor_msg_[topic_name]);
            monitor_msg_.clear();
            this->sensor_sub_.shutdown();
            return std::make_pair(Status::SUCCESS,std::move(response_str));
        }
        this->sensor_sub_.shutdown();
    });//timer done
    return fut.get();
}
```

`std::async`的第一个参数可以指定`std::launch`枚举值来指定执行策略,默认是或策略，根据线程资源来调整策略

```c++
namespace std {
enum class launch {
    async    = 0x1, // 运行新线程来执行任务
    deferred = 0x2  // 惰性求值，请求结果时才执行任务
};
}
```

packaged_task

