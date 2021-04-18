# note3

## 3.1 共享数据的问题

如要链表要删除一个节点步骤如下：

1. 找到要删除的节点
2. 使前一个节点next指向后一个节点
3. 使后一个节点prior指向前一个节点
4. 删除当前节点

在删除N过程中不确定是否有其他线程访问，可能有线程访问刚刚删除一边的节点(2)。当有线程试图删除N->next时，可能使数据结构产生永久性破坏，使程序崩溃。——这是并行中的常见错误：条件竞争。

并发中的竞争条件取决于多个线程的执行顺序。每个线程都争着完成自己的任务。大多数情况下，即使改变执行顺序，也是良性竞争，结果可以接受。

（上面是对数据结构的修改，类似于索引并发的latch，还有就是数据竞争，也就是索引并发中的lock，并发的修改一个独立的对象）。

恶性的条件竞争通常发生在对多个数据块的修改，比如对两个连接指针的修改。

`如何避免恶性的条件竞争：`最简单的方法就是对数据结构采用某种保护机制，确保只有修改线程才能看到不变量的中间状态。而对于其他线程来说，要么是修改完成了，要么是没有开始。

另一种方法就是lock-free方法，让修改操作变成原子的。

另一种方式是用事务的方式处理数据结构的更新。访问和修改都存储在事务日志中，对之前的操作进行合并，再进行提交。

最基本的方法是使用C++标准库提供的互斥量Mutex。

## 使用互斥量

在访问数据前加锁，访问结束后解锁。互斥量是C++保护数据通用机制，但也会造成死锁或对数据保护过多（或太少）。

不建议直接使用mutex，因为这样总是要先lock，函数出口unlock包含出现异常的情况。建议用mutex的RAII模板lock_guard。

`一点奇怪的地方：mutex就是放在这里就可以用，看起来像是锁住的是一段程序(操作)而不是数据——加锁时判断这个锁是否已经有程序获取，若无人获取，那么当前程序获取，执行。`

```C++
**#include <list>
#include <mutex>
#include <algorithm>

std::list<int> some_list;    // 1
std::mutex some_mutex;    // 2

void add_to_list(int new_value)
{
  std::lock_guard<std::mutex> guard(some_mutex);    // 3
  some_list.push_back(new_value);
}

bool list_contains(int value_to_find)
{
  std::lock_guard<std::mutex> guard(some_mutex);    // 4
  return std::find(some_list.begin(),some_list.end(),value_to_find) != some_list.end();
}**
```

头文件\<mutex\>，实例化std::mutex

**RAII机制，resourse acquisition is initialization，也就是资源获取就是初始化，这个前面介绍std::thread中已经讲过了，在前文中它的主要作用是解决线程join或detach的问题，把join或detach写在析构函数中，在对象销毁的时候自动调用析构函数。在这里也是一样的，如果使用mutex的成员函数lock和unlock，总是需要在前面lock，使用后unlock——为了防止忘记unlock，也是把mutex写入类中，在对象析构时自动调用unlock。也就是`std::lock_goard`。**

大多数情况需要把mutex和需要保护的数据放在同一个类中。当一个成员函数返回的是保护数据的指针或引用时，还是会破坏数据。

只是用`std::lock_guard`是不行的，防不住指针或引用。但是检查指针或引用很容易。只有没有成员函数通过返回值或者输出参数的形式向其调用者返回指向受保护数据的指针或引用，数据就是安全的。

```C++
class some_data
{
  int a;
  std::string b;
public:
  void do_something();
};

class data_wrapper
{
private:
  some_data data;
  std::mutex m;
public:
  template<typename Function>
  void process_data(Function func)
  {
    std::lock_guard<std::mutex> l(m);
    func(data);    // 1 传递“保护”数据给用户函数
  }
};

some_data* unprotected;

void malicious_function(some_data& protected_data)
{
  unprotected=&protected_data;
}

data_wrapper x;
void foo()
{
  x.process_data(malicious_function);    // 2 传递一个恶意函数
  unprotected->do_something();    // 3 在无保护的情况下访问保护数据
}
```

在上面的代码中，把保护的数据传给了func。这种情况C++无法提供任何帮助，只能由开发者使用正确的互斥锁保护数据。乐观地说还是有办法的：不要把受保护数据的指针或引用传给到互斥锁作用域之外。

链表，要确保线程能安全的删除一个节点，需要确保防止对这三个节点的并发访问，如果只是对指向每个节点的指针进行访问还是不行。

最简单的操作是用mutex保护整个链表，每次操作的时候都把整个链表锁起来。

以栈为例，讨论接口间的条件竞争。

如先判断栈是否为空，然后访问top是常见操作。但empty()返回时是正确的，返回后其他线程可以自由访问栈，这时别人可能pop()使栈为空，结果top出错。下面的代码

```C++
stack<int> s;
if (! s.empty()){    // 1
  int const value = s.top();    // 2
  s.pop();    // 3
  do_something(value);
}
```

这在非共享的栈中这么做当然是正确的。上面的问题是接口间固有的问题，而不是单个函数内对数据的保护问题——解决方法也是修改接口设计。

除了empty与top，可以观察到top()和pop()之间也有潜在的条件竞争，都引用同一个栈对象。

一个线程安全的堆栈的实现

```C++
#include <exception>
#include <memory>  // For std::shared_ptr<>

struct empty_stack: std::exception
{
  const char* what() const throw();
};

template<typename T>
class threadsafe_stack
{
public:
  threadsafe_stack();
  threadsafe_stack(const threadsafe_stack&);
  threadsafe_stack& operator=(const threadsafe_stack&) = delete; // 1 赋值操作被删除

  void push(T new_value);
  std::shared_ptr<T> pop();
  void pop(T& value);
  bool empty() const;
};
削减接口可以获得最大程度的安全,甚至限制对栈的一些操作。栈是不能直接赋值的，因为赋值操作已经删除了①(详见附录A，A.2节)，并且这里没有swap()函数。当栈为空时，pop()函数会抛出一个empty_stack异常，所以在empty()函数被调用后，其他部件还能正常工作。如选项3描述的那样，使用std::shared_ptr可以避免内存分配管理的问题，并避免多次使用new和delete操作。堆栈中的五个操作，现在就剩下三个：push(), pop()和empty()(这里empty()都有些多余)。简化接口更有利于数据控制，可以保证互斥量将操作完全锁住。下面的代码展示了一个简单的实现——封装std::stack<>的线程安全堆栈。

代码3.5 扩充(线程安全)堆栈

#include <exception>
#include <memory>
#include <mutex>
#include <stack>

struct empty_stack: std::exception
{
  const char* what() const throw() {
	return "empty stack!";
  };
};

template<typename T>
class threadsafe_stack
{
private:
  std::stack<T> data;
  mutable std::mutex m;
  
public:
  threadsafe_stack()
	: data(std::stack<T>()){}
  
  threadsafe_stack(const threadsafe_stack& other)
  {
    std::lock_guard<std::mutex> lock(other.m);
    data = other.data; // 1 在构造函数体中的执行拷贝
  }

  threadsafe_stack& operator=(const threadsafe_stack&) = delete;

  void push(T new_value)
  {
    std::lock_guard<std::mutex> lock(m);
    data.push(new_value);
  }
  
  std::shared_ptr<T> pop()
  {
    std::lock_guard<std::mutex> lock(m);
    if(data.empty()) throw empty_stack(); // 在调用pop前，检查栈是否为空
	
    std::shared_ptr<T> const res(std::make_shared<T>(data.top())); // 在修改堆栈前，分配出返回值
    data.pop();
    return res;
  }
  
  void pop(T& value)
  {
    std::lock_guard<std::mutex> lock(m);
    if(data.empty()) throw empty_stack();
	
    value=data.top();
    data.pop();
  }
  
  bool empty() const
  {
    std::lock_guard<std::mutex> lock(m);
    return data.empty();
  }
};
```

等一下，这里的pop会先加锁，接着调用empty，empty也会加锁，还是同一个锁，不会产生死锁吗？——应该是被允许的。

### 避免死锁

无所情况下，仅需要两个线程对象互相调用join()就能产生死锁。

建议：

+ 线程获取一个锁时，就别去获取第二个。当需要获取多个锁，用std::lock对获取锁的操作上锁。
+ 避免在持有锁时调用外部代码
+ 使用固定顺序获取锁
+ 使用层次锁结构

```C++
hierarchical_mutex high_level_mutex(10000); // 1
hierarchical_mutex low_level_mutex(5000);  // 2
hierarchical_mutex other_mutex(6000); // 3

int do_low_level_stuff();

int low_level_func()
{
  std::lock_guard<hierarchical_mutex> lk(low_level_mutex); // 4
  return do_low_level_stuff();
}

void high_level_stuff(int some_param);

void high_level_func()
{
  std::lock_guard<hierarchical_mutex> lk(high_level_mutex); // 6
  high_level_stuff(low_level_func()); // 5
}

void thread_a()  // 7
{
  high_level_func();
}

void do_other_stuff();

void other_stuff()
{
  high_level_func();  // 10
  do_other_stuff();
}

void thread_b() // 8
{
  std::lock_guard<hierarchical_mutex> lk(other_mutex); // 9
  other_stuff();
}
```

如果将一个hierarchical_mutex实例进行上锁，之恩给你获取更低层次实例上的锁。

上面的代码中8会出错，因为首先加了一个中层的锁，然后下面的other_stuff又要获取高层的锁这就矛盾了。会抛出一个异常或直接终止程序。但是注意hierarchical_mutex并不是C++标准的一部分，但是实现起来比较容易。

```C++
class hierarchical_mutex
{
  std::mutex internal_mutex;
  
  unsigned long const hierarchy_value;
  unsigned long previous_hierarchy_value;
  
  static thread_local unsigned long this_thread_hierarchy_value;  // 1
  
  void check_for_hierarchy_violation()
  {
    if(this_thread_hierarchy_value <= hierarchy_value)  // 2
    {
      throw std::logic_error(“mutex hierarchy violated”);
    }
  }
  
  void update_hierarchy_value()
  {
    previous_hierarchy_value=this_thread_hierarchy_value;  // 3
    this_thread_hierarchy_value=hierarchy_value;
  }
  
public:
  explicit hierarchical_mutex(unsigned long value):
      hierarchy_value(value),
      previous_hierarchy_value(0)
  {}
  
  void lock()
  {
    check_for_hierarchy_violation();
    internal_mutex.lock();  // 4
    update_hierarchy_value();  // 5
  }
  
  void unlock()
  {
    if(this_thread_hierarchy_value!=hierarchy_value)
      throw std::logic_error(“mutex hierarchy violated”);  // 9
    this_thread_hierarchy_value=previous_hierarchy_value;  // 6
    internal_mutex.unlock();
  }
  
  bool try_lock()
  {
    check_for_hierarchy_violation();
    if(!internal_mutex.try_lock())  // 7
      return false;
    update_hierarchy_value();
    return true;
  }
};
thread_local unsigned long
     hierarchical_mutex::this_thread_hierarchy_value(ULONG_MAX);  // 8
```

用thread_local表示当前线程的层级值

### std::unique_lock 灵活的锁

`std::unique_lock`实例不会总与互斥量的数据类型相关，使用起来比`std::lock_guard`更加灵活。可以用第二个参数设置初始状态。

`什么叫不会总与互斥量的数据类型相关?`

std::unique_lock的第二个参数

+ std::adopt_lock 表示互斥量已经被锁了
+ std::defer_lock 表示互斥量还没锁
+ std:: try_to_lock 尝试加锁

```C++
class some_big_object;
void swap(some_big_object& lhs,some_big_object& rhs);
class X
{
private:
  some_big_object some_detail;
  std::mutex m;
public:
  X(some_big_object const& sd):some_detail(sd){}
  friend void swap(X& lhs, X& rhs)
  {
    if(&lhs==&rhs)
      return;
    std::unique_lock<std::mutex> lock_a(lhs.m,std::defer_lock); // 1 
    std::unique_lock<std::mutex> lock_b(rhs.m,std::defer_lock); // 1 std::defer_lock 留下未上锁的互斥量
    std::lock(lock_a,lock_b); // 2 互斥量在这里上锁
    swap(lhs.some_detail,rhs.some_detail);
  }
};
```

std::unique_lock的成员函数lock()加锁，unlock()解锁，try_lock()，release解除这个unique_lock和mutex的关系。

