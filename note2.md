# 线程管理

三部分内容

+ 启动新线程
+ join和detach
+ 线程唯一标识符 thread id

## 线程管理的基础

每个程序至少有一个线程，那就是执行main\(\)函数的线程，主线程。其他线程有各自的入口函数，线程与主线程同时运行。当线程执行完入口函数后，线程也会退出。

用C++线程库启动线程，可以归结为构造std::thread对象。

```C++
void do_some_work();
std::thread my_thread(do_some_work);
```

std::thread可以用可调用类型构造，将带有函数调用符类型的实例传入std::thread类中，替代默认的构造函数。

函数对象会复制到新线程的存储空间中，函数对象的执行和调用都在线程的内存空间中进行。

`注意：当把函数对象传入到线程构造函数中，需要避免最头疼的问题，如果传递了一个临时变量，而不是一个命名的变量，C++编译器会把它解析为函数声明，而不是类型对象的定义`

```C++
std::thread my_thread(background_task());
```

上面会被解析为声明一个my_thread函数，有一个参数是background_tack\(\), 返回的是std::thread对象。

**用新同意初始化语法{}**

```C++
std::thread my_thread{background_task()};
```

启动线程，需要明确等待线程结束join还是让其自主运行detach。必须在std::thread对象销毁前作出决定，否则会终止(std::thread析构函数会调用std::terminate(),再决定触发相应异常)。

如果不等待，需要保证可访问数据的有效性。

```C++
struct func
{
  int& i;
  func(int& i_) : i(i_) {}
  void operator() ()
  {
    for (unsigned j=0 ; j<1000000 ; ++j)
    {
      do_something(i);           // 1. 潜在访问隐患：悬空引用
    }
  }
};

void oops()
{
  int some_local_state=0;
  func my_func(some_local_state);
  std::thread my_thread(my_func);
  my_thread.detach();          // 2. 不等待线程结束
}                              // 3. 新线程可能还在运行
```

`注意这里没有在讨论引用对象的问题，而是主线程中一个数据引用被一个对象使用，这个对象又参与了线程。`

处理上面这种情况，需要把数据复制到线程中，而非共享数据中。把这个对象复制到线程中。`对对象中包含的指针和引用保持谨慎。`

等待线程 join。当然上面的如果用join就没什么意义，主线程中没有事要做。在现实中，子线程开始运行，主线程继续工作，并等待子线程结束。

`join`只能等待，至于判断线程是否结束，只等待一段时间，需要其他机制。

如果要分离一个线程，创建这个线程直接分离就完了。如果要join一个线程，需要考虑join放在哪。当线程运行后产生异常，在join()调用前被抛出，意味着join调用会被跳过。`这样就需要在异常处理中调用join，避免这个问题`

```C++
struct func; // 定义在清单2.1中
void f()
{
  int some_local_state=0;
  func my_func(some_local_state);
  std::thread t(my_func);
  try
  {
    do_something_in_current_thread();
  }
  catch(...)
  {
    t.join();  // 1
    throw;
  }
  t.join();  // 2
}
```

上面的代码确保了join后结束。


还有一种方式是资源获取即初始化定义一个类，在这个类中有一个std::thread，这个类的析构函数中写一个join。定义这个线程后，用这个线程初始化这个对象

```C++
class thread_guard
{
  std::thread& t;
public:
  explicit thread_guard(std::thread& t_):
    t(t_)
  {}
  ~thread_guard()
  {
    if(t.joinable()) // 1
    {
      t.join();      // 2
    }
  }
  thread_guard(thread_guard const&)=delete;   // 3
  thread_guard& operator=(thread_guard const&)=delete;
};

struct func; // 定义在清单2.1中

void f()
{
  int some_local_state=0;
  func my_func(some_local_state);
  std::thread t(my_func);
  thread_guard g(t);
  do_something_in_current_thread();
}    // 4
```

运行到4局部对象要被逆序销毁，因此thread_guard g是第一个被销毁的，g的析构函数是在f这个线程中执行的，就算do_something_in_current_thread()有异常，上面的g也会被销毁。

析构时先判断是否joinable()，因为join和detach只能调用一次。拷贝构造和拷贝赋值都是`=delete`，不让编译器自动生成，因为直接对一个对象拷贝或赋值很危险，可能会丢失原来的线程。

如果不想等待，可以在析构函数中用detach，这样在f执行完后，调用g的析构函数，让g中的线程与f分离，在后台继续执行。

detach让线程在后台运行，主线程不能与之交互。C++运行库保证当线程退出，相关资源能够正确回收，后台线程的归属和控制C++运行库都会处理。

守护线程， fire and forgot就用到这种线程。

detach和join前都需要进行joinable检查

```C++
void edit_document(std::string const& filename)
{
  open_document_and_display_gui(filename);
  while(!done_editing())
  {
    user_command cmd=get_user_input();
    if(cmd.type==open_new_document)
    {
      std::string const new_name=get_filename_from_user();
      std::thread t(edit_document,new_name);  // 1
      t.detach();  // 2
    }
    else
    {
       process_user_input(cmd);
    }
  }
}
```

一个分离线程的例子，当打开新的文档就启动一个新线程打开新文档并分离这个线程。