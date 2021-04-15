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

## 向线程函数传递参数

```C++
void f(int i,std::string const& s);
void oops(int some_param)
{
  char buffer[1024]; // 1
  sprintf(buffer, "%i",some_param);
  std::thread t(f,3,buffer); // 2
  //std::thread t(f,3,std::string(buffer));
  t.detach();
}
```

在传入到std::thread构造函数前就构造好临时对象。否则可能出现在隐式转换过程中主线程已经结束的情况（`因为这个隐士转换的过程是在主线程中进行的`）。

相反的情况，期待传入一个引用，但整个对象都被复制了。

```C++
void update_data_for_widget(widget_id w,widget_data& data); // 1
void oops_again(widget_id w)
{
  widget_data data;
  std::thread t(update_data_for_widget,w,data); // 2
  display_status();
  t.join();
  process_widget_data(data); // 3
}
```

update_data_for_widget第二个参数期待传入一个引用，但是std::thread的构造函数不知道，`会无视函数期待的类型，盲目的拷贝(这里调用的是拷贝构造函数)`。最后update_data_for_widget中的data是内部拷贝的引用，而不是数据本身。`如在oops_again创建线程t时传入的是A的引用，但是线程的构造函数会直接构造一个A的拷贝B，然后把B的引用传给update_data_for_widget函数`。如果这个函数中不是个引用，那么还需要额外一次拷贝构造。

上面的方法并不会修改data数据——解决方法std::ref。`什么是std::bind？`

将类的成员函数作为线程函数。

```C++
class X
{
public:
  void do_lengthy_work();
};
X my_x;
std::thread t(&X::do_lengthy_work,&my_x); // 1


class X
{
public:
  void do_lengthy_work(int);
};
X my_x;
int num(0);
std::thread t(&X::do_lengthy_work, &my_x, num);
```

这种情况第二个参数必须得是对象指针(因为需要提供对象的this指针)，接下来是这个成员函数显式地参数。

还有一种情况是提供的参数只能移动，不能拷贝。move是原始对象所有权转移给另一个对象。如`std::unique_ptr`。下面展示std::move是如何将动态对象的所有权转移到线程中。

## 2.3 转移所有权

为什么需要转移所有权？

新线程返回的所有权？

```C++
void some_function();
void some_other_function();
std::thread t1(some_function);            // 1
std::thread t2=std::move(t1);            // 2
t1=std::thread(some_other_function);    // 3
std::thread t3;                            // 4
t3=std::move(t2);                        // 5
t1=std::move(t3);                        // 6 赋值操作将使程序崩溃
```

3处能用赋值是因为这是个临时对象，移动操作会隐式调用。

6处出错是因为t1已经有了一个关联线程在运行，所以这里系统直接调用std::terminate()终止程序运行。

线程所有权可以在函数外转移，可以return出去；所有权可以在函数内传递，如将`sdt::thread`作为参数，或move(t)作为参数。

如果把线程作为对象的一个参数，在析构函数中决定对这个线程进行join还是detach，移动操作就可以避免很多不必要的麻烦。

用线程分割算法的工作总量，把`std::thread`放入`std::vector`是线程自动化管理的第一步。创建**一组**线程。

## 确定线程数量

`std::thread::hardware_concurrency()`返回并发线程的数量。

`std::accumulate`是一个累加函数，三个参数(迭代器1，迭代器2，结果引用)。改写成并行版本。

先定义每个线程的最小任务数，然后根据任务总量确定线程数。——因为上下文频繁切换会降低线程性能。

怎么得到每个线程的结果的呢？用的`std::ref`，在启动所有线程后，对这些线程用join，再将结果加起来。

## 线程标识

std::thread::id类型，两种方式获取调用对象的成员函数get_id()，获取当前线程的id用std::this_thread::get_id()。可以存到容器中

std::thread::id实例常用作检测线程是否需要进行一些操作。
